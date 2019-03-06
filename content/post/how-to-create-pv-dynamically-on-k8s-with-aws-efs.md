---
title: "How to create PV dynamically on k8s with AWS EFS"
date: 2019-03-05T09:36:00+08:00
lastmod: 2019-03-06T09:31:00+08:00
draft: false
tags: ["Kubernetes", "EFS", "Persistent Volume"]
categories: ["Kubernetes"]
---

# Preface

When I was using AWS as Kubernetes cloud provider, I have two choices for persistent volume, EBS and EFS. However, EBS can only mount to one EC2 instance, which is not appropriate to my scenario, so I can only choose EFS.

My requirement is that when I need a persistent volume, I only create PVC and the related PV should be created automatically. So I summary the steps to reach my purpose in below words.

# Steps on AWS

## Create EFS on AWS

This step can be referenced to https://docs.aws.amazon.com/efs/latest/ug/getting-started.html.

I only test the case that Kubernetes cluster and EFS are in the same VPC.

## Create folder in EFS for PV

Create one EC2 instance and SSH to it, mount the EFS and create folder.

```shell
sudo yum install -y nfs-utils
sudo mkdir efs
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport ${file_sys_id}.efs.us-east-1.amazonaws.com:/ efs
sudo mkdir efs/pvs
```

`file_sys_id` can be found from AWS EFS page.

# Steps on Kubernetes cluster

## Prepare authorization

Create one service account for EFS provisioner, which will help you create PV with EFS.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: efs-provisioner
```

```shell
kubectl create -f service_account.yaml
```

Authorize the service account, reference to https://github.com/kubernetes-incubator/external-storage/blob/master/aws/efs/deploy/rbac.yaml.

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: efs-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-efs-provisioner
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
     # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: efs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-efs-provisioner
  apiGroup: rbac.authorization.k8s.io
```

This example is using `default` namespace, change it if you need to deploy in different namespace.

```shell
kubectl create -f rbac.yaml
```

## Deploy EFS provisioner

Create ConfigMap.

`file_sys_id` can be found from AWS EFS.

```shell
kubectl create configmap efs-provisioner \
--from-literal=file.system.id=${file_sys_id} \
--from-literal=aws.region=us-east-1 \
--from-literal=provisioner.name=aws-efs
```

Create Deployment.

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: efs-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: efs-provisioner
    spec:
      serviceAccount: efs-provisioner
      containers:
        - name: efs-provisioner
          image: quay.io/external_storage/efs-provisioner:v0.1.0
          env:
            - name: FILE_SYSTEM_ID
              valueFrom:
                configMapKeyRef:
                  name: efs-provisioner
                  key: file.system.id
            - name: AWS_REGION
              valueFrom:
                configMapKeyRef:
                  name: efs-provisioner
                  key: aws.region
            - name: PROVISIONER_NAME
              valueFrom:
                configMapKeyRef:
                  name: efs-provisioner
                  key: provisioner.name
          volumeMounts:
            - name: pv-volume
              mountPath: /persistentvolumes
      volumes:
        - name: pv-volume
          nfs:
            server: <file system id>.efs.us-east-1.amazonaws.com
            path: /pvs
```

```shell
kubectl apply -f deployment.yaml
```

Create StorageClass.

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: aws-efs
provisioner: aws-efs
```

```shell
kubectl create -f sc.yaml
```

## Use the created storage class to create PVC

For every PVC you make, it will make a subdirectory in EFS under `/pvs/pvc-<claim_id>`.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: efs
  annotations:
    volume.beta.kubernetes.io/storage-class: "aws-efs"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
```

```shell
kubectl create -f pvc.yaml
```

The PVC can be in same or different namespace from EFS provisioner.
