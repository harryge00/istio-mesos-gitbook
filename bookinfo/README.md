## 部署Pod应用
Istio 部署成功后，即可部署应用。对于需要加入服务网格的服务，需要添加1个 pilot-agent 容器做流量劫持。`details`的pod如下:
```
{
  "environment": {
    "SERVICES_DOMAIN": "marathon.slave.mesos",
    "INBOUND_PORTS": "9080",
    "SERVICE_NAME": "details-v1",
    "ZIPKIN_ADDRESS": "zipkin.istio.marathon.slave.mesos:31767",
    "DISCOVERY_ADDRESSS": "pilot.istio.marathon.slave.mesos:31510"
  },
  "labels": {
    "istio": "details",
    "version": "v1"
  },
  "id": "/details",
  "containers": [
    {
      "name": "details",
      "resources": {
        "cpus": 0.1,
        "mem": 512,
        "disk": 0,
        "gpus": 0
      },
      "endpoints": [
        {
          "name": "http-9080",
          "containerPort": 9080,
          "hostPort": 0,
          "protocol": [
            "tcp"
          ]
        }
      ],
      "image": {
        "kind": "DOCKER",
        "id": "istio/examples-bookinfo-details-v1:1.8.0"
      }
    },
    {
      "name": "proxy",
      "resources": {
        "cpus": 0.2,
        "mem": 512,
        "disk": 0,
        "gpus": 0
      },
      "image": {
        "kind": "DOCKER",
        "forcePullImage": true,
        "id": "hyge/proxy_debug:mesos2"
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
    "instances": 1,
    "kind": "fixed"
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
    },
    "placement": {
      "constraints": [

      ]
    }
  },
  "executorResources": {
    "cpus": 0.1,
    "mem": 32,
    "disk": 10
  },
  "volumes": [],
  "fetch": []
}
```
其中`proxy`的环境变量需要配置:

|      环境变量      | 说明                                 |
|:------------------:|:------------------------------------:|
| SERVICES_DOMAIN    |域名后缀                                  |
| INBOUND_PORTS      |指定容器流入流量的端口，可以用逗号隔开，如`8080，6443`|
| SERVICE_NAME       |服务名                                     |
| ZIPKIN_ADDRESS     |zipkin地址                                      |
| DISCOVERY_ADDRESSS |pilot grpc地址                                      |

其余的pod json 如下：

