# Ubuntu 16.04 虚拟机外网搭建 kubernetes cluster

<font size=1 color="purple">by 小呱唧</font>

### kubernetes cluster结构

![](https://raw.githubusercontent.com/smallguaji/picture/master/%E5%A4%A7%E4%BD%93%E6%9E%B6%E6%9E%84.png)



![](https://raw.githubusercontent.com/smallguaji/picture/master/k8s.png)



### 组件功能

docker源  ->   docker-ce的安装image

kubernetes源   ->   kubelet、kubeadm、kubectl的安装image

registry源   ->   api server、etcd、schedule、controller manager的安装image



**kubelet（所有节点）**：kubelet是运行在每个节点上主要的"节点代理"，每个节点都会启动kubelet进程，用来处理master节点下发到本节点的任务。kubelet通过各种机制（主要为api server）获取一组PodSpec并保证这些PodSpec中描述的容器健康运行。

主要功能有：pod管理（定期从所监听的数据源获取节点上pod/container的期望状态——包括运行什么容器、运行的副本数量、网络或存储如何配置。）、容器健康检查、容器监控

**kubeadm（master节点）**：kubeadm是kubernetes官方提供的用于快速安装kubernetes cluster集群的工具。

**kubectl（所有节点，但不是必需，在本例中需要用kubectl安装flannel）**：kubectl是kubernetes集群的命令行，通过kubectl能够对集群本身进行管理，并能够在集群上进行容器化应用的安装部署。



**api server（master节点）**：核心功能是提供了kubernetes各类资源对象（Pod、RC、service）的增、删、改、查及HTTP REST接口。

server是通过一个名为kube-apiserver的进程提供服务，该进程在master节点上，默认情况下，在本机8080端口提供REST服务。

**etcd（master节点）**：用于保存集群中所有网络配置和对象的状态信息。整个kubernetes cluster中一共有两个服务需要用到etcd来协同和存储配置，分别是：① 网络插件cni、对于其他网络插件也需要用到etcd存储网络的配置信息。② kubernetes本身，包括各种对象的状态和原信息配置。

**controller manager（master节点）**：集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点（EndPoint）、命名空间（namespace）、服务账号（ServiceAccount）的管理，当某个Node意外宕机时，controller-manager会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

**scheduler（master节点）**：根据特定的调度算法将pod调度到指定的工作节点（Node）上。



**flannel（所有节点）**：cni（网络插件）之一，负责容器间的通信、Pod之间的通信、Pod与Service之间的通信、Service与集群外部的通信。



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

<font color="blue" size=2>find / -name "{file_name}"   : 查找文件</font>

Linux时间同步有两种方案：①ntpd      ②ntp + ntpdate    本例中采用第二种方案。    ☆ntpd和ntpdate不能同时使用了端口123。

`sudo apt-get install ntp`

`sudo apt-get install ntpdate`

如果同步后时间仍不对，考虑是系统时区错误，修改系统时区

`date -R`可查看当前时区

`sudo tzselect`选择时区

永久配置系统时区

`sudo cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime`

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
kubeadm init --kubernetes-version=v1.15.0 --apiserver-advertise-address=192.168.108.130 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16
```

<font color="red">1.apiserver-advertise-address的ip地址应为apiserver所在节点的ip地址，而在kubernetes集群中，apiserver组件安装在master节点，所以该参数的值应为master节点虚拟机的ip地址。</font>

<font color="red">2. pod_network-cidr为pod网络的范围</font>

<font color="red">3. image-repository指定了初始化kubernetes集群master节点时拉取镜像的镜像仓库，如果不写该参数，则表示从其官方镜像仓库拉取镜像（master所需组件的镜像——api server、etcd、controller-Manager、schedule），而这需要翻墙才能完成，所以在本例中使用阿里云的镜像仓库。</font>

初始化成功后，控制台将返回一个token，这个token用来往集群里添加新的节点。

token的有效期为24个小时，如果在后期需要往cluster中添加新节点，需要重新生成token

`kubeadm token create`

获得CA公钥的哈希值

`openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed  's/^ .* //'`

然后将新节点添加进cluster中。



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

该命令可以显示该服务的状态以及配置文件路径



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

**2、 使用kubectl命令出现以下错误[The connection to the server localhost:8080 was refused]**

出现该错误的原因是kubectl命令需要使用kubernetes-admin来运行，解决方法为解决方法如下，将主节点中的【/etc/kubernetes/admin.conf】文件拷贝到从节点相同目录下，然后配置环境变量：

`echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile`

`source ~/.bash_profile`

https://blog.csdn.net/nklinsirui/article/details/80583971

**3、 集群搭建完成后，查看master和node的kubelet状态，发现有以下错误**

`systemctl status kubelet`   该命令可查看当前kubelet的运行状态

`journalctl -xefu kubelet`   如果kubelet运行有错误，则可以用该命令查看详细错误信息。

**master**

```shell
kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Mon 2019-07-15 19:15:55 CST; 3 weeks 0 days ago
     Docs: https://kubernetes.io/docs/home/
 Main PID: 757 (kubelet)
    Tasks: 23
   Memory: 71.8M
      CPU: 2h 26min 23.725s
   CGroup: /system.slice/kubelet.service
           └─757 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --cgroup-driver=cgroupfs --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3
~
Aug 06 13:57:07 guaji-master kubelet[757]: E0806 13:57:07.190393     757 summary_sys_containers.go:47] Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container 
Aug 06 13:57:17 guaji-master kubelet[757]: E0806 13:57:17.290830     757 summary_sys_containers.go:47] Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container 
Aug 06 13:57:27 guaji-master kubelet[757]: E0806 13:57:27.367150     757 summary_sys_containers.go:47] Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container 

