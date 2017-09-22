ingress.md

## kubectl create
```
kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/ingress/deployment.yaml
kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/ingress/service.yaml
kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/ingress/ingress-external.yaml
```

## check ingress
```
kubectl get ing nginx -o yaml 
```
## 获取AWS ELB

```
kubectl get ing nginx --template {{.status.loadBalancer.ingress}}
```

## Add DNS record in Route53

e.g.
```
name: boo.jiannc.com
A: dualstack.ab1a49df09dc411e78db10a7ed298e34-1240485153.us-east-1.elb.amazonaws.com.
```
## 验证
```
➜  ~ dig boo.jiannc.com +short
34.226.252.134
34.233.248.34
```
curl
```
curl boo.jiannc.com
```

查看ingress／nginx conf
```
kubectl -n kube-system exec `kubectl -n kube-system get pods | grep external-ingress | awk '{print $1}'` -- cat /etc/nginx/nginx.conf
...
    server {
        server_name boo.jiannc.com;
        listen 80 proxy_protocol;
        listen [::]:80 proxy_protocol;
        set $proxy_upstream_name "-";

        vhost_traffic_status_filter_by_set_key $geoip_country_code country::$server_name;

        location / {
            set $proxy_upstream_name "default-nginx-80";

            port_in_redirect off;

            client_max_body_size                    "1m";

            proxy_set_header Host                   $best_http_host;

            # Pass the extracted client certificate to the backend

            # Allow websocket connections
            proxy_set_header                        Upgrade           $http_upgrade;
            proxy_set_header                        Connection        $connection_upgrade;

            proxy_set_header X-Real-IP              $the_real_ip;
            proxy_set_header X-Forwarded-For        $the_real_ip;
            proxy_set_header X-Forwarded-Host       $best_http_host;
            proxy_set_header X-Forwarded-Port       $pass_port;
            proxy_set_header X-Forwarded-Proto      $pass_access_scheme;
            proxy_set_header X-Original-URI         $request_uri;
            proxy_set_header X-Scheme               $pass_access_scheme;

            # mitigate HTTPoxy Vulnerability
            # https://www.nginx.com/blog/mitigating-the-httpoxy-vulnerability-with-nginx/
            proxy_set_header Proxy                  "";

            # Custom headers to proxied server

            proxy_connect_timeout                   5s;
            proxy_send_timeout                      60s;
            proxy_read_timeout                      60s;

            proxy_redirect                          off;
            proxy_buffering                         off;
            proxy_buffer_size                       "4k";
            proxy_buffers                           4 "4k";

            proxy_http_version                      1.1;

            proxy_cookie_domain                     off;
            proxy_cookie_path                       off;

            # In case of errors try the next upstream server before returning an error
            proxy_next_upstream                     error timeout invalid_header http_502 http_503 http_504;

            proxy_pass http://default-nginx-80;
        }

    }
...
```
## 创建一个internal的ingress(只能内网访问)
创建internal ingress
```
kubectl create -f https://raw.githubusercontent.com/jianchen2580/k8s-example/master/ingress/ingress-internal.yaml
```
获取AWS ELB
```
kubectl get ing nginx-internal --template {{.status.loadBalancer.ingress}}
```
## Add DNS record in Route53

e.g.
```
name: boo-internal.jiannc.com
A: internal-aadc658d69dc411e78db10a7ed298e34-887477065.us-east-1.elb.amazonaws.com.
```

验证是否为内网IP

```
➜  ~ dig boo-internal.jiannc +short
192.168.57.107
192.168.43.12
```

查看ingress／nginx conf
```
kubectl -n kube-system exec `kubectl -n kube-system get pods | grep internal-ingress-nginx | awk '{print $1}'` -- cat /etc/nginx/nginx.conf

