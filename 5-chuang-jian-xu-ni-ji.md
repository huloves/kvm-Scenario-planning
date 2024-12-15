# 5 创建虚拟机

> 内核版本：linux-5.9
>
> 架构：arm64

### 5.1 创建虚拟机概述

kvm提供给用户使用的接口是设备文件`/dev/kvm`。打开该文件获取文件描述符后，通过`ioctl`操作创建虚拟机，具体的操作如下：

```c
int main(int argc, char *argv[])
{
	int sys_fd, vm_fd;

	sys_fd = open(kvm->cfg.dev, O_RDWR);

	......

	vm_fd = ioctl(sys_fd, KVM_CREATE_VM, 0);
	if (vm_fd < 0) {
		pr_err("KVM_CREATE_VM ioctl");
		ret = vm_fd;
		goto err_sys_fd;
	}

err_sys_fd:
	close(sys_fd);
	return ret;
}
```

通过`KVM_CREATE_VM`命令，内核会创建kvm虚拟机实体对象，并将kvm虚拟机实体对象与一个文件描述符绑定，文件描述符会返回给用户态，用户态程序后续使用该描述符去配置和创建虚拟机内部资源。后文将详细描述`KMV_CREATE_VM`命令在内核中的流程。在分析的过程中逐渐构建内核中虚拟机的一个视图，并通过图例展示。

### 5.2 创建虚拟机内核源码分析

`KVM_CREATE_VM`命令在内核中会执行`kvm_dev_ioctl_create_vm`函数，所以后面的分析就是针对`kvm_dev_ioctl_create_vm`函数实现的分析，具体见代码：

```c
// virt/kvm/kvm_main.c
static long kvm_dev_ioctl(struct file *filp,
			  unsigned int ioctl, unsigned long arg)
{
	long r = -EINVAL;

	switch (ioctl) {
	......
	
	case KVM_CREATE_VM:
		r = kvm_dev_ioctl_create_vm(arg);
		break;
	......
	}

out:
	return r;
}
```

在`kvm_dev_ioctl_create_vm`函数中首先调用`kvm_create_vm`函数创建一个kvm对象，`kvm_create_vm`函数完成了虚拟机的创建和初始化，其中包含虚拟机内存管理、虚拟机IO总线和一些架构相关的初始化。调用`kvm_create_vm`函数的代码如下所示：

```c
// virt/kvm/kvm_main.c
static int kvm_dev_ioctl_create_vm(unsigned long type)
{
	int r;
	struct kvm *kvm;
	struct file *file;

	kvm = kvm_create_vm(type);
	if (IS_ERR(kvm))
		return PTR_ERR(kvm);

	......
}
```

#### 5.2.1 kvm\_create\_vm函数

该函数首先分配一个struct kvm结构体，该结构就对应于内核中的虚拟机（后文简称kvm、kvm虚拟机），然后初始化kvm中的部分成员，如与该kvm虚拟机关联的进程地址空间、记录kvm虚拟机中设备的链表和各种锁，做完这些之后将kvm的用户引用计数设置为1，代码如下：

```c
// virt/kvm/kvm_main.c
static struct kvm *kvm_create_vm(unsigned long type)
{
	struct kvm *kvm = kvm_arch_alloc_vm();
	int r = -ENOMEM;
	int i;

	if (!kvm)
		return ERR_PTR(-ENOMEM);

	spin_lock_init(&kvm->mmu_lock);
	mmgrab(current->mm);
	kvm->mm = current->mm;
	kvm_eventfd_init(kvm);
	mutex_init(&kvm->lock);
	mutex_init(&kvm->irq_lock);
	mutex_init(&kvm->slots_lock);
	INIT_LIST_HEAD(&kvm->devices);

	BUILD_BUG_ON(KVM_MEM_SLOTS_NUM > SHRT_MAX);

	if (init_srcu_struct(&kvm->srcu))
		goto out_err_no_srcu;
	if (init_srcu_struct(&kvm->irq_srcu))
		goto out_err_no_irq_srcu;

	refcount_set(&kvm->users_count, 1);
	
	......
}
```

在内核中锁和引用计数的重要性不言而喻，但不是本文的重点，读者记得这些锁和引用计数在该函数进行了初始化即可。这段代码中本文重点关注三个成员：**mm**、**memslots**和**devices**，其中mm和memslots和虚拟机的内存管理相关，devices和虚拟机的设备管理相关，后续通过虚拟机物理内存管理和创建设备的情景会用到这些成员。

初始化完上述内容后，会初始化和内存管理相关的**memslots**成员，以及和设备管理相关的**buses**成员，代码如下所示：

```c
// virt/kvm/kvm_main.c
static struct kvm *kvm_create_vm(unsigned long type)
{
	......
	
	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
		struct kvm_memslots *slots = kvm_alloc_memslots();

		if (!slots)
			goto out_err_no_arch_destroy_vm;
		/* Generations must be different for each address space. */
		slots->generation = i;
		rcu_assign_pointer(kvm->memslots[i], slots);
	}

	for (i = 0; i < KVM_NR_BUSES; i++) {
		rcu_assign_pointer(kvm->buses[i],
			kzalloc(sizeof(struct kvm_io_bus), GFP_KERNEL_ACCOUNT));
		if (!kvm->buses[i])
			goto out_err_no_arch_destroy_vm;
	}
	
	......
}
```

kvm\_alloc\_memslots函数的代码如下所示：

```c
// virt/kvm/kvm_main.c
static struct kvm_memslots *kvm_alloc_memslots(void)
{
	int i;
	struct kvm_memslots *slots;

	slots = kvzalloc(sizeof(struct kvm_memslots), GFP_KERNEL_ACCOUNT);
	if (!slots)
		return NULL;

	for (i = 0; i < KVM_MEM_SLOTS_NUM; i++)
		slots->id_to_index[i] = -1;

	return slots;
}
```

到目前为止，可以得到如下的kvm虚拟机视图：

<figure><img src=".gitbook/assets/内核虚拟机视图1.drawio.png" alt=""><figcaption><p>图5.1 kvm虚拟机视图1</p></figcaption></figure>

