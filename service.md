## 创建nginx deployment
```
kb apply -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/service/nginx-deployment.yaml
```
## 创建默认的clusterIP service
```
kb apply -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/service/nginx-service.yaml
```
## 测试clusterIP service
```
kb run curl --image=radial/busyboxplus:curl -it --rm=true
If you don't see a command prompt, try pressing enter.
[ root@curl-2716574283-w4q8n:/ ]$ curl nginx-service-clusterip
```
## 创建nodePort service
```
kb apply -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/service/nginx-service-nodeport.yaml
```

## 获取node ExternalIP

```
 kb get nodes  -o yaml
```
例如：
```
kb get nodes  -o yaml
...
  status:
    addresses:
    - address: 192.168.61.121
      type: InternalIP
    - address: 34.228.23.175
      type: ExternalIP
    - address: ip-192-168-61-121.ec2.internal
      type: InternalDNS
    - address: ec2-34-228-23-175.compute-1.amazonaws.com
      type: ExternalDNS
...
```
## 本地访问测试
```
curl 34.228.23.175:31839
curl node2-ExternalIP:nodePort
```

## LoadBalance

```
kb apply -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/service/nginx-service-lb.yaml
```
获取elb
```
kb get svc nginx-service-loadbalancer --template {{.status.loadBalancer}}
```
通过elb访问nginx
```
curl a247e458e9dc211e78db10a7ed298e34-1809811406.us-east-1.elb.amazonaws.com
```