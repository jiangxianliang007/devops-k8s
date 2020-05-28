# devops-k8s
Play k8s app

### 持久化存储 -- 阿里云NAS
```
kubectl apply -f https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver/blob/master/deploy/ack/csi-provisioner.yaml
kubectl apply -f https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver/blob/master/deploy/ack/csi-plugin.yaml
```
```
kubectl get pod -o wide -n kube-system | grep csi
csi-plugin-mlgc9                           9/9     Running   0          17d   172.16.1.79       k8s-test-node-01   <none>           <none>
csi-plugin-p9hws                           9/9     Running   0          17d   172.16.1.81       k8s-test-node-02   <none>           <none>
csi-plugin-pxghp                           9/9     Running   0          17d   172.16.1.80       k8s-test-master    <none>           <none>
csi-provisioner-5d8cdd575-9tpdp            8/8     Running   0          17d   172.16.1.80       k8s-test-master    <none>           <none>
csi-provisioner-5d8cdd575-9vtx9            8/8     Running   0          17d   172.16.1.81       k8s-test-node-02   <none>           <none>

```
#### 静态存储卷挂载
静态存储卷挂载需要先在阿里云文件存储NAS 创建文件系统,点击挂载使用获取存储地址,填到 storageclass-subpath.yaml 文件的server
```
kubectl apply -f storageclass-subpath.yaml
kubectl apply -f pvc-subpath.yaml  
```

#### 动态存储卷挂载
动态存储卷挂载 需要将可用的阿里云ACCESS_KEY_ID 和 ACCESS_KEY_SECRET 加到csi-plugin.yaml 

修改 storageclass-filesystem.yaml   zoneId、vpcId、vSwitchId
```
kubectl apply -f storageclass-filesystem.yaml
kubectl apply -f pvc-filesystem.yaml
```
创建好存储卷后，部署应用时 volumes 对应到创建好的 persistentVolumeClaim

以jenkins持久化为例：

jenkins-deployment.yaml 增加volumes
```
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: nas-csi-pvc-jenkins
```

```
root@k8s-test-master:~/nas# kubectl get sc
NAME                                 PROVISIONER                       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
alicloud-nas-fs                      nasplugin.csi.alibabacloud.com    Delete          Immediate              false                  34d
alicloud-nas-subpath                 nasplugin.csi.alibabacloud.com    Retain          Immediate              false                  34d
alicloud-nas-subpath-elasticsearch   nasplugin.csi.alibabacloud.com    Retain          Immediate              true                   7d20h
alicloud-nas-subpath-jenkins         nasplugin.csi.alibabacloud.com    Retain          Immediate              false                  21d
alicloud-nas-subpath-k8s             nasplugin.csi.alibabacloud.com    Retain          Immediate              false                  34d

root@k8s-test-master:~/nas# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                   STORAGECLASS                         REASON   AGE
nas-0b2e6020-42e6-4e95-a283-58dd250be423   20Gi       RWX            Retain           Bound       default/nas-csi-pvc-k8s                 alicloud-nas-subpath-k8s                      34d
nas-1c5f9963-7d3f-47f6-ac01-29198903a547   200Gi      RWO            Retain           Bound       logging/data-es-1                       alicloud-nas-subpath-elasticsearch            5d21h
nas-51e55947-a00d-459d-b902-50ffcba71120   200Gi      RWO            Retain           Bound       logging/data-es-2                       alicloud-nas-subpath-elasticsearch            5d21h
nas-9d6c6f6b-f84e-44ba-a1c4-b79357c8ab36   20Gi       RWX            Retain           Bound       jenkins/nas-csi-pvc-jenkins             alicloud-nas-subpath-jenkins                  21d
nas-afc99fa7-b6ba-45e4-a6a7-96acb7e229f2   200Gi      RWO            Retain           Bound       logging/data-es-0                       alicloud-nas-subpath-elasticsearch            5d21h

root@k8s-test-master:~/nas# kubectl get pvc -A
NAMESPACE     NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                         AGE
default       nas-csi-pvc-k8s             Bound    nas-0b2e6020-42e6-4e95-a283-58dd250be423   20Gi       RWX            alicloud-nas-subpath-k8s             34d
jenkins       nas-csi-pvc-jenkins         Bound    nas-9d6c6f6b-f84e-44ba-a1c4-b79357c8ab36   20Gi       RWX            alicloud-nas-subpath-jenkins         21d
logging       data-es-0                   Bound    nas-afc99fa7-b6ba-45e4-a6a7-96acb7e229f2   200Gi      RWO            alicloud-nas-subpath-elasticsearch   5d21h
logging       data-es-1                   Bound    nas-1c5f9963-7d3f-47f6-ac01-29198903a547   200Gi      RWO            alicloud-nas-subpath-elasticsearch   5d21h
logging       data-es-2                   Bound    nas-51e55947-a00d-459d-b902-50ffcba71120   200Gi      RWO            alicloud-nas-subpath-elasticsearch   5d21h
```

### ingress-nginx
kubectl apply -f ./ingress

### jenkins
kubectl apply -f ./jenkins

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

### prometheus
使用 coreos 的   [kube-prometheus](https://github.com/coreos/kube-prometheus)

默认没有持久化，且只保留了30小时内的数据，自建一个 StorageClass 在 prometheus-prometheus.yaml 末尾增加存储配置

```
  retention: '90d'
  storage:
    volumeClaimTemplate:
      apiVersion: v1
      kind: PersistentVolumeClaim
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
        storageClassName: alicloud-nas-subpath-prometheus
```

### CITA
[CITA](https://github.com/citahub/cita) 是一个高性能的企业级区块链内核
 
使用官方提供的image: cita/cita-release:20.2.0-secp256k1-sha3，以下启动4个节点
  
```
kubectl create namespace cita-chain
kubectl apply -f cita/cita-storageclass-subpath.yaml
kubectl apply -f cita/cita-pvc-subpath.yaml
kubectl apply -f cita/cita-job-initchain.yaml
kubectl apply -f cita/
```
在任意k8s集群node上执行，查看区块高度是否增长
```
curl -sX POST --data '{"jsonrpc":"2.0","method":"blockNumber","params":[],"id":83}' 127.0.0.1:31337 |jq
{
  "jsonrpc": "2.0",
  "id": 83,
  "result": "0xa99"
}
```

#### 管理工具推荐：
[kuboard](https://github.com/eip-work/kuboard-press) 和 [k9s ](https://github.com/derailed/k9s)