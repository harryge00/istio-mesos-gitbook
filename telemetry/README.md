Mixer 在服务网格中分为两个组件部署：istio-policy 和 istio-telemetry 。前者负责 check policy，包括限流和黑白名单。后者负责遥测。这样分离可以使两者更好的水平扩展，前者只负责check和quota的api，后者只负责telemetry的API。如果只需要对接prometheus，那么可以只部署 istio-telemetry

## 准备工作
首先, 为了使 istio-telemetry 使用 mesosenv 和 prometheus 这两个adapter，需要创建crd：
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/install/kubernetes/helm/istio-init/files/crd-10.yaml
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


## 使用 mesosenv adapter

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
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: mesosattrgenrulerule
  namespace: default
spec:
  actions:
  - handler: handler.mesosenv
    instances:
    - attributes.mesos
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: tcpmesosattrgenrulerule
  namespace: default
spec:
  match: context.protocol == "tcp"
  actions:
  - handler: handler.mesosenv
    instances:
    - attributes.mesos
---
apiVersion: "config.istio.io/v1alpha2"
kind: mesos
metadata:
  name: attributes
  namespace: default
spec:
  # Pass the required attribute data to the adapter
  source_uid: source.uid | ""
  source_ip: source.ip | ip("0.0.0.0") # default to unspecified ip addr
  destination_uid: destination.uid | ""
  destination_port: destination.port | 0
  attribute_bindings:
    # Fill the new attributes from the adapter produced output.
    # $out refers to an instance of OutputTemplate message
    source.ip: $out.source_pod_ip | ip("0.0.0.0")
    source.uid: $out.source_pod_uid | "unknown"
    source.labels: $out.source_labels | emptyStringMap()
    source.name: $out.source_pod_name | "unknown"
    source.namespace: $out.source_namespace | "default"
    source.owner: $out.source_owner | "unknown"
    source.serviceAccount: $out.source_service_account_name | "unknown"
    source.workload.uid: $out.source_workload_uid | "unknown"
    source.workload.name: $out.source_workload_name | "unknown"
    source.workload.namespace: $out.source_workload_namespace | "unknown"
    destination.ip: $out.destination_pod_ip | ip("0.0.0.0")
    destination.uid: $out.destination_pod_uid | "unknown"
    destination.labels: $out.destination_labels | emptyStringMap()
    destination.name: $out.destination_pod_name | "unknown"
    destination.container.name: $out.destination_container_name | "unknown"
    destination.namespace: $out.destination_namespace | "default"
    destination.owner: $out.destination_owner | "unknown"
    destination.serviceAccount: $out.destination_service_account_name | "unknown"
    destination.workload.uid: $out.destination_workload_uid | "unknown"
    destination.workload.name: $out.destination_workload_name | "unknown"
    destination.workload.namespace: $out.destination_workload_namespace | "unknown"
---
apiVersion: "config.istio.io/v1alpha2"
kind: attributemanifest
metadata:
  name: kubernetes
  namespace: default
spec:
  attributes:
    source.ip:
      valueType: IP_ADDRESS
    source.labels:
      valueType: STRING_MAP
    source.metadata:
      valueType: STRING_MAP
    source.name:
      valueType: STRING
    source.namespace:
      valueType: STRING
    source.owner:
      valueType: STRING
    source.service:  # DEPRECATED
      valueType: STRING
    source.serviceAccount:
      valueType: STRING
    source.services:
      valueType: STRING
    source.workload.uid:
      valueType: STRING
    source.workload.name:
      valueType: STRING
    source.workload.namespace:
      valueType: STRING
    destination.ip:
      valueType: IP_ADDRESS
    destination.labels:
      valueType: STRING_MAP
    destination.metadata:
      valueType: STRING_MAP
    destination.owner:
      valueType: STRING
    destination.name:
      valueType: STRING
    destination.container.name:
      valueType: STRING
    destination.namespace:
      valueType: STRING
    destination.service: # DEPRECATED
      valueType: STRING
    destination.service.uid:
      valueType: STRING
    destination.service.name:
      valueType: STRING
    destination.service.namespace:
      valueType: STRING
    destination.service.host:
      valueType: STRING
    destination.serviceAccount:
      valueType: STRING
    destination.workload.uid:
      valueType: STRING
    destination.workload.name:
      valueType: STRING
    destination.workload.namespace:
      valueType: STRING
---
apiVersion: "config.istio.io/v1alpha2"
kind: stdio
metadata:
  name: handler
  namespace: default
spec:
  outputAsJson: true
---
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: requestcount
  namespace: default
