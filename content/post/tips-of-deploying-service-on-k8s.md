+++
draft = false
topics = ["Kubernetes"]
description = "Some tips from experience"
title = "Tips of deploying service on Kubernetes"
date = "2018-10-21T13:35:00+08:00"
+++

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

# About RBAC
