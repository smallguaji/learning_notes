# Docker

**<font color="brown">官方解释：应用容器引擎，属于对容器的一种封装，并提供接口。</font>**



### docker虚拟化与虚拟机虚拟化的区别

#### 传统虚拟机两种架构：

**① 在Hypervisor层上构建VM——裸金属架构**

![](https://raw.githubusercontent.com/smallguaji/picture/master/%E8%A3%B8%E9%87%91%E5%B1%9E%E6%9E%B6%E6%9E%84.png)



**② 在操作系统上构建VM——寄居模式**

![](https://raw.githubusercontent.com/smallguaji/picture/master/%E5%AF%84%E5%B1%85%E6%9E%B6%E6%9E%84.png)

传统虚拟机发布应用，虚拟化的应用不仅包含应用本身、所依赖的库文件，还要包括整个guest os。而Docker的虚拟化是操作系统层级，容器不是模拟一个完整的系统，而是对进程进行隔离，相当于在正常进程的外面套了一个保护层。

**<font color="purple">虚拟机 -> 系统</font>**

**<font color="purple">容器 -> 应用</font>**



#### Docker架构

**Docker为C/S架构，分为Docker Client和Docker Server(Docker Daemon)。client和server可以在同一个机器上，也可以为远程。client和server之间的通信可以通过stock、RESTful等接口。**



#### Docker的三个基本概念

1. 镜像 image

   ![](https://raw.githubusercontent.com/smallguaji/picture/master/%E9%95%9C%E5%83%8F.png)

   docker image是一个只读模板，是一堆只读层（read-only layer）的同一视角（UFS——union file system）

2. 容器 container

   ![](https://raw.githubusercontent.com/smallguaji/picture/master/%E5%AE%B9%E5%99%A8.png)

   容器可以理解成一个进程，容器的结构与镜像几乎相同，container也是许多层的一个集合，但container的最上层是可读可写层（read-write layer）。只有可读可写层是可以改变的，所有文件更改也都会保存在这一层上。

3. 仓库repository和仓库注册服务器registry

   Docker registry是一个提供仓库注册服务的服务器，它以容器的形式运行在集群中，

   Docker repository是镜像仓库，用于存储具体的镜像。

   registry中可以注册多个镜像仓库repository。而一个repository通常存放一类镜像，同一repository中不同镜像有不同的tag，用来表示版本。

   docker pull registry_name/namespace/image_name:tag

   默认tag为latest



在打镜像时，guest os也为镜像中的一部分，但容器比虚拟机轻量许多，其原因是在镜像中的操作系统没有内核，只有操作系统的一些组件。

且若两个镜像中具有相同镜像层，在保存镜像文件时，看似重复存储了相同的镜像层，但实际上只储存了一次，其实现机制要归功于镜像文件不同层之间的指针。



假设有三个应用的镜像文件如下图所示：

![](https://raw.githubusercontent.com/smallguaji/picture/master/%E9%95%9C%E5%83%8F%E5%AD%98%E5%82%A8.png)

而在实际存储的时候，应是如下图结构

![](https://raw.githubusercontent.com/smallguaji/picture/master/image%E5%AD%98%E5%82%A8.png)