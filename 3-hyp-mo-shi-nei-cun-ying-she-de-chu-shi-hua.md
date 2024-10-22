# 3 hyp模式内存映射的初始化

> 内核版本：linux-5.9
>
> 架构：arm64

本文作如下**约束**：未开启vhe。

本文作如下**规定**：在EL1运行linux宿主机时为host模式，在EL1运行客户机时为guest模式，在EL2运行时为hyp模式。

### 3.1 hyp模式下的页表以及恒等映射

通过前文描述可知，在kvm的场景下，有部分代码是需要在EL2异常级别运行。arm64架构中每个异常级别有自己的运行环境，即每个异常级别有自己的系统寄存器控制当前的运行环境。在运行环境中，对内存的设置至关重要，例如当前异常级别是否开启虚拟内存。

这里涉及arm64架构的规范，对初始化相关的内容作简要说明。EL1异常级别和EL2异常级别有各自的系统控制寄存器，即`SCTLR_EL1`和`SCTLR_EL2`，这两个寄存器的第0位是否开启MMU。未开启MMU则访存以物理内存直接寻址，开启后访存则需要经过MMU转换。EL1的设置本文不进行讲解，着重讲解EL2的设置，当EL2开启MMU后，转换所需要的转换表存储在`TTBR0_EL2`寄存器中。

> 这里注意区分后文会提到的控制阶段二地址转换的`VTCR_EL2`寄存器，以及阶段二地址转换所需的`VTTBR_EL2`寄存器。

Linux内核在启动时在`arch/arm64/kernel/head.S`文件中的`setup_el2`函数中对EL2的部分运行环境做了初始化，其中包含对`SCTLR_EL2`的设置。设置的代码如下所示：

```armasm
/* arch/arm64/kernel/head.S */
SYM_FUNC_START(el2_setup)
	......
	......
	......
1:	mov_q	x0, (SCTLR_EL2_RES1 | ENDIAN_SET_EL2)
	msr	sctlr_el2, x0
	......
	......
	......
SYM_FUNC_END(el2_setup)
```

代码中将`SCTLR_EL2`寄存器的值设置为`SCTLR_EL2_RES1 | ENDIAN_SET_EL2`，这两个宏的内容如下所示：

```c
/* arch/arm64/include/asm/sysreg.h */
/* SCTLR_EL2 specific flags. */
#define SCTLR_EL2_RES1	((BIT(4))  | (BIT(5))  | (BIT(11)) | (BIT(16)) | \
			 (BIT(18)) | (BIT(22)) | (BIT(23)) | (BIT(28)) | \
			 (BIT(29)))

#ifdef CONFIG_CPU_BIG_ENDIAN
#define ENDIAN_SET_EL2		SCTLR_ELx_EE
#else
#define ENDIAN_SET_EL2		0
#endif
```

根据这两个宏的值可知，linux启动时在EL2异常级别中设置EL2为未开启MMU（`SCTLR_EL2`寄存器的第0位为0）。后续在初始化KVM功能的时候会陷入到EL2中，将EL2的虚拟内存访问打开。开启EL2异常级别MMU的代码如下所示：

```armasm
/* arch/arm64/kvm/hyp/nvhe/hyp-init.S */
......
......
......
	phys_to_ttbr x4, x0
alternative_if ARM64_HAS_CNP
	orr	x4, x4, #TTBR_CNP_BIT
alternative_else_nop_endif
	msr	ttbr0_el2, x4
......
......
......
	mov_q	x4, (SCTLR_EL2_RES1 | (SCTLR_ELx_FLAGS & ~SCTLR_ELx_A))
CPU_BE(	orr	x4, x4, #SCTLR_ELx_EE)
alternative_if ARM64_HAS_ADDRESS_AUTH
	mov_q	x5, (SCTLR_ELx_ENIA | SCTLR_ELx_ENIB | \
		     SCTLR_ELx_ENDA | SCTLR_ELx_ENDB)
	orr	x4, x4, x5
alternative_else_nop_endif
	msr	sctlr_el2, x4
	isb

	/* Set the stack and new vectors */
	kern_hyp_va	x1
	mov	sp, x1
	msr	vbar_el2, x2
......
......
......
```

