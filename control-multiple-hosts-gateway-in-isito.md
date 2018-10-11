在Istio中使用多主机Gateway

***

#### 非TLS多主机环境

![gateway-multiple-hosts-in-istio](./images/gateway-multiple-hosts-in-istio.png)

- 使用azure aks环境。
- 2个域名，分别为：httpbin-a.7cb9a9b7b318440399a0.eastus.aksapp.io和httpbin-b.7cb9a9b7b318440399a0.eastus.aksapp.io。
- 为2个域名配置统一的gateway定义。
- 为2个域名分别配置virtualservice定义。
- 域名httpbin-a被路由至pod httpbin-a的/status uri。
- 域名httpbin-b被路由至pod httpbin-b的/headers uri。
- 在gateway的listnener中生成统一的监听0.0.0.0_80。
- 在gateway的route中分别生成针对httpbin-a和httpbin-b的虚拟主机。

<br/>

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-dual-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http-httpbin
      protocol: HTTP
    hosts:
    - "httpbin-a.7cb9a9b7b318440399a0.eastus.aksapp.io"
    - "httpbin-b.7cb9a9b7b318440399a0.eastus.aksapp.io"

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-dual-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http-httpbina
      protocol: HTTP
    hosts:
    - "httpbin-a.7cb9a9b7b318440399a0.eastus.aksapp.io"
  - port:
      number: 80
      name: http-httpbinb
      protocol: HTTP
    hosts:
    - "httpbin-b.7cb9a9b7b318440399a0.eastus.aksapp.io"
```

- gateway相关配置。
- 这2个gateway的配置，生成的envoy配置是一致的。

<br/>

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-a-vs
spec:
  hosts:
  - "httpbin-a.7cb9a9b7b318440399a0.eastus.aksapp.io"
  gateways:
  - httpbin-dual-gateway
  http:
  - match:
    - uri:
        prefix: /status
    route:
    - destination:
        port:
          number: 8000
        host: httpbin-a

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-b-vs
spec:
  hosts:
  - "httpbin-b.7cb9a9b7b318440399a0.eastus.aksapp.io"
  gateways:
  - httpbin-dual-gateway
  http:
  - match:
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin-b
```

- httpbin-a和httpbin-b的virtualservice配置。

<br/>

```
[~/K8s/istio/istio-azure-1.0.2/samples/httpbin]$ http http://httpbin-a.7cb9a9b7b318440399a0.eastus.aksapp.io:80/status/418
HTTP/1.1 418 Unknown
access-control-allow-credentials: true
access-control-allow-origin: *
content-length: 135
date: Thu, 11 Oct 2018 16:05:06 GMT
server: envoy
x-envoy-upstream-service-time: 3
x-more-info: http://tools.ietf.org/html/rfc2324

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`

[~/K8s/istio/istio-azure-1.0.2/samples/httpbin]$ http http://httpbin-b.7cb9a9b7b318440399a0.eastus.aksapp.io:80/headers
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-length: 1012
content-type: application/json
date: Thu, 11 Oct 2018 16:05:24 GMT
server: envoy
x-envoy-upstream-service-time: 6

{
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Content-Length": "0",
        "Host": "httpbin-b.7cb9a9b7b318440399a0.eastus.aksapp.io",
        "User-Agent": "HTTPie/0.9.9",
        "X-B3-Sampled": "1",
        "X-B3-Spanid": "4195f6aea35738e1",
        "X-B3-Traceid": "4195f6aea35738e1",
        "X-Envoy-Decorator-Operation": "httpbin-b.default.svc.cluster.local:8000/headers*",
        "X-Envoy-Internal": "true",
        "X-Istio-Attributes": "CioKHWRlc3RpbmF0aW9uLnNlcnZpY2UubmFtZXNwYWNlEgkSB2RlZmF1bHQKJwoYZGVzdGluYXRpb24uc2VydmljZS5uYW1lEgsSCWh0dHBiaW4tYgo8ChNkZXN0aW5hdGlvbi5zZXJ2aWNlEiUSI2h0dHBiaW4tYi5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsCk8KCnNvdXJjZS51aWQSQRI/a3ViZXJuZXRlczovL2lzdGlvLWluZ3Jlc3NnYXRld2F5LTdmZmI1OTc3NmItYm4yc3YuaXN0aW8tc3lzdGVtCj8KF2Rlc3RpbmF0aW9uLnNlcnZpY2UudWlkEiQSImlzdGlvOi8vZGVmYXVsdC9zZXJ2aWNlcy9odHRwYmluLWIKQQoYZGVzdGluYXRpb24uc2VydmljZS5ob3N0EiUSI2h0dHBiaW4tYi5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2Fs",
        "X-Request-Id": "63466f3e-23ae-9bf2-9c37-0dc4236551c9"
    }
}

[~/K8s/istio/istio-azure-1.0.2/samples/httpbin]$
```

- 测试结果。