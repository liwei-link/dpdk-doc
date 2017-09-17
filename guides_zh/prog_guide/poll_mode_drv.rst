..  BSD LICENSE
    Copyright(c) 2010-2015 Intel Corporation. All rights reserved.
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

.. _Poll_Mode_Driver:

轮询模式驱动
================

DPDK包含了1G、10G、40G比特和virtio半虚拟化轮询模式驱动。

轮询模式驱动(Poll Mode Driver, PMD)提供了一组API用于设备和队列的配置，以运行在用户态的BSD驱动方式提供。
另外，PMD不使用中断(除了链路状态变更中断)，而是直接访问设备的RX和TX描述符从而快速的接收，处理和传输数据包。
本节描述了PMD的要求，和以太网PMD的高级架构与通用外部API的设计原则和建议。

要求与假设
----------------------------

DPDK包处理应用有两种模型，run-to-completion 和 pipe-line:

*   在 *run-to-completion*  模型中，端口RX描述符ring通过API不断被轮询用于收包。
    然后接收的包在同一个核上被处理，处理完毕通过API再将包放到端口的TX描述符ring中用于传送。
	
*   在 *pipe-line*  模型中, 核心通过API轮询一个或更多端口的RX描述符ring。
    接收的数据包通过ring传递到另外一个核上继续处理，处理完的数据包可能会通过API被放到端口的TX描述符ring中发送。
	
在同步的 run-to-completion 模型中，每个逻辑核都会执行包处理循环，包括以下步骤:

*   通过PMD的接收API接收输入的包

*   每次只处理一个接收的包，直到转发包

*   通过PMD传输API发送输出包

相反地，在异步的 pipe-line 模型中，一些逻辑核专门用于包接收，另外一些逻辑核专门处理前面接收的包。
逻辑核间通过ring进行包的交换。包接收循环包括以下步骤:

*   通过PMD接收API接收输入包

*   通过包队列把接收的包传给专门处理包的核

包处理循环:

*   从包队列中接收包

*   处理接收的包直到重传

为了避免任何非必要的中断处理开销，执行环境中不允许使用任何异步通知机制。
必要和合理的异步通信应该尽可能通过使用ring的方式实现。

多核环境中锁竞争是个关键问题。为了解决这个问题，PMD设计尽可能多的使用 per-core 私有资源工作。
例如，PMD为每个端口在每个核上维护了分开的传输队列。
同时，端口的每个接收队列也分配给一个单独的逻辑核(lcore)轮询。

为了遵守Non-Uniform Memory Access (NUMA)，内存管理被设计成在每个逻辑核本地分配一个私有的缓冲区池，使远程内存访问降低到最少。
包缓冲区池的配置应该考虑底层物理内存的DIMM，channel和rank方面的架构。
应用必须确保在创建内存池时提供正确的参数。参见 :ref:`Mempool 库 <Mempool_Library>`

设计原则
-----------------

Ethernet* PMD的API和架构设计考虑到如下的指导方针。

PMD必须能帮助上层应用执行全局策略。反方面，NIC PMD函数不应该阻碍上层的全局策略的执行。

例如，PMD的接收和传输函数都轮询最大数量的包/描述符。
这允许静态提供run-to-completion处理栈或者通过不同的全局循环策略动态适应，比如：

*   一次只接收处理和传输一个包的零碎方式。

*   尽可能多的接收包，然后一起处理和传输所有接收的包。

*   接收一定数量的包，处理接收的包，最后把全部处理完成的包一起传输。

为了获得最佳性能，必须考虑总体软件设计和纯软件优化技术，同时兼顾低级的基于硬件的优化(CPU缓存性能，总线速率，NIC PCI带宽等等)。
包传输中的猝发式(burst-oriented)网络包传输引擎就是一个软硬件折中优化的例子。
在最初的情况下，PMD仅提供了一个 rte_eth_tx_one 函数用于在一个队列上一次传输一个包的函数。
除此以外，可以很容易创建一个 rte_eth_tx_burst 函数，该函数一次调用可以循环调用rte_eth_tx_one函数传输多个包。
然而，rte_eth_tx_burst 实际是由PMD实现的，其通过下面的优化手段最小化每个包的驱动层面的传输开销:

*   在多个包之间共享调用 rte_eth_tx_one 函数的未摊销开销(un-amortized cost)。