* ratings
```
{
    "id": "/ratings",
    "labels": {
        "istio": "ratings",
        "version": "v1"
    },
    "environment": {
        "SERVICES_DOMAIN": "marathon.slave.mesos",
        "INBOUND_PORTS": "9080",
        "SERVICE_NAME": "ratings-v1",
        "ZIPKIN_ADDRESS": "zipkin.istio.marathon.slave.mesos:31767",
        "DISCOVERY_ADDRESSS": "pilot.istio.marathon.slave.mesos:31510"
    },
    "containers": [
        {
            "name": "ratings",
            "resources": {
                "cpus": 0.1,
                "mem": 512,
                "disk": 0,
                "gpus": 0
            },
            "endpoints": [
                {
                    "name": "http-9080",
                    "containerPort": 9080,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                }
            ],
            "image": {
                "kind": "DOCKER",
                "id": "istio/examples-bookinfo-ratings-v1:1.8.0"
            }
        },
        {
            "name": "proxy",
            "resources": {
                "cpus": 0.2,
                "mem": 512,
                "disk": 0,
                "gpus": 0
            },
            "image": {
                "kind": "DOCKER",
                "id": "hyge/proxy_debug:mesos2"
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
        "placement": {
          "constraints": [
          ]
        },
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
* reviews
```
{
  "environment": {
    "SERVICES_DOMAIN": "marathon.slave.mesos",
    "INBOUND_PORTS": "9080",
    "SERVICE_NAME": "reviews-v1",
    "ZIPKIN_ADDRESS": "zipkin.istio.marathon.slave.mesos:31767",
    "DISCOVERY_ADDRESSS": "pilot.istio.marathon.slave.mesos:31510"
  },
  "labels": {
    "istio": "reviews",
    "version": "v1"
  },
  "id": "/reviews",
  "containers": [
    {
      "name": "reviews",
      "resources": {
        "cpus": 0.2,
        "mem": 1024,
        "disk": 0,
        "gpus": 0
      },
      "endpoints": [
        {
          "name": "http-9080",
          "containerPort": 9080,
          "hostPort": 0,
          "protocol": [
            "tcp"
          ]
        }
      ],
      "image": {
        "kind": "DOCKER",
        "id": "istio/examples-bookinfo-reviews-v1:1.8.0"
      }
    },
    {
      "name": "proxy",
      "resources": {
        "cpus": 0.2,
        "mem": 512,
        "disk": 0,
        "gpus": 0
      },
      "image": {
        "kind": "DOCKER",
        "id": "hyge/proxy_debug:mesos2"
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
    "instances": 1,
    "kind": "fixed"
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
    },
    "placement": {
      "constraints": [

      ]
    }
  },
  "executorResources": {
    "cpus": 0.1,
    "mem": 32,
    "disk": 10
  },
  "volumes": [],
  "fetch": []
}
```

* reviews-v2
```
{
  "environment": {
    "SERVICES_DOMAIN": "marathon.slave.mesos",
    "INBOUND_PORTS": "9080",
    "SERVICE_NAME": "reviews-v2",
    "ZIPKIN_ADDRESS": "zipkin.istio.marathon.slave.mesos:31767",
    "DISCOVERY_ADDRESSS": "pilot.istio.marathon.slave.mesos:31510"
  },
  "labels": {
    "istio": "reviews",
    "version": "v2"
  },
  "id": "/reviews-v2",
  "containers": [
    {
      "name": "reviews",
      "resources": {
        "cpus": 0.2,
        "mem": 1024,
        "disk": 0,
        "gpus": 0
      },
      "endpoints": [
        {
          "name": "http-9080",
          "containerPort": 9080,
          "hostPort": 0,
          "protocol": [
            "tcp"
          ]
        }
      ],
      "image": {
        "kind": "DOCKER",
        "id": "istio/examples-bookinfo-reviews-v2:1.8.0"
      }
    },
    {
      "name": "proxy",
      "resources": {
        "cpus": 0.2,
        "mem": 512,
        "disk": 0,
        "gpus": 0
      },
      "image": {
        "kind": "DOCKER",
        "id": "hyge/proxy_debug:mesos2"
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
    "instances": 1,
    "kind": "fixed"
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
    },
    "placement": {
      "constraints": [

      ]
    }
  },
  "executorResources": {
    "cpus": 0.1,
    "mem": 32,
    "disk": 10
  },
  "volumes": [],
  "fetch": []
}
```

* reviews-v3
```
{
  "environment": {
    "SERVICES_DOMAIN": "marathon.slave.mesos",
    "INBOUND_PORTS": "9080",
    "SERVICE_NAME": "reviews-v3",
    "ZIPKIN_ADDRESS": "zipkin.istio.marathon.slave.mesos:31767",
    "DISCOVERY_ADDRESSS": "pilot.istio.marathon.slave.mesos:31510"
  },
  "labels": {
    "istio": "reviews",
    "version": "v3"
  },
  "id": "/reviews-v3",
  "containers": [
    {
      "name": "reviews",
      "resources": {
        "cpus": 0.2,
        "mem": 1024,
        "disk": 0,
        "gpus": 0
      },
      "endpoints": [
        {
          "name": "http-9080",
          "containerPort": 9080,
          "hostPort": 0,
          "protocol": [
            "tcp"
          ]
        }
      ],
      "image": {
        "kind": "DOCKER",
        "id": "istio/examples-bookinfo-reviews-v3:1.8.0"
      }
    },
    {
      "name": "proxy",
      "resources": {
        "cpus": 0.2,
        "mem": 512,
        "disk": 0,
        "gpus": 0
      },
      "image": {
        "kind": "DOCKER",
        "id": "hyge/proxy_debug:mesos2"
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
    "instances": 1,
    "kind": "fixed"
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
    },
    "placement": {
      "constraints": [

      ]
    }
  },
  "executorResources": {
    "cpus": 0.1,
    "mem": 32,
    "disk": 10
  },
  "volumes": [],
  "fetch": []
}
```

* productpage
```
{
  "environment": {
    "SERVICES_DOMAIN": "marathon.slave.mesos",
    "INBOUND_PORTS": "9080",
    "SERVICE_NAME": "productpage-v1",
    "ZIPKIN_ADDRESS": "zipkin.istio.marathon.slave.mesos:31767",
    "DISCOVERY_ADDRESSS": "pilot.istio.marathon.slave.mesos:31510"
  },
  "labels": {
    "istio": "productpage",
    "version": "v1"
  },
  "id": "/productpage",
  "containers": [
    {
      "name": "productpage",
      "resources": {
        "cpus": 0.2,
        "mem": 1024,
        "disk": 0,
        "gpus": 0
      },
      "endpoints": [
        {
          "name": "http-9080",
          "containerPort": 9080,
          "hostPort": 31180,
          "protocol": [
            "tcp"
          ]
        }
      ],
      "image": {
        "kind": "DOCKER",
        "id": "istio/examples-bookinfo-productpage-v1:1.8.0"
      }
    },
    {
      "name": "proxy",
      "resources": {
        "cpus": 0.5,
        "mem": 1024,
        "disk": 0,
        "gpus": 0
      },
      "image": {
        "kind": "DOCKER",
        "forcePullImage": true,
        "id": "hyge/proxy_debug:mesos2"
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
    "instances": 1,
    "kind": "fixed"
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
    },
    "placement": {
      "constraints": [

      ]
    }
  },
  "executorResources": {
    "cpus": 0.1,
    "mem": 32,
    "disk": 10
  },
  "volumes": [],
  "fetch": []
}
```

将 json 分别保存后，即可部署测试
```
curl -d "@details.json" -X POST http://localhost:8080/v2/pods
curl -d "@ratings.json" -X POST http://localhost:8080/v2/pods
curl -d "@reviews.json" -X POST http://localhost:8080/v2/pods
curl -d "@reviews-v2.json" -X POST http://localhost:8080/v2/pods
curl -d "@reviews-v3.json" -X POST http://localhost:8080/v2/pods
curl -d "@productpage.json" -X POST http://localhost:8080/v2/pods
```

然后，就可以获取productpage的hostport，然后在浏览器访问啦。
```
curl -X GET \
  http://localhost:8080/v2/pods/productpage::status  |grep 'allocatedHostPort\|agentHostname'
```
其中，`allocatedHostPort` 是端口，`agentHostname`是宿主机IP。
刷新可以看到总共3种reviews：红星、黑星、没有星星。架构图如下：
![test](https://istio.io/docs/examples/bookinfo/noistio.svg)

## 生成destinationrules
在做版本控制前，需要先通过destinationrules定义版本，
```
cat << EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage.marathon.slave.mesos
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews.marathon.slave.mesos
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings.marathon.slave.mesos
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details.marathon.slave.mesos
  subsets:
  - name: v1
    labels:
      version: v1
EOF
```
subsets中定义了每个服务的版本
