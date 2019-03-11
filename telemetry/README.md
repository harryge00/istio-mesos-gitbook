Mixer 在服务网格中分为两个组件部署：istio-policy 和 istio-telemetry 。前者负责 check policy，包括限流和黑白名单。后者负责遥测。这样分离可以使两者更好的水平扩展，前者只负责check和quota的api，后者只负责telemetry的API。如果只需要对接prometheus，那么可以只部署 istio-telemetry

## 准备工作

* 首先, 为了使 istio-telemetry 使用 mesosenv 和 prometheus 这两个adapter，需要创建crd:

```
kubectl apply -f https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/install/crd-mixer.yaml
kubectl apply -f https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/install/crd-mixer-instances.yaml

```

* 另外，需要 pilot 增加--meshConfig参数，指定mesh配置文件。所以将pilot的json改为:
```
{
  "id": "/istio/pilot",
  "cmd": "/usr/local/bin/pilot-discovery discovery --httpAddr :15007 --registries Mesos --domain marathon.slave.mesos --mesosVIPDomain marathon.slave.mesos --mesosMaster http://master.mesos:8080 --kubeconfig /mnt/mesos/sandbox/kubeconfig --meshConfig /mnt/mesos/sandbox/mesh",
  "cpus": 1,
  "mem": 1024,
  "disk": 0,
  "instances": 1,
  "acceptedResourceRoles": [],
  "container": {
    "type": "DOCKER",
    "docker": {
      "forcePullImage": false,
      "image": "hyge/pilot:mesos",
      "parameters": [],
      "privileged": false
    },
    "volumes": [],
    "portMappings": [
      {
        "containerPort": 9093,
        "hostPort": 31993,
        "labels": {},
        "name": "debug",
        "protocol": "tcp",
        "servicePort": 10001
      },
      {
        "containerPort": 15010,
        "hostPort": 31510,
        "labels": {},
        "name": "grpc",
        "protocol": "tcp",
        "servicePort": 10004
      },
      {
        "containerPort": 15007,
        "hostPort": 31507,
        "labels": {},
        "name": "http",
        "protocol": "tcp",
        "servicePort": 10012
      }
    ]
  },
  "dependencies": [
    "/istio/apiserver"
  ],
  "healthChecks": [
    {
      "gracePeriodSeconds": 300,
      "ignoreHttp1xx": false,
      "intervalSeconds": 60,
      "maxConsecutiveFailures": 3,
      "path": "/ready",
      "portIndex": 0,
      "protocol": "HTTP",
      "timeoutSeconds": 20,
      "delaySeconds": 15
    }
  ],
  "networks": [
    {
      "mode": "container/bridge"
    }
  ],
  "portDefinitions": [],
  "fetch": [
    {
      "uri": "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/install/kubeconfig",
      "extract": false,
      "executable": false,
      "cache": false
    },
    {
      "uri": "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/mesh",
      "extract": false,
      "executable": false,
      "cache": false
    }
  ]
}
```
其中，https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/mesh 即为所需的mesh配置:
```
# Set the following variable to true to disable policy checks by the Mixer.
# Note that metrics will still be reported to the Mixer.
disablePolicyChecks: false
# Deprecated: mixer is using EDS
mixerCheckServer: istio-policy.marathon.slave.mesos:9091
mixerReportServer: istio-telemetry.marathon.slave.mesos:9091
```
其中指定了`mixerCheckServer` 和 `mixerReportServer`的地址。如果不需要`policy`，则只需要指定后者：
```
# Set the following variable to true to disable policy checks by the Mixer.
# Note that metrics will still be reported to the Mixer.
disablePolicyChecks: false
# Deprecated: mixer is using EDS
mixerReportServer: istio-telemetry.marathon.slave.mesos:9091
```



