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

.. _Mbuf_Library:

Mbuf库
============

mbuf库用于申请和释放DPDK应用用于存储消息的缓冲区(mbuf)。消息缓冲区存储于mempool中，使用了 :ref:`Mempool 库 <Mempool_Library>`。

rte_mbuf结构体能够承载网络包缓冲区和通用控制缓冲区(由CTRL_MBUF_FLAG标识)，也能被扩展到其他类型。
rte_mbuf头结构尽可能的小，当前仅使用了两个cache-line，但大多数时候仅使用第一个cache-line。

包缓冲区的设计
------------------------

对于包数据(包括协议头)的存储，有两种方法:

#.  元数据结构和包数据使用同一个内存缓冲区

#.  元数据结构和包数据使用不同的内存缓冲区

第一种的优势是对于一个包的内存申请/释放仅需要一次。而第二种则更加灵活，元数据结构的申请和数据包的申请完全分离。

DPDK中选择第一种方法。元数据中包含控制信息，如消息类型，长度，数据偏移，指向另外mbuf的指针（用于缓冲区链）

消息缓冲区是用于承载缓冲区链的网络包的，缓冲区链中需要多个缓冲区才能组成一个完整的包。
比如由多个mbuf组成的巨帧(mbuf之间通过next字段连接在一起)。

对于新申请的mbuf，数据区是在缓冲区起始地址后 RTE_PKTMBUF_HEADROOM 字节处，数据区地址是缓存对齐的。
消息缓冲区可用于在不同实体间承载控制信息，包，事件等。
消息缓冲区中的缓冲区指针也指向其他消息缓冲区数据段或结构体。

:numref:`figure_mbuf1` 和 :numref:`figure_mbuf2` 展示了一些场景

.. _figure_mbuf1:

.. figure:: img/mbuf1.*

   一个段的mbuf


.. _figure_mbuf2:

.. figure:: img/mbuf2.*

   三个段的mbuf


缓冲区管理器实现了一组相当标准的缓冲区访问函数来操作网络数据包。

存储在内存池中的缓冲区
------------------------------

缓冲区管理器使用 :ref:`Mempool 库 <Mempool_Library>` 申请缓冲区。
因此，可以确保包头在L3处理时能够最优的分布在不同内存通道。
mbuf中包含一个指示内存池的字段。调用 rte_ctrlmbuf_free(m) 或者 rte_pktmbuf_free(m)时，mbuf被放回原始的内存池中。

构造器
------------

API提供了数据包和控制消息mbuf的构造器。rte_pktmbuf_init() 和 rte_ctrlmbuf_init() 初始化了mbuf结构体中的部分字段，
这些字段用户不会修改(如mbuf类型，来源池，缓冲区起始地址等)。在创建池的时候，该函数作为回调函数传递给rte_mempool_create()。

申请和释放mbuf
----------------------------

申请mbuf时需要用户指定内存池，新申请的mbuf包含一个段，且长度为0。数据偏移初始化为头空间大小(RTE_PKTMBUF_HEADROOM)。

释放mbuf意味着把它归还到来源内存池。归还时mbuf中内容不会被修改(作为一个空闲mbuf)。
因此，再次申请该mbuf时，已由构造器初始化的字段无需重新初始化。

释放包含多个段的mbuf时，所有的段都会被归还到其来源内存池中。

操作mbuf
------------------

库提供了一些操作mbuf中数据的函数，例如:

    *  获取数据长度

    *  获取数据起始地址

    *  在数据段前插入数据

    *   数据段追加数据

    *   移除缓冲区起始的数据 (rte_pktmbuf_adj())

    *   移除缓冲区结尾的数据 (rte_pktmbuf_trim()) 详细参考 *DPDK API Reference* 。

元信息
----------------

网络驱动会获取一些信息并存储在mbuf中，这些信息有助于后续的包处理。
比如，VLAN，RSS哈希结果(参见 :ref:`轮询模式驱动 <Poll_Mode_Driver>`)，硬件已完成校验标识。

An mbuf also contains the input port (where it comes from), and the number of segment mbufs in the chain.
mbuf中也包含了输入端口(包来源)和链中mbuf段个数。

对于链缓冲区, 仅在链中第一个mbuf中存储元信息。

比如，IEEE1588接收上的包时间戳机制，VLAN标签和IP校验。

在发送上, 应用也可以让硬件处理一些硬件支持的操作。
比如，PKT_TX_IP_CKSUM标志从应用中卸载IPv4校验和计算，让硬件计算。

下面的例子解释了如何配置VXLAN封装的TX卸载(把VXLAN封装工作下移到硬件中):
``out_eth/out_ip/out_udp/vxlan/in_eth/in_ip/in_tcp/payload``

- 计算out_ip的校验和::

    mb->l2_len = len(out_eth)
    mb->l3_len = len(out_ip)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM
    set out_ip checksum to 0 in the packet

  This is supported on hardware advertising DEV_TX_OFFLOAD_IPV4_CKSUM.