spec:
  value: "1"
  dimensions:
    reporter: conditional((context.reporter.kind | "inbound") == "outbound", "source", "destination")
    source_workload: source.workload.name | "unknown"
    source_workload_namespace: source.workload.namespace | "unknown"
    source_principal: source.principal | "unknown"
    source_app: source.labels["istio"] | "unknown"
    source_version: source.labels["version"] | "unknown"
    destination_workload: destination.workload.name | "unknown"
    destination_workload_namespace: destination.workload.namespace | "unknown"
    destination_principal: destination.principal | "unknown"
    destination_app: destination.labels["istio"] | "unknown"
    destination_version: destination.labels["version"] | "unknown"
    destination_service: destination.service.host | "unknown"
    destination_service_name: destination.service.name | "unknown"
    destination_service_namespace: destination.service.namespace | "unknown"
    request_protocol: api.protocol | context.protocol | "unknown"
    response_code: response.code | 200
    connection_security_policy: conditional((context.reporter.kind | "inbound") == "outbound", "unknown", conditional(connection.mtls | false, "mutual_tls", "none"))
  monitored_resource_type: '"UNSPECIFIED"'
---
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: requestduration
  namespace: default
spec:
  value: response.duration | "0ms"
  dimensions:
    reporter: conditional((context.reporter.kind | "inbound") == "outbound", "source", "destination")
    source_workload: source.workload.name | "unknown"
    source_workload_namespace: source.workload.namespace | "unknown"
    source_principal: source.principal | "unknown"
    source_app: source.labels["istio"] | "unknown"
    source_version: source.labels["version"] | "unknown"
    destination_workload: destination.workload.name | "unknown"
    destination_workload_namespace: destination.workload.namespace | "unknown"
    destination_principal: destination.principal | "unknown"
    destination_app: destination.labels["istio"] | "unknown"
    destination_version: destination.labels["version"] | "unknown"
    destination_service: destination.service.host | "unknown"
    destination_service_name: destination.service.name | "unknown"
    destination_service_namespace: destination.service.namespace | "unknown"
    request_protocol: api.protocol | context.protocol | "unknown"
    response_code: response.code | 200
    connection_security_policy: conditional((context.reporter.kind | "inbound") == "outbound", "unknown", conditional(connection.mtls | false, "mutual_tls", "none"))
  monitored_resource_type: '"UNSPECIFIED"'
---
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: requestsize
  namespace: default
spec:
  value: request.size | 0
  dimensions:
    reporter: conditional((context.reporter.kind | "inbound") == "outbound", "source", "destination")
    source_workload: source.workload.name | "unknown"
    source_workload_namespace: source.workload.namespace | "unknown"
    source_principal: source.principal | "unknown"
    source_app: source.labels["istio"] | "unknown"
    source_version: source.labels["version"] | "unknown"
    destination_workload: destination.workload.name | "unknown"
    destination_workload_namespace: destination.workload.namespace | "unknown"
    destination_principal: destination.principal | "unknown"
    destination_app: destination.labels["istio"] | "unknown"
    destination_version: destination.labels["version"] | "unknown"
    destination_service: destination.service.host | "unknown"
    destination_service_name: destination.service.name | "unknown"
    destination_service_namespace: destination.service.namespace | "unknown"
    request_protocol: api.protocol | context.protocol | "unknown"
    response_code: response.code | 200
    connection_security_policy: conditional((context.reporter.kind | "inbound") == "outbound", "unknown", conditional(connection.mtls | false, "mutual_tls", "none"))
  monitored_resource_type: '"UNSPECIFIED"'
---
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: responsesize
  namespace: default
spec:
  value: response.size | 0
  dimensions:
    reporter: conditional((context.reporter.kind | "inbound") == "outbound", "source", "destination")
    source_workload: source.workload.name | "unknown"
    source_workload_namespace: source.workload.namespace | "unknown"
    source_principal: source.principal | "unknown"
    source_app: source.labels["istio"] | "unknown"
    source_version: source.labels["version"] | "unknown"
    destination_workload: destination.workload.name | "unknown"
    destination_workload_namespace: destination.workload.namespace | "unknown"
    destination_principal: destination.principal | "unknown"
    destination_app: destination.labels["istio"] | "unknown"
    destination_version: destination.labels["version"] | "unknown"
    destination_service: destination.service.host | "unknown"
    destination_service_name: destination.service.name | "unknown"
    destination_service_namespace: destination.service.namespace | "unknown"
    request_protocol: api.protocol | context.protocol | "unknown"
    response_code: response.code | 200
    connection_security_policy: conditional((context.reporter.kind | "inbound") == "outbound", "unknown", conditional(connection.mtls | false, "mutual_tls", "none"))
  monitored_resource_type: '"UNSPECIFIED"'
---
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: tcpbytesent
  namespace: default
spec:
  value: connection.sent.bytes | 0
  dimensions:
    reporter: conditional((context.reporter.kind | "inbound") == "outbound", "source", "destination")
    source_workload: source.workload.name | "unknown"
    source_workload_namespace: source.workload.namespace | "unknown"
    source_principal: source.principal | "unknown"
    source_app: source.labels["istio"] | "unknown"
    source_version: source.labels["version"] | "unknown"
    destination_workload: destination.workload.name | "unknown"
    destination_workload_namespace: destination.workload.namespace | "unknown"
    destination_principal: destination.principal | "unknown"
    destination_app: destination.labels["istio"] | "unknown"
    destination_version: destination.labels["version"] | "unknown"
    destination_service: destination.service.name | "unknown"
    destination_service_name: destination.service.name | "unknown"
    destination_service_namespace: destination.service.namespace | "unknown"
    connection_security_policy: conditional((context.reporter.kind | "inbound") == "outbound", "unknown", conditional(connection.mtls | false, "mutual_tls", "none"))
  monitored_resource_type: '"UNSPECIFIED"'
