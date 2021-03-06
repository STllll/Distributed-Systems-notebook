# 第一章 概述

## 什么是分布式计算

**分布式计算**其实就是一门计算机科学，它研究如何把一个需要非常巨大的计算能力才能解决的问题拆分成许多小的结果，然后把这些部分分配给许多计算机进行处理，最后把这些计算结果综合起来得到最终的结果。

## 什么是并行计算

**并行计算**是指同时使用多种计算资源解决计算问题的过程。并行计算是相对于串行计算来说的，所谓的并行计算分为时间和空间上的并行。时间上的并行就是指流水线技术，而空间上的并行则是指用多个处理器并发的执行计算。

![1547951039964](.\picture\1547951039964.png)

## 什么是云计算!

**云计算**实际上是分布式技术+服务化技术+资源隔离和管理技术(虚拟化)。通俗意义上的云计算是指开发者利用云API开发应用，然后上传到云上托管，并提供给用户使用。而开发者不用关系后面的运维和管理，以及机器资源分配等问题。

虚拟化和服务化是云计算的表现形式。虚拟化技术有Linux的lxc等。服务化技术主要是有软件即服务(Software-as-a-Service. SAAS)和平台即服务（Platform-as-a-Service， PAAS）两种。



## Fourinone设计思路

1.对并行计算Map/Reduce和forkjoin等思路进行对比和研究

2.侧重于小型灵活的、对业务逻辑进行并行编程的框架，基于该框架可以简单迅速建立并行的处理方式，较大程度的提高计算效率。可以在本地使用，也可以在多机使用，使用公共的内存和脚本语言配置，能嵌入式使用。

3.可以并行高效进行大数据和逻辑复杂的运算，并对外提供调用服务，所有的业务逻辑有业务开发方提供，未来并行计算平台上可高效的支持秒杀器作弊分析、炒信软件分析、评价数据分析、财务结算系统等业务逻辑运算。

4.可以尝试做得更加通用化，满足大部分并行计算需求

# 第二章 分布式并行计算的原理与实践

Fourinone架构

最初的设想是slave-master架构，master实现成一个SocketServer，master使用心跳协议检查master是否挂掉。

![1547953321057](.\picture\1547953321057.png)

但是这种架构唯一的优点我感觉就是实现起来简单，因为和单机多线程模型很像。缺点就是master太容易变成系统瓶颈了，因为master要负责分布式协调一致，还需要负责slave的容灾，还需要负责分布式的任务调度。master处于一个与整个分布式系统紧耦合的状态。所以这个架构不行。

现在fourinone的架构如下图：

![1547988712410](.\picture\1547988712410.png)

我们来描述一下整个框架的计算流程。

1.外部的程序将数据填充到手工仓库里面，也就是warehouse。（warehouse的实现是hashmap）。

2.包工头(程序里面就是Contractor抽象类的实现类)调用它实现的giveTask方法。giveTask传入承载数据的warehouse。

3.包工头的调用getWaitingWorkers系列的方法，与职介所(ParkSever)拿到可以操作的工人的对象(MigrantWorker的实现类)。

4.拿到工人对象的工头调用工人的doTask方法，传入warehouse等数据。工人执行完毕后返回。

5.工人会和职介所之间用心跳协议等方式确认工人是否存活。

在这里我们可以总结四个角色的作用。

**手工仓库(warehouse)**作为包工头和工人之间的数据承载体，他的必须能够接受任意的数量和任意类型的数据。他的底层实现是hashmap。

**包工头(Contractor)**需要实现giveTask方法。在调用getWaitingWorkers方法之后，获取空闲的工人对象。之后分配好每个工人的计算任务之后，将调用工人的doTask方法，传入warehouse。***这里doTask方法是异步方法，调用后立刻返回，包工头需要轮询工人的Status查看工人状态。***

**工人(MigrantWorker)**在自己的机器上初始化完成后，会调用waitWorking方法阻塞自己，等待工头调用doTask方法。

**职介所**就主要负责对工人的状态进行管理，知道什么时候有哪些工人空闲，那些挂掉了。



在这个架构中，我们可以看到，包工头是负责将数据流信息和控制流信息传给各个工人，这里感觉是会存在单点瓶颈的(不确定的小声逼逼)。而工人和工头之间的交互模式从上图看是直接交互的，但是fourinone还支持第二种交互方式，基于**消息中枢**的交互。

![1547991555518](.\picture\1547991555518.png)

这个不是默认的模式，也不是fourinone的推荐模式。这个模式建议是在工头和工人不在同一个局域网内，不能直接访问的情况下使用。这种模式主要有以下的弊端：

1.因为工人没有直接和工头接触，很多直接交互的优化框架是不支持的。

2.因为消息中枢是集中式的访问的，所以单点故障大家就一起爆炸。

3.不能支持工人之间的交互。如果支持工人之间的交互，那么这个队列的空间复杂度将会是平方级别的，这是不可取的。

所以不推荐这个啦。下面这个就是默认的直接交互，而这个就可以支持工人之间交互，提高了编程的自由度：

![1547992064694](.\picture\1547992064694.png)

而在这里工人和工人之间的交互用的API不是和工人与工头交互的API，这样就避免了一些潜在的死锁问题。下面是核心的交互的API。可以看到工头操作的工人是WorkLocal类型的，工人操作工人是Workman类型的。

![20190120215635](.\picture\20190120215635.png)

## 