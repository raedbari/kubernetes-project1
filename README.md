# WordPress Deployment on Kubernetes using AWS EBS and EFS

## 🛠️ Requirements

1. Launch 2 EC2 instances:
   - One as the **master**
   - The other as **node1**

2. SSH into the master node using **MobaXterm**

3. Ensure the Kubernetes cluster (using `kubeadm`) is installed and configured on both the master and node1

---

## 🔐 Step 1: Create an IAM User with EBS & EFS Permissions

## 🔑 Step 2: Generate Access Key for the User

## 🎯 Step 3: Attach Policies

Attach the following AWS managed policies to your IAM user:
- `AmazonEBSCSIDriverPolicy`
- `AmazonElasticFileSystemFullAccess`

---

## 🛡️ Step 4: Create IAM Role with Required Inline Policy

Example inline policy for EFS access:

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

---

## 📦 Step 5: Install AWS CSI Drivers

### EBS CSI Driver

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.26"
```

### EFS CSI Driver

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.7"
```

---

## 🔐 Step 6: Create Kubernetes Secret for Access Keys

```bash
kubectl create secret generic raed-project \
  --namespace kube-system \
  --from-literal "key_id=YOUR_ACCESS_KEY_ID" \
  --from-literal "access_key=YOUR_SECRET_ACCESS_KEY"
```

Make sure to update deployments to reference this secret.

### Patch EBS CSI Controller Deployment

```bash
kubectl edit deployment ebs-csi-controller -n kube-system
```

Under `containers.env`, add:

```yaml
- name: AWS_ACCESS_KEY_ID
  valueFrom:
    secretKeyRef:
      key: key_id
      name: raed-project
      optional: true
- name: AWS_SECRET_ACCESS_KEY
  valueFrom:
    secretKeyRef:
      key: access_key
      name: raed-project
      optional: true
```

Base64 encode a password (for example):

```bash
echo -n 'rraaeedd' | openssl base64
# Output: cnJhYWVlZGQ=
```

---

## 📁 Project Directory

```bash
mkdir kubernetes-project && cd kubernetes-project
```

---

## 🗄️ MySQL Deployment Files

- [secret.yaml](./Yaml-Files/secret.yaml)
- [mysql-sc.yaml](./Yaml-Files/mysql-sc.yaml)
- [mysql-pvc.yaml](./Yaml-Files/mysql-pvc.yaml)
- [mysql-app.yaml](./Yaml-Files/mysql-app.yaml)
- [mysql-svc.yaml](./Yaml-Files/mysql-svc.yaml)

---

## 🌐 WordPress Deployment

1. Create a new **EFS File System** named `wordpress`
2. Create an **Access Point** for it with the following values:
   - Root directory path: `/wordpress`
   - POSIX User ID: `1000`
   - POSIX Group ID: `1000`
   - Root Owner User ID: `1000`
   - Root Group ID: `1000`
   - POSIX Permissions: `777`
3. Update the **security group** to allow NFS access (port 2049)

### WordPress YAML Files:

- [wordpress-sc.yaml](./Yaml-Files/wordpress-sc.yaml)
- [wordpress-pv.yaml](./Yaml-Files/wordpress-pv.yaml)
- [wordpress-pvc.yaml](./Yaml-Files/wordpress-pvc.yaml)
- [wordpress-svc.yaml](./Yaml-Files/wordpress-svc.yaml)

