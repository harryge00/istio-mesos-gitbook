# Citadel
json:
```
{
  "id": "/citadel",
  "cmd": "/usr/local/bin/istio_ca --citadel-storage-namespace=istio-system --grpc-port=8060 --grpc-host-identities=istio-standalone-citadel --self-signed-ca=true --self-signed-ca-org=zmcc-org --self-signed-ca-cert-ttl=24000h --sign-ca-certs=true --workload-cert-ttl=21h --kube-config=/mnt/mesos/sandbox/kubeconfig",
  "cpus": 1,
  "mem": 1024,
  "disk": 0,
  "instances": 1,
  "acceptedResourceRoles": [
    "*"
  ],
  "container": {
    "type": "DOCKER",
    "docker": {
      "forcePullImage": false,
      "image": "michaelbi/citadel-test:1.0.5",
      "parameters": [],
      "privileged": false
    },
    "volumes": [],
    "portMappings": [
      {
        "containerPort": 8060,
        "hostPort": 0,
        "labels": {},
        "name": "grpc",
        "protocol": "tcp",
        "servicePort": 10104
      }
    ]
  },
  "networks": [
    {
      "mode": "container/bridge"
    }
  ],
  "uris": [
    "https://raw.githubusercontent.com/harryge00/mylinuxrc/master/dcos/istio/install/kubeconfig"
  ]
}
```
## 验证7层 mTLS
citadel 可以使服务之间通过7层的TLS加密通信。
* 为了验证，我们可以先部署两个服务`httpbin` 和 `sleep`：
```
{
  "environment": {
    "SERVICES_DOMAIN": "marathon.slave.mesos",
    "INBOUND_PORTS": "8000",
    "SERVICE_NAME": "httpbin",
    "ZIPKIN_ADDRESS": "zipkin.istio.marathon.slave.mesos:31767",
    "DISCOVERY_ADDRESSS": "pilot.istio.marathon.slave.mesos:31510"
  },
  "labels": {
    "istio": "httpbin"
  },
  "id": "/httpbin",
  "containers": [
    {
      "name": "httpbin",
      "resources": {
        "cpus": 0.5,
        "mem": 256,
        "disk": 0,
        "gpus": 0
      },
      "exec": {
        "command": {
          "shell": "/usr/bin/python3 /usr/local/bin/gunicorn --bind=0.0.0.0:8000 httpbin:app"
        }
      },
      "endpoints": [
        {
          "name": "http",
          "containerPort": 8000,
          "hostPort": 0,
          "protocol": [
            "tcp"
          ],
          "labels": {
            "VIP_0": "/httpbin:8000"
          }
        }
      ],
      "image": {
        "kind": "DOCKER",
        "id": "citizenstig/httpbin"
      }
    },
    {
      "name": "proxy",
      "resources": {
        "cpus": 0.5,
        "mem": 512,
        "disk": 0,
        "gpus": 0
      },
      "exec": {
        "command": {
          "shell": "mkdir /etc/certs; cp ./certs/*.pem /etc/certs; chown -R istio-proxy:istio-proxy /etc/certs/*.pem; /usr/local/bin/start.sh"
        }
      },
      "image": {
        "kind": "DOCKER",
        "id": "hyge/proxy_debug:mesos2"
      },
      "artifacts": [
        {
          "uri": "https://s3-ap-southeast-1.amazonaws.com/marathon-cmd/bundle-certs.tgz",
          "extract": true,
          "executable": false,
          "cache": false
        }
      ]
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
      "constraints": []
    }
  }
} 
```

