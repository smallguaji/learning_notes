# Ubuntu 16.04 虚拟机外网搭建 kubernetes cluster

<font size=1 color="purple">by 小呱唧</font>

### kubernetes cluster结构

![](https://raw.githubusercontent.com/smallguaji/picture/master/%E5%A4%A7%E4%BD%93%E6%9E%B6%E6%9E%84.png)



![](https://raw.githubusercontent.com/smallguaji/picture/master/k8s.png)

### 虚拟机配置

虚拟机软件：VMware Workstations Pro 15

节点配置：

​				master： guaji-master

​				node1： guaji-node1

​				node2： guaji-node2           

**配置网络**

将虚拟机网卡配置成静态ip，其原因是当电脑的网络环境发生变化时（比如从有线连接转到wifi连接或者更换办公地点时，本机的ip网段可能会发生变化，从而导致虚拟机的联网环境发生变化）

以VMware workstations为例：首先虚拟机联网方式选择为**<font color="red">NAT</font>**联网模式，使用该联网模式状态下，vmware将自动产生一个虚拟网卡 **VMnet8**，将这个网卡的ip地址改为静态地址。

将虚拟机网卡设置为静态ip后，再将虚拟机的ip设置为静态ip，其原因是，虚拟机的ip也是通过DHCP获得的，当虚拟机重启之后，ip地址将会在特定网段范围内发生改变，但如果虚拟机的ip地址发生改变，kubernetes集群将会找不到初始化时所配置的IP地址，从而导致集群启动失败。

首先查看VMware虚拟机的NAT设置。 <font color="blue">编辑 -> 虚拟网络编辑器 -> 选择VMnet8网卡 -> NAT设置</font> 可查看到子网IP地址和网关地址。在本例中分别为192.168.108.0和192.168.108.2

进入虚拟机，先查看一下本机的ip `ifconfig` 网卡应为ens33，在本例中本机ip为192.168.108.130

将配置文件<font color="purple">/etc/network/interfaces</font>修改为

```
# interfaces(5) file used by ifup(8) and ifdown(8)
auto ens33
iface ens33 inet static
address 192.168.108.130
netmask 255.255.255.0
gateway 192.168.108.2

dns-nameservers 192.168.108.2
```



### 节点安装步骤（除master节点外，其他node不需要初始化master节点及之后的步骤，但需要安装网络组建flannel）

**更新软件列表**

**<font color="orange">sudo apt-get update</font>**



**安装ntp——用于同步各节点的时间**

<font color="blue" size=2>ps -aux：查看当前进程</font>

Linux时间同步有两种方案：①ntpd      ②ntp + ntpdate    本例中采用第二种方案。    ☆ntpd和ntpdate不能同时使用了端口123。

`sudo apt-get install ntp`

`sudo apt-get install ntpdate`

1.设置开机自动启动ntp服务

chkconfig ntp on  —>   `sysv-rc-conf ntp on` （在ubuntu，chkconfig被sysv-rc-conf代替）

2.修改系统时间并将系统时间同步到硬件时间

`ntpdate 210.72.145.44`   —同步210.72.145.44的时间

`hwclock -w` —将系统时间同步到硬件时间

修改/etc/crontab配置文件   `sudo crontab -e`

在文件的最后一行写入   

`10 5 * * * root /usr/sbin/ntpdate -n 210.72.145.44; hwclock -w` 

10-表示10分；5-表示5点；三个*号-表示每天，所以10 5 * * *表示每天的5点10分；210.72.145.44这个IP地址是用作时间同步，该ip可以在ntp.org.cn中查找可用ip池。

3.重启linux计划任务

`sudo /etc/init.d/cron restart`



**更改主机名称**

两种方式：

①修改/etc/hostname（直接在文件中写入想更改为的主机名） 

②`hostnamectl set-hostname {NAME}`

两者的区别为修该配置文件需要reboot重启才能生效，而用命令更改则是即时生效。

实测在redhat系统下命令更改可以即时生效，但在ubuntu系统下仍需要reboot重启生效。



**关闭防火墙和swap** 

`systemctl stop firewalld`   关闭防火墙

`systemctl disable firewalld`    禁用防火墙 

（ubuntu默认关闭防火墙）

`swapoff -a`   关闭swap交换区，-a表示关闭所有交换设备

但swapoff只能关闭当前会话的swap，如果重启虚拟机，swap将重新打开，要永久关闭swap需要修改配置文件<font color="red">/etc/fstab</font>

`sudo sed -i.bak '/swap / s/^\(.*\)$/#\1/g' /etc/fstab`

然后 `reboot` 重启

`sudo free -m` 查看分区状态



**安装docker（由于国外网站经常被墙，本教程所有的源都用的阿里云的源）**

首先安装一些依赖组件： 

`apt-get install apt-transport-https ca-certificates curl software-properties-common -y`

添加docker镜像源的GPG-KEY（拉取docker安装包时需要gpg密钥作为打开仓库的钥匙）

`curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -`

添加docker稳定版仓储地址

`sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"`

**<font color="red" size=2>执行以上的命令会将稳定版仓储地址添加到/etc/apt/source.list文件中</font>**

`sudo apt-get update`

`sudo apt-cache madison docker-ce`  查看可安装的版本

`sudo apt-get install docker-ce=18.06.1~ce~3-0~ubuntu`  安装特定版本



**创建docker用户组，并将操作用户添加到docker用户组中**

`sudo groupadd {GROUPNAME}`   <font size=3 color="green">默认会创建一个名为docker的用户组</font>

`sudo usermod -aG {GROUPNAME} {USERNAME}`  <font size=3 color="green">默认情况下{GROUPNAME}为docker，{USERNAME}分别写为root和guaji</font>



**设置docker为开机启动**

`sudo systemctl enable docker`  或者  `sudo sysv-rc-conf docker on`



**安装kubernetes**

添加kubernetes源的GPG-KEY

`sudo curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -`

```shell
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    	deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
   	EOF
```

**<font color="red" size=2>执行以上的命令会将kubernetes源添加到/etc/apt/sources.list.d/kubernetes.list文件中</font>**

更新apt源

`sudo apt-get update`

`sudo apt-get install kubelet kubeadm kubectl`     <font size=3 color="green">默认安装最新版本，本例中安装1.15版本，安装的版本信息可通过 kubectl version查看</font>

`sudo systemctl enable kubelet`



**初始化kubernetes集群的master节点（node节点不需要此步骤）**

```shell
kubeadm init --kubernetes-version=v1.15.0 --apiserver-advertise-address=192.168.108.130 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=192.168.0.0/16
```

<font color="red">1.apiserver-advertise-address的ip地址应为apiserver所在节点的ip地址，而在kubernetes集群中，apiserver组件安装在master节点，所以该参数的值应为master节点虚拟机的ip地址。</font>

<font color="red">2. pod_network-cidr为pod网络的范围</font>

<font color="red">3. image-repository指定了初始化kubernetes集群master节点时拉取镜像的镜像仓库，如果不写该参数，则表示从其官方镜像仓库拉取镜像（master所需组件的镜像——api server、etcd、controller-Manager、schedule），而这需要翻墙才能完成，所以在本例中使用阿里云的镜像仓库。</font>

初始化成功后，控制台将返回一个token，这个token用来往集群里添加新的节点。



**配置kubectl用户**

只有配置过的用户才能用kubectl命令，在本例中，需要配置的用户为guaji和root用户（需要分别切换到这两个用户后执行以下操作）

`mkdir -p $HOME/.kube`

`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`

`sudo chown $(id -u):$(id -g) $HOME/.kube/config`



**安装通信网络**

有多种组件可以选择，本例中选择flannel，在github上下载kube-flannel.yml

`kubectl apply -f kube-flannel.yml`



**将slave节点加入集群（在slave节点中执行以下操作）**

```
kubeadm join 192.168.108.130:6443 --token kt62dw.q99dfynu1kuf4wgy --discovery-token-ca-cert-hash sha256:5404bcccc1ade37e9d80831ce82590e6079c1a3ea52a941f3077b40ba19f2c68
```

token的值和sha256的会变。



### 设置节点之间的SSH免密传输

将三个节点的ip信息写入 <font color="blue">/etc/hosts </font>三个节点的配置文件都要更改

```
192.168.108.130 guaji-master
192.168.108.131 guaji-node1
192.168.108.132 guaji-node2
```

三个节点上都安装ssh

`apt-get install ssh`

修改ssh的配置文件

`sudo gedit /etc/ssh/sshd_config`

将PermitRootLogin的值设置为yes

重启ssh 

`sysv-rc-conf restart sshd`

**配置免密传输**

生成公钥和私钥

`cd ~`

`ssh-keygen -t rsa`

在~/.ssh文件夹中会生成id_rsa.pub和id_rsa两个文件，分别为该节点的公钥和私钥。

将master的id_rsa.pub分别scp到node1和node2的~/.ssh中，并命名为authorized_keys（这个目录是ssh服务默认的，如果很闲想更改可以在ssh的/etc/ssh/sshd_config配置文件里改默认路径）

将authorized_kets的权限改为600

`chmod 600 ~/.ssh/authorized_keys`

**<font color ="pink">查看某服务的配置文件路径可用</font>** `systemctl status {SERVICENAME}` 

该命令可以显示该服务的状态已经配置文件路径

### 异常处理

**1、 master初始化时，总是在[wait-for-control-panel]阶段中断，查看错误信息，可知docker或kubelet的启动有问题。**

首先考虑是否docker cgroup和kubelet cgroup的driver不一致，修改kubelet或docker.service配置文件。docker的默认cgroup驱动为systemd，而kubelet的默认crgoup驱动为cgroupfs.

docker的crgroup driver可以通过`docker info`查看。

①更改docker的cgroup driver：

修改/etc/docker/daemon.json配置文件（没有该文件就新建一个）

```j
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

然后重新加载配置文件，重启docker

`systemctl daemon-reload`

`systemctl restart kubelet`

`systemctl restart docker`

②修改kubelet的cgroup driver

配置文件路径为<font color="purple">/etc/systemd/system/kubelet.service.d/10_kubeadm.conf</font>

```
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --cgroup-driver=cgroupfs"
```

每次重新初始化master节点之前都要先清除之前的安装结果

`kubeadm reset`



### 参考网址

阿里云官方安装docker文档： https://help.aliyun.com/document_detail/60742.html

阿里巴巴镜像开源站: http://mirrors.aliyun.com

