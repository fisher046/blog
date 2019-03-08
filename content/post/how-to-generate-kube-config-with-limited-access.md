---
title: "How to generate kubeconfig with limited access"
date: 2019-03-08T09:05:12+08:00
lastmod: 2019-03-08T09:05:12+08:00
draft: false
tags: ["Kubernetes", "RBAC"]
categories: ["Kubernetes"]
---

# Requirement

I have full access `kubeconfig` and I want to provide a limited `kubeconfig` to another user, which only has access to resources in one specific namespace.

# Steps

Create the namespace.

```shell
kubectl create namespace example
```

Create a `ServiceAccount` in previous namespace.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: example-user
  namespace: example
```

```shell
kubectl create -f service_account.yaml
```

Create `Role` and `RoleBinding` for the `ServiceAccount` in previous namespace. Then the `ServiceAccount` has the access to the resources in that namespace.

```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: example-user-full-access
  namespace: example
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: example-user-view
  namespace: example
subjects:
- kind: ServiceAccount
  name: example-user
  namespace: example
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: example-user-full-access
```

```shell
kubectl create -f rbac.yaml
```

Get `Token` name of the `ServiceAccount`.

```shell
kubectl describe sa example-user -n example #=>

Name:                example-user
Namespace:           example
Labels:              <none>
Annotations:         kubectl.kubernetes.io/last-applied-configuration:
                       {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"example-user","namespace":"example"}}
Image pull secrets:  <none>
Mountable secrets:   example-user-token-2nxm9
Tokens:              example-user-token-2nxm9
```

Get token string of the previous `Token`.

```shell
kubectl get secret example-user-token-2nxm9 -n example -o "jsonpath={.data.token}" | base64 -D
```

Get certificate from the previous `Token`, save the certificate string **without leading and ending words "-----BEGIN CERTIFICATE-----" and "-----END CERTIFICATE-----"**.

```shell
kubectl get secret example-user-token-2nxm9 -n example -o "jsonpath={.data['ca\.crt']}" | base64 -D | tr -d "\n"
```

Now create the `kubeconfig` file.

```yaml
apiVersion: v1
clusters:
  - cluster:
      certificate-authority-data: <place certificate string here>
      server: <Kubernetes cluster API endpoint>
    name: example
contexts:
  - context:
      cluster: example
      namespace: example
      user: example-user
    name: example
current-context: example
kind: Config
preferences: {}
users:
  - name: example-user
    user:
      client-key-data: <place certificate string here>
      token: <place token string here>
```

By using this `kubeconfig`, we can only access resources in `example` namespace.
