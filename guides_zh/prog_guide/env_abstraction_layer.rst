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

.. _Environment_Abstraction_Layer:

环境抽象层
=============================

环境抽象层（Environment Abstraction Layer，下文简称EAL）是对操作系统底层资源（如内存空间）的抽象，
用于DPDK应用程序访问底层资源。
EAL隐藏了不同操作系统访问底层资源的接口，
给DPDK应用程序提供了统一的访问接口。
EAL的初始化例程负责底层资源的申请，如内存，PCI设备等。

EAL中提供的典型服务有:

*	DPDK库的加载和启动:
	DPDK库和DPDK应用在编译阶段被链接成一个应用程序。而且库的加载需要一些额外的操作，这些操作由EAL完成，应用开发者无需特别关心。
	
*	CPU亲和性/分配:
	EAL能把一个执行单元分配到指定CPU上运行。
	
*	系统内存预留:
	EAL预留了各种内存，比如，用于设备交互的物理内存区域。
	
*	PCI地址抽象: EAL提供了访问PCI设备地址空间的接口。
	
*	跟踪和调试功能: 日志、dump堆栈、panic等等。
	
*	实用功能: 自旋锁和原子计数器（这类函数libc中没有提供）。
	
*	CPU特性标识: 在程序运行期间决定CPU是否支持指定的特性，如Intel® AVX。
	判断当前CPU是否和DPDK库所支持的CPU匹配。
	
*	中断处理: 注册/注销中断处理函数的接口。
	
*	Alarm Functions: Interfaces to set/remove callbacks to be run at a specific time.
	
Linux下的EAL
---------------------------------------------

在Linux中，DPDK程序以一个用户态程序运行，使用的线程库是pthread。
设备的PCI信息和地址使用linux的sysfs内核接口和内核模块（uio_pci_generic或者igb_uio）获取。
内存则是通过mmap映射到程序内存空间的。

EAL在hugetlbfs（巨页内存，提高性能） 中通过mmap()申请物理内存，这些内存是提供给DPDK服务层使用的，
如 :ref:`内存池库 <Mempool_Library>`.

在DPDK服务层初始化的时候，会调用线程亲和性设定函数（pthread提供），
让每一个执行单元绑定到一个逻辑CPU上，并让每个执行单元以一个用户线程运行。

时钟则是由CPU的TSC或者内核的HPET提供（通过mmap调用）。

EAL初始化和核心启动
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

glibc中的启动函数(入口函数)完成了程序初始化的一部分，也会检查当前CPU和DPDK程序的架构是否匹配。
然后主函数main()被调用。核心的初始化和启动是由rte_eal_init() 完成的。
它由一组pthread调用组成（具体有pthread_self(), pthread_create(), and pthread_setaffinity_np()）。

.. _figure_linuxapp_launch:

.. figure:: img/linuxapp_launch.*

   Linux环境下EAL的初始化


.. note::

	对象（如内存区域、ring、内存池、lpm表、哈希表）的初始化应该在程序初始化阶段在主核上完成，
	因为这些对象的创建和初始化函数不是线程安全的。
	但是，这些对象一旦初始化完成，对它们的使用是线程安全的。
	
多进程支持
~~~~~~~~~~~~~~~~~~~~~

Linux EAL允许以多进程模式开发应用，详细参考 :ref:`多进程支持 <Multi-process_Support>`

内存映射和内存预留
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

初始化阶段EAL会从hugetlbfs申请大量且地址连续的内存，
这些内存可以通过EAL提供的"内存区预留API"提供给上层应用使用。
API会把这个内存区对应的物理地址返回给用户。

.. note::

    内存预留使用rte_malloc提供的API。rte_malloc的内存也是从hugetlbfs文件系统中获取的。

对无hugetbls的Xen Dom0支持
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

目前的内存管理实现是基于Linux内核的巨页机制。但是，Xen Dom0不支持巨页，因此需要额外的rte_dom0_mm内核模块完成这项工作。

EAL使用IOCTL接口通知rte_dom0_mm内核模块申请指定大小内存，并获取所有内存段信息，然后EAL使用MMAP接口映射申请的内存。
对于每个内存段它的物理地址都是连续的，但是硬件地址是2MB连续的。

