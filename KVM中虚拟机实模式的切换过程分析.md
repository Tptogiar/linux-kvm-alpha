








# KVM中虚拟机实模式的切换过程分析



因为实模式和保护模式的控制在控制寄存器的cr0的PE位，所以先想到的就是找到对CR0进行修改的最终函数，再由最终函数的上游函数来理清大概的调用流程。

根据cr0这个关键词找到的相关函数有两个`kvm_set_cr0`和`svm_set_cr0`

- 目的一：`kvm_set_cr0`和`svm_set_cr0`的关系是什么
- 目的二：如果`kvm_set_cr0`，`svm_set_cr0`有调用关系，那他们各自上游函数是怎么样的，有哪些是跟模式切换强相关的





# 目的一：kvm_set_cr0和svm_set_cr0的关系是什么

- 猜测：`kvm_set_cr0`函数在目录外层，`svm_set_cr0`在子目录svm，svm和vmx是并列关系，且vmx下也有类似的函数实现`vmx_set_cr0`，猜测`kvm_set_cr0`调用`svm_set_cr0`

- 代码中有一处`static_call(kvm_x86_set_cr0)(vcpu, cr0);`但是是根据字符串查找的话，没有找到这个kvm_x86_set_cr0其他相关的了，

- 换个方向`static_call`为关键词进行查找到话，可以找到相关的说明

  ```
  /*
   * Static call support
   *
   * Static calls use code patching to hard-code function pointers into direct
   * branch instructions. They give the flexibility of function pointers, but
   * with improved performance. This is especially important for cases where
   * retpolines would otherwise be used, as retpolines can significantly impact
   * performance.
  ```

  ```
  /*
   * KVM_X86_OP() and KVM_X86_OP_NULL() are used to help generate
   * "static_call()"s. They are also intended for use when defining
   * the vmx/svm kvm_x86_ops. KVM_X86_OP() can be used for those
   * functions that follow the [svm|vmx]_func_name convention.
   * KVM_X86_OP_NULL() can leave a NULL definition for the
   * case where there is no definition or a function name that
   * doesn't match the typical naming convention is supplied.
   */
  ```

- 所以可以确定`static_call(kvm_x86_set_cr0)`在svm/vmx下会调用到`vmx_set_cr0`/`vmx_set_cr0`





# 目的二. kvm_set_cr0，svm_set_cr0他们各自上游函数是怎么样的，有哪些是跟模式切换强相关的

- 关于`svm_set_cr0`

  - 软件识别的`svm_set_cr0`的butterfly长这样

![image-20230110150020431](https://user-images.githubusercontent.com/79641956/211510141-bf4cc486-44a4-4bed-b720-5d50a41bf3f1.png)


  - 在函数`nested_vmcb02_prepare_save`，`nested_svm_vmexit`中看了一下他们调用`svm_set_cr0`的地方，发现只是在他们各自的流程中复用一下`svm_set_cr0`，调用`svm_set_cr0`不是他们的主要目的，所以应该跟模式切换流程关系不大
  - 函数`cr_trap`有专门设置cr寄存器的意图，且其函数指针被记录在`svm_exit_handlers`函数指针数组中，且索引值是`SVM_EXIT_CR0_WRITE_TRAP`
  - 除了上面两个函数之外，`svm_x86_ops`函数指针还被记录在`svm_init_ops`的`runtime_ops`中，被用于一些初始化的地方
  - 因为`static_call(kvm_x86_set_cr0)`也即调用`svm_set_cr0`，所以还需要看一下调用`static_call(kvm_x86_set_cr0)`的地方，
    - 一个是下面的`kvm_set_cr0`
    - 还有就是`enter_smm`，进入smm模式用的
    - `__set_sregs_common`函数，在向KVM API发起设置寄存器请求后，位于`virt/kvm/kvm_main.c`下的`kvm_vcpu_ioctl`函数的子流程会调用到这个函数，进而可能设置到相关的寄存器
    - `kvm_vcpu_reset`函数，reset vcpu的时候设置到比较多寄存器，cr0只是其中一个

- 关于`kvm_set_cr0`

  - `kvm_set_cr0`的butterfly如下

![image-20230110152530220](https://user-images.githubusercontent.com/79641956/211510162-19e3d489-0930-4438-a196-6cc2d7b42a7c.png)

  - `kvm_lmsw`函数，调用关系上看是给vmx下使用的；`handle_set_cr0`函数也是vmx下用的

  - `emulator_set_cr`函数，除了把指针记录在了`x86_emulate_ops`结构的`emulate_ops`变量下，最终在`arch/x86/kvm/emulate.c`文件下的相关模拟函数中被调用，其中有一个就是`rsm_enter_protected_mode`

  - `cr_interception`函数，同`cr_trap`函数一样，被注册在`svm_exit_handlers`函数指针数组中，且索引值是`SVM_EXIT_WRITE_CR0`

    - 进一步跟了一下这个`svm_exit_handlers`数组，发现比较主要的是在`svm_invoke_exit_handler`函数中调用，而该函数进一步追到`handle_exit`，`handle_exit`又在`static_call(kvm_x86_handle_exit)(vcpu, exit_fastpath)`时被调用，进一步追踪就到了svm实现的KVM API接口，`kvm_arch_vcpu_ioctl_run`，等`kvm_arch_vcpu_ioctl_xxx`函数，再上就是`virt/kvm/kvm_main.c`下的一些接口逻辑和初始化逻辑

- 所以总的来看，跟模式切换比较相关的，上游流程主要有

  - VMEXIT后，处理程序根据具体的AE( Automatic Exits)原因，进行大概的处理后，走到`svm_exit_handlers`函数指针数组中注册的某个函数里面，相关的索引有`SVM_EXIT_WRITE_CR0`，`SVM_EXIT_CR0_WRITE_TRAP`，分别对应函数`cr_interception`和`cr_trap`





# 关于cr_interception流程和cr_trap流程的区别

这里想不明白，`cr_interception`和`cr_trap`最终都会调用到`svm_set_cr0`，但是`cr_trap`的逻辑很简单，而`cr_interception`的过程比较复杂，后面还调用了emulate.c下的一些模拟函数

想不太明白的是：`cr_trap`为什么可以这么简单？以及和`cr_interception`的那边的流程的区别和联系是什么？







# 总览
![虚拟化-7](https://user-images.githubusercontent.com/79641956/211514130-3d9adacf-04f9-471d-92a5-317cf5f2dae5.jpg)
