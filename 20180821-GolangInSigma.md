---
title: "Golang 在阿里集团调度&集群管理系统 Sigma 中的实践"
date: 2018-08-21
tags: 
  - Sigma
  - Golang
categories: Sigma
---
阿里巴巴 9 年双 11 经历下来，交易额增长了 280 倍、交易峰值增长 800 多倍、系统数呈现爆发式增长。系统在支撑双 11 过程中的复杂度和支撑难度以指数级形式上升。双 11 峰值的本质是用有限的成本最大化提升用户体验和集群吞吐能力，用合理的代价解决峰值。始于 2011 年建设的调度集&群管理系统 Sigma 就是为了支撑如此庞大的体系而建立的。

本文作者李雨前（花名鹰缘），阿里巴巴技术专家，2015 年开始加入调度系统建设，参与和推动调度系统的多个版本的演进以及支持双 11 大促资源的分配和管理。
<!-- more -->
![image.png | left | 600x320.3187250996016](https://cdn.nlark.com/lark/0/2018/png/115896/1534762724285-bd27c1c7-b77d-40b2-bc78-e62c30b758e0.png "")

Sigma 有 Alikenel、SigmaSlave、SigmaMaster 三层大脑联动协作，Alikenel 部署在每一台物理机上，对内核进行增强，在资源分配、时间片分配上进行灵活的按优先级和策略调整，对任务的时延，任务时间片的抢占、不合理抢占的驱逐都能通过上层的规则配置自行决策。SigmaSlave 可以在本机进行容器 CPU 分配、应急场景处理等。通过本机 Slave 对时延敏感任务的干扰快速做出决策和响应，避免因全局决策处理时间长带来的业务损失。SigmaMaster 是一个最强的中心大脑，可以统揽全局，为大量物理机的容器部署进行资源调度分配和算法优化决策。

整个架构是面向终态的设计理念，收到请求后把数据存储到持久化存储层，调度器识别调度需求分配资源位置，Slave识别状态变化推进本地分配部署。系统整体的协调性和最终一致性非常好。我们在 2011 年开始做调度系统，2016 年用 Go 语言重写，2017 年兼容了 kubernetes API，希望结合生态的力量，共同建设和发展。

![幻灯片02.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762268289-cb593529-2672-4939-9d7c-58dfd0d530b6.png "")

## Sigma 总览

![幻灯片03.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534761854401-64836641-71f8-435d-9554-6e270d618c97.png "")

Sigma 的业务有两块，一块是对内，一块是对外，对内是所有的 BU 都接入了Sigma 的系统，我们的规模会有百万级别。Sigma 的内部有一个 logo，很容易理解，就是数学求和，Sigma 是很大的生态系统，需要很多人和系统配合完成。

这个业务分了几个层次，上层业务偏运维，下层业务偏系统，从接入层，到中心的 Master，到Slave，到最底会有一个 PouchContainer（阿里开源的富容器引擎）。

这是 

![幻灯片04.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534761876777-9cf0c78d-6486-4de6-94b3-374fe6085426.png "")

## Sigma 的架构


可以看出来，颜色有三块，一块是 Sigma 的，一个是 0 层，还有一块是关于 Fuxi 的，通俗的理解就是 Sigma 管在线，然后是 Fuxi 管离线的，中间是一个协调层，这样理解起来就跟 Mesos 比较接近一点。这个架构肯定不是最优的，但是它存在有它客观的原因。

我会从 Sigma 这边抽象的四个案例。我会给大家展示是什么的同时，讲背后我们的故事。

### 案例1

首先看一下 APIServer


![幻灯片05.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534761894300-d0b82d73-0af9-421e-be71-eba78d5f0bd1.png "")


我们平常写代码经常会接触到这些事情，在业务领域、调度领域稍稍不同，背后要做发布、扩容、销毁、启停、升级，还有云化，特别是双十一到阿里云买服务器，所以会有混合云需求。我们规模很大，就会要求所有简单的事情，怎么样让它在规模化的场景下面依然能够工作。我们调度系统跟运维有关，核心的内容就是怎么样做到运维友好。还有就是我们做在线的容器服务，肯定要做到高可用，还有一致性。

解决思路

1.数据一致性

数据的一致性上面我们是用 etcd/redis，我们会用一个实时+全量的方式做到数据的一致性

2.状态的一致性

想做到很好是很难的，但是我们把状态的一致性转为存储一致性来做，就会降低处理问题的难度。

3.简单的

我们没有追求技术看起来非常完美的方案，先把业务推起来能够用就好了。

4.高可用-无状态

我们要做到前面说的高可用，有几种方案，一个是多 Master 结构，还有无状态，还有就是快速的 failover，我们希望做到无状态。

5.降级-抢占

规模大了以后就会有一个问题，很多人都要资源，这个时候肯定会有一些稀缺的资源，系统要支持降级抢占。

6.内外兼容：一个团队两块牌子

要做到上云，所有的思考都要考虑到对内对外是一班人马，一个团队会有两个牌子，这是我们整个出发点的思路。

之后我们选择由 APIServer 把前面的发布、扩容等都放在 task，丢到 Redis 里，底层的 Worker 消费一些任务，做到无状态，得出来的结论就是架构整体的设计大于语言的选择。

数据一致性里面，如果能做到了实时，数据应该就是一致性的，为什么还要加全量来弥补这个过程？背后一个重要的原因就是物理机的硬件或者是软件会经常出故障，系统多了之后，很难保证故障的事件跟整个链路 100 闭环起来，就导致总有一些地方的数据不知道为什么不一致，有很多硬件、软件的故障导致实践的层面做不到一致性，就只好加一个全量弥补。还有另外一种方案不要全量，我们定时同步，但是有一个问题，时间窗口怎么定，定几分钟。同步的一瞬间，数据会有一些比对，这个时候加锁，成本维护会多一点。

状态一致性问题转为存储一致性的问题最好的就是转为存储一致性，我们为什么说要面向 etcd，在很大的系统里面有很多的设备系统，没有办法面向事务去做，最好是分布式的存储把这些问题兜掉。

为什么说简单够用，后面会有数据给大家看一下，要承载的量很大，没有办法做到 95%以上才上线，大家等不及，可能做到 85%就要上线了。

还有就是降级，前面我提到了阿里这么多的 BU，每一个 BU 都提了很多的预算，去采购，这就浪费很多资源，这中间有一个共享的 buffer，肯定会有抢占，这个在开源里面也会有一些策略。

内外兼容，如果是创业公司会有体会，一部分东西是基于内部的服务器，还有一些是购买上云的服务，这个时候管理起来，不可能上云的是一套管理方式，自己内部又是一套方式，至少在开发理解上是一模一样的，在设计和架构的时候要屏蔽很多的差异。基于这样的思考，我们觉得这种方式相对来说会比较好一点，不然尝试换其他的方式，问题解决可能会稍微麻烦一点。

前面讲了设计的架构，接下来讲的案例是一个真实的场景。


![幻灯片06.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534761983790-dfa0fc8b-fdd6-4940-a181-bd184f982fd1.png "")


我们在阿里要做一次发布，比如说一开始要拉镜像，就要关闭告警，可能有一些应用要下线了，还要停止应用。然后开始容器升级，还要做业务逻辑的检查，检查完之后才把告警打开。不管是上云还是内部，这套流程肯定是公共的，具体对接的时候又不一样，内部和云上不一样，云上提供一个 SDK，云上的接口的异步性的流转和内部也不一样，所以要抽出来。

内部和阿里云上面都是这样的事情，阿里云比较特别的是什么呢？在 before 会做一些处理，我们核心的架构是很固定的，在阿里主流的语言是 JAVA，我们讨论架构的时候，没有局限于上来就用 JAVA，Golang，或者别的，我们把架构想清楚，之后才选择用什么样的开发语言。

这是我们数据的表现


![幻灯片07.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762005704-686909a3-e21d-416c-95cc-c8b52938703a.png "")


这个表现是任务量，数字隐去了，发布的量非常大，平常的量比较小，可以看到它有周期性的规则，因为阿里的体量有发布窗口期，那一天整个的发布量大了，会有一个全局的管控。

### 案例2


![幻灯片08.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762040358-cf7ef19f-b27e-4457-b04e-78a1c1dbb18b.png "")

调度里面核心的模块之一是 Scheduler，在很多物理机上面选最佳的物理机，把容器布上去，常规的做法就是要有过滤链，过滤完之后会有一个权重的链，最后拿到我想要的机器上去创建这个容器。因为规模大，一个请求在一个机房里面去扫，这一个机房可能就是万级别的节点，肯定要考虑到性能的优化，所以筛选的规模比较大一点。它是一个链式的，首先要选择哪些服务器作为点，在这一块，我们的想法就是要并发起来，怎么样从工程上面做一个实践。绿色的框是顺序过程，红色的框是并发的阶段，或者是关键资源的锁的过程。每一个并发的粒度看成是一个物理机，相当于每一个物理机去筛选这个是否要，这是一种模式。


![幻灯片09.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762058674-3eda765a-ce05-4fe2-8c98-e899261b8c5d.png "")


另外一种模式是说在整个机器的顶层加一个 glock，是顺序还是并行都不用管，这个锁加在上面，前面的锁是加在下面，两者看起来好像没有太大的区别，测试下来发现性能不一样。有几个因素，一个是下面要做过滤链或者做 weight，看开销；还有每一次筛选的规模，比如一天下来有十万的请求，每个请求背景物理机是好几万的规模还是几百的规模，最后对整体的性能要求是不一样的。




![幻灯片10.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762075590-1352eec6-2845-49b5-bac8-ae6721349c97.png "")


粗粒度并发的性能比较好，我们选择了第二种场景。深入的思考的时候，我们发现第一种场景行不通，原因是有些全局的资源没有在一个协程里面更改，另外一个协程不能立即可见。

### 案例3



![幻灯片11.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762088548-2b271e28-c759-4617-95d7-ca63470bd5a9.png "")


Golang 最近几年非常的火爆，但是在阿里大的氛围里面更强调的是 JAVA 语言，我们引入 Golang 不可能一上来就大规模，需要有一个成功的案例，或者是小规模实践的过程。在这种环境下面，我们想让 Golang 有一席之地，首选的方式就是如何做到快速的打磨，跑得很慢的时候，语言构架系统可能就会被淘汰掉了。这是我们上一个版本，有大概五个链路，有一个架子，每一条链路是做的过程当中，一步一步完善出来。今天我们把链路搬出来跟 K8S 做比较的时候，很多地方都是相通的，但是具体框架的编码实践上面来说是有很大的差异。我们早些时候摸索的这些东西，可能就是业务驱动或者概念驱动，没有真正做到工程或是回馈社区的驱动。未来我们换了一种方式，我们可能是以工程的方式驱动，就是回馈社区。

### 案例4



![幻灯片12.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762106924-8e3bb0d7-8298-4ea0-96a0-359c6585f0cd.png "")

我讲一下怎么解读这个图。分几层，上面这一层更强调的怎么样编排任务，中间的这一个层是讲整个容器的；纵向有三个部分，最左边是讲怎么样调度的，中间是一个容器的引擎，再后面是容器的运行时。




![幻灯片13.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762119885-5c3c0fbc-fc9f-404c-9ce5-e147e9b75f65.png "")



在 PouchContainer 里面，官方写了很多的 Features，这些 Features 是源于阿里真实的实践。为什么叫富容器，容器经典的代表大家可能想到 Docker，那 Docker 之前呢？比如阿里的虚拟机比较流行，从虚拟机过渡到容器，这么大的规模，需要适应运维的习惯，要有这样的感知、理念，这个时候容器的技术肯定要做很厚。然后再编排，现在我们了解的有 K8S，大家觉得很完美，但是那在 K8S 之前是什么，阿里的体量并不是 2015 年那一刻才长大的，2015 年之前就很大了。

我们有一个强隔离，为什么说强隔离？我们平时和别人探讨问题的思路有两种思路，第一种思路是一上去就排查，把问题解决掉。还有就看自己的系统，拼命的证明不是我的问题。在这个时候你强隔离就好说了，问题排查或者是黑盒子，特别是规模很大的时候就需要隔离很强，每个人在我的领域范围内很容易定位。从去年到今年，阿里做了很多混部的宣传，混部没有强隔离也有问题的。

还有就是为什么要引入 P2P？最早 P2P 是在流媒体里面，为什么又跟容器关联起来了，就是因为互联网里面有很强的思维，如果你慢的话，别人就会忍无可忍的。量少的话比工作更快一点。但是规模大的时候没有办法快，在链路上面来，包括业界也是一样，连路瓶颈已经在拉镜像，由此阿里很自然的就推了 P2P 加速。

还有内核的兼容，外界有一个说法——CTO 说阿里的商业成功，掩盖了阿里的技术成功，这个话确实有道理。有些业务我们是在 2011 年的时候拿到 2.6 版本，现在业务最新的到了 4.10，那些老的业务每天服务的人群量很小，不能说这个业务不赚钱或者没有前景。把它下掉了也不行，要给一个缓冲期，该升级了，这个没有办法一步到位。这个时候要做规模化的升级，或者技术的换代，内核必须要做兼容。

前面的内容只是把阿里巴巴自己的问题解决了，并没有把这些赋能给社区，所以要做标准化的思考，这个是为了未来把好东西回馈社区，所以必须要做标准化的兼容，再反过头来比这张图，就知道这个架构怎么思考，为什么要兼容 CRI。

### 代码1

前面我说最难调的 bug 是低级错误造成的，这个低级错误什么意思呢？



![幻灯片14.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762141011-5fbe58ca-3907-474b-8704-bcc7024e777c.png "")




Golang 一开始很多人用 map 循环的时候，就受到了指针变量的误用，典型的表现就是，一个对象循环之后就会发现中间的所有的值一模一样，大家首先想到的是业务逻辑是不是不对，根本不会想到 for 循环是不是出了问题。后来我们发现我们对语言本身的理解还不是特别的到位，犯了低级的错误，这个 PPT 下面有详细的代码案例。

### 代码2



![幻灯片15.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762151785-c4db2f0f-d232-4e7a-a982-2300a5970147.png "")



还有map对象的异步序列化，现在看来本质上是我们用法不对，当时理解跟业务场景有关，我们每次筛选服务器有上万的规模，为什么选择这条服务器，要用日志把它记下来。后面的优化、迭代要进行消息分析。我们就想到异步做这个事情，把对象存在异步里面去刷盘。但是我们犯了致命的错误，就是Goroute对同一个map执行了读写并发，这样就出现了map读写的冲突。如果知道对象是共享的资源，我们就会加锁，但是这种场景下面我们没有考虑到这个资源也会导致问题。业务场景，特别是异步对象持久化的，要有这样一个意识，规模大的时候要考虑这样一个问题。

### 代码3


![幻灯片16.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762206743-e5c117f5-2f15-4ec9-a1b4-de9a7e6cd8dd.png "")



我们做的过程当中发现有一个资源泄露的问题，是场景导致的，我们起了很多 Goroute，我们有个主任务，主任务会起很多子任务，子任务做一些循环的操作。到阿里云买服务器，阿里云是异步接口，现在起很多任务去买服务器，买完之后把请求发货去，它返还我一个异步的 ID，我再请求他状态执行结果。比如说我先请求，完了之后它分配我一些资源，启动这些资源。比如发起申请，然后 start，看看这个 start 是不是完成，start 也是有一个过程。比如阿里 90 分钟再建一个淘宝，要拿到资源，肯定要很快。

我们为什么会遇到泄露呢？我们主任务要申请 200 个实例，我会发起 200 或 50 个并发的请求到阿里云，但是不可能等 200 个请求全部完成了，200 个来了 15 个就已经向业务交付，用户的体验就很好。但是这种滚动式的交付，不可能无休止的等待，你得一分钟之内完成。

除了滚动式的交付，还有总体任务时间的控制，这就涉及到两级的 Goroute，第一级 Goroute 是总任务，子任务也是 Goroute,这就会涉及到泄露的问题。假设主任务超时了，就不管子任务，子任务一直在跑就会把资源耗完。

如果大家对 K8S 里面的代码，关于定时任务的框架有所了解的话，就会有更强的体会。当时我们如果看到这个的话就会借鉴他的方式，就不会采取我们自己造轮子的方式。这个案例最重要的就是以后写 Goroute 的时候要注意是不是有泄露。

### 代码4



![幻灯片17.png | center | 747x420](https://cdn.nlark.com/lark/0/2018/png/115896/1534762230431-2d1506e2-ad73-4c6c-a714-711e0ecac5bb.png "")

我们为什么要做 Pod 打散？Pod 概念在阿里有几级，第一级是物理层级，第二级是机框打散，往上还有核心交换机的打散。比如我有两台服务器，其中有一台服务器挂掉以后，可用性一下子降到了 50%，其实创业公司还是可以接受的。但是在阿里，如果今天挂了 1%，就有非常多人投诉，所以要求非常高，所以哪怕是 1%的流量受到损失了，客服的量就一下子多很多。硬件故障和软件故障是天然的，我们无法规避怎么办呢？所有的服务不要扎堆，物理机，核心交换机上面也不要扎堆，这就是 Pod 打散。

Pod打散背后还有一层思考，比如说这台机器已经有了相同的 Pod，另外一台 Pod 一定不会往里面部署，有很强的隔离，要么是 0，要么是 1，这个打散是强制的。但是这里讲的 Pod 打散不是强制的，是尽力交付。默认情况下尽可能地打散，如果打散不了，只能在一起，但是一旦有打散的机会，一定给你打散，我们的打散是在得分之后再做的事情，这个跟 K8S 不一样，K8S 是强硬的，要打散就一定打散，要不然交付不成功。

阿里的服务量很大，服务器各个地方都有，其实它可以有一定的容忍度。假设有一万个实例，可以允许 1%的实例有波动偏差，这个时候 Pod 就可以满足它。之所以能够接受就是因为它的体量太大了，有扛波动的能力，这个跟我们接触到的 K8S 完全不一样，这是规模化的视角看到的东西。

## 3.总结

设计层面：整体架构层面优先语言选择

性能层面：任务粒度选择

数据驱动：状态一致性转移为存储一致性

语言理解：Map 异步序列化，Map 循环指针

多层并发：可控超时

调度算法：规模下 pod 打散

Pouch Container：拥抱开源，回归社区

## 【提问环节】

提问1：我想请教一下，你是利用 redis 实现数据一致性，数据一致性代替状态移植，这个什么意思？

李雨前：比如说在 K8S 里面是面向结果交付的，我要三个实例我把数字改了你就可以扩出来，但是在 K8S 没有出来之前，阿里已经存在了这些需求，那个时候是面向过程的，什么意思呢？我要拉镜像，我要关报警，这一系列的动作从容器交付周期来说，就是事务的一部分，这个时候是有状态的。比如说，你想一种方式是每一个状态都带一个 ID，一连串的串回去，不可能是并发的。每一个状态都是并发的往前做，到一个阶段再并发做另外一个阶段的事情，这个我们理解为是有状态的，这些状态我们做的时候，做不出来很好效果。

为什么要放在 redis，我们做之前尝试过一点一点去做，最后我们怎么做呢？每一个状态做完了就丢到 redis 里面去。下一个任务做的时候就知道上一个任务是什么，只要状态满足了就做下一个事情，最后你后面写很多的任务，做的事情就很简单，上一个状态任务是什么就完了，中间做什么事情，也不需要看到全局的交互关系，这个就放到存储，包括维护的一致性。


提问2：其实S igma 的思路好像跟 K8S 有点像，我不知道 Sigma 有没有像 K8S 这种封装吗？是怎么暴露服务的？

李雨前：你觉得他们两个很像，那说明你对 K8S 或者 Sigma 有不错的理解。实际上包括 Sigma，包括 mesos，包括 Omega,包括 yarn，这些东西你比较的时候，很多地方有相似之处，从整体上架构来理解，他们确实是相似的，但是我前面提到了Sigma 肯定是在 K8S 出来之前，那个时候就有自己演进的思路和方式，解决的问题都是一样的。第二个问题，我们有没有社区里面大家看到的方式方法，比如说前面讲的 PouchContainer 最后一项就是标准化的兼容，这已经反映了我们跟社区一些好的东西已经要开始吸收，把社区好的理念拿到阿里的场景进行实践，实践好了就回馈给社区，从这社区里面学到的东西有很多很多，我们也努力向社区做一些反馈。


提问3：Sigma 底层可以用 PouchContainer，Docker 这种吗？

李雨前：可以的，前面我说了现在出了一个 CRI，那个已经兼容了，实现了 CRI 的协议，你用起来是一样的，你把 PouchContainer 起来的话，你用 Remoteruntime 配一下就可以了，我推荐你看一下 PouchContainer 官方文档，里面有一些案例。