PCI访问
~~~~~~~~~~

EAL通过扫描/sys/bus/pci获取PCI总线上设备信息。
为了访问PCI设备内存，uio_pci_generic内核模块会提供/dev/uioX设备文件和sysfs资源文件，
通过这些就可以使用mmap把PCI设备内存映射到应用程序内存空间。
DPDK定制的模块igb_uio也可以用于此。两种模块都使用了uio这个内核特性（用户空间I/O）

Per-lcore和共享变量
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::
	
	lcore（逻辑核）指的是处理器的逻辑执行单元，有时也称为硬 *线程*
	
共享变量是在所有线程之间共享的，所有线程都可以访问。
Per-lcore变量则是线程本地存储，线程只能访问自己的Per-lcore变量，Per-lcore变量使用 *Thread Local Storage* (TLS) 实现。

日志
~~~~

EAL提供了日志API。
在Linux中，默认情况下，日志会被发送到syslog和控制台。
用户也可以覆盖这些日志函数，使用自定义的日志机制。

跟踪和调试功能
^^^^^^^^^^^^^^^^^^^^^^^^^

glibc中的调试函数可以把程序堆栈dump出来。
EAL提供的rte_panic()函数能够自动发出SIG_ABORT信号，这个信号能触发core文件的生成，
然后开发者可以通过gdb读取core文件排除错误。

CPU特性标识
~~~~~~~~~~~~~~~~~~~~~~~~~~

EAL能够在运行时查询CPU信息（rte_cpu_get_feature()函数）并判断哪些CPU特性可用。

用户空间中断事件
~~~~~~~~~~~~~~~~~~~~~~~~~~
.. _host_thread:

+ 主线程（Host Thread）的中断和告警的处理

EAL初始化时创建了一个专用的主线程用于检测中断（通过轮询UIO设备描述符/dev/uioX）。
开发者通过EAL提供的函数为指定中断事件注册/注销回调函数，事件发生时回调函数会被这个主线程异步调用。
对于NIC中断，EAL也提供了同样的定时回调。

.. note::

    在DPDK PMD(轮询模式驱动)中，主线程处理的中断事件只有链路状态变更（链路连接和断开通知）和设备意外移除事件。


+ Rx(接收)中断事件

PMD提供的数据包接收和发送例程允许在线程中轮询执行。
在网络吞吐量小的时候，为了降低空闲轮询可以先暂停轮询，然后等待一个“唤醒”事件的发生。
Rx中断事件可以作为首选“唤醒”事件，但也可能有其他事件作为“唤醒”事件。

EAL为事件驱动线程模式提供了事件API。
以Linux环境中的应用为例，它的事件驱动依赖于epoll。每个事件驱动的线程会监视一个epoll实例，
所关心的“唤醒”事件描述符会被加到这个epoll实例中。
事件描述符通过UIO/VFIO创建和映射到中断向量表中。
对于BSD应用，kqueue也是一种方式，只是目前还没有实现。

EAL会负责事件描述符和中断向量之间的映射，然而设备的队列和中断向量之间的映射是由设备自己完成的，
EAL无法感知到这些中断向量上面的中断事件，因此以太网设备驱动会负责把这些中断向量和事件描述符映射起来。
*（原文：EAL initializes the mapping between event file descriptors and interrupt vectors, while each device initializes the mapping
between interrupt vectors and queues. In this way, EAL actually is unaware of the interrupt cause on the specific vector.
The eth_dev driver takes responsibility to program the latter mapping.）*

.. note::

	每个队列的Rx中断事件仅在VFIO中可用（VFIO支持multiple MSI-X vector）。在UIO中，Rx中断和其他中断共享同一个中断向量，
	这种情况下，如果Rx中断和LSC(link status change)中断同时启用的话(intr_conf.lsc == 1 && intr_conf.rxq == 1)，只有前者有效。

Rx中断能够使用ethdev API进行控制/启用/关闭 - 'rte_eth_dev_rx_intr_*'。PMD不支持的操作返回失败。
intr_conf.rxq标志是用来开启设备Rx中断的。

