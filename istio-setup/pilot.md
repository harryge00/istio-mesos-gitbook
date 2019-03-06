# 安装 Istio 服务注册发现组件
以 group 形式部署`etcd, apiserver, pilot, jaeger`。 json如下：

```
{
    "id": "/istio",
    "apps": [
        {
            "id": "/istio/etcd",
            "acceptedResourceRoles": [
                "*"
            ],
            "backoffFactor": 1.15,
            "backoffSeconds": 1,
            "cmd": "/usr/local/bin/etcd -advertise-client-urls=http://0.0.0.0:2379 -listen-client-urls=http://0.0.0.0:2379",
            "container": {
                "type": "DOCKER",
                "docker": {
                    "forcePullImage": false,
                    "image": "quay.io/coreos/etcd:latest",
                    "parameters": [],
                    "privileged": false
                },
                "volumes": [],
                "portMappings": [
                    {
                        "containerPort": 2379,
                        "hostPort": 31379,
                        "labels": {},
                        "name": "etcd",
                        "protocol": "tcp",
                        "servicePort": 10000
                    }
                ]
            },
            "cpus": 0.5,
            "disk": 0,
            "env": {
                "SERVICE_IGNORE": "1"
            },
            "executor": "",
            "healthChecks": [
                {
                    "gracePeriodSeconds": 300,
                    "ignoreHttp1xx": false,
                    "intervalSeconds": 60,
                    "maxConsecutiveFailures": 3,
                    "path": "/health",
                    "portIndex": 0,
                    "protocol": "HTTP",
                    "timeoutSeconds": 20,
                    "delaySeconds": 15
                }
            ],
            "instances": 1,
            "labels": {
                "consul": ""
            },
            "maxLaunchDelaySeconds": 3600,
            "mem": 512,
            "gpus": 0,
            "networks": [
                {
                    "mode": "container/bridge"
                }
            ],
            "requirePorts": false,
            "upgradeStrategy": {
                "maximumOverCapacity": 1,
                "minimumHealthCapacity": 1
            },
            "version": "2019-03-05T16:24:26.341Z",
            "versionInfo": {
                "lastScalingAt": "2019-03-05T16:24:26.341Z",
                "lastConfigChangeAt": "2019-03-05T16:24:26.341Z"
            },
            "killSelection": "YOUNGEST_FIRST",
            "unreachableStrategy": {
                "inactiveAfterSeconds": 0,
                "expungeAfterSeconds": 0
            }
        },
        {
            "id": "/istio/zipkin",
            "cmd": null,
            "cpus": 0.5,
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
                "image": "jaegertracing/all-in-one:1.5",
                "parameters": [],
                "privileged": false
              },
              "volumes": [],
              "portMappings": [
                {
                  "containerPort": 9411,
                  "hostPort": 31767,
                  "labels": {},
                  "name": "zipkin",
                  "protocol": "tcp",
                  "servicePort": 10002
                },
                {
                  "containerPort": 16686,
                  "hostPort": 0,
                  "labels": {},
                  "name": "query-http",
                  "protocol": "tcp",
                  "servicePort": 10005
                },
                {
                  "containerPort": 14267,
                  "hostPort": 0,
                  "labels": {},
                  "name": "jaeger-collector-tchannel",
                  "protocol": "tcp",
                  "servicePort": 10006
                },
                {
                  "containerPort": 14268,
                  "hostPort": 0,
                  "labels": {},
                  "name": "jaeger-collector-http",
                  "protocol": "tcp",
                  "servicePort": 10007
                },
                {
                  "containerPort": 5775,
                  "hostPort": 0,
                  "labels": {},
                  "name": "agent-zipkin-thrift",
                  "protocol": "udp",
                  "servicePort": 10008
                },
                {
                  "containerPort": 6831,
                  "hostPort": 0,
                  "labels": {},
                  "name": "agent-compact",
                  "protocol": "tcp",
                  "servicePort": 10009
                },
                {
                  "containerPort": 6832,
                  "hostPort": 0,
                  "labels": {},
                  "name": "agent-binary",
                  "protocol": "tcp",
                  "servicePort": 10010
                }
              ]
            },
            "env": {
              "COLLECTOR_ZIPKIN_HTTP_PORT": "9411",
              "MEMORY_MAX_TRACES": "50000"
            },
            "healthChecks": [
              {
                "gracePeriodSeconds": 300,
                "ignoreHttp1xx": false,
                "intervalSeconds": 60,
                "maxConsecutiveFailures": 3,
                "path": "/",
                "portIndex": 1,
                "protocol": "HTTP",
                "timeoutSeconds": 20,
                "delaySeconds": 15
              }
            ],
            "networks": [
              {
                "mode": "container/bridge"
              }
            ]
        },
        {
            "id": "/istio/apiserver",
            "dependencies": ["/istio/etcd"],
            "backoffFactor": 1.15,
            "backoffSeconds": 1,
            "cmd": "kube-apiserver --etcd-servers http://etcd.istio.marathon.slave.mesos:31379 --service-cluster-ip-range 10.99.0.0/16 --insecure-port 8080 -v 2 --insecure-bind-address 0.0.0.0",
            "container": {
                "type": "DOCKER",
                "docker": {
                    "forcePullImage": false,
                    "image": "gcr.io/google_containers/kube-apiserver:v1.10.12",
                    "parameters": [],
                    "privileged": false
                },
                "volumes": [],
                "portMappings": [
                    {
                        "containerPort": 8080,
                        "hostPort": 31080,
                        "labels": {
                            "VIP_0": "/apiserver:10080"
                        },
                        "protocol": "tcp",
                        "servicePort": 10000
                    }
                ]
            },
            "cpus": 0.3,
            "disk": 0,
            "executor": "",
            "healthChecks": [
                {
                    "gracePeriodSeconds": 300,
                    "ignoreHttp1xx": false,
                    "intervalSeconds": 60,
                    "maxConsecutiveFailures": 3,
                    "path": "/healthz",
                    "portIndex": 0,
                    "protocol": "HTTP",
                    "timeoutSeconds": 20,
                    "delaySeconds": 15
                }
            ],
            "instances": 1,
            "labels": {
                "HAPROXY_GROUP": "external",
                "HAPROXY_0_VHOST": ""
            },
            "maxLaunchDelaySeconds": 3600,
            "mem": 640,
            "gpus": 0,
            "networks": [
                {
                    "mode": "container/bridge"
                }
            ],
            "requirePorts": false,
            "upgradeStrategy": {
                "maximumOverCapacity": 1,
                "minimumHealthCapacity": 1
            },
            "version": "2019-03-05T16:24:11.024Z",
            "versionInfo": {
                "lastScalingAt": "2019-03-05T16:24:11.024Z",
                "lastConfigChangeAt": "2019-03-05T16:24:11.024Z"
            },
            "killSelection": "YOUNGEST_FIRST",
            "unreachableStrategy": {
                "inactiveAfterSeconds": 0,
                "expungeAfterSeconds": 0
            }
        },
        {
          "id": "/istio/pilot",
          "dependencies": ["/istio/apiserver"],
          "backoffFactor": 1.15,
          "backoffSeconds": 1,
          "cmd": "/usr/local/bin/pilot-discovery discovery --httpAddr :15007 --registries Mesos --domain marathon.slave.mesos --mesosVIPDomain marathon.slave.mesos --mesosMaster http://master.mesos:8080 --kubeconfig /mnt/mesos/sandbox/kubeconfig",
          "container": {
            "portMappings": [
              {
                "containerPort": 9093,
                "hostPort": 31993,
                "protocol": "tcp",
                "name": "debug"
              },
              {
                "containerPort": 15010,
                "hostPort": 31510,
                "protocol": "tcp",
                "name": "grpc"
              },
              {
                "containerPort": 15007,
                "hostPort": 31507,
                "protocol": "tcp",
                "name": "http"
              }
            ],
            "type": "DOCKER",
            "volumes": [],
            "docker": {
              "image": "hyge/pilot:mesos",
              "forcePullImage": false,
              "parameters": []
            }
          },
          "cpus": 1,
          "fetch": [
            {
              "uri": "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/install/kubeconfig",
              "extract": false,
              "executable": false,
              "cache": false
            }
          ],
          "instances": 1,
          "maxLaunchDelaySeconds": 3600,
          "mem": 1024,
          "networks": [
            {
              "mode": "container/bridge"
            }
          ],
          "requirePorts": false,
          "upgradeStrategy": {
            "maximumOverCapacity": 1,
            "minimumHealthCapacity": 1
          },
          "killSelection": "YOUNGEST_FIRST",
          "unreachableStrategy": {
            "inactiveAfterSeconds": 0,
            "expungeAfterSeconds": 0
          },
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
          "constraints": []
        }
    ]
}
```
可以将json保存为 `groups.json`, 然后执行
```
curl -d "@groups.json" -X POST http://localhost:8080/v2/groups
```

## 验证 pilot 状态
如果 health 为绿色，则部署成功。可以验证pilot状态：
```
# curl pilot.istio.marathon.slave.mesos:31993/debug/push_status
{
    "ProxyStatus": {},
    "Start": "2019-03-05T17:40:16.861168088Z",
    "End": "0001-01-01T00:00:00Z"
}
```
表示 pilot 最新的push时间在 `2019-03-05T17:40:16.861168088Z`
