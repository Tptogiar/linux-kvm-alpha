

# 内容

- 翻一些cr0相关的代码，模式切换后的页表切换相关
- KVM虚拟机如何与Linux内存管理结合
  - KVM & 内存管理
    - VMM如何为Guest准备内存，如何上报Guest
    - VMM如何支持Guest实模式下的寻址；保护模式寻址；有了EPT后的过程
    - VM内存回收，共享(KSM)
    - qemu向KVM注册内存的过程
  - KVM & 进程管理
    - 看了一个简易的模范qemu的例子
  - 内存管理
    - 
- SPT
  - spt 如何正确高效的trace gpt





# 一些总结



## KVM & qemu如何为Guest准备内存

- **物理环境下，BIOS如何为OS准备内存信息？**

  BIOS上电后BIOS的自检程序会开始检查外设，其中就包括内存，他会检查内存插槽使用情况，大小等信息，然后将检查到的内存信息放在BIOS的数据区BDA里面；

  BIOS还会用程序实现一个用来读取数据区里内存信息的函数，并将该函数作为BIOS的0x15号中断的中断处理程序；

  当系统软件需要查询系统内存信息时，向BIOS发出0x15号中断即可，即可拿到内存信息

- **在虚拟环境下，qemu如何模拟BIOS？**

  真实的BIOS内存会被映射(MMIO)在逻辑空间的0x000F0000\~0x000FFFFF处，在虚拟环境下不需要完全模拟这个过程了，但是qemu需要模拟BIOS的内存查询函数，放在0x000F0000~0x000FFFFF的某处；

  qemu将启动的内存参数转化为e820格式的内存信息放在0x000F0000\~0x000FFFFF某处，并在内存查询函数中读取这写信息返回给系统软件；

  建立IVT（BIOS的中断向量表），将0x15中断的处理函数入口指向内存查询函数；

- 最后，guest启动，发起0x15中断，从相关的寄存器里面读取到了为其准备的内存条信息

- **那这个过程跟KVM有什么关系？**

  上文“qemu将启动的内存参数转化为e820格式的内存信息放在0x000F0000~0x000FFFFF某处”的同时，qemu还需要将相应的内存信息向KVM注册，告知KVM，当前给哪个guest分配了多少哪些内存槽(slot)，分别多大，从guest 物理内存的哪里开始(GFN)

- **为什么要告诉KVM给guest准备的内存信息？**

  因为在guest向MMU发出的内存地址是GVA，KVM截获缺页异常后需要找到其对应的GPA，然后HVA，最后再是HPA；

  从HVA到HPA这个过程是Linux原先就有的虚拟内存机制，可以比较容易实现；

  关键是GVA到GPA，到HVA的过程：因为GVA，GPA是guest的内存，而KVM如果一点都不知道guest的GPA与qemu的进程空间段有怎么的对应关系，那它就没法在VMM中实现将GPA转化为HVA了

  所以KVM是需要知道GPA与qemu进程空间段的对应关系的，而这个关系就是qemu在向KVM注册内存slot的时候告知KVM的

- **KVM如何根据GVA，找到HPA？**

  KVM在缺页异常处理函数中会先拿到VMCB中拿到guest.cr3(GPA)，根据guest曾经在KVM注册的内存槽信息(kvm_memory_slot)，找到其对应的HVA，HPA，进而找到guest页表，遍历guest页表，找到GVA对应的GPA；

  如果该GPA不存在或是不可访问，则将这一信息注入到guest里面，让guest自身处理；如果GPA处在，则类似的会根据GPA找到slot，进程找到HVA，HPA，并更新在影子页表中







## VMM如何支持Guest实模式下的寻址；保护模式寻址；有了EPT后的寻址过程

- V**MM如何支持Guest的实模式寻址？**

  在启动guest之时，host早已进去了保护模式，而且VMM不允许guest将CPU切到实模式或是非分页模式；那这样的情况下如何为guest提供一个实模式环境？

  x86体系架构的CPU支持在保护模式下运行实模式的代码——Virtual-8080模式，而KVM在保护模式下为guest提供实模式正式利用了CPU的这种模式，不过当实模式需要寻址的时候，还是需要通过分页寻址的方式进行的；

- **实模式寻址的大概过程：**

  具体的来说，VMM会为guest准备一个页表(GPA --> HPA)，而这个页表所表征的虚拟内存即为guest的物理内存，也就是说用这个页表来完成GPA到HPA的转化

  - 刚开始页表是空的，当guest发送段基址和偏移量到MMU的分段单元后，MMU的分段单元将其组合成一个物理地址GPA，然后MMU拿着这个GPA去查表
  - 因为页表是空的，会触发缺页异常，会触发VMEXIT，控制权转到了VMM这里
  - VMM根据VMCB中相关域内容，找到guest请求的GPA，根据GPA找到对应的slot，根据slot找到对应的HVA，随后分配物理内存HPA，更新页表
  - 下次guest请求相同的GPA时，由于页表中已经存在对应的映射，所以guest就可以顺利得到对应的HPA

- **保护模式下的寻址**

  保护模式下的寻址与实模式类似，不过此时建立的页表(影子页表)的映射关系(GVA --> HPA)不同于实模式下的映射关系(GPA --> HPA)；而且因为实模式下guest自身是没有页表的，而保护模式下，guest自身是有页表的，所以此时还需要考虑影子页表如何与guest自身的页表进行同步，因为如果不同步会导致出现诸如影子页表内维护了一些本不需要或是错误的映射关系的问题

