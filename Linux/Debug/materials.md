参照: https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/debug

在`linux/tools/linux工具快速教程`查看


内核调试方法: https://www.cnblogs.com/justin-y-lin/tag/%E5%86%85%E6%A0%B8/ (none)


Linux内核调试方法总结: https://www.cnblogs.com/alantu2018/p/8997149.html




linux内核调试转储工具kdump crash: http://abcdxyzk.github.io/blog/2013/08/21/debug-kdump-base/



掌握以下Linux 下常用命令行：pwd、cd、find等

1-2款趁手的编辑器，推荐掌握vim的基本操作，有时候需要在命令行编辑一些文件

gcc、gdb、ld、make等编译构建链条

objdump、nm、readif、dd 等 ELF 文件分析、烧录工具

qemu、Bochs 等硬件模拟器，我们编写的mini os一般调试阶段都是在模拟器上进行的，这些模拟器能够模拟多种CPU硬件环境、典型的如 intel x86-32

objdump：对ELF格式执行程序文件进行反编译、转换执行格式等操作的工具
nm：查看执行文件中的变量、函数的地址
readelf：分析ELF格式的执行程序文件
make：软件工程管理工具， make命令执行时，需要一个 makefile 文件，以告诉make命令如何去编译和链接程序
dd：读写数据到文件和设备中的工具




通过看qemu源码搞定

我的方法是 kvmtools + kvm 这个应该是最简单的虚拟化方案。 (to do)