+ 设备移除事件

当设备从总线上移除时会触发该事件。事件发生时其底层资源可能已经不可用了（也就是PCI映射解除）。
PMD要确保这种情况下，应用仍能够安全地使用它的回调。

设备移除事件的订阅和链接状态变更订阅一样。因此执行的环境也一样，也就是专门用于处理中断的 :ref:`主线程 <host_thread>` 。

考虑这样一种情况，应用程序去关闭一个已经发出设备移除事件的设备。这种情况下，
``rte_eth_dev_close()`` 调用会注销设备移除事件的回调。
要小心的是不要在中断处理上下文中关闭设备，应该通过其他方式去关闭设备。

黑名单
~~~~~~~~~~~~

PCI设备黑名单功能能够把特定NIC端口加入到黑名单中，DPDK会忽略黑名单中的设备。
黑名单中的端口通过PCIe*描述（Domain:Bus:Device.Function）标识。

其他
~~~~~~~~~~~~~~

Locks and atomic operations are per-architecture (i686 and x86_64).

内存段和内存区域(memzone)
------------------------------------------

物理内存映射是EAL的特性。物理内存其实会有间隙、不是连续的，因此需要使用内存描述符表，
其中存放的就是各个内存段的描述符（rte_memseg），每个描述符代表一段连续的物理内存。

除此以外，memzone分配器的任务是预留一段地址连续的物理内存。这些memzone在预留内存时以唯一的名称标识。

我们可以在应用的配置结构体（配置通过rte_eal_get_configuration()获取）中找到rte_memzone描述符表。
查找（通过名称）内存区域时返回的是包含该内存区域物理地址的描述符。

内存区域能够按照指定的对齐参数对齐预留（起始地址对齐， 默认cache line大小对齐）。
对齐大小应该是2的n次幂并且不小于cache line大小(64 bytes)。
内存区域也能够预留系统提供的两种可用的巨页（2MB和1GB）。

多线程
----------------

DPDK通常会在一个核上启动一个线程，避免线程切换的额外开销。
这会很显著地增加性能，但是缺乏灵活性并且不一定总是高效的。

我们可以通过电源管理限制CPU运行频率进而提升CPU效能。也可以把CPU的空闲周期(idle cycles)利用起来从而充分发挥CPU性能。

通过cgroup可以很容易地指定CPU利用率。这为CPU效率提升提供了另外一种方法，但是需要一个先决条件，
DPDK必须能处理每个核上面多个线程之间的上下文切换。

为了更加灵活，我们应该把线程的亲和性设置到一组而不是一个CPU上。

EAL pthread和lcore亲和性
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

术语"lcore"指的是EAL线程，事实上它是Linux/FreeBSD上的pthread。
"EAL pthreads"由EAL创建和管理，由 *remote_launch* 执行。
每一个EAL pthread都有一个叫 *_lcore_id* 的线程本地存储用于唯一标识一个线程。
因为通常pthreads和CPU是一对一地绑定，所以 *_lcore_id* 通常和CPU ID相等。

在使用多线程时，EAL线程和CPU不总是一对一绑定，EAL线程可能会对应一组CPU，这种情况下 *_lcore_id* 和CPU ID就不相等了。
为此，EAL提供了一个'--lcores'选项用于分配lcore的CPU亲和性。
你可以使用这个选项为一组lcore分配一组CPU。

参数格式:
	--lcores='<lcore_set>[@cpu_set][,<lcore_set>[@cpu_set],...]'

'lcore_set'和'cpu_set'可以是一个数，范围或者组。

数字: "digit([0-9]+)"; 范围: "<number>-<number>"; 组: "(<number|range>[,<number|range>,...])".

