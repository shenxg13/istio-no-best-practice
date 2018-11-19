# Istio基于sniHosts的TLS请求路由的相关Envoy配置解析（TCP）

## 概述

本文介绍了在Isito中直接使用TLS的两种场景：

1. 外部服务通过Igress Gateway访问Mesh内部的TLS服务。

2. Mesh内部服务通过Service Entry和Egress Gateway访问外部的TLS服务。

TLS类型的路由属于TCP，所以无Envoy route相关配置，路由均在listener中进行。

本文也为了对TCP类型的路由在Envoy中的配置进行一些解析。



## Ingress TLS

### 相关拓扑

![istio-tls-sni-1-1](../images/istio-tls-sni-1-1.png)

- 外部客户端发送相关tls请求。
- 请求被发送至lb，并由lb转发至ingressgateway pod。
- ingressgateway中的istio-proxy container根据相关snihosts路由规则将请求转发至相关nginx pod。
- 请求被nginx pod中的istio-proxy container截获，并转发至nginx container。



![istio-tls-sni-1-2](../images/istio-tls-sni-1-2.png)

- 因为tls类型的路由属于tcp，所以无envoy route相关配置，路由均在listener中进行。
- 2个nginx pod，label为别为version: v1和version: v2。
- 使用istio destinationrule为不同的label定义不同的subset。
- 使用istio gateway定义ingressgateway接收443端口的请求。
- 将ingressgateway的443端口的tls策略设置为passthrough。
- 在ingressgateway中使用istio virtualservice定义到两个节点的基于tls的envoy route。
- 指定sni名称为*.v1.nginx.internal.istio的请求被路由到subset v1。
- 指定sni名称为*.v2.nginx.internal.istio的请求被路由到subset v2。



### 相关配置

#### DestinationRule

```bash
openssl req \
-newkey rsa:4096 -nodes -sha256 -keyout 1st.nginx.key \
-out 1st.nginx.csr

echo subjectAltName = DNS:www.1st.nginx > 1st.nginx.cnf

openssl x509 \
-req -days 3655 -in 1st.nginx.csr -CA ca.crt -CAkey ca.key \
-CAcreateserial -extfile 1st.nginx.cnf -out 1st.nginx.crt

openssl req \
-newkey rsa:4096 -nodes -sha256 -keyout 2nd.nginx.key \
-out 2nd.nginx.csr

echo subjectAltName = DNS:www.2nd.nginx > 2nd.nginx.cnf

openssl x509 \
-req -days 3655 -in 2nd.nginx.csr -CA ca.crt -CAkey ca.key \
-CAcreateserial -extfile 2nd.nginx.cnf -out 2nd.nginx.crt

kubectl create secret tls sec-nginx-v1 --key ./1st.nginx.key --cert ./1st.nginx.crt
kubectl create secret tls sec-nginx-v2 --key ./2nd.nginx.key --cert ./2nd.nginx.crt
```

- 为nginx-v1 pod配置证书和私钥，并将证书和私钥生成k8s secret sec-nginx-v1。
- 为nginx-v2 pod配置证书和私钥，并将证书和私钥生成k8s secret sec-nginx-v2。



