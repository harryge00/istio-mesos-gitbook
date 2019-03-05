# 安装Istio 组件
分别部署`etcd, apiserver, zipkin`，以应用的形式

## etcd
```
{
  "id": "/etcd",
  "cmd": "/usr/local/bin/etcd -advertise-client-urls=http://0.0.0.0:2379 -listen-client-urls=http://0.0.0.0:2379",
  "cpus": 0.5,
  "mem": 512,
  "disk": 0,
  "instances": 1,
  "acceptedResourceRoles": [
    "*"
  ],
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
  "env": {
    "SERVICE_IGNORE": "1"
  },
  "networks": [
    {
      "mode": "container/bridge"
    }
  ],
  "healthChecks": [
    {
      "protocol": "HTTP",
      "path": "/health",
      "portIndex": 0,
      "gracePeriodSeconds": 300,
      "intervalSeconds": 60,
      "timeoutSeconds": 20,
      "maxConsecutiveFailures": 3
    }
  ],
  "labels": {
    "consul": ""
  }
}
```

## apiserver
```
{
  "id": "/apiserver",
  "instances": 1,
  "container": {
    "type": "DOCKER",
    "volumes": [],
    "docker": {
      "image": "gcr.io/google_containers/kube-apiserver:v1.10.12"
    },
    "portMappings": [
      {
        "containerPort": 8080,
        "hostPort": 31080,
        "labels": {
          "VIP_0": "/apiserver:10080"
        },
        "protocol": "tcp"
      }
    ]
  },
  "cpus": 0.3,
  "mem": 640,
  "requirePorts": false,
  "networks": [
    {
      "mode": "container/bridge"
    }
  ],
  "healthChecks": [
    {
      "protocol": "HTTP",
      "path": "/healthz",
      "portIndex": 0,
      "gracePeriodSeconds": 300,
      "intervalSeconds": 60,
      "timeoutSeconds": 20,
      "maxConsecutiveFailures": 3
    }
  ],
  "fetch": [],
  "constraints": [],
  "cmd": "kube-apiserver --etcd-servers http://etcd.marathon.slave.mesos:31379 --service-cluster-ip-range 10.99.0.0/16 --insecure-port 8080 -v 2 --insecure-bind-address 0.0.0.0",
  "labels": {
    "HAPROXY_GROUP": "external",
    "HAPROXY_0_VHOST": ""
  }
}
```

## Jaeger
```
{
  "id": "/zipkin",
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
}
```

## Pilot
```
{
  "id": "/pilot",
  "backoffFactor": 1.15,
  "backoffSeconds": 1,
  "cmd": "/usr/local/bin/pilot-discovery discovery --httpAddr :15007 --registries Mesos --domain marathon.slave.mesos --mesosVIPDomain marathon.slave.mesos --mesosMaster http://master.mesos:8080 --kubeconfig /mnt/mesos/sandbox/kubeconfig --marathonUser zmcc --marathonPassword Zmcc@001",
  "container": {
    "portMappings": [
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
      },
      {
        "containerPort": 9093,
        "hostPort": 31993,
        "protocol": "tcp",
        "name": "debug"
      }
    ],
    "type": "DOCKER",
    "volumes": [],
    "docker": {
      "image": "hyge/pilot:eds",
      "forcePullImage": false,
      "parameters": []
    }
  },
  "cpus": 1,
  "fetch": [
    {
      "uri": "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/kubeconfig",
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
  "healthChecks": [],
  "constraints": []
}
```
