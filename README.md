# WordPress Deployment on Kubernetes using AWS EBS and EFS

## Prerequisites

* A running EKS cluster
* `kubectl` configured to access the cluster
* AWS CLI configured with appropriate credentials

---

## Step 1: Install CSI Drivers

### EBS CSI Driver:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.26"
```

### EFS CSI Driver:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.7"
```

---

## Step 2: Create IAM User & Permissions

Create a user named `wordpress` in AWS IAM with the following permissions:

* AmazonEC2FullAccess
* AmazonElasticFileSystemFullAccess
* IAMFullAccess
* AmazonElasticContainerRegistryPublicFullAccess
* AmazonEKSWorkerNodePolicy
* AmazonEKS\_CNI\_Policy
* AmazonEC2ContainerRegistryReadOnly
* AmazonEKSClusterPolicy

Also attach the following inline policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:CreateAccessPoint",
                "elasticfilesystem:DescribeAccessPoints",
                "elasticfilesystem:DeleteAccessPoint"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## Step 3: Create Kubernetes Secret for DB Password

```bash
kubectl create secret generic mysql-pass --from-literal=password=admin123
```

---

## Step 4: MySQL with EBS

### `mysql-deployment.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  storageClassName: gp2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
        - image: mysql:5.6
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
```

---

## Step 5: WordPress with EFS

### EFS Setup in AWS Console

1. Create EFS file system and note the ID.
2. Add mount targets for each subnet.
3. Create an access point.

### Kubernetes Configuration

Replace `fs-xxx` and `fsap-xxx` with your actual File System ID and Access Point ID.

### `efs-sc.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
```

### `efs-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

### `efs-pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-xxx::fsap-xxx
```

### `wordpress-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
        - image: wordpress:4.8-apache
          name: wordpress
          env:
            - name: WORDPRESS_DB_HOST
              value: wordpress-mysql:3306
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
          ports:
            - containerPort: 80
              name: wordpress
          volumeMounts:
            - name: wordpress-persistent-storage
              mountPath: /var/www/html
      volumes:
        - name: wordpress-persistent-storage
          persistentVolumeClaim:
            claimName: efs-claim
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
spec:
  type: LoadBalancer
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
```

---

## Step 6: Access Your Application

Once deployed, use the EXTERNAL-IP from:

```bash
kubectl get svc
```

Open it in your browser to access WordPress.

---

## Notes

* This deployment uses EBS for MySQL and EFS for WordPress.
* Ensure all AWS resources (EFS, IAM roles, subnets) are correctly set up before deploying.
* Use appropriate security groups to allow traffic.

---

## Author

Prepared by Raed

---

------------------------------------------------------------------------------------
# WordPress Kubernetes Deployment with AWS EBS and EFS

## Prerequisites

1. Make sure you have installed CSI drivers for EBS and EFS:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.14"

kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.5"
```

2. Verify installation:

```bash
kubectl get csidriver
kubectl get csinode
```

## IAM Setup

1. Create an IAM **User** with policies:

* AmazonEBSCSIDriverPolicy
* AmazonElasticFileSystemFullAccess

2. Create **Access Key** for that user.

3. Create an IAM **Role** with the following permissions:
 Add permissions "AmazonEBSCSIDriverPolicy" and ...
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:DescribeAccessPoints",
                "elasticfilesystem:DescribeFileSystems",
                "elasticfilesystem:DescribeMountTargets",
                "ec2:DescribeAvailabilityZones"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:CreateAccessPoint"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:RequestTag/efs.csi.aws.com/cluster": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:TagResource"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": "elasticfilesystem:DeleteAccessPoint",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
                }
            }
        }
    ]
}
```

## Kubernetes Secret for AWS Credentials

```bash
kubectl create secret generic raed-project \
  --namespace kube-system \
  --from-literal "key_id=<AWS_ACCESS_KEY_ID>" \
  --from-literal "access_key=<AWS_SECRET_ACCESS_KEY>"
```

### Patch Deployment

Edit the EBS CSI Controller Deployment:

```bash
kubectl edit deployment ebs-csi-controller -n kube-system
```

Add under `containers.env`:

```yaml
- name: AWS_ACCESS_KEY_ID
  valueFrom:
    secretKeyRef:
      key: key_id
      name: raed-project #change it like your secret key name 
      optional: true
- name: AWS_SECRET_ACCESS_KEY
  valueFrom:
    secretKeyRef:
      key: access_key
      name: raed-project #change it like your secret key name 
      optional: true
```

## MySQL Setup with EBS

1. Encode your MySQL password:

```bash
echo -n 'rraaeedd' | openssl base64
# Output: cnJhYWVlZGQ=
```

2. Create project folder:

```bash
mkdir raed-project && cd raed-project
```

3. Create the following files:

### secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-password
type: Opaque
data:
  password: cnJhYWVlZGQ=
```

### mysql-sc.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

### mysql-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: mysql-sc
  resources:
    requests:
      storage: 5Gi
```

### mysql-app.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-app
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-password
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

### mysql-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
```

## WordPress with EFS

1. Create EFS filesystem and access point:

   * Root directory: `/wordpress`
   * POSIX User ID / Group ID: `1000`
   * Root Owner User ID / Group ID: `1000`
   * Permissions: `777`

2. Modify EFS security group and networking as needed.

### wordpress-sc.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
```

### wordpress-pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-0cd74e02bfcf0c407::fsap-0c77b53d25f7cdbfe
```

### wordpress-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-efs-pvc
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

### wordpress-app.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:php7.1-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql-svc
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-password
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-storage
        persistentVolumeClaim:
          claimName: wordpress-efs-pvc
```

### wordpress-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
```

## Verify Setup

```bash
kubectl get pv
kubectl get svc

NAME              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes        ClusterIP      10.96.0.1        <none>        443/TCP        35d
wordpress         LoadBalancer   10.96.200.129    <pending>     80:32298/TCP   6s
mysql-svc         ClusterIP      10.101.144.152   <none>        3306/TCP       20m

```

Access your WordPress app using:

```
<EC2_INSTANCE_PUBLIC_IP>:<NodePort>
```
10. Make the App Highly Available
Use a LoadBalancer in front of multiple pods and nodes.

# kubernetes-project1
