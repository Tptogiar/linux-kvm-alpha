


linux-5.15.85
<br/>


1.

在setup_vmcs_config中配置vmcs时，KVM把external interrupt exit和nmi exiting都设置为min，也就是必须打开的，

```jsx
min = PIN_BASED_EXT_INTR_MASK | PIN_BASED_NMI_EXITING; 
```

在后面创建vCPU是的init_vmcs中也没有见到把这两个关闭。那也就是说guest跑起来之后，外部中断和NMI都被host处理了，host的处理在vmx_handle_exit_irqoff中

看到函数handle_external_interrupt_irqoff里面的处理逻辑是走host IDT的，据观察，KVM好像也没有去改变host IDT指向特殊的handler，也就是此时外部中断和NMI的处理逻辑和非虚拟化环境的处理逻辑是一样的

那这样看的话，guest是永远接受不到直接的外部中断和NMI了，只能依赖注入了。那在没有开启设备直通的情况下，所有guest的外设都得是虚拟的了，因为只有是虚拟的才会调用虚拟中断控制器的接口，才能最终走到注入中断的逻辑。因为物理的外设的中断走不到guest，被host处理了。

是这样子吗

2.

posted-interrupt机制中的Posted-interrupt notification vector是一个16位的字段，只用了低8位来存储一个vector(25.6.8 Controls for APIC Virtualization )，也就是说一次只能存放一个vector，只能表示一个vector。

而发生中断之后只有vector和这个Posted-interrupt notification vector对应上才不会导致exit，可以直接传给gues。

所以对于0~255个中断号，只能允许有一个中断vector被用于posted给guest，并且不会导致vm-exit，而除了这个中断号之外其他中断号还是照样得vm-exit？

如果是这样的话感觉怪怪的，那么多中断，只能挑一个来优化性能，其他还是照样得exit

比较合理的不应该是设置成一个bitmap，每个bit表示一个要post给guest的中断，这样VMM就可以自己选择要哪些中断post给guest，类似Exception bitmap那样

是这里有什么理解不对的地方吗

3.

post机制中为什么需要去更新这个kvm_kernel_irq_routing_entry(posted_intr.c中的pi_update_irte)，看不懂这个函数涉及到的是什么流程

3.

posted-interrupt机制和interrupt-remapping是得同时开启的吗，因为看他们代码似乎交织在一起，理不清他们的关系。posted-interrupt 是可以独立存在的？

4.

为什么interrupt remapping(irq_remapping.c)”相关的逻辑是放在“dervers/iommu”下的了？而不是在“arch/x86/kvm”目录下了

5.

VFIO和dma remapping的关系