```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: 192.168.0.61/istio-example/nginx
        ports:
        - containerPort: 443
        volumeMounts:
        - mountPath: /etc/nginx/conf.d/
          readOnly: true
          name: conf
        - mountPath: /etc/nginx/html/
          readOnly: true
          name: index
        - mountPath: /etc/nginx/certs
          readOnly: true
          name: certs
      volumes:
      - name: conf
        configMap:
          name: cm-nginx-v1
          items:
            - key: default.conf
              path: default.conf
      - name: index
        configMap:
          name: cm-nginx-v1
          items:
            - key: index.html
              path: index.html
      - name: certs
        secret:
          secretName: sec-nginx-v1
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-nginx-v1
data:
  default.conf: |
    server {
      listen       443 ssl;
      server_name  loalhost;

      location / {
        root   /etc/nginx/html/;
        index  index.html index.htm;
      }

      ssl_certificate      /etc/nginx/certs/tls.crt;
      ssl_certificate_key  /etc/nginx/certs/tls.key;
      ssl_verify_depth     2;

      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
        root   /usr/share/nginx/html;
      }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
      body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
      }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <h1>nginx01</h1>
    <p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>
    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>
    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
        version: v2
    spec:
      containers:
      - name: nginx
        image: 192.168.0.61/istio-example/nginx
        ports:
        - containerPort: 443
        volumeMounts:
        - mountPath: /etc/nginx/conf.d/
          readOnly: true
          name: conf
        - mountPath: /etc/nginx/html/
          readOnly: true
          name: index
        - mountPath: /etc/nginx/certs
          readOnly: true
          name: certs
      volumes:
      - name: conf
        configMap:
          name: cm-nginx-v2
          items:
            - key: default.conf
              path: default.conf
      - name: index
        configMap:
          name: cm-nginx-v2
          items:
            - key: index.html
              path: index.html
      - name: certs
        secret:
          secretName: sec-nginx-v2
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-nginx-v2
data:
  default.conf: |
    server {
      listen       443 ssl;
      server_name  loalhost;

      location / {
        root   /etc/nginx/html/;
        index  index.html index.htm;
      }

      ssl_certificate      /etc/nginx/certs/tls.crt;
      ssl_certificate_key  /etc/nginx/certs/tls.key;
      ssl_verify_depth     2;

      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
        root   /usr/share/nginx/html;
      }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
      body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
      }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <h1>nginx02</h1>
    <p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>
    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>
    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  ports:
  - port: 443
    name: https
  selector:
    app: nginx
```

- 使用configmap cm-nginx-v1定制nginx-v1的配置文件和测试页面。
- 使用configmap cm-nginx-v2定制nginx-v2的配置文件和测试页面。
- 配置nginx-v1 deployment，挂载configmap cm-nginx-v1和secret sec-nginx-v1。
- 配置nginx-v2 deployment，挂载configmap cm-nginx-v1和secret sec-nginx-v1。
- 配置nginx service，端口为443，名称为https。



```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-sni
spec:
  host: nginx.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

- destinationrule相关配置。
- 为cluster nginx.default.svc.cluster.local定义subset。
- 将label为version: v1的endpoint deployment定义为subset v1。
- 将label为version: v2的endpoint deployment定义为subset v2。



```json
{
        "name": "outbound|443|v1|nginx.default.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|443|v1|nginx.default.svc.cluster.local"
        },
        "connectTimeout": "1.000s",
        "circuitBreakers": {
            "thresholds": [
                {}
            ]
        }
    }

{
        "name": "outbound|443|v2|nginx.default.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|443|v2|nginx.default.svc.cluster.local"
        },
        "connectTimeout": "1.000s",
        "circuitBreakers": {
            "thresholds": [
                {}
            ]
        }
    }
