# Ingress Gateway
在一个典型的网格中，通常有一个或多个用于终结外部 TLS 链接，将流量引入网格的负载均衡器（我们称之为 gateway）。 #然后流量通过边车网关（sidecar gateway）流经内部服务。下图描绘了网关在网格中的使用情况
![gateways](https://istio.io/blog/2018/v1alpha3-routing/gateways.svg)

其中 Gateway 是一个独立于平台的抽象，用于对流入专用中间设备的流量进行建模。下图描述了跨多个配置资源的控制流程。
![virtualservices-destrules](https://istio.io/blog/2018/v1alpha3-routing/virtualservices-destrules.svg)

## 前提条件
需要先部署好 [bookinfo](../bookinfo/README.md)

## Ingress 部署
Ingressgateway 同样以pod的方式部署：
```
{
    "id": "/istio-ingressgateway",
    "labels": {
        "istio": "ingressgateway"
    },
    "environment": {
        "SERVICES_DOMAIN": "marathon.slave.mesos"
    },
    "containers": [
        {
            "name": "istio-ingressgateway",
            "exec": {
                "command": {
                    "shell": "/usr/local/bin/pilot-agent proxy  router  -v  \"2\"  --discoveryRefreshDelay  '1s'   --drainDuration  '45s'  --parentShutdownDuration  '1m0s'  --connectTimeout  '10s' --serviceregistry Mesos --serviceCluster  istio-ingressgateway  --zipkinAddress  zipkin.istio.marathon.slave.mesos:31767 --proxyAdminPort  \"15000\"  --controlPlaneAuthPolicy  NONE  --discoveryAddress  pilot.istio.marathon.slave.mesos:31510"
                }
            },
            "resources": {
                "cpus": 0.2,
                "mem": 512,
                "disk": 0,
                "gpus": 0
            },
            "endpoints": [
                {
                    "name": "http2",
                    "containerPort": 80,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "https",
                    "containerPort": 443,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "tcp",
                    "containerPort": 31400,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "tcp-pilot-grpc-tls",
                    "containerPort": 15011,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "tcp-citadel-grpc-tls",
                    "containerPort": 8060,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "tcp-dns-tls",
                    "containerPort": 853,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "http2-prometheus",
                    "containerPort": 15030,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "http2-grafana",
                    "containerPort": 15031,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                }
            ],
            "image": {
                "kind": "DOCKER",
                "id": "hyge/proxy_debug:1.0.5"
            }
        }
    ],
    "networks": [
        {
            "name": "mesos-bridge",
            "mode": "container"
        }
    ],
    "scaling": {
        "kind": "fixed",
        "instances": 1
    },
    "scheduling": {
        "backoff": {
            "backoff": 1,
            "backoffFactor": 1.15,
            "maxLaunchDelay": 3600
        },
        "upgrade": {
            "minimumHealthCapacity": 1,
            "maximumOverCapacity": 1
        },
        "killSelection": "YOUNGEST_FIRST",
        "unreachableStrategy": {
            "inactiveAfterSeconds": 0,
            "expungeAfterSeconds": 0
        }
    },
    "executorResources": {
        "cpus": 0.1,
        "mem": 32,
        "disk": 10
    }
}
```

ingressgateway和bookinfo启动后，我们可以配置80端口的gateway:

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: productpage-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "productpage.example.com"
EOF
```

给gateway配置路由
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage-gateway
spec:
  hosts:
  - "productpage.example.com"
  gateways:
  - productpage-gateway
  http:
  - match:
    - uri:
        prefix: /
    - uri:
        prefix: /productpage
    route:
    - destination:
        port:
          number: 9080
        host: productpage.marathon.slave.mesos
EOF
```

## Ingress 访问

首先需要获取ingress pod的 hostIP 以及 host port：
```
# curl localhost:8080/v2/pods/istio-ingressgateway::status
 {
    "id": "/istio-ingressgateway",
    "spec": {
        "id": "/istio-ingressgateway",
        "labels": {
            "istio": "ingressgateway"
        },
        ...
    "status": "STABLE",
    "statusSince": "2019-01-25T02:25:08.892Z",
    "instances": [
        {
            "id": "istio-ingressgateway.instance-6cbf6f2d-2048-11e9-a4ed-560a19cd1873",
            ...
            "agentHostname": "172.31.26.199",  # Host IP
            "agentId": "539c3f6d-bd72-4024-96fb-183d71ec1f27-S576",
           ...
            "networks": [
                {
                    "name": "mesos-bridge",
                    "addresses": [
                        "172.31.254.33"
                    ]
                }
            ],
            "containers": [
                {
                    "name": "istio-ingressgateway",
                    "status": "TASK_RUNNING",
                    "statusSince": "2019-01-25T02:25:08.892Z",
                    "conditions": [],
                    "containerId": "istio-ingressgateway.instance-6cbf6f2d-2048-11e9-a4ed-560a19cd1873.istio-ingressgateway",
                    "endpoints": [
                        {
                            "name": "http2",
                            "allocatedHostPort": 31705 # Host Port
                        },
                        {
                            "name": "https",
                            "allocatedHostPort": 31706
                        },
                        {
                            "name": "tcp",
                            "allocatedHostPort": 31707
                        },
                        {
                            "name": "tcp-pilot-grpc-tls",
                            "allocatedHostPort": 31708
                        },
                        {
                            "name": "tcp-citadel-grpc-tls",
                            "allocatedHostPort": 31709
                        },
                        {
                            "name": "tcp-dns-tls",
                            "allocatedHostPort": 31710
                        },
                        {
                            "name": "http2-prometheus",
                            "allocatedHostPort": 31711
                        },
                        {
                            "name": "http2-grafana",
                            "allocatedHostPort": 31712
                        }
                    ],
                   ...
}
```
`http2` 的 `allocatedHostPort` 是 31705, `host ip(agentHostname)`为172.31.26.199。
则我们可以通过 `curl  -HHost:productpage.example.com  172.31.26.199:31705` 来访问productpage，如果成功获取html，则说明ingress正常

## 浏览器访问Ingress
curl 访问ingress时，通过了在header中加入 `Host:productpage.example.com` 来指定需要ingress导流的后端服务，但是通过浏览器访问后端服务时，指定header是不现实的。所以考虑使用通配符`*`来替换gateway和virtualservice中的`hosts`：
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: productpage-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage-gateway
spec:
  hosts:
  - "*"
  gateways:
  - productpage-gateway
  http:
  - match:
    - uri:
        prefix: /
    - uri:
        prefix: /productpage
    route:
    - destination:
        port:
          number: 9080
        host: productpage.marathon.slave.mesos
EOF
```
之后即可通过 `172.31.26.199:31705` 在浏览器中访问productpage
