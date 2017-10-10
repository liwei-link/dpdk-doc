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

链路聚合轮询模式驱动库
=====================================

除了物理设备和虚拟硬件的轮询模式驱动(PMD)，DPDK中也包含一个纯软件的库用于把物理PMD绑定在一起
创建一个逻辑PMD。

.. figure:: img/bond-overview.*

   聚合PMD


Linux聚合驱动把多个NIC接口聚合成一个在服务器和交换机之间的逻辑接口。与之类似，
链路聚合PMD库(librte_pmd_bond)则支持把多个具有相同速率和双工模式的 ``rte_eth_dev`` 端口聚合起来。
然后，聚合PMD会根据操作模式处理这些接口，操作模式提供指定特性的支持，比如，冗余链路，容错和/或负载均衡。

librte_pmd_bond导出C语言API用于创建聚合设备，还有配置和管理聚合设备及其从设备。

.. note::

    链路聚合PMD库在构建配置文件中默认是启用的，可以通过设置 
    ``CONFIG_RTE_LIBRTE_PMD_BOND=n`` 并重新编译来关闭。

链路聚合模式总览
---------------------------

当前链路聚合PMD库支持以下操作模式:

*   **轮询模式 (Mode 0):**

.. figure:: img/bond-mode-0.*

   轮询模式(Mode 0)


    该模式为包传输提供负载均衡和容错功能，数据包按照从第一个可用设备到最后一个可用设备顺序传输。
    数据包从设备中大量出队，然后以轮询方式发送。该模式不保证数据包能够顺序接受，下游应用应该能
    处理乱序包。
	
*   **主备模式 (Mode 1):**

.. figure:: img/bond-mode-1.*

   主备模式 (Mode 1)


    该模式下同时只有一个设备激活，仅在主设备出错时另外一个从设备才会被激活，从而提容错功能。
    逻辑聚合接口的MAC地址仅在一个NIC端口上对外可见，这样做是为了避免网络交换出现混淆。

*   **均衡异或模式 (Balance XOR, Mode 2):**

.. figure:: img/bond-mode-2.*

   均衡异或模式 (Balance XOR, Mode 2)


    该模式提供传输负载均衡(基于选择的传输策略)和容错。默认策略(L2)是基于包的源和目的MAC地址
    以及聚合设备可用从设备数。传输策略也可以基于L2+L3，该策略也会把源和目的IP用于计算传输的从设备端口。
    最后支持的策略是基于L3+L4的，该策略使用源IP、目的IP、源TCP/UDP端口和目的TCP/UDP端口。

.. note::
    包的颜色用于标识不同的流分类。


*   **广播模式 (Mode 3):**

.. figure:: img/bond-mode-3.*

   广播模式 (Mode 3)


   该模式通过在所有从设备端口上传输包提供容错。

*   **链路聚合 802.3AD (Mode 4):**

.. figure:: img/bond-mode-4.*

   链路聚合 802.3AD (Mode 4)


    该模式根据802.3ad提供动态链路聚合。xxx并监控用于输出流量的聚合组。（It negotiates and 
    monitors aggregation groups that share the same speed and duplex settings 
    using the selected balance transmit policy for balancing outgoing traffic.）

    该模式的DPDK实现对应用有一些额外的要求:

    #. 需要在小于100ms的间隔内调用 ``rte_eth_tx_burst`` 和 ``rte_eth_rx_burst`` 。

    #. 调用 ``rte_eth_tx_burst`` 必须要有大小至少为2xN的缓冲区，N是从设备的数量。
       该空间是LACP帧需要的。另外LACP包会包含到统计信息中，但不会返回给应用。

*   **传输负载均衡模式 (Mode 5):**

.. figure:: img/bond-mode-5.*

   传输负载均衡模式 (Mode 5)


   该模式提供自适应传输负载均衡。它会根据计算出的负载动态改变传输设备。每100ms收集一次统计信息，每10ms调度一次。


实现细节
----------------------

librte_pmd_bond聚合的设备和 *DPDK API Reference* 中描述的以太网PMD设备API兼容。

链路聚合库支持在程序启动时通过 ``--vdev`` 选项创建聚合设备。
也可以在程序运行期间通过C API  ``rte_eth_bond_create`` 函数创建聚合。

可以通过API ``rte_eth_bond_slave_add`` / ``rte_eth_bond_slave_remove`` 
动态增加和移除聚合设备的从设备。

在从设备增加到聚合设备中后，使用 ``rte_eth_dev_stop`` 停止从设备，然后使用
 ``rte_eth_dev_configure`` 重新配置设备，使用 ``rte_eth_tx_queue_setup`` /
``rte_eth_rx_queue_setup`` API和聚合设备的参数重新配置RX和TX队列。
如果聚合设备开启了RSS，那么从设备上也会开启并配置RSS。

为聚合设备的RSS设置多队列模式使其具有完整的RSS能力，所有从设备有相同的RSS配置。
这样可以为客户应用提供从设备透明的RSS配置。