---
apiVersion: "config.istio.io/v1alpha2"
kind: metric
metadata:
  name: tcpbytereceived
  namespace: default
spec:
  value: connection.received.bytes | 0
  dimensions:
    reporter: conditional((context.reporter.kind | "inbound") == "outbound", "source", "destination")
    source_workload: source.workload.name | "unknown"
    source_workload_namespace: source.workload.namespace | "unknown"
    source_principal: source.principal | "unknown"
    source_app: source.labels["istio"] | "unknown"
    source_version: source.labels["version"] | "unknown"
    destination_workload: destination.workload.name | "unknown"
    destination_workload_namespace: destination.workload.namespace | "unknown"
    destination_principal: destination.principal | "unknown"
    destination_app: destination.labels["istio"] | "unknown"
    destination_version: destination.labels["version"] | "unknown"
    destination_service: destination.service.name | "unknown"
    destination_service_name: destination.service.name | "unknown"
    destination_service_namespace: destination.service.namespace | "unknown"
    connection_security_policy: conditional((context.reporter.kind | "inbound") == "outbound", "unknown", conditional(connection.mtls | false, "mutual_tls", "none"))
  monitored_resource_type: '"UNSPECIFIED"'
---
apiVersion: "config.istio.io/v1alpha2"
kind: prometheus
metadata:
  name: handler
  namespace: default
spec:
  metrics:
  - name: requests_total
    instance_name: requestcount.metric.default
    kind: COUNTER
    label_names:
    - reporter
    - source_app
    - source_principal
    - source_workload
    - source_workload_namespace
    - source_version
    - destination_app
    - destination_principal
    - destination_workload
    - destination_workload_namespace
    - destination_version
    - destination_service
    - destination_service_name
    - destination_service_namespace
    - request_protocol
    - response_code
    - connection_security_policy
  - name: request_duration_seconds
    instance_name: requestduration.metric.default
    kind: DISTRIBUTION
    label_names:
    - reporter
    - source_app
    - source_principal
    - source_workload
    - source_workload_namespace
    - source_version
    - destination_app
    - destination_principal
    - destination_workload
    - destination_workload_namespace
    - destination_version
    - destination_service
    - destination_service_name
    - destination_service_namespace
    - request_protocol
    - response_code
    - connection_security_policy
    buckets:
      explicit_buckets:
        bounds: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
  - name: request_bytes
    instance_name: requestsize.metric.default
    kind: DISTRIBUTION
    label_names:
    - reporter
    - source_app
    - source_principal
    - source_workload
    - source_workload_namespace
    - source_version
    - destination_app
    - destination_principal
    - destination_workload
    - destination_workload_namespace
    - destination_version
    - destination_service
    - destination_service_name
    - destination_service_namespace
    - request_protocol
    - response_code
    - connection_security_policy
    buckets:
      exponentialBuckets:
        numFiniteBuckets: 8
        scale: 1
        growthFactor: 10
  - name: response_bytes
    instance_name: responsesize.metric.default
    kind: DISTRIBUTION
    label_names:
    - reporter
    - source_app
    - source_principal
    - source_workload
    - source_workload_namespace
    - source_version
    - destination_app
    - destination_principal
    - destination_workload
    - destination_workload_namespace
    - destination_version
    - destination_service
    - destination_service_name
    - destination_service_namespace
    - request_protocol
    - response_code
    - connection_security_policy
    buckets:
      exponentialBuckets:
        numFiniteBuckets: 8
        scale: 1
        growthFactor: 10
  - name: tcp_sent_bytes_total
    instance_name: tcpbytesent.metric.default
    kind: COUNTER
    label_names:
    - reporter
    - source_app
    - source_principal
    - source_workload
    - source_workload_namespace
    - source_version
    - destination_app
    - destination_principal
    - destination_workload
    - destination_workload_namespace
    - destination_version
    - destination_service
    - destination_service_name
    - destination_service_namespace
    - connection_security_policy
  - name: tcp_received_bytes_total
    instance_name: tcpbytereceived.metric.default
    kind: COUNTER
    label_names:
    - reporter
    - source_app
    - source_principal
    - source_workload
    - source_workload_namespace
    - source_version
    - destination_app
    - destination_principal
    - destination_workload
    - destination_workload_namespace
    - destination_version
    - destination_service
    - destination_service_name
    - destination_service_namespace
    - connection_security_policy
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: promhttp
  namespace: default
spec:
  match: context.protocol == "http" || context.protocol == "grpc"
  actions:
  - handler: handler.prometheus
    instances:
    - requestcount.metric
    - requestduration.metric
    - requestsize.metric
    - responsesize.metric
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: promtcp
  namespace: default
spec:
  match: context.protocol == "tcp"
  actions:
  - handler: handler.prometheus
    instances:
    - tcpbytesent.metric
    - tcpbytereceived.metric
EOF
```
