# 6 配置并使能虚拟中断控制器

> 内核版本：linux-5.9
>
> 架构：arm64

本章分析虚拟中断控制器的配置和使能，中断控制器为VGICv2。在KVM实现的虚拟机管理器中，虚拟中断依赖硬件GIC对虚拟化扩展的支持。一个支持虚拟化扩展的GIC示意图如图6.1所示：

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption><p>图6.1 GIC逻辑分区示意图</p></figcaption></figure>

图6.1选自《IHI0048B\_b\_gic\_architecture\_specification》的第2章。关于对图6.1的介绍，建议阅读规范的第2章内容，大概十页的内容，可以对GICv2有个整体的认识。本文在图6.1中需要注意的是Virtual interface control、CPU interface和Distributor。其中Distributor首先接收到中断进行分发。Virtual interface control和CPU interface对于每一个Processor都有一对，负责对中断进行虚拟化，分发给不同的虚拟机，并共同维护虚拟中断的状态。

GIC硬件和虚拟机管理器软件共同实现虚拟中断的示意图如图6.2所示：

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption><p>图6.2 支持虚拟化的ARM处实现的GIC</p></figcaption></figure>

图6.2选自《IHI0048B\_b\_gic\_architecture\_specification》的第5章。图6.2中具体的内容可查看手册的第5章，本文不作详细分析。在图6.2中可以看到，当一个硬件中断发生后，首先到达Distributor。对于可以进行虚拟化的中断，会被传送到Hypervisor（软件），Hypervisor和Virtual interface control中的列表寄存器（List Register）协同完成将中断发送到Virtual CPU interface。Virtual CPU interface收到中断后，在条件允许的情况下就会将中断注入到虚拟机中。这里的条件与普通中断发送到处理器的条件相同，具体可查看手册。

从图6.2中可以看出，要实现虚拟中断需要硬件和软件的配合。描述一个中断的开始到结束，使用中断的状态机较为合适，在GICv2中，中断的状态机如图6.3所示：

<figure><img src=".gitbook/assets/image (5).png" alt=""><figcaption><p>图6.3 GICv2中断状态机</p></figcaption></figure>

图6.3选自《IHI0048B\_b\_gic\_architecture\_specification》的第3.2.4节，具体的状态转换推荐阅读规范。如图6.3，一个中断发生后，直到该中断的状态回到Inactive时，才可以算该中断结束。在最后中断结束步骤中有两个动作：优先级下降（Priority drop）和中断停用（Interrupt deactivation）。

通常，在不开启虚拟化的场景下，结束一个中断只需要处理器写入中断结束寄存器（GICC\_EOIR）即可，也就是说处理器写入中断结束寄存器后就完了优先级下降和中断停用。但是在开启了虚拟化的场景下，会将优先级下降和中断停用的步骤分开：通过写入GICC\_EOIR完成优先级下降，通过写入GICC\_DIR完成中断停用。

这个配合中很重要的一个流程是Hyperivor需要先接管中断处理，然后将中断注入给需要的虚拟机，虚拟机内处理完中断后，中断才能结束。

> 只需要处理器写入GICC\_EOIR（中断结束寄存器）就能结束中断也需要配置，如何配置本文不作介绍。
