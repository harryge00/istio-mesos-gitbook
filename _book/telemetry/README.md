Mixer 在服务网格中分为两个组件部署：istio-policy 和 istio-telemetry 。前者负责 check policy，包括限流和黑白名单。后者负责遥测。这样分离可以使两者更好的水平扩展，前者只负责check和quota的api，后者只负责telemetry的API。如果只需要对接prometheus，那么可以只部署 istio-telemetry

## 准备工作
首先需要创建 crd
```
cat <<EOF | kubectl apply -f -
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: checknothings.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: checknothing
    istio: mixer-instance
spec:
  group: config.istio.io
  names:
    kind: checknothing
    plural: checknothings
    singular: checknothing
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: deniers.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: denier
    istio: mixer-adapter
spec:
  group: config.istio.io
  names:
    kind: denier
    plural: deniers
    singular: denier
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: rules.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: istio.io.mixer
    istio: core
spec:
  group: config.istio.io
  names:
    kind: rule
    plural: rules
    singular: rule
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: adapters.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: adapter
    istio: mixer-adapter
spec:
  group: config.istio.io
  names:
    kind: adapter
    plural: adapters
    singular: adapter
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: instances.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: instance
    istio: mixer-instance
spec:
  group: config.istio.io
  names:
    kind: instance
    plural: instances
    singular: instance
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: templates.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: template
    istio: mixer-template
spec:
  group: config.istio.io
  names:
    kind: template
    plural: templates
    singular: template
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: handlers.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: handler
    istio: mixer-handler
spec:
  group: config.istio.io
  names:
    kind: handler
    plural: handlers
    singular: handler
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: attributemanifests.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: istio.io.mixer
    istio: core
spec:
  group: config.istio.io
  names:
    kind: attributemanifest
    plural: attributemanifests
    singular: attributemanifest
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: memquotas.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: memquota
    istio: mixer-adapter
spec:
  group: config.istio.io
  names:
    kind: memquota
    plural: memquotas
    singular: memquota
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---

kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: quotas.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: quota
    istio: mixer-instance
spec:
  group: config.istio.io
  names:
    kind: quota
    plural: quotas
    singular: quota
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: mesosenvs.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: mesosenv
    istio: mixer-adapter
spec:
  group: config.istio.io
  names:
    kind: mesosenv
    plural: mesosenvs
    singular: mesosenv
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: mesoses.config.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: mixer
    package: adapter.template.mesos
    istio: mixer-instance
spec:
  group: config.istio.io
  names:
    kind: mesos
    plural: mesoses
    singular: mesos
    categories:
    - istio-io
    - policy-istio-io
  scope: Namespaced
  version: v1alpha2
EOF
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
                    "shell": "/usr/local/bin/mixs server --address unix:///sock/mixer.socket --configStoreURL=k8s:///mnt/mesos/sandbox/kubeconfig --configDefaultNamespace=default --trace_zipkin_url=http://zipkin.marathon.slave.mesos:31767/api/v1/spans"
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
                "id": "hyge/mixer_debug:mesosenv2",
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
                    "uri": "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/kubeconfig",
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
                    "shell": "/usr/local/bin/mixs server --address unix:///sock/mixer.socket --configStoreURL=k8s:///mnt/mesos/sandbox/kubeconfig --configDefaultNamespace=default --trace_zipkin_url=http://zipkin.marathon.slave.mesos:31767/api/v1/spans"
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
                    "uri": "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/kubeconfig",
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
