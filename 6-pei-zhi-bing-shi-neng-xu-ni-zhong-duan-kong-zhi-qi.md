# 6 配置并使能虚拟中断控制器

> 内核版本：linux-5.9
>
> 架构：arm64

### 6.1 VGICv2概述

本章分析虚拟中断控制器的配置和使能，中断控制器为VGICv2。在KVM实现的虚拟机管理器中，虚拟中断依赖硬件GIC对虚拟化扩展的支持。一个支持虚拟化扩展的GIC示意图如图6.1所示：

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption><p>图6.1 GIC逻辑分区示意图</p></figcaption></figure>

图6.1选自《IHI0048B\_b\_gic\_architecture\_specification》的第2章。关于对图6.1的介绍，建议阅读规范的第2章内容，大概十页的内容，可以对GICv2有个整体的认识。本文在图6.1中需要注意的是Virtual interface control、CPU interface和Distributor。其中Distributor首先接收到中断进行分发。Virtual interface control和CPU interface对于每一个Processor都有一对，负责对中断进行虚拟化，分发给不同的虚拟机，并共同维护虚拟中断的状态。

GIC硬件和虚拟机管理器软件共同实现虚拟中断的示意图如图6.2所示：

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption><p>图6.2 支持虚拟化的ARM处实现的GIC</p></figcaption></figure>

图6.2选自《IHI0048B\_b\_gic\_architecture\_specification》的第5章。图6.2中具体的内容可查看手册的第5章，本文不作详细分析。在图6.2中可以看到，当一个硬件中断发生后，首先到达Distributor。对于可以进行虚拟化的中断，会被传送到Hypervisor（软件），Hypervisor和Virtual interface control中的列表寄存器（List Register）协同完成将中断发送到Virtual CPU interface。Virtual CPU interface收到中断后，在条件允许的情况下就会将中断注入到虚拟机中。这里的条件与普通中断发送到处理器的条件相同，具体可查看手册。

从图6.2中可以看出，要实现虚拟中断需要硬件和软件的配合。一个中断结束时，需要执行两个动作：优先级下降（Priority drop）和中断停用（Interrupt deactivation）。这两个动作需要软硬件协同完成。

通常，在**不开启虚拟化的场景**下，结束一个中断只需要处理器写入中断结束寄存器（GICC\_EOIR）即可，也就是说处理器写入中断结束寄存器后就完了优先级下降和中断停用。但是在**开启了虚拟化的场景**下，会将优先级下降和中断停用的步骤分开：通过写入GICC\_EOIR完成优先级下降，通过写入GICC\_DIR完成中断停用。引用GICv2手册中的内容：这意味着管理程序（Hypervisor）对GICC\_EOIR寄存器的访问会降低CPU接口的运行优先级，但不会停用中断。在写入EOI寄存器后，CPU接口上的运行优先级降低，从而允许后续的中断信号被发送到处理器。这样的配置提高了灵活性。

这个配合中很重要的一个流程是Hyperivor需要先接管中断处理，然后将中断注入给需要的虚拟机。在kvm的场景下，一个物理中断到达后，host linux首先写入GICC\_EOIR寄存器完成优先级下降，然后进行中断注入相关工作，最后写入GICC\_DIR寄存器完成中断停用。

> 这两种配置取决于GICC\_CTLR.EOImodeNS位为0还是1。
>
> 在手册中，对于Hypervior将GICC\_CTLR.EOImodeNS设置为1，GICV\_CTLR.EOImodeNS设置为0的场景，ARM推荐Hypervisor执行优先级下降，虚拟机写入GICV\_EOIR寄存器执行中断停用。但是笔者看代码host linux写入了GICC\_EOIR和GICC\_DIR。在Qemu中经过测试，对于虚拟时钟中断，host linux是否写入GICC\_DIR都可以正常运行虚拟机。

关于GICv2硬件目前先介绍这些内容，其他的硬件相关内容到具体情境中再进行介绍。

### 6.2 VGICv2探测

### 6.3 创建VGICv2设备

> 未完待续\~
