---
title: "Tips of deploying service on Kubernetes"
date: 2018-10-21T13:35:00+08:00
lastmod: 2018-10-21T13:35:00+08:00
draft: false
tags: ["Kubernetes"]
categories: ["Kubernetes"]
---

# Preface

These are some tips or something should be considered when you want to deploy your service on Kubernetes.

# About Pod

## How to decide what things should be put in one Pod?

When designing a Pod, you should consider below 4 types container.

- Function container
- Init container
- Monitor container
- Operator container

{{< fluid_imgs
  "pod|/img/post/pod.png|"
>}}

Function container is the service.

Init container is responsible for initializing the service. For example, the configuration. It would be started and completed before function container is started.

Monitor container should be existed if there is monitor API in service.

Operator container should be existed if there is operator API in service.

## Use InitContainer to do initialization with ConfigMap

A good example is how Jenkins Helm Charts handle ConfigMap with InitContainer.

- Write configuration template and script both into ConfigMap.
- Create a Volume with ConfigMap.
- Mount the Volume to InitContainer and run configuration script to convert template to a real configuration file (still in the Volume).
- Mount the Volume to function container and the configuration is there.

https://github.com/helm/charts/blob/master/stable/jenkins/templates/jenkins-master-deployment.yaml

## StatefulSet

When you have below requirements, consider using StatefulSet.

- Need fixed container identifier (hostname).
- Need persistent volume.
- Need sequence deploying and deleting.

For a StatefulSet with N replicas, when Pods are being deployed, they are created sequentially, in order from {0..N-1}.

For a StatefulSet with N replicas, when Pods are being deleted, they are terminated in reverse order, from {N-1..0}.

## Readiness and Liveness

Readiness is used to know when a Container is ready to start accepting traffic. When a Pod is not ready, it is removed from Service load balancers.

Liveness is used to know when to restart a Container.

See an example.

{{< highlight YAML >}}
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
{{< /highlight >}}

# About Service

## Headless Service with Selector

See below case. We need to go to the specific pod to get job status.

{{< fluid_imgs
  "job|/img/post/job.png|"
>}}

For headless services that define selectors, the endpoints controller creates Endpoints records in the API, and modifies the DNS configuration to return A records (addresses) that point directly to the Pods backing the Service.

See Kafka Helm Chart as an example:
https://github.com/helm/charts/blob/master/incubator/kafka/templates/service-headless.yaml

# About Ingress

## Basic function

{{< fluid_imgs
  "ingress|/img/post/ingress.png|"
>}}

- Externally-reachable URLs
- Load balance traffic
- Terminate SSL
- Name based virtual hosting

See a simple snippet.

{{< highlight YAML >}}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        backend:
          serviceName: test
          servicePort: 80
{{< /highlight >}}

## Ingress controller

- GCE and Nginx supported by Kubernetes.
- F5.
- Kong.
- Traefik.
- Nginx supported by Nginx,Inc.
- HAProxy.
- Istio.

## Why need more controllers?

Suppose you have some requirements like:

- Select route rule based on header.
- Select route rule based on header with regex.
- Set weight to different Services.
- Combine weight and header to select route rule.
- Route request with customized rule.

## Why Istio is much powerful?

Istio Gateway is a new resource defined by Custom Resource Definition(CRD), while others are limited by Ingress synax.

An Gated Launch sample:

Define a Gateway resource.

{{< highlight YAML >}}
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: book-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
{{< /highlight >}}

Define a Virtual Service.

{{< highlight YAML >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: book-public
spec:
  gateways:
  - book-gateway
  hosts:
  - '*'
  http:
  - match:
    - headers:
        user:
          exact: new
    route:
    - destination:
        host: new-book-svc
        port:
          number: 80
      weight: 50
    - destination:
        host: old-book-svc
        port:
          number: 80
      weight: 50
  - route:
    - destination:
        host: old-book-svc
        port:
          number: 80
{{< /highlight >}}

# Resource

If you don't know how to get started, see examples in https://github.com/helm/charts.