聚合设备会存储RSS设置(也就是RETA、RSS哈希函数和RSS key)用于设置从设备。
这使聚合设备RSS配置成为整个聚合设备的配置，而不是单独在从设备中存储。
但要保证数据一致性和提供更加严格的错误校验。

聚合设备的RSS哈希函数集合是所有绑定从设备支持的RSS哈希函数最大集。RETA大小是所有RETA大小的最大公约数，
因此很容易通过参数使用，即使从设备的RETA大小各不相同。如果聚合设备没有设置RSS Key，
那么从设备上的RSS Key不会改变，并且设备使用默认key。

所有设置都是通过聚合端口API来管理的，并且总是朝一个方向传播(从聚合设备到从设备)

链路状态变更中断/轮询
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

链路聚合设备可以使用 ``rte_eth_dev_callback_register`` API注册链路状态变更回调函数，
聚合设备状态变更时回调函数会被调用。比如，聚合设备有三个从设备，其中一个从设备激活时聚合设备激活，
当所有从设备不活动时，聚合设备状态变为不活动。当单个从设备变更状态并且不满足前面的条件时不会有回调通知。
如果想要单独监测从设备，需要直接为从设备注册回调函数。

链路聚合库也支持没有实现链路状态变更中断的设备，它是通过周期性地轮询设备状态实现的，
``rte_eth_bond_link_monitoring_set`` API用于设置轮询周期，默认轮询周期是10ms。
当一个设备加入到聚合设备中时，通过 ``RTE_PCI_DRV_INTR_LSC`` 标识设备是通过中断获得链路变更还是通过轮询监测设备状态。

要求 / 限制
~~~~~~~~~~~~~~~~~~~~~~~~~~

当前的实现只支持有相同速率和双工模式的设备加入到同一个聚合设备中。
聚合设备从第一个激活的从设备继承这两个属性。后面加入的从设备必须支持这些属性/参数。

聚合设备在启动前必须至少有一个从设备。

为了高效地使用聚合设备动态RSS配置特性，所有从设备应该支持RSS能力，
并且对所有从设备至少有一个通用的可用哈希函数。当所有从设备都支持相同Key大小时，
才可以修改RSS Key。

为了防止从设备处理包的方式不一致，一旦设备加入到聚合设备中，
RSS配置都应该通过聚合设备API来操作，而不要直接操作从设备。

和其他PMD一样，导出函数都是无锁的，同一对象不能由不同逻辑核同时调用无锁函数操作。

PMD接收函数不应该直接在已加入聚合设备的从设备上调用，
因为直接从从设备上读取数据包会导致该数据包无法在聚合设备上读取到。

配置
~~~~~~~~~~~~~

链路聚合设备通过 ``rte_eth_bond_create`` API创建，该函数需要一个唯一设备名，聚合模式，
和聚合设备资源所在的socket ID。聚合设备其他可配置参数有: 从设备、主设备、用户定义的MAC地址
和传输策略(如果设备处于均衡异或模式)。

从设备
^^^^^^^^^^^^^

聚合设备最大支持 ``RTE_MAX_ETHPORTS`` 个同速率同双工模式的从设备。
以太网设备可以作为从设备加入到聚合设备中。从设备加入到聚合设备时会使用
聚合设备的配置重新配置从设备。

从聚合设备中删除一个从设备时，从设备原始MAC地址会被恢复。

主设备
^^^^^^^^^^^^^

主设备是聚合设备工作于备份模式的默认端口。仅在当前主端口不可用时其他端口才会被使用。
如果用户不指定主设备，那么第一个加入的设备就是主设备。

MAC 地址
^^^^^^^^^^^

用户可以为聚合设备配置指定MAC地址，并且该地址会被一些/所有(依据操作模式)从设备继承。
如果设备处于备份模式，那么只有主设备拥有用户指定MAC地址，所有其他从设备保持原有MAC。
在模式0、2、3、4中，所有从设备的MAC都被配置成聚合设备的MAC地址。

如果用户未定义MAC地址，那么聚合设备默认使用主设备MAC地址。

均衡异或传输策略
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

运行于均衡异或模式的聚合设备支持三种传输策略。2层，2+3层，3+4层

*   **2层:**   以太网MAC地址均衡是均衡异或聚合模式默认的传输策略。对包的源和目的
    MAC地址做异或运算，然后计算该值的模，从而获得传输该包的从设备。

*   **2 + 3层:** 以太网MAC地址和IP地址均衡使用包的源/目的MAC和源/目的IP的组合决
    定传输包的从设备。

*   **3 + 4层:**  IP地址和UDP端口均衡使用包的源/目的IP和源/目的UDP端口的组合决
    定传输包的从设备。

负载均衡的所有策略都支持802.1Q VLAN以太网包，IPv4，IPv6和UDP协议。

使用链路聚合设备
--------------------------

librte_pmd_bond支持两种设备创建方式，一种是使用C语言API，另一种是在应用启动时
使用EAL命令行静态的配置聚合设备。通过命令行选项使用链路聚合功能无需了解API，
比如，可用于给那些不使用C API的应用增加聚合的功能，比如活动备份。

