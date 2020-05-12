# devops-k8s
Play k8s app


### EFK
Elasticsearch、Fluentd 和 Kibana（EFK）技术栈

```
cd efk
kubectl apply -f namespace-logging.yaml
htpasswd -c /root/kibana-auth admin
kubectl -n logging create secret generic kibana-auth --from-file=/root/kibana-auth
kubectl apply -f elasticsearch-storageclass-subpath.yaml 
kubectl apply -f elasticsearch-service.yaml
kubectl apply -f elasticsearch-statefulset.yaml 
kubectl apply -f fluentd-configmap.yaml
kubectl apply -f fluentd-daemonset.yaml
kubectl apply -f kibana-deployment.yaml
kubectl apply -f kibana-service.yaml
kubectl apply -f kibana-ingress-yaml  
```