## 部署istio-telemetry/policy和prometheus
* istio-policy
```istio-policy.json
{
    "id": "/istio-policy",
    "labels": {
        "istio-mixer-type": "policy",
        "istio": "istio-policy"
    },
    "environment": {
        "SERVICES_DOMAIN": "marathon.slave.mesos",
        "GODEBUG": "gctrace=2",
        "KUBECONFIG": "/mnt/mesos/sandbox/kubeconfig",
        "POD_NAME": "istio-policy",
        "POD_NAMESPACE": "default"
    },
    "containers": [
        {
            "name": "mixer",
            "exec": {
                "command": {
                    "shell": "/usr/local/bin/mixs server --address unix:///sock/mixer.socket --configStoreURL=k8s:///mnt/mesos/sandbox/kubeconfig --configDefaultNamespace=default --trace_zipkin_url=http://zipkin.istio.marathon.slave.mesos:31767/api/v1/spans"
                }
            },
            "resources": {
                "cpus": 0.5,
                "mem": 1024,
                "disk": 0,
                "gpus": 0
            },
            "endpoints": [
                {
                    "name": "grpc-mixer",
                    "containerPort": 9091,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "grpc-mixer-mtls",
                    "containerPort": 15004,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "http-monitoring",
                    "containerPort": 9093,
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                }
            ],
            "image": {
                "kind": "DOCKER",
                "id": "hyge/mixer_debug:mesosenv",
                "forcePull": true
            },
            "volumeMounts": [
                {
                    "name": "uds-socket",
                    "mountPath": "/sock"
                }
            ],
            "artifacts": [
                {
                    "uri": "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/install/kubeconfig",
                    "extract": true,
                    "executable": false,
                    "cache": false
                }
            ]
        },
        {
            "name": "proxy",
            "exec": {
                "command": {
                    "shell": "/usr/local/bin/pilot-agent proxy --discoveryAddress pilot.istio.marathon.slave.mesos:31510 --serviceCluster istio-policy --domain marathon.slave.mesos --serviceregistry Mesos --templateFile /etc/istio/proxy/envoy_policy_mesos.yaml.tmpl --controlPlaneAuthPolicy NONE"
                }
            },
            "resources": {
                "cpus": 0.5,
                "mem": 1024,
                "disk": 0,
                "gpus": 0
            },
            "image": {
                "kind": "DOCKER",
                "id": "hyge/proxy_debug:mesos2",
                "forcePull": true
            },
            "volumeMounts": [
                {
                    "name": "uds-socket",
                    "mountPath": "/sock"
                }
            ]
        }
    ],
    "volumes": [
        {
            "name": "uds-socket"
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

* istio-telemetry
```
{
    "id": "/istio-telemetry",
    "labels": {
        "istio-mixer-type": "telemetry",
        "istio": "istio-telemetry"
    },
    "environment": {
        "POD_NAMESPACE": "default",
        "GODEBUG": "gctrace=2",
        "SERVICES_DOMAIN": "marathon.slave.mesos",
        "POD_NAME": "istio-telemetry",
        "KUBECONFIG": "/mnt/mesos/sandbox/kubeconfig"
    },
    "containers": [
        {
            "name": "mixer",
            "exec": {
                "command": {
                    "shell": "/usr/local/bin/mixs server --address unix:///sock/mixer.socket --configStoreURL=k8s:///mnt/mesos/sandbox/kubeconfig --configDefaultNamespace=default --trace_zipkin_url=http://zipkin.istio.marathon.slave.mesos:31767/api/v1/spans --log_output_level=adapters:debug,api:debug"
                }
            },
            "resources": {
                "cpus": 0.5,
                "mem": 1024,
                "disk": 0,
                "gpus": 0
            },
            "endpoints": [
                {
                    "name": "grpc-mixer",
                    "containerPort": 9091,
                    "hostPort": 31131,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "grpc-mixer-mtls",
                    "containerPort": 15004,
                    "hostPort": 31504,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "http-monitoring",
                    "containerPort": 9093,
                    "hostPort": 31093,
                    "protocol": [
                        "tcp"
                    ]
                },
                {
                    "name": "prometheus",
                    "containerPort": 42422,
                    "hostPort": 31422,
                    "protocol": [
                        "tcp"
                    ]
                }
            ],
            "image": {
                "kind": "DOCKER",
                "id": "hyge/mixer_debug:mesosenv",
                "forcePull": true
            },
            "volumeMounts": [
                {
                    "name": "uds-socket",
                    "mountPath": "/sock"
                }
            ],
            "artifacts": [
                {
                    "uri": "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/install/kubeconfig",
                    "extract": true,
                    "executable": false,
                    "cache": false
                }
            ]
        },
        {
            "name": "proxy",
            "exec": {
                "command": {
                    "shell": "/usr/local/bin/pilot-agent proxy --discoveryAddress pilot.istio.marathon.slave.mesos:31510 --serviceCluster istio-telemetry --domain marathon.slave.mesos --serviceregistry Mesos --templateFile /etc/istio/proxy/envoy_telemetry_mesos.yaml.tmpl --controlPlaneAuthPolicy NONE"
                }
            },
            "resources": {
                "cpus": 0.5,
                "mem": 1024,
                "disk": 0,
                "gpus": 0
            },
            "image": {
                "kind": "DOCKER",
                "id": "hyge/proxy_debug:mixer",
                "forcePull": true
            },
            "volumeMounts": [
                {
                    "name": "uds-socket",
                    "mountPath": "/sock"
                }
            ]
        }
    ],
    "volumes": [
        {
            "name": "uds-socket"
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

* prometheus
```
{
  "id": "/prometheus",
  "cmd": "/bin/prometheus --storage.tsdb.retention=6h  --config.file=/mnt/mesos/sandbox/prometheus-mesos.yaml",
  "cpus": 0.4,
  "mem": 1024,
  "disk": 0,
  "instances": 1,
  "portDefinitions": [],
  "acceptedResourceRoles": [
    "*"
  ],
  "container": {
    "type": "DOCKER",
    "docker": {
      "forcePullImage": false,
      "image": "prom/prometheus:v2.3.1",
      "parameters": [],
      "privileged": false
    },
    "volumes": [],
    "portMappings": [
      {
        "containerPort": 9090,
        "hostPort": 31909,
        "labels": {},
        "name": "prometheus",
        "protocol": "tcp",
        "servicePort": 10003
      }
    ]
  },
  "fetch": [
    {
      "uri": "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/install/prometheus-mesos.yaml",
      "extract": false,
      "executable": false,
      "cache": false
    }
  ],
  "labels": {
    "app": "prometheus"
  },
  "networks": [
    {
      "mode": "container/bridge"
    }
  ]
}
```


## 使用 mesosenv adapter
为了使用mesosenv来从marathon获取attributes，需要先配置adapter：
```
cat << EOF| kubectl apply -f -
apiVersion: "config.istio.io/v1alpha2"
kind: mesosenv
metadata:
  name: handler
  namespace: default
spec:
  marathon_address: master.mesos:8080
  http_basic_auth_user: zmcc
  http_basic_auth_password: Zmcc@001
EOF
```
`marathon_address` 是marathon地址，http_basic_auth 是用户名和密码，免密则可以跳过。

## Prometheus 查询
刷新bookinfo，服务会自动向istio-telemetry汇报请求的响应时间，请求源、目的地址，请求的大小、http头部等信息。然后即可在prometheus中查询，打开prometheus 宿主机IP:31909。
常见查询sql如下：
```
# 目的地址为 v2 reviews 的请求总数
istio_requests_total{destination_service="reviews.marathon.slave.mesos", destination_version="v2"}
# 目的地址为 productpage v1 的请求总数
istio_requests_total{destination_service="productpage.marathon.slave.mesos", destination_version="v1"}
# 目的地址为 productpage 的平均每5分钟请求速率
rate(istio_requests_total{destination_service=~"productpage.*", response_code="200"}[5m])
```