- **保护模式下的大概寻址过程：**

  - 同实模式一样，刚开始影子页表是空的，当guest发送GVA到MMU之后，会触发缺页异常，陷入VMM
  - VMM找到guest的cr3，拿到guest的gpt，遍历guest的页表
  - 如果此时guest内GVA到GPA的映射还没有建立，或是该entry是只读的，VMM会件缺页异常注入给guest，让guest自己处理，guest收到这样的异常后，会给GVA分配对应的GPA，并尝试修改页表，这个操作会被VMM拦截，以便于VMM更新对应的影子页表
  - 如果此时guest内GVA到GPA的映射已经建立，则VMM会读取对应的GPA，在找到对应的slot，进而找到HVA，并分配对应的HPA，更新影子页表
  - 下次guest请求相同的GVA时，由于影子页表中已经存在对应的映射，所以guest就可以顺利得到对应的HPA

- **影子页表是如何trace guest内的gpt的？**

  - 影子页表自身防止被修改：影子页表的基址在cr3上，有可能会被guest修改到，所以需要把影子页表对应的页面设置为只读，当guest试图修改是，就会触发异常而陷入VMM

  - 当guest试图切换或修改其自身的页表：

    - 更新cr3：guest在切换页表的时候需要更新cr3，而这需要执行敏感指令，导致陷入VMM，进而VMM也可以趁机切换影子页表；
    - INVlPG指令：同理当guest试图使用INVlPG指令修改页表项时，也会因为敏感指令而陷入VMM
    - 修改页表项：VMM会将影子页表的访问权限(CPL)设置得比guest低，导致guest试图修改页表时，就会触发缺页异常

  - 当发生缺页异常：

    对于一条事先不存在的映射(GVA->GPA->HPA)，首先要经过guest来给GVA分配GPA，之后才是VMM分配HPA并设置影子页表，所以VMM在设置影子页表的时候可以根据guest的页表中对应的entry的字段值来设置影子页表来达到trace guest内的gpt的目的，其中对于guest的pte有这么几种情况：

    - present位为0，这种情况影子页表不需要建立对应的entry，等下次触发缺页异常就好了
    - present位，acceessed位，dirty位均为1，影子页表中对应的entry需要将acceessed位，dirty位设置为1，这样与guest的也是一致的
    - acceessed位为0，影子页表中对应的entry需要设置present位为0，这样当guest试图修改acceessed位时，便会触发缺页异常，陷入VMM
    - dirty位为0，影子页表中对应的entry需要设置为只读，这样当guest试图写内容时，也会因为异常而陷入VMM

- **EPT支持下的寻址过程（32位，二级页表）**

  - guest从cr3中取出页目录的基址，根据GVA中的字段值，索引到对应的PDE
  - 由于这个PDE是个GPA，要真正访问到内容，需要使用EPT，转化为HPA，也即某个页表的地址
  - 再根据GVA中的字段值，索引到对应的PTE(GPA)，使用EPT，转化为HPA
  - 在加上GVA中的偏移量，得到最终的GPA，使用EPT，转化为HPA，也即最终的访问的HPA

- **EPT/NPT 与 影子页表**

  - 影子页表

    大多数内存访问可以没有在VMM介入的情况下访问；

    任何guest页表的修改需要VMEXIT后VMM同步为影子页表的修改；

    每次guest内部切换进程，都需要切换影子页表；

    每个guest内部进程都需要额外维护一张影子页表；

  - EPT/NPT

    缺页异常后不需要VMEXIT，CPU上下文切换小；

    gpt，ept页表分别维护，没有spt同步开销

    一个guest一张EPT页表，而不是一个进程一个EPT页表

    guest内部进程切换不需要切EPT





## VM内存回收，共享(KSM)

- VM内存回收

  VMM会使用一个伪设备驱动程序，载入到guest中，并通过它与guest进行交互；

  当VMM需要回收内存时，使用它向guest申请内存，guest的内存管理机制会给它分配内存，它将分配到的内存的GFN告知到VMM，VMM回收这部分内存，并guest对应的页表中将相应的HPA标记为缺失；

  这样，即使guest向它要回那部分原先被分配出去的内存，也不会有什么问题，因为真正访问的时候会触发缺页异常而陷入VMM，VMM可以把这部分内容返还回去

- 内存共享

  VMM会定时扫描GPA，对页面进行初步的哈希计算，并将结果放在哈希表中，如有出现哈希冲突，而进一步对比冲突的两个页面，如果两个页面完全相同，这回收其中一个页面，更新页表，使得两个entry同时指向同一个HPN；

  当guest要修改页面内容的时候，在拷贝页面成为两个，分别给两个guest使用

  
  
  ## qemu向KVM注册内存的大概过程

- qemu主要是通过KVM API与KVM交互的，交互的方式向设备文件/dev/kvm发起相应的ioctl

  内存注册接口对应`KVM_SET_USER_MEMORY_REGION`命令，并传入`kvm_userspace_memory_region`结构体

  ```
  struct kvm_userspace_memory_region {
         __u32 slot;
         __u32 flags;
         __u64 guest_phys_addr;
         __u64 memory_size; /* bytes */
         __u64 userspace_addr; /* start of the userspace allocated memory */
  };
  ```

- 虚拟机启动时，qemu会在其虚拟空间内申请一段连续的内存空间，并根据启动参数虚拟出一些内存条slot信息，并向KVM进行注册

- KVM用kvm_memory_slot 结构体来记录某个guest注册过的内存条，其中可以根据base_gfn(guest的GPN)，npages(页数)，userspace_addr(在qemu中的HVA)字段可以在知道某个具体的GPN情况下，找到对应的slot，进而找到对应的HVA，在根据qemu进程的页表找到对应的HPA

  ```
  struct kvm_memory_slot {
         gfn_t base_gfn;
         unsigned long npages;
         unsigned long *dirty_bitmap;
         struct kvm_arch_memory_slot arch;
         unsigned long userspace_addr;
         u32 flags;
         short id;
         u16 as_id;
  };
  ```

  





