*   利用猝发式硬件特性(数据预取到缓存，NIC的头/尾寄存器的使用)使 rte_eth_tx_burst 函数最小化每个包处理的CPU周期数，
    例如，避免不必要的ring传输描述符的内存读取，或者有组织的使用指针数组。* (原文: Enable the rte_eth_tx_burst function to take advantage of burst-oriented hardware features (prefetch data in cache, use of NIC head/tail registers)
    to minimize the number of CPU cycles per packet, for example by avoiding unnecessary read memory accesses to ring transmit descriptors,
    or by systematically using arrays of pointers that exactly fit cache line boundaries and sizes.) *

*   应用猝发式软件优化技术，消除不可避免的操作，比如，ring索引回绕管理。

猝发式函数也被PMD广泛使用的服务API引入。
这尤其适用于用于填充NIC ring的缓冲区分配器，这种缓冲区分配器可以一次分配或释放多个缓冲区。
比如，mbuf_multiple_alloc 函数在接收包时就会返回一组指向 rte_mbuf 缓冲区的指针，其提高了PMD接收轮询函数的速度。

逻辑核，内存和NIC队列的关系
--------------------------------------------------

DPDK支持NUMA意味着当处理器的逻辑核或者接口利用本地内存时会获得更好的性能。
因此，与本地 PCIe* 接口相关的mbuf申请应该从本地内存池中申请。
如果可能的话，缓冲区应该保持在本地处理器上，这样可以获得最佳的性能。并且RX和TX缓冲区应该从本地内存中申请。

run-to-completion 模型在处理本地内存数据也会表现地更好。对于所有的逻辑核处于同一个处理器的 pipe-line 模型同样适用。

多个逻辑核不应该共享接收或者传输队列，因为这需要全局锁，必然会导致性能下降。

设备标识和配置
---------------------------------------

设备标识
~~~~~~~~~~~~~~~~~~~~~

每个NIC端口都是由PCI标识符(bus/bridge, device, function)唯一确定，P
CI标识符是在DPDK初始化时通过PCI probing/enumeration函数分配的。
在PCI标识符基础上，NIC端口又被分配了两个其他的标识符:

*   PMD API函数中用于指定NIC端口的端口索引。

*   控制台消息中用于指定端口的端口名，可用于管理或者调试。为了简化使用，端口名中包含了端口索引。

设备配置
~~~~~~~~~~~~~~~~~~~~

NIC端口的配置包括以下操作:

*   申请PCI资源

*   把硬件重置(发布一个全局重置)到默认状态

*   设置物理设备和链路(Set up the PHY and the link)

*   初始化统计计数器

PMD API也必须提供用于启动/停止端口的 all-multicast 特性，和设置/取消端口混杂模式的函数。

一些硬件卸载特性必须在端口初始化时通过指定的配置参数单独地配置。
比如接收端扩展(RSS)和数据中心桥(DCB)特性。

热配置(On-the-Fly Configuration)
~~~~~~~~~~~~~~~~~~~~~~~~

设备所有可以热配置(也就是，在不停止设备情况下配置)的特性不需要PMD API提供专门的函数设置。

需要的是设备PCI寄存器的映射地址，驱动程序外的函数使用该地址可以配置这些特性。

为此，PMD API提供了一个函数，该函数可在驱动程序外获取设备信息(包括PCI厂商标识符，PCI设备标识符，PCI设备寄存器映射地址和驱动名称)，
这些信息可用于设置设备的特性。

这种方式的优势是可以给予特性配置，启用和关闭API充分的自由。

举例，参考testpmd应用中，Intel® 82576 和 Intel® 82599 的 IEEE1588 特性配置。

其他的特性如端口的 L3/L4 5-Tuple 包过滤也可以以同样方式配置。
以太网的流控(帧暂停)可以配置在每个端口上。详细信息参考 testpmd 源码。
同样，NIC 的 L4 (UDP/TCP/ SCTP)校验和卸载也能根据单独的包开启(只要mbuf配置正确)。详细参考 `Hardware Offload`_ 。

传输队列的配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each transmit queue is independently configured with the following information:

*   The number of descriptors of the transmit ring

*   The socket identifier used to identify the appropriate DMA memory zone from which to allocate the transmit ring in NUMA architectures

*   The values of the Prefetch, Host and Write-Back threshold registers of the transmit queue

