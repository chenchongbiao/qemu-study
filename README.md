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

![img](images/CPU异常中断.jpg)

## **半虚拟化**

半虚拟化的思想是：修改操作系统内核，替换掉不能虚拟化的指令，通过超级调用(`hypercall`)直接和底层的虚拟化层 `Hypervisor`来通讯，`Hypervisor`同时也提供了超级调用接口来满足其他关键内核操作，比如内存管理、中断和时间保持。

这种做法省去了全虚拟化中的捕获和模拟，大大提高了效率。所以像 `Xen`这种半虚拟化技术，客户机操作系统都是有一个专门的定制内核版本，和 `x86`、`mips`、`arm`这些内核版本等价。这样以来，就不会有捕获异常、翻译、模拟的过程了，性能损耗非常低。这就是 `Xen`这种半虚拟化架构的优势。这也是为什么 `Xen`只支持虚拟化 `Linux`，无法虚拟化 `windows`原因，因为微软不改代码。

## **硬件辅助的全虚拟化**

2005年后，`CPU`厂商 `Intel`和 `AMD`开始支持虚拟化了。`Intel`引入了 `Intel-VT(Virtualization Technology)`技术。 这种 `CPU`，有 `VMX root operation`和 `VMX non-root operation`两种模式，两种模式都支持 `ring0~ring3`共4个运行级别。这样，`VMM`可以运行在 `VMX root operation`模式下，`Guest OS`运行在 `VMX non-root operation`模式下。

而且两种操作模式可以互相转换。运行在 `VMX root operation`模式下的 `VMM`通过显式调用 `VMLAUNCH`或 `VMRESUME`指令切换到 `VMX non-root operation`模式，硬件自动加载 `Guest OS`的上下文，于是 `Guest OS`获得运行，这种转换称为 `<b>VM entry</b>`。`Guest OS`运行过程中遇到需要 `VMM`处理的事件，例如外部中断或缺页异常，或者主动调用 `VMCALL`指令调用 `VMM`的服务的时候(与系统调用类似)，硬件自动挂起 `Guest OS`，切换到 `VMX root operation`模式，恢复 `VMM`的运行，这种转换称为 `<b>VM exit</b>`。`VMX root operation`模式下软件的行为与在没有 `VT-x`技术的处理器上的行为基本一致；而 `VMX non-root operation`模式则有很大不同，最主要的区别是此时运行某些指令或遇到某些事件时，发生 `VM exit`。

也就说，硬件这层就做了些区分，这样全虚拟化下，那些靠**捕获异常-翻译-模拟**的实现就不需要了。而且CPU厂商，支持虚拟化的力度越来越大，靠硬件辅助的全虚拟化技术的性能逐渐逼近半虚拟化，再加上全虚拟化不需要修改客户操作系统这一优势，全虚拟化技术应该是未来的发展趋势。

|                         | 利用二进制翻译的全虚拟化           | 硬件辅助虚拟化                                                                        | 操作系统协助/半虚拟化                                                                                          |
| ----------------------- | ---------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 实现技术                | BT和直接执行                       | 遇到特权指令转到root模式执行                                                          | Hypercall                                                                                                      |
| 客户操作系统修改/兼容性 | 无需修改客户操作系统，最佳兼容性   | 无需修改客户操作系统，最佳兼容性                                                      | 客户操作系统需要修改来支持hypercall，因此它不能运行在物理硬件本身或其他的hypervisor上，兼容性差，不支持Windows |
| 性能                    | 差                                 | 全虚拟化下，CPU需要在两种模式之间切换，带来性能开销；但是，其性能在逐渐逼近半虚拟化。 | 好。半虚拟化下CPU性能开销几乎为0，虚机的性能接近于物理机。                                                     |
| 应用厂商                | VMware Workstation/QEMU/Virtual PC | VMware ESXi/Microsoft Hyper-V/Xen 3.0/KVM                                             | Xen                                                                                                            |

# **KVM的CPU虚拟化**

## **vCPU概述**