```
{
  "environment": {
    "SERVICES_DOMAIN": "marathon.slave.mesos",
    "INBOUND_PORTS": "80",
    "SERVICE_NAME": "sleep",
    "ZIPKIN_ADDRESS": "zipkin.istio.marathon.slave.mesos:31767",
    "DISCOVERY_ADDRESSS": "pilot.istio.marathon.slave.mesos:31510"
  },
  "labels": {
    "istio": "sleep"
  },
  "id": "/sleep",
  "containers": [
    {
      "name": "sleep",
      "resources": {
        "cpus": 0.2,
        "mem": 128,
        "disk": 0,
        "gpus": 0
      },
      "exec": {
        "command": {
          "shell": "sleep 3650d"
        }
      },
      "endpoints": [
        {
          "name": "http",
          "containerPort": 80,
          "hostPort": 0,
          "protocol": [
            "tcp"
          ],
          "labels": {
            "VIP_0": "/sleep:80"
          }
        }
      ],
      "image": {
        "kind": "DOCKER",
        "id": "pstauffer/curl"
      }
    },
    {
      "name": "proxy",
      "resources": {
        "cpus": 0.5,
        "mem": 512,
        "disk": 0,
        "gpus": 0
      },
      "exec": {
        "command": {
          "shell": "mkdir /etc/certs; cp ./certs/*.pem /etc/certs; chown -R istio-proxy:istio-proxy /etc/certs/*.pem; /usr/local/bin/start.sh"
        }
      },
      "image": {
        "kind": "DOCKER",
        "id": "hyge/proxy_debug:mesos2"
      },
      "artifacts": [
        {
          "uri": "https://s3-ap-southeast-1.amazonaws.com/marathon-cmd/bundle-certs.tgz",
          "extract": true,
          "executable": false,
          "cache": false
        }
      ]
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
      "constraints": []
    }
  }
}
```

通过 nsenter 进入 sleep的namespace：

```
$ ps aux|grep "serviceCluster sleep"
root     27467  0.0  0.0  46980  1508 ?        S    02:39   0:00 su istio-proxy -c /usr/local/bin/pilot-agent proxy --discoveryAddress pilot.istio.marathon.slave.mesos:31510  --domain marathon.slave.mesos --serviceregistry Mesos --serviceCluster sleep --zipkinAddress zipkin.istio.marathon.slave.mesos:31767 --configPath /var/lib/istio

$ nsenter -m -u -n -p -i -t 27467 /bin/sh
root@b729422b-0a96-4479-a843-27243ed5d5df:/# curl httpbin.marathon.slave.mesos:8000/ip
{
  "origin": "127.0.0.1"
}
```

* 给httpbin启动mtls规则

```
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: httpbin
spec:
  peers:
  - mtls: {}
  targets:
  # Must be short name of service
  - name: httpbin
    # subsets: "v1"
    ports:
    # name also supported
    # TODO: this should be target port (container)
    - number: 8000
```

再次 nsenter 进入 sleep的namespace
```
root@b729422b-0a96-4479-a843-27243ed5d5df:/# curl httpbin.marathon.slave.mesos:8000/ip
upstream connect error or disconnect/reset before headers
```
此时，由于服务间必须要用tls通信，而目前没有指定sleep服务的envoy启用mtls，所以会报错。我们还可以通过pilot的debug接口验证mtls的启用情况：
```
curl pilot.istio.marathon.slave.mesos:31993/debug/authenticationz

[
...
{
    "host": "httpbin.marathon.slave.mesos",
    "port": 8000,
    "authentication_policy_name": "httpbin/default",
    "destination_rule_name": "-",
    "server_protocol": "mTLS",
    "client_protocol": "HTTP",
    "TLS_conflict_status": "CONFLICT"
  },
...
]

```
可以看到`httpbin`在server端已经启用了mtls。但是客户端(sleep)还是http，所以`TLS_conflict_status`显示`CONFLICT`

* 使 envoy 使用mtls

为了使访问`httpbin`的客户端也启用mtls，配置如下`DestinationRule`：
```
cat <<EOF | kubectl apply -f -
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "httpbin"
  namespace: "default"
spec:
  host: "httpbin.marathon.slave.mesos"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF
```

此时，再次nsenter 进入sleep，就可以通过mtls 来访问 httpbin:
```
root@b729422b-0a96-4479-a843-27243ed5d5df:/# curl  httpbin.marathon.slave.mesos:8000/ip
{
  "origin": "127.0.0.1"
}
```

## 小结
在启用mtls后，`sleep`和`httpbin`之间的通信会被加密，加密的证书来自`https://s3-ap-southeast-1.amazonaws.com/marathon-cmd/bundle-certs.tgz`。我们还可以通过tcpdump来验证抓包的确是加密过的。
另外，本文中的证书是每个服务通过uri fetch得到。istio 将会在`1.1.0`版引入 `SDS(secret discovery services)`，由pilot自动分发证书。但是目前的稳定版仍然是`1.0.6`，所以自动分发证书的功能会考虑在之后的稳定版引入mesos
