# CNI 集成
Mesos 源码中已经有Mesos CNI: https://github.com/apache/mesos/tree/master/src/slave/containerizer/mesos/isolators/network/cni
所以不管是rpm安装或者编译，都会产生cni插件的目录。rpm安装的方式，CNI插件在 `/usr/libexec/mesos`
```
$ ls /usr/libexec/mesos
mesos-cni-port-mapper  mesos-default-executor  mesos-executor  mesos-io-switchboard    mesos-tcp-connect
mesos-containerizer    mesos-docker-executor   mesos-fetcher   mesos-logrotate-logger  mesos-usage
```
但是这些插件依赖于 https://github.com/containernetworking/plugins 官方的二进制。所以我们需要在从节点添加来自于containernetworking的二进制文件，并在 mesos-slave 配置中指定。

```
wget https://github.com/containernetworking/plugins/releases/download/v0.7.4/cni-plugins-amd64-v0.7.4.tgz
tar -xzvf cni-plugins-amd64-v0.7.4.tgz -C /etc/cni
mkdir /etc/cniconf
cat > /etc/cniconf/ucr-default-bridge.conf << EOF
{
  "name": "mesos-bridge",
  "type" : "mesos-cni-port-mapper",
  "excludeDevices" : ["ucr-br0"],
  "chain": "UCR-DEFAULT-BRIDGE",
  "delegate": {
    "type": "bridge",
    "bridge": "ucr-br0",
    "isDefaultGateway": true,
    "forceAddress": false,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
      "type": "host-local",
      "subnet": "172.31.254.0/24",
      "routes": [
      { "dst":
        "0.0.0.0/0" }
      ]
    }
  }
}
EOF
```
配置mesos-slave
```
echo "MESOS_NETWORK_CNI_CONFIG_DIR=/etc/cniconf" >>  /etc/default/mesos-slave
echo "MESOS_NETWORK_CNI_PLUGINS_DIR=/usr/libexec/mesos:/etc/cni" >>  /etc/default/mesos-slave
systemctl daemon-reload
systemctl restart mesos-slave
```