*   The *minimum* transmit packets to free threshold (tx_free_thresh).
    When the number of descriptors used to transmit packets exceeds this threshold, the network adaptor should be checked to see if it has written back descriptors.
    A value of 0 can be passed during the TX queue configuration to indicate the default value should be used.
    The default value for tx_free_thresh is 32.
    This ensures that the PMD does not search for completed descriptors until at least 32 have been processed by the NIC for this queue.

*   The *minimum*  RS bit threshold. The minimum number of transmit descriptors to use before setting the Report Status (RS) bit in the transmit descriptor.
    Note that this parameter may only be valid for Intel 10 GbE network adapters.
    The RS bit is set on the last descriptor used to transmit a packet if the number of descriptors used since the last RS bit setting,
    up to the first descriptor used to transmit the packet, exceeds the transmit RS bit threshold (tx_rs_thresh).
    In short, this parameter controls which transmit descriptors are written back to host memory by the network adapter.
    A value of 0 can be passed during the TX queue configuration to indicate that the default value should be used.
    The default value for tx_rs_thresh is 32.
    This ensures that at least 32 descriptors are used before the network adapter writes back the most recently used descriptor.
    This saves upstream PCIe* bandwidth resulting from TX descriptor write-backs.
    It is important to note that the TX Write-back threshold (TX wthresh) should be set to 0 when tx_rs_thresh is greater than 1.
    Refer to the Intel® 82599 10 Gigabit Ethernet Controller Datasheet for more details.

The following constraints must be satisfied for tx_free_thresh and tx_rs_thresh:

*   tx_rs_thresh must be greater than 0.

*   tx_rs_thresh must be less than the size of the ring minus 2.

*   tx_rs_thresh must be less than or equal to tx_free_thresh.

*   tx_free_thresh must be greater than 0.

*   tx_free_thresh must be less than the size of the ring minus 3.

*   For optimal performance, TX wthresh should be set to 0 when tx_rs_thresh is greater than 1.

One descriptor in the TX ring is used as a sentinel to avoid a hardware race condition, hence the maximum threshold constraints.

.. note::

    When configuring for DCB operation, at port initialization, both the number of transmit queues and the number of receive queues must be set to 128.

Free Tx mbuf on Demand
~~~~~~~~~~~~~~~~~~~~~~

Many of the drivers do not release the mbuf back to the mempool, or local cache,
immediately after the packet has been transmitted.
Instead, they leave the mbuf in their Tx ring and
either perform a bulk release when the ``tx_rs_thresh`` has been crossed
or free the mbuf when a slot in the Tx ring is needed.

An application can request the driver to release used mbufs with the ``rte_eth_tx_done_cleanup()`` API.
This API requests the driver to release mbufs that are no longer in use,
independent of whether or not the ``tx_rs_thresh`` has been crossed.
There are two scenarios when an application may want the mbuf released immediately:

* When a given packet needs to be sent to multiple destination interfaces
  (either for Layer 2 flooding or Layer 3 multi-cast).
  One option is to make a copy of the packet or a copy of the header portion that needs to be manipulated.
  A second option is to transmit the packet and then poll the ``rte_eth_tx_done_cleanup()`` API
  until the reference count on the packet is decremented.
  Then the same packet can be transmitted to the next destination interface.
  The application is still responsible for managing any packet manipulations needed
  between the different destination interfaces, but a packet copy can be avoided.
  This API is independent of whether the packet was transmitted or dropped,
  only that the mbuf is no longer in use by the interface.

* Some applications are designed to make multiple runs, like a packet generator.
  For performance reasons and consistency between runs,
  the application may want to reset back to an initial state
  between each run, where all mbufs are returned to the mempool.
  In this case, it can call the ``rte_eth_tx_done_cleanup()`` API
  for each destination interface it has been using
  to request it to release of all its used mbufs.

To determine if a driver supports this API, check for the *Free Tx mbuf on demand* feature
in the *Network Interface Controller Drivers* document.

Hardware Offload
~~~~~~~~~~~~~~~~

Depending on driver capabilities advertised by
``rte_eth_dev_info_get()``, the PMD may support hardware offloading
feature like checksumming, TCP segmentation or VLAN insertion.

