# mesos-container
为了能使用 mesos-container， 需要在 mesos-slave 配置`isolation, image_providers, containerizers` 等：
一种办法是在 ` /etc/default/mesos-slave` 中添加如下内容：
```
MESOS_IMAGE_PROVIDERS=docker
MESOS_CONTAINERIZERS=docker,mesos
MESOS_ISOLATION=cgroups/cpu,cgroups/mem,disk/du,network/cni,filesystem/linux,docker/runtime,volume/sandbox_path,volume/secret,posix/rlimits,namespaces/pid,linux/capabilities,cgroups/devices
```
也可以在 `/etc/mesos-slave` 目录下单独添加
```
echo "docker" > /etc/mesos-slave/image_providers
echo "docker,mesos" > /etc/mesos-slave/containerizers
echo "cgroups/cpu,cgroups/mem,disk/du,network/cni,filesystem/linux,docker/runtime,volume/sandbox_path,volume/secret,posix/rlimits,namespaces/pid,linux/capabilities,cgroups/devices" > /etc/mesos-slave/isolation
```
其中 isolation 定义了容器间资源是如何隔离的，网络如何配置，以及安全规则如何应用。具体可以参见[http://mesos.apache.org/documentation/latest/mesos-containerizer/](http://mesos.apache.org/documentation/latest/mesos-containerizer/)。添加这些isolation，也有助于通过nsenter进入mesos-container进行调试。
