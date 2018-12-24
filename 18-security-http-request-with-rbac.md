# Istio数据面配置解析18：使用RBAC对Http请求进行授权



[TOC]



## 概述

本文介绍了在Isito中使用RBAC对Http请求进行授权：

1. 1.0版本针对Http的RBAC授权的配置会被加到Server端inbound listener的envoy.http_connection_manager这个filter的envoy.filters.http.rbac中。
2. 1.0版本只支持针对Http的RBAC授权。
3. 针对source.principal的授权，需要为Client端挂载证书和密钥。



## 相关拓扑

![18-security-http-request-with-rbac-1](./images/18-security-http-request-with-rbac-1.png)

- 在destinationrule中启用针对server端的istio_mutual的配置。
- 为server端配置rbacconfig，开启rbac授权。
- 为server端配置servicerole，为角色授权。
- 为server端配置servicerolebinding，将角色与被授权对象关联。
- 在本文中，将基于source.principal进行授权，也就是针对client端的证书信息进行授权。



![18-security-http-request-with-rbac-2](./images/18-security-http-request-with-rbac-2.png)

- 使用istio rbacconfig定义nginx.default.svc.cluster.local启用rbac授权。
- 使用istio servicerole定义到nginx.default.svc.cluster.local服务的带有version: v1标签的端点的角色权限。
- 使用istio servicerole定义到nginx.default.svc.cluster.local服务的带有version: v2标签的端点的角色权限。
- 使用istio servicerolebinding定义将cluster.local/ns/default/sa/sleep-v1与v1角色绑定。
- 使用istio servicerolebinding定义将cluster.local/ns/default/sa/sleep-v2与v2角色绑定。
- 使用istio destinationrule为到nginx.default.svc.cluster.local服务的请求挂载证书。



## 准备

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sleep-v1
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sleep-v2
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sleep-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sleep
        version: v1
    spec:
      serviceAccountName: sleep-v1
      containers:
      - name: sleep
        image: 192.168.0.61/istio-example/alpine-curl
        command: ["/bin/sleep","7200"]
        imagePullPolicy: IfNotPresent
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sleep-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sleep
        version: v2
    spec:
      serviceAccountName: sleep-v2
      containers:
      - name: sleep
        image: 192.168.0.61/istio-example/alpine-curl
        command: ["/bin/sleep","7200"]
        imagePullPolicy: IfNotPresent
```

- 准备sleep-v1和sleep-v2用于client端。
- 为sleep-v1和sleep-v2准备相应的serviceaccount。



```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  ports:
  - port: 80
    name: http
  selector:
    app: nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx
---
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
      serviceAccountName: nginx
      containers:
      - name: nginx
        image: 192.168.0.61/istio-example/nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/nginx/conf.d/
          readOnly: true
          name: conf
        - mountPath: /etc/nginx/html/
          readOnly: true
          name: index
      volumes:
      - name: conf
        configMap:
          name: nginx-v1
          items:
            - key: default.conf
              path: default.conf
      - name: index
        configMap:
          name: nginx-v1
          items:
            - key: index.html
              path: index.html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-v1