这段代码展示了hyp模式初始化时内存相关的内容，首先设置`ttbr0_el2`寄存器为运行在hyp模式下使用的页表，然后设置了`sctlr_el2`寄存器，其中包含了打开MMU。这段代码展示的内容除了打开MMU，还在打开MMU之后又展示了些代码。这里是要展示在EL2异常级别中，**开启MMU前后都有代码运行。**这里就存在一个问题：打开虚拟内存访问的代码是提前存在的，也就是说打开虚拟内存访问的代码是在未开启MMU的情况下运行的，打开之后紧跟着的代码是在开启MMU的情况下运行的，那么如何保证开启MMU前后的代码可以正常运行？

解决该问题的方式就是将这段代码进行恒等映射，Linux虚拟化相关需要进行恒等映射的代码被安排在`__hyp_idmap_text_start`到`__hyp_idmap_text_end`，可以在链接脚本中看到：

```c
/* arch/arm64/kernel/vmlinux.lds.S */
__hyp_idmap_text_start = .;			\
*(.hyp.idmap.text)				\
__hyp_idmap_text_end = .;			\
```

> 注：该文件用于生成最终使用的链接脚本。

到这里就得到了hyp模式内存映射初始化需要的内容：

1. Linux在hyp模式下使用虚拟内存，所以需要分配和设置hyp模式下使用的页表；
2. 由于打开MMU前后都有代码要运行，所以需要对这部分代码提前进行恒等映射。

### 3.2 hyp模式下的页表以及恒等映射的设置

只关注hyp模式内存映射初始化相关的内容，函数调用图如图3.1所示。

<figure><img src=".gitbook/assets/hyp模式内存映射初始化函数调用图.drawio.png" alt=""><figcaption><p>图3.1 hyp模式内存映射初始化函数调用图</p></figcaption></figure>

如图3.1所示，在`kvm_mmu_init`函数中，完成了hyp模式下使用的PGD的分配，以及需要恒等映射代码的映射。下面详细分析该函数的实现：

```c
/* arch/arm64/kvm/mmu.c */
int kvm_mmu_init(void)
{
	int err;

	hyp_idmap_start = __pa_symbol(__hyp_idmap_text_start);
	hyp_idmap_start = ALIGN_DOWN(hyp_idmap_start, PAGE_SIZE);
	hyp_idmap_end = __pa_symbol(__hyp_idmap_text_end);
	hyp_idmap_end = ALIGN(hyp_idmap_end, PAGE_SIZE);
	hyp_idmap_vector = __pa_symbol(__kvm_hyp_init);
	
	......
}
```

该函数首先获得hyp模式下需要恒等映射代码的起始地址和结束地址，并且记录了了hyp模式下初始化异常向量表的地址。

```c
/* arch/arm64/kvm/mmu.c */
int kvm_mmu_init(void)
{
	......
	
	/*
	 * We rely on the linker script to ensure at build time that the HYP
	 * init code does not cross a page boundary.
	 */
	BUG_ON((hyp_idmap_start ^ (hyp_idmap_end - 1)) & PAGE_MASK);

	kvm_debug("IDMAP page: %lx\n", hyp_idmap_start);
	kvm_debug("HYP VA range: %lx:%lx\n",
		  kern_hyp_va(PAGE_OFFSET),
		  kern_hyp_va((unsigned long)high_memory - 1));
	
	if (hyp_idmap_start >= kern_hyp_va(PAGE_OFFSET) &&
	    hyp_idmap_start <  kern_hyp_va((unsigned long)high_memory - 1) &&
	    hyp_idmap_start != (unsigned long)__hyp_idmap_text_start) {
		/*
		 * The idmap page is intersecting with the VA space,
		 * it is not safe to continue further.
		 */
		kvm_err("IDMAP intersecting with HYP VA, unable to continue\n");
		err = -EINVAL;
		goto out;
	}
	
	......

out:
	free_hyp_pgds();
	return err;
}
```

