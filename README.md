# Getting Started
---

### Intro

Kubernetes or K8s (because it's K + [8 characters] + s) is an orchestrator for Linux containers. In short, you can describe an application that consists of several microservices, a webserver or a load balancer, a database and a store, in just a few files (or even one file) and deploy it with just one CLI command.

K8s is an "easy to learn, hard to master" tool. It has a huge (and awesome) documentation with lots of useful info and examples: https://kubernetes.io/docs/concepts/

Before you get to know K8s (Kubernetes) better, you need an environment where you can deploy various applications via K8s manifests.

There are several options:
* A self-hosted K8s cluster running on either bare metal servers or virtual machines. Creating such a cluster is a time consuming process which also requiers certin knowledge and skills. That said, it's not the best choice for a playground
* A cloud K8s cluster such as AWS EKS, Google GKE, Azure AKS, etc. Basically, all these cloud providers use the same K8s services; however, behind the scenes K8s itself uses resources specific to each cloud provider. For example, Ingress on AWS uses AWS ALB and Ingress on GCP uses GCP HTTP Load Balancer. While K8s makes the process of configuring these load balancers unfied via the same set of Ingress directives which are then transformed into API calls that only a certain cloud provider understands, there still may be differences in how the balancers perform and what features thay have because they are different programs. It is relatively easy to spin up such a cluster, however I would recommend this option only if you have a trial in any of these clouds
* Local mini K8s cluster: Kubernetes IN Docker or kind, CodeReady Containers or CRC (OpenShift 4.x), Minishift (OpenShift 3.x) but currently Minikube is #1 choise for a K8s playground

## Install Minikube

### Set Shell variables

* for macOS (x86-64)

```
os=darwin
arch=amd64
```

* for macOS (ARM64)

```
os=darwin
arch=arm64
```

* for Linux (x86-64)

```
os=linux
arch=amd64
```

* for Linux (ARM64)

```
os=linux
arch=arm64
```

### Download a Minikube binary

```
curl -L -o /usr/local/bin/minikube "https://storage.googleapis.com/minikube/releases/latest/minikube-${os}-${arch}"
chmod 0755 /usr/local/bin/minikube
```

### Start Minikube

> Get Started guide: https://minikube.sigs.k8s.io/docs/start/

```
minikube start
```

### Check the current context

```
kubectl config current-context
```

If your current context is not `minikube`

```
kubectl config get-contexts
kubectl config set-context minikube
```