在应用中使用轮询模式驱动
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用librte_pmd_bond库API可以在应用中动态创建并管理链路聚合设备。``rte_eth_bond_create`` 
用于创建链路聚合设备，该API需要一个唯一设备名，聚合模式(用于初始化设备)和设备资源所在
socket ID。聚合设备创建成功后必须使用通用以太网设备配置API ``rte_eth_dev_configure`` 
进行配置，RX 和 TX 队列也必须在使用 ``rte_eth_tx_queue_setup`` / ``rte_eth_rx_queue_setup`` 
设置后才能使用。

``rte_eth_bond_slave_add`` / ``rte_eth_bond_slave_remove`` 用于动态地向/从聚合设备中
增加/删除从设备。聚合设备在使用 ``rte_eth_dev_start`` 启动前至少要有一个从设备。

聚合设备的链路状态是由它的从设备决定的，如果所有从设备都是down的或者所有从设备都从聚
合设备中删除了，那么聚合设备的状态才会down。

``rte_eth_bond_mode_set/ get``, ``rte_eth_bond_primary_set/get``,
``rte_eth_bond_mac_set/reset`` 和 ``rte_eth_bond_xmit_policy_set/get`` 
这些API用于配置或查询聚合设备的控制参数。

通过EAL命令行使用链路聚合设备
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

聚合设备可以在应用启动时通过使用 ``--vdev`` EAL命令行选项创建。设备名称必须以
net_bond开头，后面紧随数字或字母，每个设备名必须唯一。每个设备可以有多个选项，
选项之间用逗号分隔。可以通过多次使用 ``--vdev`` 选项来定义多个设备。

设备名和设备选项之间必须用逗号分隔:

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,bond_opt0=..,bond opt1=..'--vdev 'net_bond1,bond _opt0=..,bond_opt1=..'

链路聚合EAL选项
^^^^^^^^^^^^^^^^^^^^^^^^

选项定义方法可以有多种形式，只要遵守下面的两条规则:

*   唯一设备名，以net_bondX形式提供，这里的X可以是数组和/或字母任意组合，
    名字长度不超过32字符。

*   至少要提供一个从设备给链路聚合设备。

*   提供聚合设备的操作模式。

不同的选项有:

*   mode: 整数，定义设备聚合模式。当前支持的模式有0,1,2,3,4,5(轮询,活动备份,均衡,
    广播,链路聚合,传输负载均衡)。

.. code-block:: console

        mode=2

*   slave: 定义要加入到聚合设备中的PMD设备。该选项可以多次使用，用于增加多个PMD设备。
    物理设备使用PCI地址指定，格式: domain:bus:devid.function。

.. code-block:: console

        slave=0000:0a:00.0,slave=0000:0a:00.1

*   primary: (可选)定义主设备端口，在活动备份模式下用于数据TX/RX的主设备。主设
    备也用于在用户未指定MAC地址时为聚合设备提供MAC地址。默认的主设备是第一个加
    入到聚合设备中的设备。主设备必须是聚合设备中的一个设备。

.. code-block:: console

        primary=0000:0a:00.0

*   socket_id: (可选)指定申请聚合设备资源的socket。

.. code-block:: console

        socket_id=0

*   mac: (可选)聚合设备的MAC地址，会覆盖主设备的值。

.. code-block:: console

        mac=00:1e:67:1d:fd:1d

*   xmit_policy: (可选)定义聚合设备处于均衡模式时选用的传输策略。如用户未指定，
    默认l2(2层转发)，其他可用策略有l23和l34。

.. code-block:: console

        xmit_policy=l23

*   lsc_poll_period_ms: (可选)设备(不支持lsc中断)状态轮询周期(毫秒)。

.. code-block:: console

        lsc_poll_period_ms=100

*   up_delay: (可选) 设备状态变更为up时，延迟通知的时间(毫秒)，默认0。

.. code-block:: console

        up_delay=10

*   down_delay: (可选) 设备状态变更为down时，延迟通知的时间(毫秒)，默认0。

.. code-block:: console

        down_delay=50

使用实例
^^^^^^^^^^^^^^^^^

创建具有两个从设备，轮询模式的聚合设备:

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,mode=0, slave=0000:00a:00.01,slave=0000:004:00.00' -- --port-topology=chained

创建具有两个从设备，轮询模式，指定MAC地址的聚合设备:

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,mode=0, slave=0000:00a:00.01,slave=0000:004:00.00,mac=00:1e:67:1d:fd:1d' -- --port-topology=chained

活动备份模式的聚合设备，有两个从设备，并且通过PCI地址指定一个为主设备:

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,mode=1, slave=0000:00a:00.01,slave=0000:004:00.00,primary=0000:00a:00.01' -- --port-topology=chained

均衡模式的聚合设备，有两个从设备，传输策略是3+4层转发:

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,mode=2, slave=0000:00a:00.01,slave=0000:004:00.00,xmit_policy=l34' -- --port-topology=chained
	