然后在需要debug时打印地址信息，然后对获取的地址进行检查，若检查失败则返回错误码。

```c
/* arch/arm64/kvm/mmu.c */
int kvm_mmu_init(void)
{
	......
	
	hyp_pgd = (pgd_t *)__get_free_pages(GFP_KERNEL | __GFP_ZERO, hyp_pgd_order);
	if (!hyp_pgd) {
		kvm_err("Hyp mode PGD not allocated\n");
		err = -ENOMEM;
		goto out;
	}
	
	......

out:
	free_hyp_pgds();
	return err;
}
```

然后分配一页作为hyp模式下使用的PGD，若分配失败则返回错误码。

```c
/* arch/arm64/kvm/mmu.c */
int kvm_mmu_init(void)
{
	......
	
	if (__kvm_cpu_uses_extended_idmap()) {
		
		......
		
	} else {
		err = kvm_map_idmap_text(hyp_pgd);
		if (err)
			goto out;
	}

	io_map_base = hyp_idmap_start;
	return 0;
out:
	free_hyp_pgds();
	return err;
}
```

当前情景不包含`__kvm_cpu_uses_extended_idmap`为真的情况，在else语句中调用`kvm_map_idmap_text`函数对需要恒等映射的代码进行映射。

`kvm_map_idmap_text`函数的代码如下：

```c
static int kvm_map_idmap_text(pgd_t *pgd)
{
	int err;

	/* Create the idmap in the boot page tables */
	err = 	__create_hyp_mappings(pgd, __kvm_idmap_ptrs_per_pgd(),
				      hyp_idmap_start, hyp_idmap_end,
				      __phys_to_pfn(hyp_idmap_start),
				      PAGE_HYP_EXEC);
	if (err)
		kvm_err("Failed to idmap %lx-%lx\n",
			hyp_idmap_start, hyp_idmap_end);

	return err;
}
```

该函数中主要调用`__create_hyp_mappings`函数进行映射，`__create_hyp_mappings`函数内容不进行详细介绍，大致内容为：在`hyp_pgd`中设置`hyp_idmap_start`内存地址到`hyp_idmap_end`内存地址的页表项。

### 3.3 hyp模式栈以及代码的映射

除了需要恒等映射的代码，在思考一个问题：编写hyp模式需要运行的代码和host模式运行的代码都是同时编译进一个可执行文件里的，那么hyp模式下如何运行到对应的代码。问题的解答比较简单：将hyp模式需要运行的代码映射进hyp模式的虚拟地址空间即可，这里就不需要恒等映射了，直接映射虚拟地址即可。并且hyp模式函数调用也要提前设置好栈空间。

不过有一个问题要注意：在arm64架构的Linux下host模式的虚拟地址，开头以0xf开始；hyp模式的虚拟地址以0x0开始。举例说明：host模式下的虚拟地址0xFFFFFFF000000000，在hyp模式下对应的虚拟地址应该为0x000000F000000000。

这部分代码在图3.1中的`init_hyp_mode`函数中。下面进行详细分析：

```c
/* arch/arm64/kvm/arm.c */
static int init_hyp_mode(void)
{
	int cpu;
	int err = 0;

	/*
	 * Allocate Hyp PGD and setup Hyp identity mapping
	 */
	err = kvm_mmu_init();
	if (err)
		goto out_err;

	/*
	 * Allocate stack pages for Hypervisor-mode
	 */
	for_each_possible_cpu(cpu) {
		unsigned long stack_page;

		stack_page = __get_free_page(GFP_KERNEL);
		if (!stack_page) {
			err = -ENOMEM;
			goto out_err;
		}

		per_cpu(kvm_arm_hyp_stack_page, cpu) = stack_page;
	}
	
	......
}
```