data:
  default.conf: |
    server {
      listen       80;
      server_name  loalhost;

      location / {
        root   /etc/nginx/html/;
        index  index.html index.htm;
      }

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
    <h1>v1!</h1>
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
      serviceAccountName: nginx
      containers:
      - name: nginx
        image: 192.168.0.61/istio-example/nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/nginx/conf.d/
          readOnly: true
          name: conf
        - mountPath: /etc/nginx/html/
          readOnly: true
          name: index
      volumes:
      - name: conf
        configMap:
          name: nginx-v2
          items:
            - key: default.conf
              path: default.conf
      - name: index
        configMap:
          name: nginx-v2
          items:
            - key: index.html
              path: index.html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-v2
data:
  default.conf: |
    server {
      listen       80;
      server_name  loalhost;

      location / {
        root   /etc/nginx/html/;
        index  index.html index.htm;
      }

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
    <h1>v2!</h1>
    <p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>
    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>
    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
```

- 准备nginx-v1和nginx-v2用于server端。
- 为nginx-v1和nginx-v2准备相应的service以及serviceaccount。



## 相关配置

### RbacConfig

```yaml
apiVersion: "rbac.istio.io/v1alpha1"
kind: RbacConfig
metadata:
  name: default
  namespace: istio-system
spec:
  mode: 'ON_WITH_INCLUSION'
  inclusion:
    services: ["nginx.default.svc.cluster.local"]
```

- rbacconfig相关配置。
- 目前istio的rbaconfig为全局配置，每个mesh中只能配置一个。
- rbacconfig的name需要配置为default。
- rbacconfig的namespace需要配置为istio-system。
- rbacconfig的模式配置为ON_WITH_INCLUSION，即只针对inclusion中定义的服务启用rbac。
- 在inclusion中，配置针对nginx.default.svc.cluster.local启用rbac授权。



### ServiceRole和ServiceRoleBinding

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRole
metadata:
  name: sr-v1
  namespace: default
spec:
  rules:
  - services: ["nginx.default.svc.cluster.local"]
    methods: ["GET"]
    constraints:
    - key: "destination.labels[version]"
      values: ["v1"]
---
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRole
metadata:
  name: sr-v2
  namespace: default
spec:
  rules:
  - services: ["nginx.default.svc.cluster.local"]
    methods: ["GET"]
    constraints:
    - key: "destination.labels[version]"
      values: ["v2"]
```

- servicerole相关配置。
- 在servicerole sr-v1中，针对nginx.default.svc.cluster.local的destination.labels[version]包含v1的pod，授予http get的权限。
- 在servicerole sr-v2中，针对nginx.default.svc.cluster.local的destination.labels[version]包含v2的pod，授予http get的权限。



```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRoleBinding
metadata:
  name: srb-v1
  namespace: default
spec:
  subjects:
  - user: "cluster.local/ns/default/sa/sleep-v1"
  roleRef:
    kind: ServiceRole
    name: "sr-v1"
---
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRoleBinding
metadata:
  name: srb-v2
  namespace: default
spec:
  subjects:
  - user: "cluster.local/ns/default/sa/sleep-v2"
  roleRef:
    kind: ServiceRole
    name: "sr-v2"
```

- servicerolebinding相关配置。
- 将user，也就是source.principal，为cluster.local/ns/default/sa/sleep-v1，也就是sleep-v1应用，授予sr-v1的角色，也就是可以以http get的方式，访问nginx-v1。
- 将user，也就是source.principal，为cluster.local/ns/default/sa/sleep-v2，也就是sleep-v2应用，授予sr-v2的角色，也就是可以以http get的方式，访问nginx-v2。



```json
{
        "name": "10.234.1.48_80",
        "address": {
            "socketAddress": {
                "address": "10.234.1.48",
                "portValue": 80
            }
        },
        "filterChains": [
            {
                "filterChainMatch": {
                    "transportProtocol": "raw_buffer"
                },
                "filters": [
                    {
                        "name": "envoy.http_connection_manager",
…
                            "http_filters": [
                                {
                                    "config": {
                                        "policy": {
                                            "peers": [
                                                {
                                                    "mtls": {
                                                        "mode": "PERMISSIVE"
                                                    }
                                                }
                                            ]
                                        }
                                    },
                                    "name": "istio_authn"
                                },
                                {
                                    "config": {
                                        "rules": {
                                            "policies": {
                                                "sr-v1": {
                                                    "permissions": [
                                                        {
                                                            "and_rules": {
                                                                "rules": [
                                                                    {
                                                                        "or_rules": {
                                                                            "rules": [
                                                                                {
                                                                                    "header": {
                                                                                        "exact_match": "GET",
                                                                                        "name": ":method"
…
                                                    "principals": [
                                                        {
                                                            "and_ids": {
                                                                "ids": [
                                                                    {
                                                                        "metadata": {
                                                                            "filter": "istio_authn",
                                                                            "path": [
                                                                                {
                                                                                    "key": "source.principal"
                                                                                }
                                                                            ],
                                                                            "value": {
                                                                                "string_match": {
                                                                                    "exact": "cluster.local/ns/default/sa/sleep-v1"
```

- 在nginx-v1的inbound listener中，会增加相应的filter，针对source.principal为cluster.local/ns/default/sa/sleep-v1的角色，授予get的权限。



```json
{
        "name": "10.234.0.70_80",
        "address": {
            "socketAddress": {
                "address": "10.234.0.70”,
                "portValue": 80
            }
        },
        "filterChains": [
            {
                "filterChainMatch": {
                    "transportProtocol": "raw_buffer"
                },
                "filters": [
                    {
                        "name": "envoy.http_connection_manager",
…
                            "http_filters": [
                                {
                                    "config": {
                                        "policy": {
                                            "peers": [
                                                {
                                                    "mtls": {
                                                        "mode": "PERMISSIVE"
                                                    }
                                                }
                                            ]
                                        }
                                    },
                                    "name": "istio_authn"
                                },
                                {
                                    "config": {
                                        "rules": {
                                            "policies": {
                                                "sr-v1": {
                                                    "permissions": [
                                                        {
                                                            "and_rules": {
                                                                "rules": [
                                                                    {
                                                                        "or_rules": {
                                                                            "rules": [
                                                                                {
                                                                                    "header": {
                                                                                        "exact_match": "GET",
                                                                                        "name": ":method"
…
                                                    "principals": [
                                                        {
                                                            "and_ids": {
                                                                "ids": [
                                                                    {
                                                                        "metadata": {
                                                                            "filter": "istio_authn",
                                                                            "path": [
                                                                                {
                                                                                    "key": "source.principal"
                                                                                }
                                                                            ],
                                                                            "value": {
                                                                                "string_match": {
                                                                                    "exact": "cluster.local/ns/default/sa/sleep-v2”
```

- 在nginx-v2的inbound listener中，会增加相应的filter，针对source.principal为cluster.local/ns/default/sa/sleep-v2的角色，授予get的权限。



### DestinationRule

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr
spec:
  host: nginx.default.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

- destinationrule相关配置。
- 因为是针对source.principal进行授权，所以要在client端挂载相应证书。
- 针对到server端的请求，启用ISTIO_MUTUAL，挂载相应证书和密钥。



```json
{
        "name": "outbound|80||nginx.default.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|80||nginx.default.svc.cluster.local"
        },
        "connectTimeout": "1.000s",
        "circuitBreakers": {
            "thresholds": [
                {}
            ]
        },
        "tlsContext": {
            "commonTlsContext": {
                "tlsCertificates": [
                    {
                        "certificateChain": {
                            "filename": "/etc/certs/cert-chain.pem"
                        },
                        "privateKey": {
                            "filename": "/etc/certs/key.pem"
                        }
                    }
                ],
                "validationContext": {
                    "trustedCa": {
                        "filename": "/etc/certs/root-cert.pem"
                    },
                    "verifySubjectAltName": [
                        "spiffe://cluster.local/ns/default/sa/nginx"
                    ]
                },
                "alpnProtocols": [
                    "istio"
                ]
            },
            "sni": "outbound|80||nginx.default.svc.cluster.local"
        }
    }
```

- 在sleep-v1和sleep-v2的和服务nginx.default.svc.cluster.local相关的outbound cluster中，会挂载相应的证书和密钥。



## 测试结果

```bash
/ # curl http://nginx
RBAC: access denied/ #
/ #
/ # curl http://nginx
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
<h1>v1!</h1>
<p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ #
```

- sleep-v1可以正常访问nginx-v1的服务，但在访问nginx-v2时，会发生RBAC: access denied。



```bash
/ # curl http://nginx
RBAC: access denied/ #
/ #
/ # curl http://nginx
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
<h1>v2!</h1>
<p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ #
```

- sleep-v2可以正常访问nginx-v2的服务，但在访问nginx-v1时，会发生RBAC: access denied。