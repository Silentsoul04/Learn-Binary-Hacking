其他
========================================

ioctl
----------------------------------------
在QEMU-KVM中，用户空间的QEMU是通过 ioctl 与内核空间的KVM模块进行通讯的。

QOM
----------------------------------------
QOM (QEMU Object Module) 是用C语言实现的一种面向对象的编程模型，是设备模拟的基础，QEMU中所有的设备，包括CPU、内存、PCIe、外设都是基于QOM来实现的。

TCG
----------------------------------------
TCG是Tiny Code Generator的简称，在QEMU中，它将虚拟出来的系统的指令转化成真正硬件支持的指令中的从中间代码到硬件支持的机器代码。