```

- envoy cluster相关配置。
- 在destinationrule定义完成后，envoy会生成2个subset相关的cluster，分别为outbound|443|v1|nginx.default.svc.cluster.local和outbound|443|v2|nginx.default.svc.cluster.local。



```json
{"svc": "nginx.default.svc.cluster.local:https", "ep": [
{
    "endpoint": {
      "Family": 0,
      "Address": "10.234.0.58",
      "Port": 443,
      "ServicePort": {
        "name": "https",
        "port": 443,
        "protocol": "HTTPS"
      },
…
    "labels": {
      "app": "nginx",
      "pod-template-hash": "3875363210",
      "version": "v2"
    },
…
    "endpoint": {
      "Family": 0,
      "Address": "10.234.0.59",
      "Port": 443,
      "ServicePort": {
        "name": "https",
        "port": 443,
        "protocol": "HTTPS"
      },
…
    "labels": {
      "app": "nginx",
      "pod-template-hash": "1126943858",
      "version": "v1"
    },
```

- envoy endpoint相关配置。
- 这2个cluster会与envoy endpoint相关联。



#### Gateway和VirtualService

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gw-sni
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
    hosts:
    - "*"
```

- gateway相关配置。
- 为ingress gateway设置监听。
- 打开ingress gateway的443端口进行监听，端口类型为https。
- tls类型为passthrough。
- 所有host均可以注册到这个监听。



```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-sni-gw
spec:
  hosts:
  - "*.v1.nginx.internal.istio"
  - "*.v2.nginx.internal.istio"
  gateways:
  - gw-sni
  tls:
  - match:
    - sniHosts:
      - "*.v1.nginx.internal.istio"
    route:
    - destination:
        host: nginx.default.svc.cluster.local
        port: 
          number: 443
        subset: v1
  - match:
    - sniHosts:
      - "*.v2.nginx.internal.istio"
    route:
    - destination:
        host: nginx.default.svc.cluster.local
        port: 
          number: 443
        subset: v2
```

- ingress gateway层面virtualservice相关配置。
- ingress gateway在接收到外部lb转发的请求后，按照sni的信息，对请求进行路由。
- sni为\*.v1.nginx.internall.istio的443端口的请求，被转发至nginx.default.svc.cluster.local的v1版本的443端口。
- sni为\*.v2.nginx.internal.istio的443端口的请求，被转发至nginx.default.svc.cluster.local的v2版本的443端口。



```yaml
{
        "name": "0.0.0.0_443",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 443
            }
        },
        "filterChains": [
            {
                "filterChainMatch": {
                    "serverNames": [
                        "*.v1.nginx.internal.istio"
                    ]
                },
…
                    {
                        "name": "envoy.tcp_proxy",
                        "config": {
                            "cluster": "outbound|443|v1|nginx.default.svc.cluster.local",
                            "stat_prefix": "outbound|443|v1|nginx.default.svc.cluster.local"
                        }
                    }
…
            {
                "filterChainMatch": {
                    "serverNames": [
                        "*.v2.nginx.internal.istio"
                    ]
                },
…
                    {
                        "name": "envoy.tcp_proxy",
                        "config": {
                            "cluster": "outbound|443|v2|nginx.default.svc.cluster.local",
                            "stat_prefix": "outbound|443|v2|nginx.default.svc.cluster.local"
                        }
                    }
```

- ingress gateway envoy listener相关配置。
- 因为virtualservice类型为tls，所以路由会在listener中直接进行。
- ingress gateway envoy listener的filterchain会将servername为\*.v1.nginx.internal.istio的请求全部转发至outbound|443|v1|nginx.default.svc.cluster.local。
- egress gateway envoy listener的filterchain会将servername为\*.v2.nginx.internal.istio的请求全部转发至outbound|443|v2|nginx.default.svc.cluster.local。



### 测试结果

```bash
[~]$ curl https://api.v1.nginx.internal.istio -k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
  body {
    width: 35em;
    margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif;
  }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<h1>nginx01</h1>
<p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[~]$

[~]$ curl https://api.v2.nginx.internal.istio -k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
  body {
    width: 35em;
    margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif;
  }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<h1>nginx02</h1>
<p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[~]$
```

- 测试结果。

- 到api.v1.nginx.internal.istio的请求被正确转发至nginx-v1。
- 到api.v2.nginx.internal.istio的请求被正确转发至nginx-v2。



## Egress TLS

### 相关拓扑

![egress-serviceentry-egressgateway-structure](../images/istio-tls-sni-2-1.png)

- sleep pod中的sleep container发送相关tls请求。
- 请求被sleep pod中的istio-proxy container截获，并根据路由规则转发至egressgateway pod。
- egressgateway中的istio-proxy container根据相关snihosts路由规则将请求转发至相关外部服务。



![egress-serviceentry-egressgateway-mapping](../images/istio-tls-sni-2-2.png)

- 因为tls类型的路由属于tcp，所以无envoy route相关配置，路由均在listener中进行。
- 使用istio serviceentry定义外部的nginx服务到envoy cluster和endpoint中的节点。
- 两个节点分别对应nginx01 vm和nginx02 vm。
- 为外部的nginx服务在serviceentry中定义label。
- 使用istio destinationrule为不同的label定义不同的subset。
- 在mesh中使用istio virtualservice定义所有到*.nginx.external.istio的请求均被路由至egressgateway。
- 使用istio gateway定义egressgateway接收443端口的请求。
- 将egressgateway的443端口的tls策略设置为passthrough。
- 在egressgateway中使用istio virtualservice定义到两个节点的基于tls的envoy route。
- 指定sni名称为*.v1.nginx.external.istio的请求被路由到nginx01。
- 指定sni名称为*.v2.nginx.external.istio的请求被路由到nginx02。



### 相关配置

#### ServiceEntry和DestinationRule

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: se-sni
spec:
  hosts:
  - "*.nginx.external.istio"
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: STATIC
  endpoints:
  - address: 192.168.0.63
    ports:
      https: 443
    labels:
      version: v1
  - address: 192.168.0.64
    ports:
      https: 443
    labels:
      version: v2
```

- serviceentry相关配置。
- 在serviceentry中定义cluster的fqdn为*.nginx.external.istio。
- 为serviceentry添加2个endpoint，分别为192.168.0.63的443端口，和192.168.0.64的443端口。
- 192.168.0.63的label定义为version: v1。
- 192.168.0.64的label定义为version: v2。



```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-sni
spec:
  host: "*.nginx.external.istio"
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

- destinationrule相关配置。
- 为cluster *.nginx.external.istio定义subset。
- 将label为version: v1的endpoint定义为subset v1。
- 将label为version: v2的endpoint定义为subset v2。



```json
{
        "name": "outbound|443|v1|*.nginx.external.istio",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|443|v1|*.nginx.external.istio"
        },
        "connectTimeout": "1.000s",
        "circuitBreakers": {
            "thresholds": [
                {}
            ]
        }
    }


{
        "name": "outbound|443|v2|*.nginx.external.istio",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|443|v2|*.nginx.external.istio"
        },
        "connectTimeout": "1.000s",
        "circuitBreakers": {
            "thresholds": [
                {}
            ]
        }
    }
```

- envoy cluster相关配置。
- 在serviceentry和destinationrule定义完成后，envoy会生成2个subset相关的cluster，分别为outbound|443|v1|\*.nginx.external.istio和outbound|443|v2|\*.nginx.external.istio。



```json
{"svc": "*.nginx.external.istio:https", "ep": [
{
    "endpoint": {
      "Family": 0,
      "Address": "192.168.0.63",
      "Port": 443,
      "ServicePort": {
        "name": "https",
        "port": 443,
        "protocol": "HTTPS"
      },
…
    "labels": {
      "version": "v1"
    }
…
{
    "endpoint": {
      "Family": 0,
      "Address": "192.168.0.64",
      "Port": 443,
      "ServicePort": {
        "name": "https",
        "port": 443,
        "protocol": "HTTPS"
      },
…
    "labels": {
      "version": "v2"
    }
```

- envoy endpoint相关配置。
- 这2个cluster会与envoy endpoint相关联。



#### Mesh层面VirtualService

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-sni-mesh
spec:
  hosts:
  - "*.nginx.external.istio"
  tls:
  - match:
    - sniHosts:
      - "*.nginx.external.istio"
      port: 443
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        port:
          number: 443
```

- mesh层面virtualservice相关配置。
- virtualservice的路由类型为tls。
- 将所有到*.nginx.external.istio的443端口的请求全部转发至istio-egressgateway.istio-system.svc.cluster.local，也就是istio的egress gateway。



```json
{
        "name": "0.0.0.0_443",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 443
            }
        },
        "filterChains": [
            {
                "filterChainMatch": {
                    "serverNames": [
                        "*.nginx.external.istio"
                    ]
                },
…
                    {
                        "name": "envoy.tcp_proxy",
                        "config": {
                            "cluster": "outbound|443||istio-egressgateway.istio-system.svc.cluster.local",
                            "stat_prefix": "outbound|443||istio-egressgateway.istio-system.svc.cluster.local"
                        }
                    }
```

- envoy listener相关配置。
- 因为virtualservice类型为tls，所以路由会在listener中直接进行。
- envoy listener的filterchain会将所有servername为*.nginx.external.istio的请求全部转发至outbound|443||istio-egressgateway.istio-system.svc.cluster.local这个cluster。



#### Gateway和Gateway层面VirtualService

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gw-sni
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
    hosts:
    - "*"
```

- gateway相关配置。
- 为egress gateway设置监听。
- 打开egress gateway的443端口进行监听，端口类型为https。
- tls类型为passthrough。
- 所有host均可以注册到这个监听。



```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-sni-gw
spec:
  hosts:
  - "*.v1.nginx.external.istio"
  - "*.v2.nginx.external.istio"
  gateways:
  - gw-sni
  tls:
  - match:
    - sniHosts:
      - "*.v1.nginx.external.istio"
    route:
    - destination:
        host: "*.nginx.external.istio"
        port: 
          number: 443
        subset: v1
  - match:
    - sniHosts:
      - "*.v2.nginx.external.istio"
    route:
    - destination:
        host: "*.nginx.external.istio"
        port: 
          number: 443
        subset: v2
```

- egress gateway层面virtualservice相关配置。
- egress gateway在接收到mesh内部的请求后，按照sni的信息，对请求进行路由。
- sni为\*.v1.nginx.external.istio的443端口的请求，被转发至\*.nginx.external.istio的v1版本的443端口。
- sni为\*.v2.nginx.external.istio的443端口的请求，被转发至\*.nginx.external.istio的v2版本的443端口。



```json
{
        "name": "0.0.0.0_443",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 443
            }
        },
        "filterChains": [
            {
                "filterChainMatch": {
                    "serverNames": [
                        "*.v1.nginx.external.istio"
                    ]
                },
…
                    {
                        "name": "envoy.tcp_proxy",
                        "config": {
                            "cluster": "outbound|443|v1|*.nginx.external.istio",
                            "stat_prefix": "outbound|443|v1|*.nginx.external.istio"
                        }
                    }
…
            {
                "filterChainMatch": {
                    "serverNames": [
                        "*.v2.nginx.external.istio"
                    ]
                },
…
                    {
                        "name": "envoy.tcp_proxy",
                        "config": {
                            "cluster": "outbound|443|v2|*.nginx.external.istio",
                            "stat_prefix": "outbound|443|v2|*.nginx.external.istio"
                        }
                    }
```

- egress gateway envoy listener相关配置。
- 因为virtualservice类型为tls，所以路由会在listener中直接进行。
- egress gateway envoy listener的filterchain会将servername为\*.v1.nginx.external.istio的请求全部转发至outbound|443|v1|\*.nginx.external.istio这个cluster。
- egress gateway envoy listener的filterchain会将servername为\*.v2.nginx.external.istio的请求全部转发至outbound|443|v2|\*.nginx.external.istio这个cluster。



### 测试结果

```bash
/ # curl https://api.v1.nginx.external.istio --resolve api.v1.nginx.external.istio:443:10.235.0.11 -k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<h1>first!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ #

/ # curl https://api.v2.nginx.external.istio --resolve api.v2.nginx.external.istio:443:10.235.0.11 -k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<h1>second!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ #
```

- 测试结果。

- 到api.v1.nginx.external.istio的请求被正确转发至nginx01。
- 到api.v2.nginx.external.istio的请求被正确转发至nginx02。