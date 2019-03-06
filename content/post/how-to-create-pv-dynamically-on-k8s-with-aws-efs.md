+++
draft = false
categories = ["Kubernetes"]
description = "A method to create PV dynamically on k8s with AWS EFS."
title = "How to create PV dynamically on k8s with AWS EFS"
date = "2019-03-05T09:36:00+08:00"
+++

# Prerequisites

1. Create EFS on AWS

   Selected Subnets should contain the k8s cluster subnets.

   Selected Security Groups should ensure the EFS can be accessible.

2. Create one EC2 instance in the same VPC as EFS.

3. SSH to the instance, mount the EFS and create folder.

{{< highlight Bash >}}
sudo yum install -y nfs-utils
sudo mkdir efs
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport ${file_sys_id}.efs.us-east-1.amazonaws.com:/ efs
sudo mkdir efs/pvs
{{< /highlight >}}

`file_sys_id` can be found from AWS EFS console.

# Prepare authorization

1. Create `ServiceAccount`.

{{< highlight YAML >}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: efs-provisioner
{{< /highlight >}}

{{< highlight Bash >}}
kubectl create -f service_account.yaml
{{< /highlight >}}

2. Create `RBAC` rules.

{{< highlight YAML >}}
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
{{< /highlight >}}

{{< highlight Bash >}}
kubectl create -f rbac.yaml
{{< /highlight >}}

# Deploy EFS provisioner to k8s cluster

1. Create `ConfigMap`.

{{< highlight Bash >}}
kubectl create configmap efs-provisioner \
--from-literal=file.system.id=${file_sys_id} \
--from-literal=aws.region=us-east-1 \
--from-literal=provisioner.name=aws-efs
{{< /highlight >}}

2. Create `Deployment`.

{{< highlight YAML >}}
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
{{< /highlight >}}

{{< highlight Bash >}}
kubectl apply -f deployment.yaml
{{< /highlight >}}

# Create `StorageClass` on k8s cluster

{{< highlight YAML >}}
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: aws-efs
provisioner: aws-efs
{{< /highlight >}}

{{< highlight Bash >}}
kubectl create -f sc.yaml
{{< /highlight >}}

# Create PVC

Note: For every `PersistentVolumeClaim` you make, it will make a subdirectory in EFS under `/pvs/pvc-<claim_id>`.

{{< highlight YAML >}}
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
{{< /highlight >}}

{{< highlight Bash >}}
kubectl create -f pvc.yaml
{{< /highlight >}}

# Use PVC

{{< highlight YAML >}}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: efs-test
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: efs-test
    spec:
      containers:
      - name: efs-test
        image: alpine:latest
        volumeMounts:
        - mountPath: /efs_vol
          name: efs
      volumes:
        - name: efs
          persistentVolumeClaim:
            claimName: efs
{{< /highlight >}}
