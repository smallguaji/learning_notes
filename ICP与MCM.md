# ICP与MCM

### IBM Cloud

**icp：** IBM Cloud Private

**iks：** IBM kubernetes Service



**pmc：** Platform Management Console   私有云管理组件

**cam：** Cloud Automation Management  ICP组件，通过可视化脚本配置cluste，基于VM



### MCM（Multi-Cloud Management）多云管理

#### Visibility

①  管理不同的云，并且实现determine allocation

②  VA：扫描docker image 仓库，icp3.2之上版本支持第三方仓库

③  Multi-tenant Core（多租户）Service in Single Cluster via namespaces：one user can access multiple namespace。

④  轻量安装kubernetes：kubernetes的部分组件使用率较低或在某场景可省略。

⑤  Mutation policy controller：突变定制控制器。

![](https://github.com/smallguaji/picture/raw/master/mcm.png)



#### App迁移和cluster扩展

![](https://raw.githubusercontent.com/smallguaji/picture/master/%E9%9B%86%E7%BE%A4%E5%A4%8D%E5%88%B6.png)

**实现机制**

①  在cluster2中预先配置好app的image，但不运行。

②  GLB会定时去检查时候有应用的容器发生故障（默认30s检查一次），若检测到app死掉，则立刻在cluster2中运行该app的镜像文件。

![](https://raw.githubusercontent.com/smallguaji/picture/master/%E9%9B%86%E7%BE%A4%E5%A4%8D%E5%88%B6.png)

当当前集群无法负荷过多的业务量时，GLB将自动创建多个相同集群。



#### Governance：placement and compliance policies

**police分发**

![](https://raw.githubusercontent.com/smallguaji/picture/master/policy%E5%88%86%E5%8F%91.png)



#### service的跨cluster发现

![](https://raw.githubusercontent.com/smallguaji/picture/master/service%E8%B7%A8%E9%9B%86%E7%BE%A4%E5%8F%91%E7%8E%B0.png)

该场景发生在某个cluster中的app需要调用另一个cluster的service。



#### istio plugin

![](https://raw.githubusercontent.com/smallguaji/picture/master/istio.png)

