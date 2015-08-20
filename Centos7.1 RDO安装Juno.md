安装kilo一直有问题，就安装的juno，最好能使用本地源，不然老报错，重复执行。
只有一台服务器，安装的centos7.1，安装成功后做以下操作：

1，配置固定ip，然后执行
```
systemctl stop NetworkManager
systemctl disable NetworkManager
systemctl enable network
```
将网卡重启下：
```
ifdown <interface_name> && systemctl start network
```
2,安装国内源，比如163
3，更新系统，并重启
```
sudo yum update -y
sudo yum install -y https://rdoproject.org/repos/rdo-release.rpm
sudo yum install -y openstack-packstack
```
4,设置selinux为允许：setenforce 0
5,创建一个vg名字cinder-volumes，作为cinder后端存储
6，生产配置文件并修改
```
#packstack --gen-answer-file=juno.txt
```
修改内容有：
>CONFIG_KEYSTONE_ADMIN_PW=admin

>CONFIG_CINDER_VOLUMES_CREATE=n

7，开始安装
packstack --answer-file=juno.txt
安装很耗时，还会出错，一般根据日志能找到原因，除非是官方的bug。
8,安装完后要配置网络
配置br-ex桥接文件
```
# cat ifcfg-br-ex 
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=192.168.1.236
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
ONBOOT=yes
```
配置网卡文件
```
# cat ifcfg-enp9s0f0 
TYPE="Ethernet"
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-ex
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
NAME="enp9s0f0"
UUID="aafbda5c-01af-4431-81a3-00c4aef19f75"
DEVICE="enp9s0f0"
ONBOOT="yes"
```
然后执行以下命令：
```
ovs-vsctl add-port br-ex enp9s0f0 && systemctl restart network
```

9,安装完之后的网络默认是vxlan，我在创建自己的flat网络时一直报错。后来查看日志发现需要修改配置文件
修改的配置文件为 /etc/neutron/plugins/ml2/ml2_conf.ini
```

[ml2]
...
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = gre
mechanism_drivers = openvswitch
...
tenant_network_types = local,flat,vlan,gre,vxlan
```
10,系统默认虚机存放路径的修改，修改/etc/nova/nova.conf
```
# Where instances are stored on disk (string value)
#instances_path=$state_path/instances
instances_path=/home/instances
```

