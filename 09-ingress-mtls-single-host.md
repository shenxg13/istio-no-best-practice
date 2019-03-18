# Istio数据面配置解析09：使用Ingress Gateway对单主机mTLS请求进行路由



[TOC]



## 概述

本文介绍了在Istio中接收请求的场景：使用Ingress Gateway对单主机mTLS请求进行路由。



## 相关拓扑

![09-ingress-mtls-single-host-1](./images/09-ingress-mtls-single-host-1.png)

- 使用azure aks环境。
- ingress gateway的service类型为loadbalancer。
- ingress gateway的service enternal ip为104.211.54.62。
- 通过该external ip对应的域名，访问ingress gateway svc。



![09-ingress-mtls-single-host-2](./images/09-ingress-mtls-single-host-2.png)

- 使用istio gateway定义ingressgateway中的envoy listener。
- 将名称为httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io的主机与gateway绑定。
- 针对443端口启用mtls并加载mtls相关配置。
- 使用istio virtualservice定义ingressgateway中的envoy route。
- 将virtualservice与之前定义的gateway绑定。
- 将名称为httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io的主机与路由绑定。



## 相关配置

###  Gateway和VirtualService

```bash
openssl req \
-newkey rsa:4096 -nodes -sha256 -keyout ca.key \
-x509 -days 3655 -out ca.crt

openssl req \
-newkey rsa:4096 -nodes -sha256 -keyout httpbin-mtls.key \
-out httpbin-mtls.csr

echo subjectAltName = DNS:httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io > extfile-httpbin-mtls.cnf

openssl x509 \
-req -days 3655 -in httpbin-mtls.csr -CA ca.crt -CAkey ca.key \
-CAcreateserial -extfile extfile-httpbin-mtls.cnf -out httpbin-mtls.crt

openssl req \
-newkey rsa:4096 -nodes -sha256 -keyout client.key \
-out client.csr

echo subjectAltName = DNS:is5.istio.client > client-extfile.cnf

openssl x509 \
-req -days 3655 -in client.csr -CA ca.crt -CAkey ca.key \
-CAcreateserial -extfile client-extfile.cnf -out client.crt

kubectl create -n istio-system secret tls istio-ingressgateway-certs --key ./httpbin-mtls.key --cert ./httpbin-mtls.crt
kubectl create -n istio-system secret generic istio-ingressgateway-ca-certs --from-file ./ca.crt
```

- server端自签名证书相关配置。
- client端自签名证书相关配置。
- k8s secret相关配置。



```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-mtls-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https-httpbin
      protocol: HTTPS
    tls:
      mode: MUTUAL
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
      privateKey: /etc/istio/ingressgateway-certs/tls.key
      caCertificates: /etc/istio/ingressgateway-ca-certs/ca.crt
      subjectAltNames:
      - is5.istio.client
    hosts:
    - "httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io"
```

- gateway相关配置。
- 新建监听端口443。
- 在443中启用mtls。
- 指定443的key和cert。
- 指定443的ca cert。
- 指定允许连接443的san。



```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-tls-vs
spec:
  hosts:
  - "httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io"
  gateways:
  - httpbin-mtls-gateway
  http:
  - match:
    - uri:
        prefix: /status
    route:
    - destination:
        port:
          number: 8000
        host: httpbin.default.svc.cluster.local
```

- virtualservice相关配置。
- 配置相关路由。



```json
{
        "name": "outbound|8000||httpbin.default.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|8000||httpbin.default.svc.cluster.local"
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
- ingressgateway中会生成httpbin相关cluster。



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
                        "httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io"
                    ]
                },
                "tlsContext": {
                    "commonTlsContext": {
                        "tlsCertificates": [
                            {
                                "certificateChain": {
                                    "filename": "/etc/istio/ingressgateway-certs/tls.crt"
                                },
                                "privateKey": {
                                    "filename": "/etc/istio/ingressgateway-certs/tls.key"
                                }
                            }
                        ],
                        "validationContext": {
                            "trustedCa": {
                                "filename": "/etc/istio/ingressgateway-ca-certs/ca.crt"
                            },
                            "verifySubjectAltName": [
                                "is5.istio.client"
                            ]
                        },
                        "alpnProtocols": [
                            "h2",
                            "http/1.1"
                        ]
                    },
                    "requireClientCertificate": true
                },
…
                            "rds": {
                                "config_source": {
                                    "ads": {}
                                },
                                "route_config_name": "https.443.https-httpbin"
                            },
```

- 443端口的envoy listener相关配置。
- 在gateway和virtualservice定义完成后，envoy会生成443端口的监听，相关路由为https.443.https-httpbin。
- **在listener中会加载mtls相关证书和密钥。**
- **mtls流量在listener中被卸载。**



```json
"name": "https.443.https-httpbin",
        "virtualHosts": [
            {
                "name": "httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io:443",
                "domains": [
                    "httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io",
                    "httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io:443"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/status"
                        },
                        "route": {
                            "cluster": "outbound|8000||httpbin.default.svc.cluster.local",
                            "timeout": "0.000s",
                            "maxGrpcTimeout": "0.000s"
                        },
```

- envoy route相关配置。
- 到httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io的443端口的相关http请求，会被转发至outbound|8000||httpbin.default.svc.cluster.local。



## 测试结果

```bash
[~/K8s/istio/istio-azure-1.0.2/samples/httpbin/ssl]$ http https://httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io/status/418 --verify no --cert ./client.crt --cert-key ./client.key
HTTP/1.1 418 Unknown
access-control-allow-credentials: true
access-control-allow-origin: *
content-length: 135
date: Sun, 04 Nov 2018 15:28:47 GMT
server: envoy
x-envoy-upstream-service-time: 6
x-more-info: http://tools.ietf.org/html/rfc2324

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`

[~/K8s/istio/istio-azure-1.0.2/samples/httpbin/ssl]
```

- https测试结果。
- 通过https mtls方式访问httpbin.6491dea3ce6b4d17b109.eastus.aksapp.io，可以正常访问httpbin pod。