- 计算out_ip 和 out_udp的校验和::

    mb->l2_len = len(out_eth)
    mb->l3_len = len(out_ip)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM | PKT_TX_UDP_CKSUM
    set out_ip checksum to 0 in the packet
    set out_udp checksum to pseudo header using rte_ipv4_phdr_cksum()

  This is supported on hardware advertising DEV_TX_OFFLOAD_IPV4_CKSUM
  and DEV_TX_OFFLOAD_UDP_CKSUM.

- 计算in_ip的校验和::

    mb->l2_len = len(out_eth + out_ip + out_udp + vxlan + in_eth)
    mb->l3_len = len(in_ip)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM
    set in_ip checksum to 0 in the packet

  This is similar to case 1), but l2_len is different. It is supported
  on hardware advertising DEV_TX_OFFLOAD_IPV4_CKSUM.
  Note that it can only work if outer L4 checksum is 0.

- 计算in_ip 和 in_tcp的校验和::

    mb->l2_len = len(out_eth + out_ip + out_udp + vxlan + in_eth)
    mb->l3_len = len(in_ip)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM | PKT_TX_TCP_CKSUM
    set in_ip checksum to 0 in the packet
    set in_tcp checksum to pseudo header using rte_ipv4_phdr_cksum()

  This is similar to case 2), but l2_len is different. It is supported
  on hardware advertising DEV_TX_OFFLOAD_IPV4_CKSUM and
  DEV_TX_OFFLOAD_TCP_CKSUM.
  Note that it can only work if outer L4 checksum is 0.

- segment inner TCP::

    mb->l2_len = len(out_eth + out_ip + out_udp + vxlan + in_eth)
    mb->l3_len = len(in_ip)
    mb->l4_len = len(in_tcp)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CKSUM | PKT_TX_TCP_CKSUM |
      PKT_TX_TCP_SEG;
    set in_ip checksum to 0 in the packet
    set in_tcp checksum to pseudo header without including the IP
      payload length using rte_ipv4_phdr_cksum()

  This is supported on hardware advertising DEV_TX_OFFLOAD_TCP_TSO.
  Note that it can only work if outer L4 checksum is 0.

- 计算out_ip, in_ip, in_tcp的校验和::

    mb->outer_l2_len = len(out_eth)
    mb->outer_l3_len = len(out_ip)
    mb->l2_len = len(out_udp + vxlan + in_eth)
    mb->l3_len = len(in_ip)
    mb->ol_flags |= PKT_TX_OUTER_IPV4 | PKT_TX_OUTER_IP_CKSUM  | \
      PKT_TX_IP_CKSUM |  PKT_TX_TCP_CKSUM;
    set out_ip checksum to 0 in the packet
    set in_ip checksum to 0 in the packet
    set in_tcp checksum to pseudo header using rte_ipv4_phdr_cksum()

  This is supported on hardware advertising DEV_TX_OFFLOAD_IPV4_CKSUM,
  DEV_TX_OFFLOAD_UDP_CKSUM and DEV_TX_OFFLOAD_OUTER_IPV4_CKSUM.

标志的列表和意义在mbuf API文档(rte_mbuf.h)中有详细描述。也可以参考testpmd源码(特别是 csumonly.c 文件)

.. _direct_indirect_buffer:

直接和间接缓冲区
---------------------------

直接缓冲区完全分开和独立的。间接缓冲区和直接缓冲区行为相似但是它的缓冲区指针和数据偏移却是指向另外一个直接缓冲区的。
间接缓冲区用于包复制和分段，因为间接缓冲区可以重用多个缓冲区的同一个包数据。

通过使用rte_pktmbuf_attach()函数可以让缓冲区“附加”到一个直接缓冲区上，操作完成后“附加”缓冲区就变成间接缓冲区。
每个缓冲区都有一个引用计数的字段，当间接缓冲区附加到直接缓冲区上时，直接缓冲区的引用计数增加。
类似地，当间接缓冲区和直接缓冲分离时，直接缓冲区的引用计数减少。
如果引用计数器等于0，意味着该缓冲区不会再被使用，释放它。

处理间接缓冲区时有些注意事项。
首先，间接缓冲区不能附加到间接缓冲区上，
尝试把缓冲区A附加到已经附加到缓冲区C上的间接缓冲区B上，rte_pktmbuf_attach()会自动地把A附加到C上，等于克隆了B。
其次，一个缓冲区要想变成间接的，他的引用计数必须等于1，也就是它不能被其他间接缓冲区引用。
最后，间接缓冲区不能重复附加到直接缓冲区(除非首先分离)

虽然附加/分离操作可以通过rte_pktmbuf_attach() 和 rte_pktmbuf_detach()被调用，但建议使用高级的 rte_pktmbuf_clone() 函数，
它会正确的初始化间接缓冲区，克隆多段的缓冲区。

由于间接缓冲区被认为没有持有任何数据，间接缓冲区内存池应该配置成减少内存占用。
间接缓冲区内存区初始化的示例(也就是间接缓冲区使用示例)在示例程序中可以找到，如IPv4多播示例程序。

调试
-----

调试模式(CONFIG_RTE_MBUF_DEBUG is enabled)下，mbuf库函数在执行任何操作前都要进行合法性(如，缓冲区损坏，错误类型等)检查。

使用案例
---------

所有网络应用都应该使用mbuf传输网络包。
