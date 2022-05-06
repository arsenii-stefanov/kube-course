## Stateful Web App

* Deploy the application and database

```
cd apps/003-stateful-app/
kubectl apply -f .
kubectl apply -f database/
kubectl -n stateful-apps get configmap,secret,pvc,pv,deployment,sts,pod,svc,ingress -o wide
```

* Check volume sharing and init script

```
kubectl -n stateful-apps logs -f -l app=my-app -c app
kubectl -n stateful-apps exec -it $(kubectl -n stateful-apps get pod -l app=my-app | grep -v NAME | awk '{print $1}') -c nginx -- sh
ls -lah /var/www/html
ls -lah /var/run/php
cat /var/www/html/SETUP_SCRIPT_OUTPUT.txt
```

* Check MySQL connection via Service's DNS name

```
kubectl -n stateful-apps exec -it $(kubectl -n stateful-apps get pod -l app=my-app | grep -v NAME | awk '{print $1}') -c app -- bash
apt update
apt install -y mycli
# mycli -h "$WORDPRESS_DB_HOST" -u "$WORDPRESS_DB_USER" -p${WORDPRESS_DB_PASSWORD} "$WORDPRESS_DB_NAME"
mycli -h mysql.stateful-apps.svc.cluster.local -u "$WORDPRESS_DB_USER" -p${WORDPRESS_DB_PASSWORD} "$WORDPRESS_DB_NAME"
```

* Add the website's domain name to '/etc/hosts' and start a Minikube tunnel

```
echo '127.0.0.1 my-wp-site.example' | sudo tee -a /etc/hosts
minikube tunnel
```

* Open your browser and navigate to http://my-wp-site.example

* Clean up

```
kubectl delete -f .
kubectl delete -f database/
kubectl get pv
kubectl delete pv <persistent_volume_name>
```