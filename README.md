# CPU虚拟化

## 虚拟CPU带来的问题

`x86`处理器提供 `ring0~ring3`四个特权级别给操作系统和应用程序来访问硬件，`ring0`权限最高，`ring3`权限最低：

* 操作系统（内核）需要直接访问硬件和内存，因此它的代码需要运行在最高运行级别 `ring0`上，这样它可以使用特权指令，控制中断、修改页表、访问设备等等。
* 应用程序的代码运行在最低运行级别上 `ring3`上，不能做受控操作。如果要做，比如要访问磁盘，写文件，那就要通过执行系统调用，执行系统调用的时候，CPU的运行级别会发生从 `ring3`到 `ring0`的切换，并跳转到系统调用对应的内核代码位置执行，这样内核就为你完成了设备访问，完成之后再从 `ring0`返回 `ring3`。这个过程也称作用户态和内核态的切换。

**虚拟CPU在这里遇到一个难题，因为 `Host OS`是工作在 `ring0`的，`Guest OS`就无法也在 `ring0`了，但是 `Guest OS`本身不知道这一点，以前执行访问设备等特权指令，现在还是执行特权指令，但是没有执行权限肯定会出错。**

虚拟机怎么通过 `VMM/Hypervisor`实现 `Guest CPU`对硬件的访问，根据其原理不同有三种实现技术：

1. 全虚拟化/`Full-Virtualization`
2. 半虚拟化/`Para-Virtualization`
3. 硬件辅助的全虚拟化

## **基于二进制翻译的全虚拟化**

客户操作系统运行在 `ring1`，它在执行特权指令时，会触发CPU异常(`Exception`)，然后 `VMM`捕获这个异常，在异常里面做翻译，模拟，最后返回到客户操作系统内，客户操作系统认为自己的特权指令工作正常，继续运行。但是这个性能损耗非常的大，简单的一条指令，本来执行完结束，现在却要通过复杂的异常处理过程。

异常 `Exception`(内中断)，中断 `Interrupt`(外中断)，陷阱 `Trap`(软中断)的区别：

![img]()

# 参考

[Qemu学习-CPU虚拟化](https://zhuanlan.zhihu.com/p/521162423)
