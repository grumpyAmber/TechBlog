---
title: "深度解析 PouchContainer 的富容器技术"
date: 2018-08-29
tags: 
  - PouchContainer
  - RichContainer
categories: PouchContainer
---

PouchContainer 是阿里巴巴集团开源的高效、轻量级企业级富容器引擎技术，拥有隔离性强、可移植性高、资源占用少等特性。可以帮助企业快速实现存量业务容器化，同时提高超大规模下数据中心的物理资源利用率。

PouchContainer 源自阿里巴巴内部场景，诞生初期，在如何为互联网应用保驾护航方面，倾尽了阿里巴巴工程师们的设计心血。PouchContainer 的强隔离、富容器等技术特性是最好的证明。在阿里巴巴的体量规模下，PouchContainer 对业务的支撑得到双 11 史无前例的检验，开源之后，阿里容器成为一项普惠技术，定位于「助力企业快速实现存量业务容器化」。


初次接触容器技术时，阿里巴巴内部有着惊人规模的存量业务，如何通过技术快速容器化存量业务，是阿里容器技术当年在内部铺开时的重点难题。发展到今天，开源容器技术逐渐普及，面对落地，相信不少存在大量存量业务的企业，同样为这些业务的如何容器化而犯愁。云原生领域，CNCF 基金会推崇的众多先进理念，绝大多数都建立在业务容器化的基础之上。倘若企业业务在云原生的入口容器化方面没有踩准步点，后续的容器编排、Service Mesh 等行业开源技术红利更是无从谈起。

通过七年的实践经验，阿里巴巴容器技术 PouchContainer 用事实向行业传递这样的信息 —— 富容器是实现企业存量业务快速容器化的首选技术。

## 什么是富容器

富容器是企业打包业务应用、实现业务容器化过程中，采用的一种容器模式。此模式可以帮助企业IT技术人员打包业务应用时，几乎不费吹灰之力。通过富容器技术打包的业务应用可以达到以下两个目的：

* 容器镜像实现业务的快速交付
* 容器环境兼容企业原有运维体系

技术角度而言，富容器提供了有效路径，帮助业务在单个容器镜像中除了业务应用本身之外，还打包更多业务所需的运维套件、系统服务等；同时相比于较为简单的单进程容器，富容器在进程组织结构层面，也有着巨大的变革：容器运行时内部自动运行 systemd 等管家进程。如此一来，富容器模式下的应用，有能力在不改变任何业务代码、运维代码的情况下，像在物理机上运行一模一样。可以说，这是一种更为通用的「面向应用」的模式。

换言之，富容器在保障业务交付效率的同时，在开发和运维层面对应用没有任何的侵入性，从而有能力帮助 IT 人员更多聚焦业务创新。

## 适用场景

富容器的适用场景极广。可以说企业几乎所有的存量业务，都可以采纳富容器作为容器化方案首选。容器技术流行之前，有接近二十年的时间，企业 IT 服务运行在裸金属或者虚拟机中。企业业务的稳定运行，有非常大的功劳来源于运维工作，如果细分，包括「基础设施运维」以及「业务运维」。所有的应用运行，都依赖于物理资源；所有的业务稳定，都仰仗于监控系统、日志服务等运维体系。那么，我们有理由相信，在业务容器化过程中，企业坚决不能对运维体系置之不理，否则后果可想而知。

因此，存量业务容器化过程中，需要考虑兼容企业原有运维体系的场景，都在 PouchContainer 富容器技术的使用范围之内。

## 富容器技术实现

既然可以业务兼容原有运维体系，那么富容器技术又是通过什么样的技术来实现的呢？下图清晰的描述了富容器技术的内部情况。