```bash
qemu-kvm通过对/dev/kvm的一系列ioctl命令来控制虚拟机和监视虚拟机的状态，例如：
int kvmfd = open("/dev/kvm", O_RDWR|O_LARGEFILE);
ret = ioctl(kvmfd, KVM_GET_API_VERSION, 0);
int vmfd = ioctl(kvmfd, KVM_CREATE_VM, 0);   
int vcpufd = ioctl(vmfd, KVM_CREATE_VCPU, 0);
```

一个 `KVM`虚拟机即一个Linux `qemu-kvm`进程，与其他Linux进程一样被Linux进程调度器调度。

`KVM`虚拟机包括虚拟内存、虚拟CPU和虚机 I/O设备，其中，内存和 `CPU`的虚拟化由 `KVM`内核模块负责实现，I/O设备的虚拟化由 `Qwmu`负责实现。

`KVM`户机系统的内存是 `qemu-kvm`进程的地址空间的一部分。

`KVM`虚拟机的 `vCPU`作为线程运行在 `qemu-kvm`进程的上下文中。

`vCPU`、`Qemu`进程、Linux进程调度和物理CPU之间的逻辑关系：

![img](images/CPU之间的逻辑关系.jpg)

通过上面的学习已经得知 `kvm`属于硬件辅助的全虚拟化，由于支持虚拟化的 `CPU`中都增加了新的功能。以 `Intel-VT`技术为例，它增加了两种运行模式：`VMX root`模式和 `VMX non-root`模式。通常来讲，`Host OS`和 `VMM`运行在 `VMX root`模式中，`Guest OS`及其应用运行在 `VMX non-root`模式中。因为两个模式都由各自的 `ring0~ring3`，因此，客户机可以运行在它所需要的 `ring`中(OS运行在 `ring0`，应用运行在 `ring3`中)，`VMM`也运行在其需要的 `ring`中(对 `KVM`来说，`QEMU`运行在 `ring3`，`KVM`运行在 `ring0`)。`CPU`在两种模式之间的切换称为 `VMX`切换。 **从 `root mode`进入 `non-root mode`，称为 `VM entry`；从 `non-root mode`进入 `root mode`，称为 `VM exit`。可见，`CPU`受控制地在两种模式之间切换，轮流执行 `VMM code`和 `Guest OS code`** 。

对 `KVM`虚拟机来说，运行在 `root mode`下的 `VMM`在需要执行 `Guest OS`指令时执行**`VMLAUNCH/VMRESUME`指令**将 `CPU`转换到 `non-root mode`，开始执行客户机代码，即 `VM entry`过程；在 `Guest OS`需要退出 `non-root mode`时(主动调用** `VMCALL`指令**或者遇到外中断或者遇到缺页等异常)，`CPU`自动切换到 `root mode`，即 `VM exit`过程。**可见，`KVM`** **客户机代码是受 `VMM`控制直接运行在物理 `CPU`上的。`Qemu`只是通过 `KVM`控制虚机的代码被 `CPU`执行，但是它们本身并不执行其代码。也就是说，`CPU`并没有真正的被虚拟化成虚拟的 `CPU`给客户机使用。**

几个概念：

* `socket`：颗，插入 `CPU`的个数。
* `core`：核，每个 `CPU`中的物理内核。
* `thread`：超线程，通常来说，一个 `CPU core`只提供一个 `thread`，这时客户机就只看到一个 `CPU`；但是，超线程技术实现了 `CPU`核的虚拟化，一个核被虚拟化出多个逻辑 `CPU`，可以同时运行多个线程。

