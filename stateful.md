## 创建headless service

Zookeeper Stateful Set 需要创建一个headless的service来控制域名, 包括两个端口2888/3888，server／2888端口是为了follow与leader通信用，3888端口是为了leader选举

```
apiVersion: v1
kind: Service
metadata:
  name: zk-headless
  labels:
    app: zk-headless
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
```


## Stateful Set
Stateful Set 配置必须匹配Headless Service(.spec.serviceName), 同时必须提供replicas的数量,必须为奇数（3/5/7...)

## 查看stateful pod创建 (zk-0/1/2依次创建)
```
kubectl get pods -w -l app=zk
```
## 获取StatefulSet Pod的hostnames
出于leader选举的目的，Pod hostname会hardcode在其它的zk节点里，kubectl exec查看hostname
```
for i in 0 1 2; do kubectl exec zk-$i -- hostname; done
```
```
zk-0
zk-1
zk-2
```

StatefulSet controller提供给Pod一个唯一的 hostname，基于pod创建的顺序. hostnames 格式为 <statefulset name>-<ordinal index>. 这里replicas为3,  StatefulSet controller创建了3个 Pods zk-0, zk-1, and zk-2.

## 查看zookeeper的myid 文件
```
for i in 0 1 2; do echo "myid zk-$i";kubectl exec zk-$i -- cat /var/lib/zookeeper/data/myid; done
```

## 获取pod的FQDN (Fully Qualified Domain Name) 
```
for i in 0 1 2; do kubectl exec zk-$i -- hostname -f; done
```
```
zk-0.zk-headless.default.svc.cluster.local
zk-1.zk-headless.default.svc.cluster.local
zk-2.zk-headless.default.svc.cluster.local
```
上述A记录解析到Pod的IP，Pod被调度后IP地址变化，但是A record不会变

查看ZK的zoo.cfg
```
kubectl exec zk-0 -- cat /opt/zookeeper/conf/zoo.cfg
```
## 测试zk集群是否正常
zk-0写入数据
```
kubectl exec zk-0 zkCli.sh create /hello world
```
zk-1读／hello
```
kubectl exec zk-1 zkCli.sh get /hello
```
## 测试数据持久化
把zk集群删除
```
kubectl delete statefulset zk
```
```
kubectl get pods -w -l app=zk
```

重新apply yaml
```
kubectl apply -f https://k8s.io/docs/tutorials/stateful-application/zookeeper.yaml
```
```
statefulset "zk" created
Error from server (AlreadyExists): error when creating "zookeeper.yaml": services "zk-headless" already exists
Error from server (AlreadyExists): error when creating "zookeeper.yaml": configmaps "zk-config" already exists
Error from server (AlreadyExists): error when creating "zookeeper.yaml": poddisruptionbudgets.policy "zk-budget" already exists
```
```
kubectl get pods -w -l app=zk
```
检查之前创建的数据是否还在
```
kubectl exec zk-2 zkCli.sh get /hello
```
因为StatefulSet controller为每个pod创建PersistentVolumeClaim
```
volumeClaimTemplates:
  - metadata:
      name: datadir
      annotations:
        volume.alpha.kubernetes.io/storage-class: anything
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi
```
所以在zk集群重建时原来pod的PersistentVolumes被重新mount了，数据不会丢失
## ConfigMap注入配置
```
kubectl get cm zk-config -o yaml
apiVersion: v1
data:
  client.cnxns: "60"
  ensemble: zk-0;zk-1;zk-2
  init: "10"
  jvm.heap: 256M
  purge.interval: "0"
  snap.retain: "3"
  sync: "5"
  tick: "2000"
```

ConfigMap通过ENV的方式将配置注入到zk StatefulSet’s Pod
```
env:
        - name : ZK_ENSEMBLE
          valueFrom:
            configMapKeyRef:
              name: zk-config
              key: ensemble
        - name : ZK_HEAP_SIZE
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: jvm.heap
        - name : ZK_TICK_TIME
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: tick
```
zk的entrypoint会调用bash script, zkGenConfig.sh, 在launch  ZooKeeper server process前从环境变量产生zk的配置文件：
```
 command:
        - sh
        - -c
        - zkGenConfig.sh && zkServer.sh start-foreground
```
检查zk的环境变量
```
for i in 0 1 2; do kubectl exec zk-$i env | grep ZK_*;echo""; done
```
其中log4j也是通过zkGenConfig.sh生成的：
```
kubectl exec zk-0 cat /usr/etc/zookeeper/log4j.properties
kubectl logs zk-0 --tail 20
```
## 配置非特权用户
通过SecurityContext控制zk user为普通用户

StatefulSet:
```
securityContext:
  runAsUser: 1000
  fsGroup: 1000
```
查看zk当前的user
```
kubectl exec zk-0 -- ps -elf
```
因为fsGroup设置为1000， PersistentVolumes（/var/lib/zookeeper/data）的group为1000，使得zk user就可以访问这个目录
```
kubectl exec -ti zk-0 -- ls -ld /var/lib/zookeeper/data
```
## Stateful容错性
Restart Policies 控制pod的重启策略，对于stateful的pod应该设置为always
查看process tree
```
kubectl exec zk-0 -- ps -ef
```
打开两个terminal
TerminalA
```
kubectl get pod -w -l app=zk
```
在另外一个terminal，查看pod的重启状态
```
kubectl exec zk-0 -- pkill java
```
在另外一个terminal，delete zk-0的 zkOk.sh
```
kubectl exec zk-0 -- rm /opt/zookeeper/bin/zkOk.sh
```
livenessProbe（/bin/zkOk.sh）检查失败，pod重启

*测试node挂掉*

获取当前pod的分布：
```
for i in 0 1 2; do kubectl get pod zk-$i --template {{.spec.nodeName}}; echo ""; done
```
```
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values: 
                    - zk-headless
              topologyKey: "kubernetes.io/hostname"
```
因为podAntiAffinity的限制，app不会被调度到同一个node

cordon某一个运行pod的node（让该node驱逐该节点上的pods）
```
kubectl cordon < node name >
```
获取PodDisruptionBudget
```
kubectl get poddisruptionbudget zk-budget
```
```
NAME        MIN-AVAILABLE   ALLOWED-DISRUPTIONS   AGE
zk-budget   2               1                     1h
```
min-available表示zk至少有两个节点在线才能正常服务
新开一个terminal，观察pod
```
kubectl get pods -w -l app=zk
```
查看最新的pod的调度情况
```
for i in 0 1 2; do kubectl get pod zk-$i --template {{.spec.nodeName}}; echo ""; done
```
测试之前持久化的数据是否丢失
```
kubectl exec zk-0 zkCli.sh get /hello
```
