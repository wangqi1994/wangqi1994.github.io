---
title: "yarn调优-container"
catalog: true
toc_nav_num: true
date: 2020-05-18 12:00:00
subtitle: "container调优"
article_type: 0 # 0 原创，1 转载
tags:
- [ruozedata]
- [Hadoop]
categories:
- [ruozedata]
- [Hadoop]
updateDate: 2020-05-18 12:00:00
top: 0  #置顶
---

关于Yarn的调优 就是调container  
首先明确一个container容器的概念，这是一个虚拟的概念，其实是一组memory+cpu vcore资源的组合,是用来运行task任务的 

生产如何调优container参数？

假设128G  16物理core
#### 1.1 系统装完之后， 内存消耗1G
#### 1.2 系统预留20%内存（包含1G）
128G*20%=25.6G==26G

为什么预留内存？
oom-kill机制
Linux系统防止夯住
给未来部署组件预留空间


#### 1.3 假设只有DN NM节点
生产上部署一般遵循存储技术一体，90%以上都是dn和nn放一起，
就是计算时发现本节点有数据不需要去其他节点拉取，节省网络io，这种一般叫做 数据本地化 

DN=2G 在生产上一般设置2G即可  
在实际中业务高峰时用了900M，
（由于是分布式存储，且存在多个dn节点，每秒一般没有1G的流量，况且还有一定的缓存在发挥作用）
当然在资源要求不是很严格且内存充足，不需要扣扣搜搜的时候，可以设置3G/4G

NM=4G 生产上一般4G
一般资源比较紧缺时，就是在NM这里抠资源

去除掉系统预留、dn和nm之外，剩余资源全部给真正干活的小弟container用

128-26=102-2-4=96G 

#### 1.4 container内存设置，基于内存考虑

设置yarn-site.xml文件中的参数
```shell
[hadoop@hadoop001 hadoop]$ pwd
/home/hadoop/app/hadoop/etc/hadoop
[hadoop@hadoop001 hadoop]$ vi yarn-site.xml 

    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>96</value> # 该参数控制所有container内存之和,生产上需要调整，默认只有8G
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>1</value> #container容器内存最小值，默认1G 
    </property> #在极限情况下，会存在96个container，每个内存1G
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>96</value> #container容器内存最大值，官方默认8G，应该改为上面参数的最大值 
    </property> #在极限情况下，会存在1个container，内存96G，这种情况少
```

container容器的内存会根据实际情况自动增加，默认1G递增 
cdh中yarn配置的 但是一般不调整

#### 1.5 container vcore 基于cpu考虑
目前的CPU被划分成虚拟CPU（CPU virtual Core），这里的虚拟CPU是YARN自己引入的概念，
初衷是，考虑到不同节点的CPU性能可能不同，每个CPU具有的计算能力也是不一样的，
比如某个物理CPU的计算能力可能是另外一个物理CPU的2倍，
这时候，你可以通过为第一个物理CPU多配置几个虚拟CPU弥补这种差异。
用户提交作业时，可以指定每个任务需要的虚拟CPU个数。在YARN中，CPU相关配置参数如下：
（1）yarn.nodemanager.resource.cpu-vcores
表示该节点上YARN可使用的虚拟CPU个数，默认是8，
注意，目前推荐将该值设值为与物理CPU核数数目相同。
如果你的节点CPU核数不够8个，则需要调减小这个值，而YARN不会智能的探测节点的物理CPU总数。
（2）yarn.scheduler.minimum-allocation-vcores
单个任务可申请的最小虚拟CPU个数，默认是1，如果一个任务申请的CPU个数少于该数，则该对应的值改为这个数。
（3）yarn.scheduler.maximum-allocation-vcores
单个任务可申请的最多虚拟CPU个数，默认是32。
默认情况下，YARN是不会对CPU资源进行调度的，你需要配置相应的资源调度器。

**举例说明**：
第一台机器 强悍     pcore:vcore=1:2  16core:32vcore
第二台机器 不强悍   pcore:vcore=1:1  16core:16core

现在生产上基本上不考虑设置强悍程度
统一 设置 pcore:vcore=1:2

为什么要设置 vcore 1:2呢？
container需要vcore  
并发任务是靠vcore
```shell
    <property>
        <name>yarn.nodemanager.resource.pcores-vcores-multiplier</name>
        <value>2</value>  # pcore:vcore=1:2
    </property> 
    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>32</value> # 可用虚拟vcore数
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-vcores</name>
        <value>1</value> #可用虚拟vcore最小数，极限情况下 是32个
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-vcores</name>
        <value>32</value> #可用虚拟vcore最大数，极限情况下 是1个
    </property> 
```

#### 1.6 生产如何设置 基于生产经验
cloudera公司推荐，一个container的vcore最好不要超过5个，那么设置4

yarn.scheduler.maximum-allocation-vcores 4 极限情况下，只有8个container 


#### 1.7 整合memory cpu

确定 4 vcore  8个container
```shell
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>96</value> # 该参数控制所有container内存之和,生产上需要调整，默认只有8G
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>1</value> #container容器内存最小值，默认1G 
    </property> #在极限情况下，会存在96个container，每个内存1G
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>12</value> #container容器内存最大值，官方默认8G，应该改为上面参数的最大值 
    </property> # 极限情况container 8个,所以设置为12G
```
但是spark计算时内存有些指标比较大，那么这个参数必然调大，
那么这种理想化，完美化的设置必然被打破，到时以memory为主，可以提升内存的量

```shell
    <property>
        <name>yarn.nodemanager.resource.pcores-vcores-multiplier</name>
        <value>2</value>  # pcore:vcore=1:2
    </property> 
    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>32</value> # 可用虚拟vcore数
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-vcores</name>
        <value>1</value> #可用虚拟vcore最小数，极限情况下 是32个
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-vcores</name>
        <value>4</value> #可用虚拟vcore最大数，极限情况下 是1个
    </property> 
```
**生产上机器内存大小最好不要超过32G,调优参数不是越大越好**
30-32g比较神奇
https://blog.csdn.net/weixin_44641024/article/details/103248842

补充设置：
yarn.nodemanager.pmem-check-enabled  false #是否将对容器实施物理内存限制 一般关闭
yarn.nodemanager.vmem-check-enabled  false
yarn.nodemanager.vmem-pmem-ratio      2.1  就是有个虚拟内存的概念 一般不要 不要调整这个参数


#### 1.7 笔试题
假如该节点还有其他组件，比如HBase RS节点
256G内存
cpu物理core 32核

DN
NM
HBase RS=30G 

请问以上6个参数如何设置？
预留256*20%=51.8G
暂定52G
dn=2G
nm=4G
HBase RS=30G 
container 总内存256-52-2-4-30=168G
物理core32，设置pcore：vcore=1：2
vcore =64
一般设置vcore最大4，那么container有16个
所以每个内存最大是10G

```shell
# cpu参数
    <property>
        <name>yarn.nodemanager.resource.pcores-vcores-multiplier</name>
        <value>2</value>  # pcore:vcore=1:2
    </property> 
    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>64</value> # 可用虚拟vcore数
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-vcores</name>
        <value>1</value> #可用虚拟vcore最小数，极限情况下 是32个
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-vcores</name>
        <value>4</value>
    </property> 

#内存参数
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>168</value> # 该参数控制所有container内存之和,生产上需要调整，默认只有8G
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>1</value> #container容器内存最小值，默认1G 
    </property> #在极限情况下，会存在96个container，每个内存1G
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>10</value> #container容器内存最大值，官方默认8G，应该改为上面参数的最大值 
    </property> # 极限情况container 16个,所以设置为10G
```