```

解决这个问题需要在配置文件/etc/systemd/system/kubelet.service.d/10-kubeadm.conf中添加环境参数

```
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --cgroup-driver=cgroupfs"

Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"

Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"

Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"

Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"

Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"

# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically

EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env

# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.

EnvironmentFile=-/etc/default/kubelet

ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

然后生效配置文件，再重启kubelet服务

`systemctl daemon-reload`

`systemctl restart kubelet`

**node**

```shell
kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Fri 2019-07-05 21:37:10 CST; 1 months 1 days ago
     Docs: https://kubernetes.io/docs/home/
 Main PID: 11934 (kubelet)
    Tasks: 18
   Memory: 43.9M
      CPU: 1h 23min 42.991s
   CGroup: /system.slice/kubelet.service
           └─11934 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.1
Aug 06 13:56:58 guaji-node1 kubelet[11934]: E0806 13:56:58.582830   11934 reflector.go:125] object-"kube-system"/"kube-proxy": Failed to list *v1.ConfigMap: Get https://192.168.108.130:6443/api/v1/namespaces/kube-system/configmaps?fieldSelector=metadata.name%3Dkube-proxy&limit=500&resourceVersion=0: dial tcp 192.168.108.130:6443: connect: no route to host
Aug 06 13:56:58 guaji-node1 kubelet[11934]: E0806 13:56:58.582904   11934 reflector.go:125] object-"kube-system"/"kube-flannel-cfg": Failed to list *v1.ConfigMap: Get https://192.168.108.130:6443/api/v1/namespaces/kube-system/configmaps?fieldSelector=metadata.name%3Dkube-flannel-cfg&limit=500&resourceVersion=0: dial tcp 192.168.108.130:6443: connect: no route to host
Aug 06 13:56:58 guaji-node1 kubelet[11934]: E0806 13:56:58.582976   11934 reflector.go:125] object-"kube-system"/"flannel-token-65mps": Failed to list *v1.Secret: Get https://192.168.108.130:6443/api/v1/namespaces/kube-system/secrets?fieldSelector=metadata.name%3Dflannel-token-65mps&limit=500&resourceVersion=0: dial tcp 192.168.108.130:6443: connect: no route to host
```

如果报错的节点为HA的cluster中的某一个master节点，则原因为加入master节点时没有创建用户。具体操作看上文【配置kubectl用户】步骤。

如果在不是HA的集群中的slave节点抛出这样的问题，则属于正常状况，如果强迫症想修复这个问题，可以将master节点中 $HOME/.kube 目录复制到其他node中的相同路径下。



`etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key endpoint health`



### 参考网址

阿里云官方安装docker文档： https://help.aliyun.com/document_detail/60742.html

阿里巴巴镜像开源站: http://mirrors.aliyun.com