如果'\@cpu_set'没有提供, 默认和'lcore_set'相同。

    ::

    	比如, "--lcores='1,2@(5-7),(3-5)@(0,2),(0,6),7-8'" 启动9个线程;
    	    lcore 0 runs on cpuset 0x41 (cpu 0,6);
    	    lcore 1 runs on cpuset 0x2 (cpu 1);
    	    lcore 2 runs on cpuset 0xe0 (cpu 5,6,7);
    	    lcore 3,4,5 runs on cpuset 0x5 (cpu 0,2);
    	    lcore 6 runs on cpuset 0x41 (cpu 0,6);
    	    lcore 7 runs on cpuset 0x80 (cpu 7);
    	    lcore 8 runs on cpuset 0x100 (cpu 8).

使用这个选项，每个给定的lcore会分配给相关的CPU。
该选项和启用核列表选项'-l'兼容。

非EAL线程支持
~~~~~~~~~~~~~~~~~~~~~~~

在DPDK应用中用户可以创建线程（也就是非EAL线程）。
在非EAL线程中， *_lcore_id* 总是LCORE_ID_ANY。
由于很多基础库需要使用 *_lcore_id* ，因此在非EAL线程中，有的库会使用其他的唯一ID（比如，线程ID），
有的库则不受影响，还有些库会受限使用（比如，定时器和内存池库）。

所有的影响看这里 :ref:`known_issue_label`

公共线程API
~~~~~~~~~~~~~~~~~

``rte_thread_set_affinity()`` 和 ``rte_thread_get_affinity()`` 用于设置和获取与亲和性相关的TLS。

这些TLS包括 *_cpuset* 和 *_socket_id*:

*	*_cpuset* 存储的是CPU和线程绑定关系的位图。

*	*_socket_id* 存储的是CPU集合的NUMA节点。如果CPU集合中的CPU属于其他NUMA节点，那么 *_socket_id* 就设置为SOCKET_ID_ANY。


.. _known_issue_label:

已知问题
~~~~~~~~~~~~

+ rte_mempool

  rte_mempool在内存池中使用了per-lcore缓存。
  在非EAL线程中调用 ``rte_lcore_id()`` 会返回非法值。
  目前，当在非EAL线程中使用rte_mempool的put/get操作时，不会使用默认的内存池缓存，但这会导致性能下降。
  用户自己创建的缓存可以通过函数 ``rte_mempool_generic_put()`` 和 ``rte_mempool_generic_get()`` 在非EAL环境中使用。
  这两个函数会接收一个参数用于指定使用的缓存。
  
+ rte_ring

  rte_ring支持多生产者入队和多消费者出队并且是非抢占的，
  由于rte_mempool使用了rte_ring，所以这使得rte_mempool也是非抢占的。

  .. note::

    "非抢占"约束意味着:

    - 同一个ring，一个线程的多生产者入队操作不可被另一个线程多生产者入队操作抢占。
    - 同一个ring，一个线程的多消费者出队操作不可被另一个线程多消费者出队操作抢占。

    开启抢占会导致第二个线程一直自旋，直到第一个线程再次被调度执行。
    而且，如果第一个线程被高优先级的任务抢占可能会导致死锁。

  这并不意味着rte_ring无法使用，简单的说，应该尽量不要在同一个核心的多个线程上使用。

  1. 可以用于单生产者或单消费者的情况。
  2. 可以用于使用SCHED_OTHER(cfs)调度策略的多生产者/消费者线程中。 *注意：使用者应该意识到这会导致性能损耗*
  3. 禁止用于使用SCHED_FIFO或者SCHED_RR调度策略的多生产者/消费者线程中。

+ rte_timer

  不允许在非EAL线程中调用 ``rte_timer_manager()`` 。但是可以在非EAL线程中重置/停止定时器。

+ rte_log

  在非EAL线程中，只用全局日志等级可用，没有线程日志等级和日志类型可用。

+ 其他

  非EAL线程中不支持rte_ring、rte_mempool和rte_timer调试统计功能。

cgroup控制
~~~~~~~~~~~~~~