![image.png | center | 747x368](https://cdn.yuque.com/lark/0/2018/png/65333/1526059474453-02e322a1-f33f-4de2-a238-26e0501b3106.png "")


富容器技术可以完全百分百兼容社区的 OCI 镜像，容器启动时将镜像的文件系统作为容器的 rootfs。运行模式上，功能层面，除了内部运行进程，同时还包括容器启停时的钩子方法（prestart hook 和 poststop hook）。

### 富容器内部运行进程

如果从内部运行进程的角度来看待 PouchContainer 的富容器技术，我们可以把内部运行进程分为 4 类：

* pid=1 的 init 进程
* 容器镜像的 CMD
* 容器内部的系统 service 进程
* 用户自定义运维组件

#### pid=1 的 init 进程

富容器技术与传统容器最明显的差异点，即容器内部运行一个 init 进程，而传统的容器（如 docker 容器等）将容器镜像中指定的 CMD 作为容器内 pid=1 的进程。PouchContainer 的富容器模式可以运行从三种 init 进程中选择：

* systemd
* sbin/init
* dumb-init

众所周知，传统容器作为一个独立运行环境，内部进程的管理存在一定的弊端：比如无法回收僵尸进程，导致容器消耗太多进程数、消耗额外内存等；比如无法友好管理容器内部的系统服务进程，导致一些业务应用所需要的基本能力欠缺等，比如 cron 系统服务、syslogd 系统服务等；比如，无法支持一些系统应用的正常运行，主要原因是某些系统应用需要调用 systemd 来安装 RPM 包……

富容器的 init 进程在运维模式上，毫无疑问可以解决以上问题，给应用带来更好的体验。init 进程在设计时就加入了可以 wait 消亡进程的能力，即可以轻松解决上图中业务进程运行过程中诞生的 Zombie 僵尸进程；同时管理系统服务也是它的本职工作之一。如果一来，一些最为基本的传统运维能力，init 进程即帮助用户解决了大半，为运维体系做好了坚实的基础。

#### 容器镜像的CMD

容器镜像的 CMD，也就是传统意义上我们希望在容器内部运行的业务。比如，用户在容器化一个 Golang 的业务系统打包成镜像时，肯定会在 Dockerfile 中将该业务系统的启动命令指定为 CMD，从而保证未来通过该镜像运行容器起，会执行这条 CMD 命令运行业务系统。

当然，容器镜像的 CMD 代表业务应用，是整个富容器的核心部分，所有的运维适配都是为了保障业务应用更加稳定的运行。

#### 容器内系统 service 进程

服务器编程发展了数十年，很多的业务系统开发模式均基于裸金属上的 Linux 操作系统，或者虚拟化环境的下的 Linux 环境。长此以往，很多业务应用的开发范式，会非常频繁地与系统服务进程交互。比如，使用 Java 编程语言编写的应用程序，很有可能通过 log4j 来配置日志的管理方式，也可以通过<span data-type="color" style="color:rgb(0, 0, 0)"> </span>log4j.properties 配置把应用日志重定向到运行环境中的 syslogd，倘若应用运行环境中没有 syslogd 的运行，则极有可能影响业务的启动运行；再比如，业务应用需要通过 crond 来管理业务需要的周期性任务，倘若应用运行环境中没有 crond 系统守护进程，业务应用也就不可能通过 crontab 来配置周期任务；再比如，容器内部的 sshd 系统服务系统，可以快速帮助运维工程师快速进度应用运行现场，定位并解决问题等。

PouchContainer 的富容器模式，考虑到了行业大量有需求和系统服务交付的应用，富容器内部的 init 进程有能力非常方面的原生管理多种系统服务进程。

#### 用户自定义运维组件

系统服务的存在可以辅助业务的正常运行，但是很多情况下这还不够，企业自身针对基础设施以及应用配备的运维组件，同时起到为业务保驾护航的作用。比如，企业运维团队需要统一化的为业务应用贴近配置监控组件；运维团队必须通过自定义的日志 agent 来管理容器内部的应用日志；运维团队需要自定义自己的基础运维工具，以便要求应用运行环境符合内部的审计要求等。

正因为富容器内部存在 init 进程，用户自定义的运维组件，可以如往常健康稳定的运行，提供运维能力。

### 富容器启停执行 hook

最终富容器内部运行的任务进程，可以保障应用的运行时稳定正常，然而对于运维团队而言，负责内容的范畴往往要比单一的运行时广得多。通俗而言，运维的职责还需要覆盖运行时之前的环境准备工作，以及运行时结束后的善后工作。对于应用而言，也就是我们通常意义上提到的 prestart hook 以及  poststop hook。

PouchContainer 的富容器模式，可以允许用户非常方便的指定应用的启停执行 hook： prestart hook 以及 poststop hook。 运维团队指定 prestart hook，可以帮助应用在运行之前，在容器内部做符合运维需求的一些初始化操作，比如：初始化网络路由表、获取应用执行权限、下载运行时所需的证书等。运维团队指定 poststop hook，可以帮助应用在运行结束或者异常退出之后，执行统一的善后工作，比如，对中间数据的清理以便下一次启动时的纯净环境；倘若是异常退出的话，可以即时汇报出错信息，满足运维需求等。

我们可以发现，富容器内部的启停 hook，对容器的运维能力又做了一层拔高，大大释放了运维团队对应用的灵活管理能力。

## 总结

经过阿里巴巴内部大量业务的锤炼，PouchContainer 已经帮助超大体量的互联网公司实现了所有在线业务的容器化。毫无疑问，富容器技术是最为实用、对应用开发以及应用运维没有任何侵入性的一项技术。开源的PouchContainer 更是希望技术可以普惠行业，帮助大量的企业在存量业务的容器化方面，赢得自己的时间，快速拥抱云原生技术，大步迈向数字化转型。


