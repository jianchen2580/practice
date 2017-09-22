
# Pod Hand-on

## Two-Container Pod

创建一个运行两个container的pod
```
kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/pod/two-container-pod.yaml
```
查看Pod信息：
```
kubectl get pod two-containers  -o yaml
kubectl exec two-containers -c nginx-container -- ls /usr/share/nginx/html
kubeclt exec two-containers -c nginx-container -- cat  /usr/share/nginx/html/index.html
```
通过busybox Pod访问nginx Pod
```
kubectl run curl --image=radial/busyboxplus:curl -it --rm=true
If you don't see a command prompt, try pressing enter.
[ root@curl-2716574283-5jkbk:/ ]$
[ root@curl-2716574283-5jkbk:/ ]$ curl 100.111.83.6 # pod IP
```
查看nginx log
```
kb log -f two-containers -c nginx-container
```

## Liveness/Readiness Probe
### 创建pod

```
kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/pod/exec-liveness.yaml
```

### 观察pod的events（30秒内）

```
kubectl describe pod liveness-exec
```
输出表面目前没有failed liveness probes
```
FirstSeen    LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
24s       24s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "gcr.io/google_containers/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "gcr.io/google_containers/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
```

*30秒后再次查看事件*

```
kubectl describe pod liveness-exec

FirstSeen LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
37s       37s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "gcr.io/google_containers/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "gcr.io/google_containers/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
2s        2s      1   {kubelet worker0}   spec.containers{liveness}   Warning     Unhealthy   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

liveness probes 失败，container被杀死后重启 （需要几十秒）
```
kubectl get pod liveness-exec
NAME            READY     STATUS    RESTARTS   AGE
liveness-exec   1/1       Running   1          1m
```

### liveness http request

```
kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/pod/http-liveness.yaml
```
container是一个简单的go http server, 10s后返回500
server.go
```
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```
10s后 liveness probe 失败
```
kubectl describe pod liveness-http
```

## TCP liveness probe (略)

## Pod lifecycle 

创建Pod
```
kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/pod/lifecycle-events.yaml
```
```
kubectl get pod lifecycle-demo
```
```
kubectl exec -it lifecycle-demo -- /bin/bash
```
验证postStart 创建message file
```
root@lifecycle-demo:/# cat /usr/share/message
```

