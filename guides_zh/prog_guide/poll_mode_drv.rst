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

每个传输队列的配置都是独立的，可配置的信息有:

*   传输队列描述符数量

*   socket标识符，该传输队列ring的内存所在socket

*   传输队列的预取(Prefetch)、主机(Host)和回写阀值(Write-Back threshold)寄存器值

*   传输包释放最小阀值(tx_free_thresh)。
    当传输包的描述符数量到达阀值，网络适配器应该检查是否有回写描述符。
    配置TX队列时可以传0使用默认阀值。默认 tx_free_thresh 是32。
    这可以确保PMD在处理了32个描述符后再搜寻由NIC处理完成的描述符。

*   RS位最小阀值。设置传输描述符的 报告状态(RS) 位需要的最小传输描述符数。
    注意，该参数仅对Intel 10 GbE 网络适配器有效。
    如果从上一个设置了RS位的描述符到传输包的第一个描述符之间的描述符个数达到了传输RS为阀值(tx_rs_thresh)，
    那就为传输包的最后一个描述符设置RS位。
    简言之，该参数控制了哪个传输描述符被网络适配器回写到主机内存。
    配置TX队列时可以传0使用默认阀值。默认 tx_rs_thresh 是32。
    这可以确保在网络适配器回写最近使用的描述符前至少使用了32个描述符。
    TX描述符回写能够节省上游 PCIe* 带宽。
    特别要注意的是，TX回写阀值(TX wthresh)在 tx_rs_thresh 比1大时应该设置为0。
    详细参考 Intel® 82599 10 Gigabit Ethernet Controller 手册
	
tx_free_thresh 和 tx_rs_thresh 必须满足以下限制:

*   tx_rs_thresh 大于0

*   tx_rs_thresh 小于 size(ring) - 2

*   tx_rs_thresh 小于或等于 tx_free_thresh

*   tx_free_thresh 大于0

*   tx_free_thresh 小于 size(ring) - 3

*   为了获得最佳性能，TX回写阀值(TX wthresh)在 tx_rs_thresh 比1大时应该设置为0

TX ring中的一个描述符作为哨兵，用于消除硬件竞争条件，因为存在最大阀值限制。

.. note::

    在端口初始化配置DCB时，传输队列和接收队列数都必须设置为128。

按需释放Tx mbuf
~~~~~~~~~~~~~~~~~~~~~~

很多驱动在包传输完成后不会立即把mbuf释放会内存池或者本地缓存。
而是，放在Tx ring中，要么在到达 ``tx_rs_thresh`` 时执行批量释放，要么在Tx ring空间不足时释放。

应用可以通过 ``rte_eth_tx_done_cleanup()`` API 让驱动释放已用过的mbuf。
该API让驱动释放不再使用的mbuf，且不依赖 ``tx_rs_thresh`` 是否到达。
应用要求立即释放mbuf的场景有两个:

* 数据包需要发送到多个目的接口(二层泛洪或者三层多播)。
  一种选择是拷贝数据包或者拷贝需要操作的包头部分。
  第二种选择是传输数据包，然后轮询 ``rte_eth_tx_done_cleanup()`` API，直到包的引用计数减少。
  然后同一个数据包可以传输到下一个目的接口。
  应用仍需处理数据包发送到不同目的接口的操作，但是可以省去包拷贝操作。
  该API不在乎包是被传输了或者被丢弃，只知道网络接口不再使用该mbuf。

* 有些应用设计成多次运行，比如包生成器。
  为了性能和不同运行之间一致性，应用在不同运行之间可能会需要重置到初始状态，
  初始状态所有的mbuf都归还到内存池。
  这种情况下，可以为每个目的接口调用  ``rte_eth_tx_done_cleanup()`` API让其释放用过的mbuf。

可以在 *Network Interface Controller Drivers* 文档中查看 *Free Tx mbuf on demand* 特性来判断驱动是否支持该API。

硬件卸载(Hardware Offload)
~~~~~~~~~~~~~~~~

依赖 ``rte_eth_dev_info_get()`` 获得的驱动能力，PMD可以支持像校验和、TCP分段和VLAN嵌入(VLAN insertion)这样的硬件卸载特性。

这些硬件卸载特性的支持依赖rte_mbuf结构中专用的状态位和PMD的接收和传输函数的正确处理。
标志位列表和详细描述可以在 mbuf API文档中和 :ref:`Mbuf 库 <Mbuf_Library>`, 的 "元信息"节中找到。

轮询模式驱动API
--------------------

概论
~~~~~~~~~~~~

默认，PMD导出的函数都是无锁函数(即假定这些函数不会用于不同核同时操作同一个对象)。
比如，PMD接收函数不能被两个核同时用于在同一个端口的同一个接收队列上接收数据。
当然，该函数可以被不同核同时用于不同接收队列上接收数据。
这要上层应用强制遵守该规则。

如果需要多核并发访问共享队列的话，可以调用基于PMD无锁函数构建的专用的内联有锁函数保护共享队列。

通用数据包表示
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

数据包使用rte_mbuf结构体表示，该结构体是包含了所有必需的管理信息的通用元数据结构体。
其包含了和硬件卸载特性相关的字段和状态位，如IP头的校验和计算或者VLAN标签。