所以一个 `kvm`虚拟机的 `vCPU`数目为 `thread*core*socket`，例如 `-smp 5,sockets=5,cores=1,threads=1`，所以 `vCPU`数目为 `5`，`vCPU`作为 `QEMU`线程被 `Linux`作为普通的线程/轻量级进程调度到物理的 `CPU`核上。(关于 `-smp`：[-smp用法](https://link.zhihu.com/?target=https%3A//zhensheng.im/2015/11/22/qemu-smp%25E5%258F%2582%25E6%2595%25B0%25E8%25A7%25A3%25E9%2587%258A%25E5%258F%258A%25E5%259C%25A8libvirt%25E4%25B8%25AD%25E7%259A%2584%25E8%25AE%25BE%25E7%25BD%25AE%25E5%25A4%259Acpu-%25E5%25A4%259A%25E6%25A0%25B8%25E5%25BF%2583.meow.html)

## **Guest模式**

一个普通的 `Linux`内核有两种执行模式：内核模式和用户模式。为了支持带有虚拟化功能的 `CPU`，`KVM`向 `Linux`内核增加了第三种模式即客户机模式 `Guest`，该模式对应于 `CPU`的 `VMMX non-root mode`。

`KVM`内核模块作为 `User mode`和 `Guest mode`之间的桥梁：

* `User mode`中的 `Qemu-kvm`会通过 `ioctl`命令来运行虚拟机。
* `KVM`收到该请求后，它先做一些准备工作，比如将 `vCPU`上下文加载到 `VMCS(virtual machine control structure)`等，然后驱动 `CPU`进入 `VMX non-root`模式，开始执行客户机代码。

三种模式的分工为：

* `Guest`模式：执行客户机系统非 `I/O`代码，并在需要的时候驱动 `CPU`退出该模式。
* `Kernel`模式：负责将 `CPU`切换到 `Guest mode`执行 `Guest OS`代码，并在 `CPU`退出 `Guest mode`时回到 `Kenerl`模式，将客户机的 `I/O`请求转发给 `User`模式让 `Qemu`完成。
* `User`模式：用 `ioctl(/dev/kvm)`与 `kvm`交互，代替客户机系统执行 `I/O`操作。

`Qemu-KVM`相比原生 `Qemu`的改动：

* 原生的 `Qemu`通过指令翻译实现 `CPU`的完全虚拟化，但是修改后的 `Qemu-KVM`会调用 `ioctl`命令来调用 `KVM`模块。
* 原生的 `Qemu`是单线程实现，`Qemu-KVM`是多线程实现。

主机 `Linux`将一个虚拟视作一个 `Qemu`进程，该进程包括下面几种线程：

* `I/O`线程用于管理模拟设备。
* `vCPU`线程用于运行 `Guest`代码。
* 其它线程，比如处理 `event loop`，`offloaded tasks`等的线程。

# Qemu

Qemu可以看成一款虚拟机，他可以模拟很多CPU架构。比如Alpha, ARM, Cris, i386, M68K, PPC, Sparc, [Mips](https://so.csdn.net/so/search?q=Mips&spm=1001.2101.3001.7020)等；以及大部分的硬件设备，也就可以模拟出不同的目标系统。

# Qemu主要原理与机制

Qemu可以实现目标平台的仿真，但是arm平台的程序怎么能在我们电脑上运行呢？这是Qemu主要要做的事情，翻译，那如何进行翻译呢。大致上就是下面这样：

客户代码 ——>中间代码 ——> 主机代码

那就会有人问了为什么要转换成中间代码，直接转换成主机代码不香吗？问得好。
在Qemu中执行翻译这部分流程的是TCG模块，因为Qemu是支持跨平台的，他不仅可以在linux下执行arm的程序还可以在ppc下等等不同的平台下执行，这里为了通用性方面的考虑，先转换成中间代码，在将中间代码转换成主机代码。

## Qemu的两种运行模式

Qemu 有两种运行模式，一种是全系统模拟（system mode），一种是用户态模拟(user mode)。
 从名字就可以看出来system mode肯定是模拟全了，可以直接跑操作系统之类的。user mode肯定就弱一点，跑个进程之类的。
 放个图吧，两种用户态模式：


# 参考

[Qemu学习-CPU虚拟化](https://zhuanlan.zhihu.com/p/521162423)

[Qemu学习笔记](https://blog.csdn.net/qq_44117740/article/details/119411029)
