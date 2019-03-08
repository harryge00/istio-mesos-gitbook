# mixer
Mixer 在服务网格中分为两个组件部署：`istio-policy` 和 `istio-telemetry` 。前者负责 check policy，包括限流和黑白名单。后者负责遥测。这样分离可以使两者更好的水平扩展，前者只负责check和quota的api，后者只负责telemetry的API。如果只需要对接prometheus，那么可以只部署 `istio-telemetry`

## istio-policy
如果不需要使用限流和policy check的功能，可以跳过此步骤。
istio-policy 的 json 如下：
```
{
    "id": "/istio-policy",
    "labels": {
        "istio-mixer-type": "policy",
        "istio": "istio-policy"
    },
    "environment": {
        "SERVICES_DOMAIN": "marathon.slave.mesos",
        "GODEBUG": "gctrace=2",
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
                    "uri": "https://gist.githubusercontent.com/harryge00/09fed16607a25c47952047d8d7983fe2/raw/3d856e99a9a215c0b5cb0daa4c819745cd35c327/kubeconfig",
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
                    "shell": "/usr/local/bin/pilot-agent proxy --discoveryAddress master.mesos:15010 --serviceCluster istio-policy --domain marathon.slave.mesos --serviceregistry Mesos --templateFile /etc/istio/proxy/envoy_policy_mesos.yaml.tmpl --controlPlaneAuthPolicy NONE"
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

### istio-policy验证之黑白名单

在验证功能前，需要确保bookinfo已经部署完成，并先创建 CRD(CustomResourceDefinition)。
```
kubectl apply -f https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/install/crd-mixer-policy.yaml
kubectl apply -f https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/yamls/virtualservice-details-reviews-v2.yaml
```
之后可以创建白名单，允许用户名为alice 和 bob的用户能访问`ratings`这个服务。

```
cat << EOF|kubectl apply -f -
apiVersion: config.istio.io/v1alpha2
kind: listchecker
metadata:
  name: whitelist
spec:
  # providerUrl: ordinarily black and white lists are maintained
  # externally and fetched asynchronously using the providerUrl.
  overrides: ["alice", "bob"]  # overrides provide a static list
  blacklist: false
---
apiVersion: config.istio.io/v1alpha2
kind: listentry
metadata:
  name: appversion
spec:
  value: request.headers["end-user"]
---
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: checkversion
spec:
  match: destination.service == "ratings.marathon.slave.mesos"
  actions:
  - handler: whitelist.listchecker
    instances:
    - appversion.listentry
EOF
```
之后，刷新productpage 页面，可以看到星星不显示了。但是登录为`alice` 或者 `bob`，星星就又能显示。

## istio-telemetry
istio-telemetry的json如下：
```
{
    "id": "/istio-telemetry",
    "labels": {
        "istio-mixer-type": "telemetry",
        "istio": "istio-telemetry"
    },
    "environment": {
        "SERVICES_DOMAIN": "marathon.slave.mesos",
        "GODEBUG": "gctrace=2",
        "POD_NAME": "istio-telemetry",
        "POD_NAMESPACE": "default"
    },
    "containers": [
        {
            "name": "mixer",
            "exec": {
                "command": {
                    "shell": "/usr/local/bin/mixs server --address unix:///sock/mixer.socket --configStoreURL=fs:/mnt/mesos/sandbox/kubeconfig --configDefaultNamespace=default --trace_zipkin_url=http://zipkin.marathon.slave.mesos:31767/api/v1/spans"
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
                    "hostPort": 0,
                    "protocol": [
                        "tcp"
                    ]
                }
            ],
            "image": {
                "kind": "DOCKER",
                "id": "istio/mixer_debug:mesosenv"
            },
            "volumeMounts": [
                {
                    "name": "uds-socket",
                    "mountPath": "/sock"
                }
            ],
            "artifacts": [
                {
                    "uri": "https://gist.githubusercontent.com/harryge00/09fed16607a25c47952047d8d7983fe2/raw/3d856e99a9a215c0b5cb0daa4c819745cd35c327/kubeconfig",
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
                    "shell": "/usr/local/bin/pilot-agent proxy --serviceCluster istio-telemetry --domain marathon.slave.mesos --serviceregistry Mesos --templateFile /etc/istio/proxy/envoy_telemetry_mesos.yaml.tmpl --controlPlaneAuthPolicy NONE"
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
        "placement": {
          "constraints": [
            {
              "fieldName": "hostname",
              "operator": "LIKE",
              "value": "172.31.17.24"
            }
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

prometheus 的json如下：注意需要通过URI加载它的配置文件prometheus-mesos.yaml
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
在istio-telemetry部署后，需要部署CRD来告知istio-telemetry，使用那些adapter(mesosenv)，生成哪些指标：
```
kubectl apply -f https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/yamls/mesosenv-default.yaml
```
之后即可在prometheus中查询如下指标：
```