下面是一个使用cgroup控制使用率的简单例子，其中有两个线程（t0和t1）在同一个CPU（$cpu）上做包I/O操作。
我们希望CPU的50%的时间用于包IO。

  .. code-block:: console

    mkdir /sys/fs/cgroup/cpu/pkt_io
    mkdir /sys/fs/cgroup/cpuset/pkt_io

    echo $cpu > /sys/fs/cgroup/cpuset/cpuset.cpus

    echo $t0 > /sys/fs/cgroup/cpu/pkt_io/tasks
    echo $t0 > /sys/fs/cgroup/cpuset/pkt_io/tasks

    echo $t1 > /sys/fs/cgroup/cpu/pkt_io/tasks
    echo $t1 > /sys/fs/cgroup/cpuset/pkt_io/tasks

    cd /sys/fs/cgroup/cpu/pkt_io
    echo 100000 > pkt_io/cpu.cfs_period_us
    echo  50000 > pkt_io/cpu.cfs_quota_us


Malloc
------

EAL提供用于申请任意大小内存的API。

该API的目的是提供一个类似于malloc的函数，可以从操作系统的巨页内存中申请内存，
还有简化程序的移植。 *DPDK API Reference* 手册中叙述了可用的函数。

显然，这些内存申请操作不应该在数据处理过程中进行，因为它们比基于内存池的内存申请操作慢很多，
而且在内存申请和释放的过程中还使用了锁。

更多有关rte_malloc()函数的描述请查看  *DPDK API Reference*

Cookies
~~~~~~~

当启用调试模式（CONFIG_RTE_MALLOC_DEBUG is enabled）时，
申请的内存会包含覆写保护域用于标识缓冲区溢出。

对齐和NUMA约束
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

rte_malloc()接收一个对齐参数n(n必须是2的幂)，申请的内存将对齐于n的倍数。

在支持NUMA的系统当中，rte_malloc()将从本地NUMA socket申请内存。
DPDK中也提供了直接从指定NUMA socket或者其他核心所在NUMA socket（比如，为其他核申请内存）中申请内存的API。

使用案例
~~~~~~~~~

该API用于在初始化阶段需要使用像malloc函数的应用中。

在应用运行时为了快速申请和释放内存应该使用内存池库代替真正的内存申请和释放操作。

内部实现
~~~~~~~~~~~~~~~~~~~~~~~

数据结构
^^^^^^^^^^^^^^^

malloc库中有两种内部使用的数据结构类型:

*   struct malloc_heap - 用于记录每个socket（per-socket basis）上空闲内存

*   struct malloc_elem - 内存申请的基本元素，还用于malloc库内空闲内存记录

结构体: malloc_heap
""""""""""""""""""""""

malloc_heap结构体用于管理每个socket上空闲内存。每个NUMA节点有一个malloc_heap结构体，
通过这个结构体我们可以给该NUMA节点上的线程申请内存。
但这并不保证该内存仅会被该NUMA节点上的线程使用。
一个很烂的设计: 总是在固定节点或总是在随机节点上申请内存。

malloc_heap结构体的关键字段:

*   lock - 锁保证堆访问的同步性。由于堆中的空闲内存是用链表记录的，
    所以需要使用锁防止两个线程同时操作该链表。

*   free_head - 指向该堆空闲内存链表的第一个元素。

.. note::

    malloc_heap结构体不会记录使用中的内存块，因为除了释放内存，malloc库不会对它们做任何操作。

.. _figure_malloc_heap:

.. figure:: img/malloc_heap.*

   malloc库中堆（malloc_heap）和元素（malloc_elem）的示例


.. _malloc_elem:

结构体: malloc_elem
""""""""""""""""""""""

malloc_elem结构体用于各种内存块的通用头结构。上图中有三种内存块用到该结构体:

#.  空闲或已申请内存块头 - 正常用法

#.  内存块里的填充头

#.  内存段（memseg）的结束标记

.. note::

    上面三种使用方法中有个别字段没有描述，没有描述的字段其值是未定义的，比如，
    在填充头中仅"state"和"pad"字段有合法值。

该结构体中最重要的字段如下: 

*   heap - 该指针指向该内存块所属的堆。在内存块释放的使用，
    通过该指针把该空闲内存块加到堆空闲列表。

*   prev - 该指针指向内存段（memseg）中的前一个元素（内存块）。
    在释放内存块时，通过这个指针找到前一个块，判断前一个内存块是否是空闲的，
    如果是空闲的则将这两块内存合并成一个大块内存。