The support of these offload features implies the addition of dedicated
status bit(s) and value field(s) into the rte_mbuf data structure, along
with their appropriate handling by the receive/transmit functions
exported by each PMD. The list of flags and their precise meaning is
described in the mbuf API documentation and in the in :ref:`Mbuf Library
<Mbuf_Library>`, section "Meta Information".

Poll Mode Driver API
--------------------

Generalities
~~~~~~~~~~~~

By default, all functions exported by a PMD are lock-free functions that are assumed
not to be invoked in parallel on different logical cores to work on the same target object.
For instance, a PMD receive function cannot be invoked in parallel on two logical cores to poll the same RX queue of the same port.
Of course, this function can be invoked in parallel by different logical cores on different RX queues.
It is the responsibility of the upper-level application to enforce this rule.

If needed, parallel accesses by multiple logical cores to shared queues can be explicitly protected by dedicated inline lock-aware functions
built on top of their corresponding lock-free functions of the PMD API.

Generic Packet Representation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A packet is represented by an rte_mbuf structure, which is a generic metadata structure containing all necessary housekeeping information.
This includes fields and status bits corresponding to offload hardware features, such as checksum computation of IP headers or VLAN tags.

The rte_mbuf data structure includes specific fields to represent, in a generic way, the offload features provided by network controllers.
For an input packet, most fields of the rte_mbuf structure are filled in by the PMD receive function with the information contained in the receive descriptor.
Conversely, for output packets, most fields of rte_mbuf structures are used by the PMD transmit function to initialize transmit descriptors.

The mbuf structure is fully described in the :ref:`Mbuf Library <Mbuf_Library>` chapter.

Ethernet Device API
~~~~~~~~~~~~~~~~~~~

The Ethernet device API exported by the Ethernet PMDs is described in the *DPDK API Reference*.

Extended Statistics API
~~~~~~~~~~~~~~~~~~~~~~~

The extended statistics API allows a PMD to expose all statistics that are
available to it, including statistics that are unique to the device.
Each statistic has three properties ``name``, ``id`` and ``value``:

* ``name``: A human readable string formatted by the scheme detailed below.
* ``id``: An integer that represents only that statistic.
* ``value``: A unsigned 64-bit integer that is the value of the statistic.

Note that extended statistic identifiers are
driver-specific, and hence might not be the same for different ports.
The API consists of various ``rte_eth_xstats_*()`` functions, and allows an
application to be flexible in how it retrieves statistics.

Scheme for Human Readable Names
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A naming scheme exists for the strings exposed to clients of the API. This is
to allow scraping of the API for statistics of interest. The naming scheme uses
strings split by a single underscore ``_``. The scheme is as follows:

* direction
* detail 1
* detail 2
* detail n
* unit

Examples of common statistics xstats strings, formatted to comply to the scheme
proposed above:

* ``rx_bytes``
* ``rx_crc_errors``
* ``tx_multicast_packets``

The scheme, although quite simple, allows flexibility in presenting and reading
information from the statistic strings. The following example illustrates the
naming scheme:``rx_packets``. In this example, the string is split into two
components. The first component ``rx`` indicates that the statistic is
associated with the receive side of the NIC.  The second component ``packets``
indicates that the unit of measure is packets.

A more complicated example: ``tx_size_128_to_255_packets``. In this example,
``tx`` indicates transmission, ``size``  is the first detail, ``128`` etc are
more details, and ``packets`` indicates that this is a packet counter.

Some additions in the metadata scheme are as follows:

* If the first part does not match ``rx`` or ``tx``, the statistic does not
  have an affinity with either receive of transmit.

* If the first letter of the second part is ``q`` and this ``q`` is followed
  by a number, this statistic is part of a specific queue.

An example where queue numbers are used is as follows: ``tx_q7_bytes`` which
indicates this statistic applies to queue number 7, and represents the number
of transmitted bytes on that queue.

API Design
^^^^^^^^^^

The xstats API uses the ``name``, ``id``, and ``value`` to allow performant
lookup of specific statistics. Performant lookup means two things;

* No string comparisons with the ``name`` of the statistic in fast-path
* Allow requesting of only the statistics of interest

The API ensures these requirements are met by mapping the ``name`` of the
statistic to a unique ``id``, which is used as a key for lookup in the fast-path.
The API allows applications to request an array of ``id`` values, so that the
PMD only performs the required calculations. Expected usage is that the
application scans the ``name`` of each statistic, and caches the ``id``
if it has an interest in that statistic. On the fast-path, the integer can be used
to retrieve the actual ``value`` of the statistic that the ``id`` represents.

