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


创建新内存池时，用户可以选择是否使用该特性。

.. _mempool_local_cache:

本地缓存
-----------

In terms of CPU usage, the cost of multiple cores accessing a memory pool's ring of free buffers may be high
since each access requires a compare-and-set (CAS) operation.
To avoid having too many access requests to the memory pool's ring,
the memory pool allocator can maintain a per-core cache and do bulk requests to the memory pool's ring,
via the cache with many fewer locks on the actual memory pool structure.
In this way, each core has full access to its own cache (with locks) of free objects and
only when the cache fills does the core need to shuffle some of the free objects back to the pools ring or
obtain more objects when the cache is empty.

While this may mean a number of buffers may sit idle on some core's cache,
the speed at which a core can access its own cache for a specific memory pool without locks provides performance gains.

The cache is composed of a small, per-core table of pointers and its length (used as a stack).
This internal cache can be enabled or disabled at creation of the pool.

The maximum size of the cache is static and is defined at compilation time (CONFIG_RTE_MEMPOOL_CACHE_MAX_SIZE).

:numref:`figure_mempool` shows a cache in operation.

.. _figure_mempool:

.. figure:: img/mempool.*

   A mempool in Memory with its Associated Ring

Alternatively to the internal default per-lcore local cache, an application can create and manage external caches through the ``rte_mempool_cache_create()``, ``rte_mempool_cache_free()`` and ``rte_mempool_cache_flush()`` calls.
These user-owned caches can be explicitly passed to ``rte_mempool_generic_put()`` and ``rte_mempool_generic_get()``.
The ``rte_mempool_default_cache()`` call returns the default internal cache if any.
In contrast to the default caches, user-owned caches can be used by non-EAL threads too.

Mempool Handlers
------------------------

This allows external memory subsystems, such as external hardware memory
management systems and software based memory allocators, to be used with DPDK.

There are two aspects to a mempool handler.

* Adding the code for your new mempool operations (ops). This is achieved by
  adding a new mempool ops code, and using the ``MEMPOOL_REGISTER_OPS`` macro.

* Using the new API to call ``rte_mempool_create_empty()`` and
  ``rte_mempool_set_ops_byname()`` to create a new mempool and specifying which
  ops to use.

Several different mempool handlers may be used in the same application. A new
mempool can be created by using the ``rte_mempool_create_empty()`` function,
then using ``rte_mempool_set_ops_byname()`` to point the mempool to the
relevant mempool handler callback (ops) structure.

Legacy applications may continue to use the old ``rte_mempool_create()`` API
call, which uses a ring based mempool handler by default. These applications
will need to be modified to use a new mempool handler.

For applications that use ``rte_pktmbuf_create()``, there is a config setting
(``RTE_MBUF_DEFAULT_MEMPOOL_OPS``) that allows the application to make use of
an alternative mempool handler.


Use Cases
---------

All allocations that require a high level of performance should use a pool-based memory allocator.
Below are some examples:

*   :ref:`Mbuf Library <Mbuf_Library>`

*   :ref:`Environment Abstraction Layer <Environment_Abstraction_Layer>` , for logging service

*   Any application that needs to allocate fixed-sized objects in the data plane and that will be continuously utilized by the system.