*   next_free - 该指针用于把空闲内存块链接成一个空闲链表。仅用于正常（空闲或使用中的）内存块; 
    在 ``malloc()`` 中用于寻找合适的空闲块，在 ``free()`` 中用于把新释放的内存块加入到空闲列表中。

*   state - 该字段有三个值: ``FREE``, ``BUSY`` 和 ``PAD``。
    前两个值代表的是正常内存块的状态; 最后一个值PAD表示该malloc_elem是哑头（不代表任何正常内存块），
    因为对齐约束，这种块只是用来填充的，该结构体位于填充区域的结尾，
    因为填充的存在，该内存块内数据的起始地址并不是块的地址。在这种情况下，
    填充头被用于定位该内存块实际的头。对于内存段（memseg）结束结构，state总是 ``BUSY``，
    这样可以防止内存段结束元素被 ``free()`` 合并。

*   pad -  存放的是内存块中从起始位置开始填充的长度。在正常块的头中，
    该值加上头结束地址得到数据区域的起始地址，也就是 ``malloc()`` 的返回值。
    填充块的哑头中这个字段存储的也是填充长度，哑头的地址减去该值得到实际内存块头的地址。

*   size - 数据块的长度，包括头的长度。对于内存段结束结构，该值为零。
    对于将要释放的内存块，这个值被用来作为"next"指针识别下一个内存块的位置。
    如果下一个内存块是 ``FREE`` 的，那么这两个内存块会被合并成一个。

内存申请
^^^^^^^^^^^^^^^^^

在EAL初始化时，所有的内存段（memseg）被加入到malloc堆中，
并且会在每个段的尾部放置一个带有 ``BUSY`` 状态的哑头（如果开启了 ``CONFIG_RTE_MALLOC_DEBUG`` 哑头中也会包含一个哨兵元素），
每个段的开头放置状态为 ``FREE`` 的 :ref:`element header<malloc_elem>` 。
然后把 ``FREE`` 的元素加入到malloc堆的 ``free_list`` 中。

当程序调用像malloc这样的函数时，malloc函数会首先从调用线程中索引 ``lcore_config`` 结构，
然后判断该线程的NUMA节点。NUMA节点用于从malloc堆数组中索引具体的堆，
然后把具体的堆作为参数传递给 ``malloc_heap_alloc()`` 函数。

``malloc_heap_alloc()`` 会扫描堆的空闲列表，尝试找到一个合适的（大小、对齐、boundary等约束）空闲块。

当找到合适的空闲内存块时，会计算出返回给用户的指针。
在计算返回给用户的指针前，内存的cache-line会立即被malloc_elem头填充。
（原文：The cache-line of memory immediately preceding this pointer is filled with a struct malloc_elem header.）
由于对齐和boundary约束，元素的开头和/或结尾会有空闲空间，这会导致下面的行为:

#. 尾部空间检查
   如果尾部空间足够大，即大于128字节，会分割一个空闲元素出去。否则忽略这段空间（空间浪费）。

#. 起始空间检查
   如果起始空间很小，即小于等于128字节，会在其中放置一个填充头，其余空间会被浪费掉。
   否则会分割出一个空闲元素。

从已存在元素尾部申请内存的好处是不用调整空闲列表 - the existing element
on the free list just has its size pointer adjusted, and the following element
has its "prev" pointer redirected to the newly created element.

内存释放
^^^^^^^^^^^^^^

释放内存时需要把指向数据区的指针传递给释放函数。
用这个指针减去 ``malloc_elem`` 的大小得到内存块的头结构体。
如果内存块头的类型是 ``PAD`` ，再从指针中减去填充长度得到整个内存块的真确的头结构体。

从这个头结构体中可以获取该内存块所属的堆指针和前一个元素指针，
并且通过大小字段我们可以计算出下一个元素的指针。
这些前后的元素会被检查是否 ``FREE`` ，如果是空闲的则会被合并当前内存块中。
这意味着不会有两个 ``FREE`` 内存块相邻，因为它们总会被合并到一个块中。
