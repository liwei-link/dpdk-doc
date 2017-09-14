..  BSD LICENSE
    Copyright(c) 2010-2014 Intel Corporation. All rights reserved.
    All rights reserved.

    Redistribution and use in source and binary forms, with or without
    modification, are permitted provided that the following conditions
    are met:

    * Redistributions of source code must retain the above copyright
    notice, this list of conditions and the following disclaimer.
    * Redistributions in binary form must reproduce the above copyright
    notice, this list of conditions and the following disclaimer in
    the documentation and/or other materials provided with the
    distribution.
    * Neither the name of Intel Corporation nor the names of its
    contributors may be used to endorse or promote products derived
    from this software without specific prior written permission.

    THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
    "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
    LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
    A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
    OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
    SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
    LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
    DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
    THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
    (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
    OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

**Part 1: Architecture Overview**

概述
========

本节将给予读者DPDK的全局总览。

DPDK主要的目的是提供一个简单的计算框架，用于数据平面应用的包处理。
用户可以通过示例代码理解其中的技术、构建应用原型或者增加自己的协议栈。
Alternative ecosystem options that use the DPDK are available.

DPDK框架通过环境抽象层（EAL）为指定环境创建了一组功能库，EAL可能被绑定到Intel架构(32位或64位)、
Linux用户态编译器或者一种特定平台。环境的创建是通过Makefile和配置文件完成的。
一旦EAL创建完成，用户可以通过链接EAL的库创建自己的应用。
DPDK也提供EAL之外其他的库，包括Hash、Longest Prefix Match（LPM）和Ring库。
示例应用展示了DPDK各种特性的使用方法。

DPDK为数据包处理实现了一种run-to-completion模型，这种模型在调用数据平面应用之前，
就需要预先申请所有资源。run-to-completion模型在逻辑处理核心上运行执行单元。
这个模型没有调度器，所有设备的访问都是通过轮询的方式。不使用中断的主要原因是中断处理的性能损耗。

除了run-to-completion模型，流水线（pipeline）模型也会用于核心之间传递数据包或者消息（通过rings）。
这使得数据包的处理能够在各个阶段执行，并且可以更加高效地在各个核心上运行。

开发环境
-----------------------

DPDK开发环境的安装需要Linux和相关的工具链，如一个或者多个编译器、汇编器、make工具、
编辑器和各种用于创建DPDK组件和库的依赖库。

一旦指定环境和架构的库创建完成，就可以用于创建用户的数据平面应用。

当为Linux用户态创建应用的时候，需要glibc库。对于DPDK应用，RTE_SDK和RTE_TARGET这两个环境变量必须在编译前配置好。
可以通过下面的示例设置这两个环境变量：

.. code-block:: console

    export RTE_SDK=/home/user/DPDK
    export RTE_TARGET=x86_64-native-linuxapp-gcc

开发环境的设置参见 *DPDK Getting Started Guide*

环境抽象层
-----------------------------

环境抽象层（EAL）屏蔽特定环境差异，提供了一组通用的接口/服务给应用程序使用。
EAL提供的服务有：

*   DPDK加载和启动

*   多进程和多线程的支持

*   CPU核心亲和性/分配

*   系统内存申请和释放

*   原子/锁操作

*   时间相关

*   PCI总线访问

*   跟踪和调试功能

*   CPU特性标识

*   中断处理

*   Alarm operations

*   内存管理 (malloc)

看这里 :ref:`Environment Abstraction Layer <Environment_Abstraction_Layer>`.

核心组件
---------------

*核心组件* 是一组库，它们提供了高性能数据包处理应用所需的要素。

.. _figure_architecture-overview:

.. figure:: img/architecture-overview.*

   核心组件架构


Ring管理 (librte_ring)
~~~~~~~~~~~~~~~~~~~~~~~~~~

ring通过一个有限大小的表实现了无锁，多生产者，多消费者，先进先出（FIFO）的API。
和无锁队列相比其有一些优势：实现更容易，适用于大部分操作和更加快速。
ring被 :ref:`Memory Pool Manager (librte_mempool) <Mempool_Library>` 使用，
并且可以作为一种通用的通信机制为核心之间或同一核心上的执行单元提供通信。

看这里 :ref:`Ring Library <Ring_Library>`.

内存池管理 (librte_mempool)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

内存池管理器是负责为对象池申请内存的。
内存池由名字标识，使用ring存储可用的对象。
其提供了一些可选的服务，比如per-core对象缓存和对齐辅助（alignment helper，保证对象在所有RAM通道上均匀分布）

看这里  :ref:`Mempool Library <Mempool_Library>`.

网络包缓存管理 (librte_mbuf)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

mbuf库提供了用于创建和销毁DPDK消息缓存的工具。
DPDK消息缓存在程序启动是创建，并存储在内存池中。

这个库提供了申请/释放mbuf、ctrlmbuf和pktmbuf的API。
ctrlmbuf（manipulate control message buffers）是通用消息缓存，pktmbuf（packet buffers）用于承载网络包。

看这里 :ref:`Mbuf Library <Mbuf_Library>`.

定时器管理 (librte_timer)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

该库为DPDK执行单元提供了定时器服务、异步执行函数的能力。
它能够周期性或者一次性地调用函数。定时器库使用EAL提供的定时器接口获取时间基准。
并且能够按需要在每个核心上初始化。

定时器完整描述 :ref:`Timer Library <Timer_Library>`.

以太网轮询模式驱动架构
---------------------------------------

DPDK包含了1 GbE, 10 GbE and 40GbE的轮询模式驱动，和半虚拟化的virtio以太网控制器。
virtio以太网控制器能在没有异步和中断信号机制的情况下工作。

看这里  :ref:`Poll Mode Driver <Poll_Mode_Driver>`.

数据包转发算法
-----------------------------------

DPDK包括哈希(librte_hash)和最长前缀匹配(LPM,librte_lpm)库，用于支持包转发算法。

看这里 :ref:`Hash Library <Hash_Library>` and  :ref:`LPM Library <LPM_Library>` for more information.

librte_net
----------

librte_net库是一组IP协议定义和相关操作的宏。基于FreeBSD* IP栈代码，包含IP头所使用的协议号，
IP相关的宏，IPv4/IPv6头结构体，TCP, UDP和SCTP头结构体。