rte_mbuf数据结构以一种通用的方式包含了由网络控制器提供的卸载特性。
对于输入包，rte_mbuf 结构体的大部分字段由PMD接收函数根据接收描述符中包含的信息填充。
相反，对于输出包，rte_mbuf 结构体的大部分字段被PMD传输函数用来初始化传输描述符。

mbuf结构体完整描述 :ref:`Mbuf 库 <Mbuf_Library>`

以太网设备API
~~~~~~~~~~~~~~~~~~~

PMD导出的以太网设备API请参考  *DPDK API Reference*

扩展统计API
~~~~~~~~~~~~~~~~~~~~~~~

扩展统计API允许PMD暴露所有可用的统计，包括设备特有的统计。
每个统计项有三个属性 ``name``, ``id`` 和 ``value``:

* ``name``: 按照某种方案定义的易于阅读的字符串
* ``id``: 仅代表某种统计项的整数
* ``value``: 无符号64位整数的统计值

注意扩展统计是驱动特有的，所以不同端口可能会有不同的扩展统计。
该API由各种  ``rte_eth_xstats_*()`` 函数组成，并且允许应用灵活接收统计信息。

易于阅读的名称定义方案
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

对于统计项暴露给API客户端的名称是有具体的命名方案的。这允许API抓取感兴趣的统计项。
该命名方案中使用一个下划线 ``_`` 分割字符串。方案如下:

* 方向
* 细节 1
* 细节 2
* 细节 n
* 单位

遵守上述命名方案的统计项名称的例子:

* ``rx_bytes``
* ``rx_crc_errors``
* ``tx_multicast_packets``

这种方案简单，信息展示灵活。比如 ``rx_packets``，分成两部分，
第一部分 ``rx`` 指明该统计项统计的是NIC的接收侧。第二部分 ``packets`` 指明统计单位是包。

一个复杂的例子 ``tx_size_128_to_255_packets``，``tx`` 指明是传输, ``size``  第一个统计细节, 
``128`` 等是细节的细节 ``packets`` 指明这是个包计数器。

其他:

* 如果名称中第一部分既不是 ``rx`` 也不是 ``tx``，那么该统计项就和接收或传输没有关系。

* 如果第二部分的第一个字母是 ``q``，并且 ``q`` 后面紧跟着一个数字，则该统计项是指定队列的统计。

使用队列号的例子: ``tx_q7_bytes`` 是队列7传输的字节数统计信息。

API设计
^^^^^^^^^^

The xstats API uses the ``name``, ``id``, and ``value`` to allow performant
lookup of specific statistics. Performant lookup means two things;
统计API使用 ``name``, ``id``, 和 ``value`` 执行特定统计项的查询。查询意味着两件事:

* 快速路径中没有统计项名称的字符串比较

* 允许仅查询感兴趣的统计项

API通过把 ``name`` 映射到唯一 ``id`` 保证满足要求。``id`` 用作快速路径查找中的key。
API允许应用请求一组 ``id``，以便PMD仅执行请求的计算。预期的使用方式是应用扫描每个统计项的 ``name``,
并缓存感兴趣统计项的 ``id``。在快速路径中，可以通过整数 ``id`` 来获取代表的统计项实际 ``value``。

API函数
^^^^^^^^^^^^^

扩展统计API是由少量函数组成的，这些函数可用于获取统计项数量和这些统计项的名称，ID以及值。

* ``rte_eth_xstats_get_names_by_id()``: 返回统计项数量. 当给该函数传递 ``NULL`` 参数时，
  该函数返回所有可用统计项数量。

* ``rte_eth_xstats_get_id_by_name()``: 根据名称搜索ID，如果找到了就设置整数 ``id``。

* ``rte_eth_xstats_get_by_id()``: 根据提供的 ``id`` 数组，填充对应的 ``uint64_t`` 值数组。
  如果 ``id`` 数组是NULL，则返回所有可用的统计项。


应用的使用方法
^^^^^^^^^^^^^^^^^

假设应用想要查看丢包数。如果没有丢包，因为性能原因应用不会读取任何其他指标。
如果有丢包，应用会有个特有的统计项集合。该统计项集合允许应用决定下一步执行什么。
下面的代码片段展示了扩展统计API如何达到这个目标。

第一步获取所有统计名称并列出来:

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

应用已经获得了PMD暴露的所有统计项名称。应用可以决定哪些统计项是感兴趣的，
并通过这些统计项名称查询并缓存id:

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

API给应用提供了极大灵活性，因此应用可以通过 ``id`` 数组查询多个统计项。
这可以降低获取统计项的函数调用开销，并使应用的多统计项查询更加容易。

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


扩展统计API的成组查询允许应用创建“统计组”，统计组里的这些ID对应的值使用一个API调用即可查询到。
最终，应用可以通过以下方式达成目标，应用先不断监测单个统计项(本例中就是"rx_errors")，
如果该统计项显示出有丢包发生，则应用通过给函数 ``rte_eth_xstats_get_by_id`` 传递一组统计项ID，
从而获取更多的统计信息。

