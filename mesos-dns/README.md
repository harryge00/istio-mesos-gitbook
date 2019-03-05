# Mesos-DNS 部署
Mesos-dns是 mesos 服务发现工具，能查找app的Ip，端口号以及master，leader等信息。在Apache Mesos 集群中,允许应用程序和服务通过域名系统(DNS)来相互定位。Mesos-DNS 的特点是轻量、无状态,易于部署和维护。Mesos-DNS 将每个集群中正在运行的应用程序的域名转换成IP 地址和端口号。这使得任何链接到集群中运行的服务都可以通过DNS 进行查找。 Mesos-DNS是简单和无状态的，它不需要一致性机制，持久存储或日志复制，因为应用所需的心跳，健康监控或生命周期管理等功能都已经由Master、Agent和框架所提供。Mesos-DNS可以通过使用像Marathon这样的框架来实现容错，可以监控应用程序运行状况并在应用出现故障时重新启动它。Mesos-DNS因故障重新启动时，无需进一步协调它会从Master节点检索最新的应用状态，然后处理DNS请求。可以通过增加Master节点，即可提高可用性并为大规模集群中的DNS请求提供负载均衡。

## Mesos-DNS 安装
Mesos-DNS 可以装在Mesos集群的任意一台机器上，或者是在同一网络的一台专用机器上。为了实现HA，可以在多个master上分别部署mesos-dns 实例。
```
wget https://github.com/mesosphere/mesos-dns/releases/download/v0.6.0/mesos-dns-v0.6.0-linux-amd64
sudo mv mesos-dns-v0.6.0-linux-amd64 /usr/local/bin/
```

接下来，按照链接 http://mesosphere.github.io/mesos-dns/docs/configuration-parameters.html 说明为你的集群创建一个配置文件。可以这样启动Mesos-DNS
```
sudo su
mkdir /etc/mesos-dns
cat > /etc/mesos-dns/config.json << EOF
{
  "zk": "zk://10.101.160.15:2181/mesos",
  "masters": ["10.101.160.15:5050", "10.101.160.16:5050", "10.101.160.17:5050"],
  "refreshSeconds": 60,
  "mesosCredentials": {
    "principal": "admin",
    "secret": "password"
  },
  "mesosAuthentication": "basic",
  "ttl": 60,
  "domain": "mesos",
  "port": 53,
  "externalon": false,
  "timeout": 5
}
EOF

# Create Systemd service
cat > /usr/lib/systemd/system/mesos-dns.service << EOF
[Unit]
Description=Mesos-DNS
After=network.target
Wants=network.target

[Service]
ExecStart=/usr/local/bin/mesos-dns -config=/etc/mesos-dns/config.json
Restart=on-failure
RestartSec=20

[Install]
WantedBy=multi-user.target
EOF

# Start mesos-dns 
systemctl daemon-reload
systemctl start mesos-dns
```


### 设置从节点 
允许Mesos任务使用Mesos-DNS作为主DNS服务，必须在每个从节点上修改文件 /etc/resolv.conf增加新的nameserver。例如，如果多个mesos-dns运行的服务器地址是`10.181.64.11，10.181.64.12，10.181.64.13`， 则应该在每个从节点/etc/resolv.conf开始加上三行nameserver 10.181.64.1X。可以通过运行下面的命令实现：
```
sudo sed -i '1s/^/nameserver 10.181.64.11\n /' /etc/resolv.conf
sudo sed -i '2s/^/nameserver 10.181.64.12\n /' /etc/resolv.conf
sudo sed -i '3s/^/nameserver 10.181.64.13\n /' /etc/resolv.conf
```
这些条目的顺序将确定从节点连接Mesos-DNS实例的顺序。可以通过设置options rotate在nameserver之间选择负载均衡的轮循机制, 如
```
# cat /etc/resolv.conf 
options rotate
nameserver 10.181.64.11
nameserver 10.181.64.12
nameserver 10.181.64.13
```

#### 持久化 /etc/resolv.conf
由于重启后/etc/resolv.conf可能被覆盖，为了持久化，可以在`/etc/sysconfig/network-scripts/ifcfg-lo`中添加dns服务器地址：
```
DNS1=10.181.64.11
DNS2=10.181.64.12
DNS3=10.181.64.13
```


### 验证mesos-dns
我们可以在 marathon 上创建1个应用，比如nginx，然后在从节点应该能获取到对应的dns解析记录
```
nslookup nginx.marathon.slave.mesos
Server:		172.31.23.220
Address:	172.31.23.220#53

Name:	nginx.marathon.slave.mesos
Address: 172.31.27.226
```