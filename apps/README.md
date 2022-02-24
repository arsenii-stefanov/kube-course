## Getting Started

* Install Minikube

```
### Mac (x86-64)
os=darwin
arch=amd64

### Mac (ARM64)
os=darwin
arch=arm64

### Linux (x86-64)
os=linux
arch=amd64

### Linux (ARM64)
os=linux
arch=arm64

curl -L -o /usr/local/bin/minikube "https://storage.googleapis.com/minikube/releases/latest/minikube-${os}-${arch}"
chmod 0755 /usr/local/bin/minikube
```

* Start Minikube

> Get Started guide: https://minikube.sigs.k8s.io/docs/start/

```
minikube start
kubectl config current-context
```

If your current context is not `minikube`

```
kubectl config get-contexts
kubectl config set-context minikube
```

## Pod
-----

* Apply the K8s manifest

```
cd ./001-pod

kubectl get namespace
kubectl apply -f Pod.yaml
kubectl get pod -o wide
kubectl describe pod test-pod-1
kubectl logs -f test-pod-1 -c web-server
```

* Try out labels

```
kubectl get pod -l app=my-app,type=web-app
kubectl get pod -l app=my-app
```

* Check the networking INSIDE the Pod

```
kubectl exec -it test-pod-1 -c app -- sh
apk add curl net-tools
netstat -tlpn
curl 127.0.0.1:80
```

* Check the networking OUTSIDE the Pod

```
kubectl exec -it test-pod-2 -- sh
apk add curl net-tools
netstat -tlpn
curl 172.17.0.4:80
```

* Clean up

```
kubectl delete -f Pod.yaml
```


## Deployment
-----

* Ensure that the NGINX Ingress Controller is installed

```
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

* Apply the K8s manifests

```
cd ./002-stateless-web-app

kubectl get namespace

kubectl apply -f 001_Namespace.yaml
kubectl get ns

kubectl apply -f 002_Deployment.yaml 
kubectl -n stateless-apps get deployment,replicaset,pod -o wide

kubectl apply -f 003_Service_NodePort.yaml

kubectl -n stateless-apps get deployment,replicaset,pod,svc -o wide
kubectl -n stateless-apps describe svc my-app

kubectl -n stateless-apps get deployment,replicaset,pod,svc,ingress -o wide
kubectl -n stateless-apps describe ingress my-app

kubectl apply -f 005_Service_ClusterIP.yaml
kubectl -n stateless-apps describe svc my-app-cluster-ip

kubectl apply -f 006_Ingress_ClusterIP.yaml
kubectl -n stateless-apps describe ingress my-app-cluster-ip

kubectl -n stateless-apps get deployment,replicaset,pod,svc,ingress -o wide
```

* Open a new Terminal tab and start a tunnel

```
minikube tunnel
```

* Check the Ingress

```
curl -H "Host: site.example" 127.0.0.1   ### this command should return a 404 error
curl -H "Host: site.ingress-via-node-port.example" 127.0.0.1   ### this command should return a default NGINX page
curl -H "Host: site.ingress-via-cluster-ip.example" 127.0.0.1   ### this command should return a default NGINX page
```

* Clean up

```
kubectl delete -f ./
```