`kvm_mmu_init`函数中分配hyp模式的PGD，以及进行恒等映射后，为每一个CPU分配一个在hyp模式下运行的空间，并保存在`kvm_arm_hyp_stack_page`这个per-cpu变量中。

```c
/* arch/arm64/kvm/arm.c */
static int init_hyp_mode(void)
{
	......

	/*
	 * Map the Hyp-code called directly from the host
	 */
	err = create_hyp_mappings(kvm_ksym_ref(__hyp_text_start),
				  kvm_ksym_ref(__hyp_text_end), PAGE_HYP_EXEC);
	if (err) {
		kvm_err("Cannot map world-switch code\n");
		goto out_err;
	}

	err = create_hyp_mappings(kvm_ksym_ref(__start_rodata),
				  kvm_ksym_ref(__end_rodata), PAGE_HYP_RO);
	if (err) {
		kvm_err("Cannot map rodata section\n");
		goto out_err;
	}

	err = create_hyp_mappings(kvm_ksym_ref(__bss_start),
				  kvm_ksym_ref(__bss_stop), PAGE_HYP_RO);
	if (err) {
		kvm_err("Cannot map bss section\n");
		goto out_err;
	}

	......
}
```

然后将`__hyp_text_start`到`__hyp_text_end`、`__start_rodata`到`__end_rodata`、`__bss_start`到`__bss_stop`的地址空间映射到hyp模式的地址空间中。这里可以看到希望在hyp模式下运行的代码会被安排在`__hyp_text_start`到`__hyp_text_end`。

```c
/* arch/arm64/kvm/arm.c */
static int init_hyp_mode(void)
{
	......

	err = kvm_map_vectors();
	if (err) {
		kvm_err("Cannot map vectors\n");
		goto out_err;
	}

	......
}
```

当前情景不讨论arm64向量的问题，假设向量相关的配置为n，则`kvm_map_vectors`函数直接返回0。

```c
/* arch/arm64/kvm/arm.c */
static int init_hyp_mode(void)
{
	......

	/*
	 * Map the Hyp stack pages
	 */
	for_each_possible_cpu(cpu) {
		char *stack_page = (char *)per_cpu(kvm_arm_hyp_stack_page, cpu);
		err = create_hyp_mappings(stack_page, stack_page + PAGE_SIZE,
					  PAGE_HYP);

		if (err) {
			kvm_err("Cannot map hyp stack\n");
			goto out_err;
		}
	}

	for_each_possible_cpu(cpu) {
		kvm_host_data_t *cpu_data;

		cpu_data = per_cpu_ptr(&kvm_host_data, cpu);
		err = create_hyp_mappings(cpu_data, cpu_data + 1, PAGE_HYP);

		if (err) {
			kvm_err("Cannot map host CPU state: %d\n", err);
			goto out_err;
		}
	}

	......
}
```

然后将每一个CPU的栈空间映射到和hyp模式的地址空间中，随后将每一个CPU的CPU数据映射进hyp模式的地址空间中。该CPU数据用于保存CPU上下文，hyp模式下保存和恢复的上下文信息就保存在这里，这里说明hyp模式和host模式都可感知到相同的CPU上下文，该结构的作用在这里有个印象即可，后续会结合情景进行分析。

```c
/* arch/arm64/kvm/arm.c */
static int init_hyp_mode(void)
{
	......

	err = hyp_map_aux_data();
	if (err)
		kvm_err("Cannot map host auxiliary data: %d\n", err);

	return 0;

	......
}
```

当前情景不讨论CONFIG\_ARM64\_SSBD配置的情况，所以`hyp_map_aux_data`函数直接返回0。

到这里hyp模式内存映射的初始化就完成了，进入hyp模式后开启MMU前后能正常运行，并且后续在hyp模式在进行函数调用（与host代码同时编译）也可以正常运行。
