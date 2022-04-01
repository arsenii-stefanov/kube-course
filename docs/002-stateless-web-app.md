## Stateless Web App
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

```
kubectl -n stateless-appss rollout history deployment my-app
kubectl -n dev-tools rollout undo deployment my-app --to-revision=1
```

* Clean up

```
kubectl delete -f ./
```
