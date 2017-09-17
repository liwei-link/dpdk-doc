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

.. _Mempool_Library:

内存池库
===============

内存池是固定大小对象的分配器。
在DPDK中使用名称标识内存池，使用mempool处理器存储空闲对象。默认的mempool处理器基于ring。
其提供一些可选服务，如per-core对象缓存和对齐辅助（alignment helper，保证对象在所有RAM通道上均匀分布）。

该库由 :ref:`Mbuf 库 <Mbuf_Library>` 引用。

Cookies
-------

在调试模式(启用 CONFIG_RTE_LIBRTE_MEMPOOL_DEBUG)中，所申请内存块的首尾会增加额外信息（cookies）。
然后申请的对象包含覆写保护字段用于调试缓冲区溢出。

统计
-----

在调试模式(启用 CONFIG_RTE_LIBRTE_MEMPOOL_DEBUG)中,
对象的get/put统计信息存储在mempool结构中。
统计信息是per-lcore的，避免了统计计数器的并行访问。

内存对齐限制
----------------------------

根据硬件内存配置，可以通过给对象之间增加填充提高性能。
目的是确保每个对象起始地址分布在内存的不同通道(channel)和rank，使得内存通道负载均衡。

这对L3转发和流量分类时的包缓存性能提升尤其显著。
由于仅需要处理前64字节，所以可以通过把缓存包对象起始地址分布在不同内存通道上来提升性能。

The number of ranks on any DIMM is the number of independent sets of DRAMs that can be accessed for the full data bit-width of the DIMM.
The ranks cannot be accessed simultaneously since they share the same data path.
The physical layout of the DRAM chips on the DIMM itself does not necessarily relate to the number of ranks.

运行应用时，通过EAL提供的参数配置可以增加内存通道和rank的数量。

.. note::

    命令行必须为处理器提供内存通道数量。

:numref:`figure_memory-management` 和 :numref:`figure_memory-management2` 是不同DIMM架构的对齐示例。

.. _figure_memory-management:

.. figure:: img/memory-management.*

   Two Channels and Quad-ranked DIMM Example


该示例中，假设一个包由16个64字节块组成(现实情况并非如此)。

Intel® 5520芯片组有三个通道，因此大部分情况对象之间不需要填充(除非对象的大小是 n x 3 x 64 字节块)。

.. _figure_memory-management2:

.. figure:: img/memory-management2.*

   Three Channels and Two Dual-ranked DIMM Example


创建新内存池时，用户可以选择是否使用对齐特性。

.. _mempool_local_cache:

本地缓存
-----------

在CPU使用方面，多核访问内存池ring的代价可能很高，因为每次都需要compare-and-set (CAS)操作。
为了减少内存池ring的访问，内存池分配器维护一个per-core缓存并对内存池ring做bulk请求(Bulk dequeue/Bulk enqueue ),
在实际的内存池结构中使用的是带有很少锁的缓存。每个核可以自由地访问自己的空闲对象缓存(有锁),
在缓存满的时候把部分空闲对象刷回内存池ring中，缓存空时会从内存池ring中获取一些对象放在缓存中。

虽然这会导致对象在核的缓存中闲置，暂时无法被其他核利用，但当前核心访问自己缓存的速度很快，这会提供更好的性能。

该缓存由一个小per-core指针表和长度构成，并且作为一个栈来使用。在创建池的时候可以选择启用或关闭。

缓存大小是固定的，在编译时由 CONFIG_RTE_MEMPOOL_CACHE_MAX_SIZE 定义。

:numref:`figure_mempool` 展示了缓存操作.

.. _figure_mempool:

.. figure:: img/mempool.*

   内存中的mempool和相关的Ring

除了内部默认的per-lcore本地缓存，应用也可以通过 ``rte_mempool_cache_create()``, ``rte_mempool_cache_free()`` 和 ``rte_mempool_cache_flush()`` 创建和管理额外的缓存。
通过 ``rte_mempool_generic_put()`` and ``rte_mempool_generic_get()`` 存取对象。 ``rte_mempool_default_cache()`` 调用返回默认的内部缓存(如果有的话)。
与默认缓存相比，用户自己的缓存可以用于非EAL线程。

Mempool Handlers
------------------------

DPDK可以使用外部内存子系统，如硬件内存管理系统和基于内存分配器的软内存系统。

mempool handler 的两个方面

* 使用宏 ``MEMPOOL_REGISTER_OPS`` 增加新的内存池操作(ops)代码。

* 使用新API  ``rte_mempool_create_empty()`` 和 ``rte_mempool_set_ops_byname()``
  创建mempool并指定ops。

相同应用中可能会使用多个不同的mempool handler。使用函数 ``rte_mempool_create_empty()`` 
创建mempool，然后使用 ``rte_mempool_set_ops_byname()`` 设置相关的mempool handler回调(ops)结构。

遗留的(Legacy)应用中可能仍使用旧的API ``rte_mempool_create()`` ，这个API默认使用基于mempool handler的ring。
这些应用需要改成使用新的mempool handler。

在使用 ``rte_pktmbuf_create()`` 的应用中，有个配置项(``RTE_MBUF_DEFAULT_MEMPOOL_OPS``)，通过它可以选择mempool handler

使用案例
---------

需要高性能内存申请的地方都应该使用基于池的内存分配器。
如以下案例:

*   :ref:`Mbuf 库 <Mbuf_Library>`

*   :ref:`环境抽象层EAL <Environment_Abstraction_Layer>` , 的日志服务

*   需要在数据平面上频繁处理大小固定对象的应用
