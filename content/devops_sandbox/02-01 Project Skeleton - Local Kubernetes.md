---
title: 02-01 Project Skeleton - Local Kubernetes
fileName: 02-01 Project Skeleton - Local Kubernetes
tags:
- kind
- k8s
categories:
- 
date: 2023-02-14
lastMod: 2023-02-16
ShowPostNavLinks: true
ShowBreadCrumbs: true
ShowReadingTime: true
---
# Description

The goal of the following sections will be to set up a basic infrastructure that will be used as a base for further developments. As such, I will try to put in place a local [Kubernetes](https://kubernetes.io/) (k8s) equivalent, on which I will deploy a [PostgreSQL](https://www.postgresql.org/) instance and a [Kafka](https://kafka.apache.org/) bus. I've chosen those 2 components because I'm used to them in my day-to-day work.

# Installation of a k8s equivalent

k8s is used in a lot of places, and I was looking forward to deepening my understanding of it. Which is why I choose a local variant to deploy the different components, even if I miss some features.

As for local versions of k8s, many are available:

  + [k3s](https://k3s.io/)

  + [MicroK8s](https://microk8s.io/)

  + [minikube](https://github.com/kubernetes/minikube)

  + [kind](https://kind.sigs.k8s.io/)

  + and probably others...

I've never heard of `kind` before (neither `microK8s`), so I decided to try it out. All I need is docker on my machine, which is well within my capabilities. I was initially going to install `k3s`, but seeing `sudo` usage in the examples made me hesitate...

The installation was easy, I just had to follow the [installation guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation):

```bash
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
$ chmod +x ./kind
# I just changed the destination folder
$ mv ./kind ~/.local/bin/kind
```

Once installed I specified a configuration file for my future cluster. [Mr. Sandman](https://www.youtube.com/watch?v=CX45pYvxDiA) was playing in my head while doing it, so there might be reference to it here and there:

```yaml
# a cluster with 3 control-plane nodes and 3 workers
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: sandman
nodes:
- role: control-plane
```

Then I applied the configuration to create my cluster (it can take some time):

```bash
$ kind create cluster --config config.yaml
$ kind get clusters
kind
sandman
```

*Voilà*, I have my local cluster to work on!