API Functions
^^^^^^^^^^^^^

The API is built out of a small number of functions, which can be used to
retrieve the number of statistics and the names, IDs and values of those
statistics.

* ``rte_eth_xstats_get_names_by_id()``: returns the names of the statistics. When given a
  ``NULL`` parameter the function returns the number of statistics that are available.

* ``rte_eth_xstats_get_id_by_name()``: Searches for the statistic ID that matches
  ``xstat_name``. If found, the ``id`` integer is set.

* ``rte_eth_xstats_get_by_id()``: Fills in an array of ``uint64_t`` values
  with matching the provided ``ids`` array. If the ``ids`` array is NULL, it
  returns all statistics that are available.


Application Usage
^^^^^^^^^^^^^^^^^

Imagine an application that wants to view the dropped packet count. If no
packets are dropped, the application does not read any other metrics for
performance reasons. If packets are dropped, the application has a particular
set of statistics that it requests. This "set" of statistics allows the app to
decide what next steps to perform. The following code-snippets show how the
xstats API can be used to achieve this goal.

First step is to get all statistics names and list them:

.. code-block:: c

    struct rte_eth_xstat_name *xstats_names;
    uint64_t *values;
    int len, i;

    /* Get number of stats */
    len = rte_eth_xstats_get_names_by_id(port_id, NULL, NULL, 0);
    if (len < 0) {
        printf("Cannot get xstats count\n");
        goto err;
    }

    xstats_names = malloc(sizeof(struct rte_eth_xstat_name) * len);
    if (xstats_names == NULL) {
        printf("Cannot allocate memory for xstat names\n");
        goto err;
    }

    /* Retrieve xstats names, passing NULL for IDs to return all statistics */
    if (len != rte_eth_xstats_get_names_by_id(port_id, xstats_names, NULL, len)) {
        printf("Cannot get xstat names\n");
        goto err;
    }

    values = malloc(sizeof(values) * len);
    if (values == NULL) {
        printf("Cannot allocate memory for xstats\n");
        goto err;
    }

    /* Getting xstats values */
    if (len != rte_eth_xstats_get_by_id(port_id, NULL, values, len)) {
        printf("Cannot get xstat values\n");
        goto err;
    }

    /* Print all xstats names and values */
    for (i = 0; i < len; i++) {
        printf("%s: %"PRIu64"\n", xstats_names[i].name, values[i]);
    }

The application has access to the names of all of the statistics that the PMD
exposes. The application can decide which statistics are of interest, cache the
ids of those statistics by looking up the name as follows:

.. code-block:: c

    uint64_t id;
    uint64_t value;
    const char *xstat_name = "rx_errors";

    if(!rte_eth_xstats_get_id_by_name(port_id, xstat_name, &id)) {
        rte_eth_xstats_get_by_id(port_id, &id, &value, 1);
        printf("%s: %"PRIu64"\n", xstat_name, value);
    }
    else {
        printf("Cannot find xstats with a given name\n");
        goto err;
    }

The API provides flexibility to the application so that it can look up multiple
statistics using an array containing multiple ``id`` numbers. This reduces the
function call overhead of retrieving statistics, and makes lookup of multiple
statistics simpler for the application.

.. code-block:: c

    #define APP_NUM_STATS 4
    /* application cached these ids previously; see above */
    uint64_t ids_array[APP_NUM_STATS] = {3,4,7,21};
    uint64_t value_array[APP_NUM_STATS];

    /* Getting multiple xstats values from array of IDs */
    rte_eth_xstats_get_by_id(port_id, ids_array, value_array, APP_NUM_STATS);

    uint32_t i;
    for(i = 0; i < APP_NUM_STATS; i++) {
        printf("%d: %"PRIu64"\n", ids_array[i], value_array[i]);
    }


This array lookup API for xstats allows the application create multiple
"groups" of statistics, and look up the values of those IDs using a single API
call. As an end result, the application is able to achieve its goal of
monitoring a single statistic ("rx_errors" in this case), and if that shows
packets being dropped, it can easily retrieve a "set" of statistics using the
IDs array parameter to ``rte_eth_xstats_get_by_id`` function.
