# 安装calio网络插件 (忽略，通过kops已安装)

*建议在自己创建的namespace下练习，避免命名冲突*

```
$ kubectl config set-context $(kubectl config current-context) --namespace=<insert-namespace-name-here>
```
# 首先部署一个nginx服务

```
$ kubectl run nginx --image=nginx --replicas=2
$ kubectl expose deployment nginx --port=80
```

此时，通过其他Pod是可以访问nginx服务的
```
$ kubectl run curl --image=radial/busyboxplus:curl -i -t
Waiting for pod default/curl-2421989462-wxrcd to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget --spider --timeout=1 nginx
Connecting to nginx (100.71.110.244:80)
/ #
```
# default namespace的DefaultDeny Network Policy
开启default namespace的DefaultDeny Network Policy后，其他Pod（包括namespace外部）不能访问nginx了：

```
# annotate适用于v1.6/v1.7，v1.7+支持创建默认拒绝策略, 需替换default成你们自己的namespace
$ kubectl annotate ns default "net.beta.kubernetes.io/network-policy={\"ingress\": {\"isolation\": \"DefaultDeny\"}}"
```

```
$ kubectl run busybox --rm -ti --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.100.0.16:80)
wget: download timed out
/ #
```

最后再创建一个运行带有access=true的Pod访问的网络策略

```
kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/network_policy/access-nginx.yaml
networkpolicy "access-nginx" created
```

# 不带access=true标签的Pod还是无法访问nginx服务
```
$ kubectl run busybox --rm -ti --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget --spider --timeout=1 nginx 
Connecting to nginx (10.100.0.16:80)
wget: download timed out
/ #
```

# 而带有access=true标签的Pod可以访问nginx服务

```
$ kubectl run busybox --rm -ti --labels="access=true" --image=busybox /bin/sh
Waiting for pod default/busybox-472357175-y0m47 to be running, status is Pending, pod ready: false

Hit enter for command prompt

/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.100.0.16:80)
/ #
```

最后开启nginx服务的外部访问：
```
$ kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/network_policy/nginx-external-policy.yaml
```