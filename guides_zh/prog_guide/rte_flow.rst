..  BSD LICENSE
    Copyright 2016 6WIND S.A.
    Copyright 2016 Mellanox.

    Redistribution and use in source and binary forms, with or without
    modification, are permitted provided that the following conditions
    are met:

    * Redistributions of source code must retain the above copyright
    notice, this list of conditions and the following disclaimer.
    * Redistributions in binary form must reproduce the above copyright
    notice, this list of conditions and the following disclaimer in
    the documentation and/or other materials provided with the
    distribution.
    * Neither the name of 6WIND S.A. nor the names of its
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

.. _Generic_flow_API:

通用流API (rte_flow)
===========================

综述
--------

该API提供了通用的硬件配置方法，使硬件能够根据用于定义的规则匹配特定出入口流量，
改变硬件工作方式和查询相关计数器。

该API中所有符号名都是以 *rte_flow* 为前缀，并且定义在 ``rte_flow.h`` 文件中。

- 可以匹配的对象有包数据(协议头，负载)和属性(比如，物理网口，虚拟设备功能ID)

- 操作有丢弃包，传递到指定队列，传递到虚拟/物理设备处理函数或者端口，
  执行隧道卸载(tunnel offloads)，增加标签等。

该API是用来替换旧过滤框架(其包含了旧框架所有功能和过滤类型)的，比旧过滤框架更高级一些。
该API为所有PMD提供了清楚且通用的使用接口。

旧应用中API更新方法，参考 `API migration`_。

流规则
---------

描述
~~~~~~~~~~~

流规则是规则属性的组合，规则属性包括匹配模式和动作列表。流规则构成了该API的基础。

一个流规则能包含多个不同的动作(如，计数，转发前封包和解包)，
不需要多个流规则来实现多个动作。
也不需要应用处理不同动作顺序的硬件实现细节。

流规则支持优先级，比如一个包匹配两个流规则，那么优先级高的规则会被优先处理。
但是，无法保证硬件支持多优先级。当支持时，可用的优先级往往比较少，
这就是为什么优先级可以有PMD在软件层面实现(比如可以通过重新排序规则来模拟)。

为了尽量保持硬件无关性，默认认为所有规则有相同的优先级，
也就是重叠规则(一个包匹配多个过滤器)的动作执行顺序未定义。

PMD可以拒绝在同一个优先级上创建重叠规则(比如，匹配模式已存在)。

因此在给定优先级上没有重叠规则时，其结果是可预见的。这样的规则在所有协议层都可以完美工作。

流规则也可以分组，流规则优先级被分配到他们所属的组上。因此，
给定组内的流规则要么在其他组的前面处理，要么在其他组后面处理。

在内部，一个规则多动作的支持可能是基于非默认硬件优先级实现的，
因此这两种特性可能不会同时对应用可用。

考虑到设备允许的模式/动作组合不能预先知道，会导致暴露大量无用的功能。
DPDK提供了从当设备配置状态中验证给定规则的方法。

这可以让应用在初始化时(启动数据路径前)检查它需要的规则类型是否支持。
该方法可以在任何时候使用，前提条件是规则所需的资源应该存在(比如，一个目标RX队列应先被配置好)

每一个定义好的规则都和由PMD管理的不透明句柄(opaque handle)相关联，应用应该维护它。
这些句柄可以用于查询和管理规则，比如获取计数器或其他数据，销毁规则。

为避免在PMD侧出现资源泄露，应用在释放相关资源(如，队列和端口)前必须明确销毁句柄。

以下章节包括:

- **属性** (由 ``struct rte_flow_attr`` 表示): 流规则的属性，比如方向(入口或出口)和优先级。

- **模式项** (由 ``struct rte_flow_item`` 表示): 匹配模式的一部分，匹配对象是特定包数据或者流量属性。
  它描述了模式本身的属性，比如反向匹配。

- **匹配模式**: 要查找的流量属性，任意项组合。

- **动作** (由 ``struct rte_flow_action`` 表示): 包匹配时要执行的操作。

属性
~~~~~~~~~~

属性: 组
^^^^^^^^^^^^^^^^

可以给流规则指定组号进行分组。组号越小优先级越高。组0拥有最高优先级。

虽然是可选的，但还是鼓励应用尽可能多的把类似的规则分为一组，
这样可以充分利用硬件能力(比如，最佳匹配)和解决(打破)限制(比如，一组只能有一个匹配模式)。

注意，无法保证对多组的支持。

属性: 优先级
^^^^^^^^^^^^^^^^^^^

优先级可以分配给流规则。在规则组中，优先级值越小优先级越高，0是最高的。

组8中优先级0的规则的优先级总是低于组0中优先级8的规则的。

组和优先级是任意的，并且取决于应用，它们不必是连续的，也不必从0开始。
但是，不同设备的组和优先级的最大数量可能受已存在的流规则影响。

如果一个包匹配多条规则，结果是未定义的。它可能被意处理，
比如复制，甚至引发不可恢复的错误。

注意，无法保证对多优先级的支持。

属性: 流量方向
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

流规则可以应用到出口/入口流量。

我们至少要为流规则指定一个方向。但不限于一个方向，同一个流规则可以应用到两个方向上。

不建议同时在两个方向上指定一个规则，但在一些案例中是合理的(比如，共享计数器)

模式项(Pattern item)
~~~~~~~~~~~~

模式项分为两类:

- 协议头和包数据的匹配 (ANY, RAW, ETH, VLAN, IPV4,
  IPV6, ICMP, UDP, TCP, SCTP, VXLAN, MPLS, GRE 等), 通常和一个结构体相关。

- 元数据和模式处理匹配 (END, VOID, INVERT, PF,
  VF, PORT 等), 通常没有结构体与之相关。

模式项详情结构体用于匹配指定的协议(或模式的属性)。文档中会描述每个模式项。

目前，一个模式项类型可以有三个相关结构体:

- ``spec``: 匹配值 (如，IPv4地址)。

- ``last``: ``spec`` 字段的范围上限。

- ``mask``: 位掩码应用于 ``spec`` 和 ``last`` ，目的是区分哪些值算在内或者哪些值排除在外。
  (比如，用于匹配IPv4前缀)。

使用限制和规范:

- 只设置 ``mask`` 或者 ``last`` ，不设置 ``spec`` 是错误的。

- ``last`` 等于0或者等于相关的  ``spec`` 会被忽略；因为这无法指定一个范围。
  不支持比 ``spec`` 小的非零值。

- 设置 ``spec`` 而不设置 ``mask`` ，PMD会使用该模式项默认的mask
  (常量 ``rte_flow_item_{name}_mask`` )。
  

- 当这些都不设置(假设模式项类型支持)时等价于空(零) ``mask`` 宽(未指定)匹配。

- ``mask`` 是一个简单的位掩码，在解析 ``spec`` 和 ``last`` 内容前应用。
  (如使用不当会产生不可预期的结果) 比如，对于IPv4地址，``spec`` 为 *10.1.2.3*，
  ``last`` 为 *10.3.4.5* , ``mask`` 为 *255.255.0.0* ，
  那么最终有效范围变成 *10.1.0.0* to *10.3.255.255*。

模式项匹配以太网头的例子:

.. _table_rte_flow_pattern_item_example:

.. table:: Ethernet 项

   +----------+----------+--------------------+
   | Field    | Subfield | Value              |
   +==========+==========+====================+
   | ``spec`` | ``src``  | ``00:01:02:03:04`` |
   |          +----------+--------------------+
   |          | ``dst``  | ``00:2a:66:00:01`` |
   |          +----------+--------------------+
   |          | ``type`` | ``0x22aa``         |
   +----------+----------+--------------------+
   | ``last`` | 未指定                        |
   +----------+----------+--------------------+
   | ``mask`` | ``src``  | ``00:ff:ff:ff:00`` |
   |          +----------+--------------------+
   |          | ``dst``  | ``00:00:00:00:ff`` |
   |          +----------+--------------------+
   |          | ``type`` | ``0x0000``         |
   +----------+----------+--------------------+

无掩码位代表可以是任意值(如下的 ``?``)，因此，像下面的以太网头可以匹配到:

- ``src``: ``??:01:02:03:??``
- ``dst``: ``??:??:??:??:01``
- ``type``: ``0x????``

匹配模式
~~~~~~~~~~~~~~~~

匹配模式中与协议相关的模式项是从最底层协议开始匹配的(组成一个模式项栈)。该限制不会应用到元模式项，
元模式项可以放在模式中任何位置，并且不会影响到最终的匹配模式。

匹配模式由END项终结。

例子:

.. _table_rte_flow_tcpv4_as_l4:

.. table:: TCPv4 as L4

   +-------+----------+
   | Index | Item     |
   +=======+==========+
   | 0     | Ethernet |
   +-------+----------+
   | 1     | IPv4     |
   +-------+----------+
   | 2     | TCP      |
   +-------+----------+
   | 3     | END      |
   +-------+----------+

|

.. _table_rte_flow_tcpv6_in_vxlan:

.. table:: TCPv6 in VXLAN

   +-------+------------+
   | Index | Item       |
   +=======+============+
   | 0     | Ethernet   |
   +-------+------------+
   | 1     | IPv4       |
   +-------+------------+
   | 2     | UDP        |
   +-------+------------+
   | 3     | VXLAN      |
   +-------+------------+
   | 4     | Ethernet   |
   +-------+------------+
   | 5     | IPv6       |
   +-------+------------+
   | 6     | TCP        |
   +-------+------------+
   | 7     | END        |
   +-------+------------+

|

.. _table_rte_flow_tcpv4_as_l4_meta:

.. table:: TCPv4 as L4 with meta items

   +-------+----------+
   | Index | Item     |
   +=======+==========+
   | 0     | VOID     |
   +-------+----------+
   | 1     | Ethernet |
   +-------+----------+
   | 2     | VOID     |
   +-------+----------+
   | 3     | IPv4     |
   +-------+----------+
   | 4     | TCP      |
   +-------+----------+
   | 5     | VOID     |
   +-------+----------+
   | 6     | VOID     |
   +-------+----------+
   | 7     | END      |
   +-------+----------+

上面的例子说明了只要保持包数据模式项的正确的叠放顺序，
那么元模式项是不会影响包数据模式项的。所以上面的例子中匹配模式和"TCPv4 as L4"一样。

.. _table_rte_flow_udpv6_anywhere:

.. table:: UDPv6 anywhere

   +-------+------+
   | Index | Item |
   +=======+======+
   | 0     | IPv6 |
   +-------+------+
   | 1     | UDP  |
   +-------+------+
   | 2     | END  |
   +-------+------+

如果PMD支持，省略栈底的一个或几个协议层(如上例未指定以太网层)，
则会查找包中任意位置。

这种情况下未指定封装负载(比如，VXLAN负载)要匹配哪种包，可能是封装的内部包，外部包或者两者都是。

.. _table_rte_flow_invalid_l3:

.. table:: 无效, 缺少L3协议

   +-------+----------+
   | Index | Item     |
   +=======+==========+
   | 0     | Ethernet |
   +-------+----------+
   | 1     | UDP      |
   +-------+----------+
   | 2     | END      |
   +-------+----------+

上面的匹配模式中因为在L2(Ethernet)和L4(UDP)之间未指定L3协议，所以是无效的。
层级的缺失只能发生在栈顶或栈底。

元模式项类型
~~~~~~~~~~~~~~~

元模式项匹配的是元数据，或者影响模式处理而不是直接匹配包数据，大多数的元模式项不需要指定数据结构。
这个例外可以让元模式项处在栈中任何位置并不会产生副作用。

元模式项: ``END``
^^^^^^^^^^^^^

模式项列表的结束标志。防止超范围的模式项处理，从而结束模式匹配。

- 为了方便起见，它的数值为0。
- PMD必须支持。
- 忽略 ``spec``, ``last`` 和 ``mask``。

.. _table_rte_flow_item_end:

.. table:: END

   +----------+---------+
   | Field    | Value   |
   +==========+=========+
   | ``spec`` | ignored |
   +----------+---------+
   | ``last`` | ignored |
   +----------+---------+
   | ``mask`` | ignored |
   +----------+---------+

元模式项: ``VOID``
^^^^^^^^^^^^^^

该元模式项是个占位符。PMD在处理时会忽略并丢弃该元模式项。

- PMD必须支持。
- 忽略 ``spec``, ``last`` 和 ``mask``。

.. _table_rte_flow_item_void:

.. table:: VOID

   +----------+---------+
   | Field    | Value   |
   +==========+=========+
   | ``spec`` | ignored |
   +----------+---------+
   | ``last`` | ignored |
   +----------+---------+
   | ``mask`` | ignored |
   +----------+---------+

该类型的一个使用案例是流规则快速共享通用前缀，不用重新申请内存，仅更新模式项类型:

.. _table_rte_flow_item_void_example:

.. table:: TCP, UDP or ICMP as L4

   +-------+--------------------+
   | Index | Item               |
   +=======+====================+
   | 0     | Ethernet           |
   +-------+--------------------+
   | 1     | IPv4               |
   +-------+------+------+------+
   | 2     | UDP  | VOID | VOID |
   +-------+------+------+------+
   | 3     | VOID | TCP  | VOID |
   +-------+------+------+------+
   | 4     | VOID | VOID | ICMP |
   +-------+------+------+------+
   | 5     | END                |
   +-------+--------------------+

元模式项: ``INVERT``
^^^^^^^^^^^^^^^^

反向匹配, 也就是处理与该模式不匹配的包。

- 忽略 ``spec``, ``last`` and ``mask``。

.. _table_rte_flow_item_invert:

.. table:: INVERT

   +----------+---------+
   | Field    | Value   |
   +==========+=========+
   | ``spec`` | ignored |
   +----------+---------+
   | ``last`` | ignored |
   +----------+---------+
   | ``mask`` | ignored |
   +----------+---------+

使用案例，仅匹配非TCPv4包:

.. _table_rte_flow_item_invert_example:

.. table:: Anything but TCPv4

   +-------+----------+
   | Index | Item     |
   +=======+==========+
   | 0     | INVERT   |
   +-------+----------+
   | 1     | Ethernet |
   +-------+----------+
   | 2     | IPv4     |
   +-------+----------+
   | 3     | TCP      |
   +-------+----------+
   | 4     | END      |
   +-------+----------+

元模式项: ``PF``
^^^^^^^^^^^^

匹配发送到设备物理功能的包。

如果底层设备功能无法正常接收匹配流量，则使用该元模式项可以防止流量到达该设备，
除非流规则包含 `Action: PF`_ 。默认数据包不会在设备实例间复制。

- 该元模式项如果应用到VF设备很可能会返回错误或者根本不会匹配到任何流量。

- 可以和任何数量的  `Item: VF`_ 组合匹配PF和VF流量。
  
- ``spec``, ``last`` 和 ``mask`` 不允许设置。

.. _table_rte_flow_item_pf:

.. table:: PF

   +----------+-------+
   | Field    | Value |
   +==========+=======+
   | ``spec`` | unset |
   +----------+-------+
   | ``last`` | unset |
   +----------+-------+
   | ``mask`` | unset |
   +----------+-------+

元模式项: ``VF``
^^^^^^^^^^^^

匹配发送到设备虚拟功能的包。

如果底层设备功能无法正常接收匹配流量，则使用该元模式项可以防止流量到达该设备，
除非流规则包含 `Action: VF`_ 。默认数据包不会在设备实例间复制。

- 该元模式项如果应用到VF设备很可能会返回错误或者根本不会匹配到任何流量。

- 可以和任何数量的  `Item: VF`_ 组合匹配PF和VF流量。
  
- ``spec``, ``last`` 和 ``mask`` 不允许设置。

- 如果让VF设备去匹配发送到不同VF上的流量时，很可能会返回错误或者根本不会匹配到任何流量。
  
- 可以通过多次指定该元模式项去匹配发送到多个VF上的流量。
  
- 可以和PF模式项组合匹配FP和VF流量。

- 默认 ``mask`` 匹配任何VF。

.. _table_rte_flow_item_vf:

.. table:: VF

   +----------+----------+---------------------------+
   | Field    | Subfield | Value                     |
   +==========+==========+===========================+
   | ``spec`` | ``id``   | destination VF ID         |
   +----------+----------+---------------------------+
   | ``last`` | ``id``   | 上限                      |
   +----------+----------+---------------------------+
   | ``mask`` | ``id``   | 0值匹配任何VF ID          |
   +----------+----------+---------------------------+

元模式项: ``PORT``
^^^^^^^^^^^^^^

匹配来自底层设备物理端口的数据包。

第一个PORT模式项覆盖了和DPDK相关联(port_id)的物理端口。
可以提供多个该模式项匹配多个物理端口。

注意，当物理端口不在DPDK控制下时，就不必绑定到DPDK输入端口(port_id)上。
每个设备会有一个port_id，他们不必从0开始，也可能不是连续的。

作为设备的属性，合法的端口号应该通过其他方法获得。

- 默认 ``mask`` 匹配任何端口号。

.. _table_rte_flow_item_port:

.. table:: PORT

   +----------+-----------+--------------------------------+
   | Field    | Subfield  | Value                          |
   +==========+===========+================================+
   | ``spec`` | ``index`` | 物理端口号                     |
   +----------+-----------+--------------------------------+
   | ``last`` | ``index`` | 上限                           |
   +----------+-----------+--------------------------------+
   | ``mask`` | ``index`` | 0值匹配任何端口号              |
   +----------+-----------+--------------------------------+

数据匹配项类型
~~~~~~~~~~~~~~~~~~~~~~~~

这些类型基本上都是协议头和相关位掩码的定义。这些类型必须按照从最低到最高协议层排列成一个栈，
组成一个匹配模式。

下面的列表并不是全面的，未来可能会有新协议加入进来。

数据模式项: ``ANY``
^^^^^^^^^^^^^

匹配任何协议，而不是仅匹配当前协议层，一个单个的ANY也可以代表多个协议层。

当在包中任意层查找一个协议时，``ANY`` 通常作为第一个模式项。

- 默认 ``mask`` 代表任意层数。

.. _table_rte_flow_item_any:

.. table:: ANY

   +----------+----------+--------------------------------------+
   | Field    | Subfield | Value                                |
   +==========+==========+======================================+
   | ``spec`` | ``num``  | 覆盖的层数                           |
   +----------+----------+--------------------------------------+
   | ``last`` | ``num``  | 上限                                 |
   +----------+----------+--------------------------------------+
   | ``mask`` | ``num``  | 0代表任意层数                        |
   +----------+----------+--------------------------------------+

VXLAN TCP负载匹配的例子，外部的L3(IPv4 or IPv6)和L4(UDP)都是由第一个 ANY 匹配。
内部的L3 (IPv4 or IPv6)由第二个 ANY 匹配:

.. _table_rte_flow_item_any_example:

.. table:: 使用 ANY 通配符匹配VXLAN中的TCP

   +-------+------+----------+----------+-------+
   | Index | Item | Field    | Subfield | Value |
   +=======+======+==========+==========+=======+
   | 0     | Ethernet                           |
   +-------+------+----------+----------+-------+
   | 1     | ANY  | ``spec`` | ``num``  | 2     |
   +-------+------+----------+----------+-------+
   | 2     | VXLAN                              |
   +-------+------------------------------------+
   | 3     | Ethernet                           |
   +-------+------+----------+----------+-------+
   | 4     | ANY  | ``spec`` | ``num``  | 1     |
   +-------+------+----------+----------+-------+
   | 5     | TCP                                |
   +-------+------------------------------------+
   | 6     | END                                |
   +-------+------------------------------------+

数据模式项: ``RAW``
^^^^^^^^^^^^^

Matches a byte string of a given length at a given offset.

Offset is either absolute (using the start of the packet) or relative to the
end of the previous matched item in the stack, in which case negative values
are allowed.

If search is enabled, offset is used as the starting point. The search area
can be delimited by setting limit to a nonzero value, which is the maximum
number of bytes after offset where the pattern may start.

Matching a zero-length pattern is allowed, doing so resets the relative
offset for subsequent items.

- This type does not support ranges (``last`` field).
- Default ``mask`` matches all fields exactly.

.. _table_rte_flow_item_raw:

.. table:: RAW

   +----------+--------------+-------------------------------------------------+
   | Field    | Subfield     | Value                                           |
   +==========+==============+=================================================+
   | ``spec`` | ``relative`` | look for pattern after the previous item        |
   |          +--------------+-------------------------------------------------+
   |          | ``search``   | search pattern from offset (see also ``limit``) |
   |          +--------------+-------------------------------------------------+
   |          | ``reserved`` | reserved, must be set to zero                   |
   |          +--------------+-------------------------------------------------+
   |          | ``offset``   | absolute or relative offset for ``pattern``     |
   |          +--------------+-------------------------------------------------+
   |          | ``limit``    | search area limit for start of ``pattern``      |
   |          +--------------+-------------------------------------------------+
   |          | ``length``   | ``pattern`` length                              |
   |          +--------------+-------------------------------------------------+
   |          | ``pattern``  | byte string to look for                         |
   +----------+--------------+-------------------------------------------------+
   | ``last`` | if specified, either all 0 or with the same values as ``spec`` |
   +----------+----------------------------------------------------------------+
   | ``mask`` | bit-mask applied to ``spec`` values with usual behavior        |
   +----------+----------------------------------------------------------------+

Example pattern looking for several strings at various offsets of a UDP
payload, using combined RAW items:

.. _table_rte_flow_item_raw_example:

.. table:: UDP payload matching

   +-------+------+----------+--------------+-------+
   | Index | Item | Field    | Subfield     | Value |
   +=======+======+==========+==============+=======+
   | 0     | Ethernet                               |
   +-------+----------------------------------------+
   | 1     | IPv4                                   |
   +-------+----------------------------------------+
   | 2     | UDP                                    |
   +-------+------+----------+--------------+-------+
   | 3     | RAW  | ``spec`` | ``relative`` | 1     |
   |       |      |          +--------------+-------+
   |       |      |          | ``search``   | 1     |
   |       |      |          +--------------+-------+
   |       |      |          | ``offset``   | 10    |
   |       |      |          +--------------+-------+
   |       |      |          | ``limit``    | 0     |
   |       |      |          +--------------+-------+
   |       |      |          | ``length``   | 3     |
   |       |      |          +--------------+-------+
   |       |      |          | ``pattern``  | "foo" |
   +-------+------+----------+--------------+-------+
   | 4     | RAW  | ``spec`` | ``relative`` | 1     |
   |       |      |          +--------------+-------+
   |       |      |          | ``search``   | 0     |
   |       |      |          +--------------+-------+
   |       |      |          | ``offset``   | 20    |
   |       |      |          +--------------+-------+
   |       |      |          | ``limit``    | 0     |
   |       |      |          +--------------+-------+
   |       |      |          | ``length``   | 3     |
   |       |      |          +--------------+-------+
   |       |      |          | ``pattern``  | "bar" |
   +-------+------+----------+--------------+-------+
   | 5     | RAW  | ``spec`` | ``relative`` | 1     |
   |       |      |          +--------------+-------+
   |       |      |          | ``search``   | 0     |
   |       |      |          +--------------+-------+
   |       |      |          | ``offset``   | -29   |
   |       |      |          +--------------+-------+
   |       |      |          | ``limit``    | 0     |
   |       |      |          +--------------+-------+
   |       |      |          | ``length``   | 3     |
   |       |      |          +--------------+-------+
   |       |      |          | ``pattern``  | "baz" |
   +-------+------+----------+--------------+-------+
   | 6     | END                                    |
   +-------+----------------------------------------+

This translates to:

- Locate "foo" at least 10 bytes deep inside UDP payload.
- Locate "bar" after "foo" plus 20 bytes.
- Locate "baz" after "bar" minus 29 bytes.

Such a packet may be represented as follows (not to scale)::

 0                     >= 10 B           == 20 B
 |                  |<--------->|     |<--------->|
 |                  |           |     |           |
 |-----|------|-----|-----|-----|-----|-----------|-----|------|
 | ETH | IPv4 | UDP | ... | baz | foo | ......... | bar | .... |
 |-----|------|-----|-----|-----|-----|-----------|-----|------|
                          |                             |
                          |<--------------------------->|
                                      == 29 B

Note that matching subsequent pattern items would resume after "baz", not
"bar" since matching is always performed after the previous item of the
stack.

数据模式项: ``ETH``
^^^^^^^^^^^^^

Matches an Ethernet header.

- ``dst``: destination MAC.
- ``src``: source MAC.
- ``type``: EtherType.
- Default ``mask`` matches destination and source addresses only.

数据模式项: ``VLAN``
^^^^^^^^^^^^^^

Matches an 802.1Q/ad VLAN tag.

- ``tpid``: tag protocol identifier.
- ``tci``: tag control information.
- Default ``mask`` matches TCI only.

数据模式项: ``IPV4``
^^^^^^^^^^^^^^

Matches an IPv4 header.

Note: IPv4 options are handled by dedicated pattern items.

- ``hdr``: IPv4 header definition (``rte_ip.h``).
- Default ``mask`` matches source and destination addresses only.

数据模式项: ``IPV6``
^^^^^^^^^^^^^^

Matches an IPv6 header.

Note: IPv6 options are handled by dedicated pattern items.

- ``hdr``: IPv6 header definition (``rte_ip.h``).
- Default ``mask`` matches source and destination addresses only.

数据模式项: ``ICMP``
^^^^^^^^^^^^^^

Matches an ICMP header.

- ``hdr``: ICMP header definition (``rte_icmp.h``).
- Default ``mask`` matches ICMP type and code only.

数据模式项: ``UDP``
^^^^^^^^^^^^^

Matches a UDP header.

- ``hdr``: UDP header definition (``rte_udp.h``).
- Default ``mask`` matches source and destination ports only.

数据模式项: ``TCP``
^^^^^^^^^^^^^

Matches a TCP header.

- ``hdr``: TCP header definition (``rte_tcp.h``).
- Default ``mask`` matches source and destination ports only.

数据模式项: ``SCTP``
^^^^^^^^^^^^^^

Matches a SCTP header.

- ``hdr``: SCTP header definition (``rte_sctp.h``).
- Default ``mask`` matches source and destination ports only.

数据模式项: ``VXLAN``
^^^^^^^^^^^^^^^

Matches a VXLAN header (RFC 7348).

- ``flags``: normally 0x08 (I flag).
- ``rsvd0``: reserved, normally 0x000000.
- ``vni``: VXLAN network identifier.
- ``rsvd1``: reserved, normally 0x00.
- Default ``mask`` matches VNI only.

数据模式项: ``E_TAG``
^^^^^^^^^^^^^^^

Matches an IEEE 802.1BR E-Tag header.

- ``tpid``: tag protocol identifier (0x893F)
- ``epcp_edei_in_ecid_b``: E-Tag control information (E-TCI), E-PCP (3b),
  E-DEI (1b), ingress E-CID base (12b).
- ``rsvd_grp_ecid_b``: reserved (2b), GRP (2b), E-CID base (12b).
- ``in_ecid_e``: ingress E-CID ext.
- ``ecid_e``: E-CID ext.
- Default ``mask`` simultaneously matches GRP and E-CID base.

数据模式项: ``NVGRE``
^^^^^^^^^^^^^^^

Matches a NVGRE header (RFC 7637).

- ``c_k_s_rsvd0_ver``: checksum (1b), undefined (1b), key bit (1b),
  sequence number (1b), reserved 0 (9b), version (3b). This field must have
  value 0x2000 according to RFC 7637.
- ``protocol``: protocol type (0x6558).
- ``tni``: virtual subnet ID.
- ``flow_id``: flow ID.
- Default ``mask`` matches TNI only.

数据模式项: ``MPLS``
^^^^^^^^^^^^^^

Matches a MPLS header.

- ``label_tc_s_ttl``: label, TC, Bottom of Stack and TTL.
- Default ``mask`` matches label only.

数据模式项: ``GRE``
^^^^^^^^^^^^^^

Matches a GRE header.

- ``c_rsvd0_ver``: checksum, reserved 0 and version.
- ``protocol``: protocol type.
- Default ``mask`` matches protocol only.

动作
~~~~~~~

Each possible action is represented by a type. Some have associated
configuration structures. Several actions combined in a list can be affected
to a flow rule. That list is not ordered.

They fall in three categories:

- Terminating actions (such as QUEUE, DROP, RSS, PF, VF) that prevent
  processing matched packets by subsequent flow rules, unless overridden
  with PASSTHRU.

- Non-terminating actions (PASSTHRU, DUP) that leave matched packets up for
  additional processing by subsequent flow rules.

- Other non-terminating meta actions that do not affect the fate of packets
  (END, VOID, MARK, FLAG, COUNT).

When several actions are combined in a flow rule, they should all have
different types (e.g. dropping a packet twice is not possible).

Only the last action of a given type is taken into account. PMDs still
perform error checking on the entire list.

Like matching patterns, action lists are terminated by END items.

*Note that PASSTHRU is the only action able to override a terminating rule.*

Example of action that redirects packets to queue index 10:

.. _table_rte_flow_action_example:

.. table:: Queue action

   +-----------+-------+
   | Field     | Value |
   +===========+=======+
   | ``index`` | 10    |
   +-----------+-------+

Action lists examples, their order is not significant, applications must
consider all actions to be performed simultaneously:

.. _table_rte_flow_count_and_drop:

.. table:: Count and drop

   +-------+--------+
   | Index | Action |
   +=======+========+
   | 0     | COUNT  |
   +-------+--------+
   | 1     | DROP   |
   +-------+--------+
   | 2     | END    |
   +-------+--------+

|

.. _table_rte_flow_mark_count_redirect:

.. table:: Mark, count and redirect

   +-------+--------+-----------+-------+
   | Index | Action | Field     | Value |
   +=======+========+===========+=======+
   | 0     | MARK   | ``mark``  | 0x2a  |
   +-------+--------+-----------+-------+
   | 1     | COUNT                      |
   +-------+--------+-----------+-------+
   | 2     | QUEUE  | ``queue`` | 10    |
   +-------+--------+-----------+-------+
   | 3     | END                        |
   +-------+----------------------------+

|

.. _table_rte_flow_redirect_queue_5:

.. table:: Redirect to queue 5

   +-------+--------+-----------+-------+
   | Index | Action | Field     | Value |
   +=======+========+===========+=======+
   | 0     | DROP                       |
   +-------+--------+-----------+-------+
   | 1     | QUEUE  | ``queue`` | 5     |
   +-------+--------+-----------+-------+
   | 2     | END                        |
   +-------+----------------------------+

In the above example, considering both actions are performed simultaneously,
the end result is that only QUEUE has any effect.

.. _table_rte_flow_redirect_queue_3:

.. table:: Redirect to queue 3

   +-------+--------+-----------+-------+
   | Index | Action | Field     | Value |
   +=======+========+===========+=======+
   | 0     | QUEUE  | ``queue`` | 5     |
   +-------+--------+-----------+-------+
   | 1     | VOID                       |
   +-------+--------+-----------+-------+
   | 2     | QUEUE  | ``queue`` | 3     |
   +-------+--------+-----------+-------+
   | 3     | END                        |
   +-------+----------------------------+

As previously described, only the last action of a given type found in the
list is taken into account. The above example also shows that VOID is
ignored.

动作类型
~~~~~~~~~~~~

Common action types are described in this section. Like pattern item types,
this list is not exhaustive as new actions will be added in the future.

动作: ``END``
^^^^^^^^^^^^^^^

End marker for action lists. Prevents further processing of actions, thereby
ending the list.

- Its numeric value is 0 for convenience.
- PMD support is mandatory.
- No configurable properties.

.. _table_rte_flow_action_end:

.. table:: END

   +---------------+
   | Field         |
   +===============+
   | no properties |
   +---------------+

动作: ``VOID``
^^^^^^^^^^^^^^^^

Used as a placeholder for convenience. It is ignored and simply discarded by
PMDs.

- PMD support is mandatory.
- No configurable properties.

.. _table_rte_flow_action_void:

.. table:: VOID

   +---------------+
   | Field         |
   +===============+
   | no properties |
   +---------------+

动作: ``PASSTHRU``
^^^^^^^^^^^^^^^^^^^^

Leaves packets up for additional processing by subsequent flow rules. This
is the default when a rule does not contain a terminating action, but can be
specified to force a rule to become non-terminating.

- No configurable properties.

.. _table_rte_flow_action_passthru:

.. table:: PASSTHRU

   +---------------+
   | Field         |
   +===============+
   | no properties |
   +---------------+

Example to copy a packet to a queue and continue processing by subsequent
flow rules:

.. _table_rte_flow_action_passthru_example:

.. table:: Copy to queue 8

   +-------+--------+-----------+-------+
   | Index | Action | Field     | Value |
   +=======+========+===========+=======+
   | 0     | PASSTHRU                   |
   +-------+--------+-----------+-------+
   | 1     | QUEUE  | ``queue`` | 8     |
   +-------+--------+-----------+-------+
   | 2     | END                        |
   +-------+----------------------------+

动作: ``MARK``
^^^^^^^^^^^^^^^^

Attaches an integer value to packets and sets ``PKT_RX_FDIR`` and
``PKT_RX_FDIR_ID`` mbuf flags.

This value is arbitrary and application-defined. Maximum allowed value
depends on the underlying implementation. It is returned in the
``hash.fdir.hi`` mbuf field.

.. _table_rte_flow_action_mark:

.. table:: MARK

   +--------+--------------------------------------+
   | Field  | Value                                |
   +========+======================================+
   | ``id`` | integer value to return with packets |
   +--------+--------------------------------------+

动作: ``FLAG``
^^^^^^^^^^^^^^^^

Flags packets. Similar to `Action: MARK`_ without a specific value; only
sets the ``PKT_RX_FDIR`` mbuf flag.

- No configurable properties.

.. _table_rte_flow_action_flag:

.. table:: FLAG

   +---------------+
   | Field         |
   +===============+
   | no properties |
   +---------------+

动作: ``QUEUE``
^^^^^^^^^^^^^^^^^

Assigns packets to a given queue index.

- Terminating by default.

.. _table_rte_flow_action_queue:

.. table:: QUEUE

   +-----------+--------------------+
   | Field     | Value              |
   +===========+====================+
   | ``index`` | queue index to use |
   +-----------+--------------------+

动作: ``DROP``
^^^^^^^^^^^^^^^^

Drop packets.

- No configurable properties.
- Terminating by default.
- PASSTHRU overrides this action if both are specified.

.. _table_rte_flow_action_drop:

.. table:: DROP

   +---------------+
   | Field         |
   +===============+
   | no properties |
   +---------------+

动作: ``COUNT``
^^^^^^^^^^^^^^^^^

Enables counters for this rule.

These counters can be retrieved and reset through ``rte_flow_query()``, see
``struct rte_flow_query_count``.

- Counters can be retrieved with ``rte_flow_query()``.
- No configurable properties.

.. _table_rte_flow_action_count:

.. table:: COUNT

   +---------------+
   | Field         |
   +===============+
   | no properties |
   +---------------+

Query structure to retrieve and reset flow rule counters:

.. _table_rte_flow_query_count:

.. table:: COUNT query

   +---------------+-----+-----------------------------------+
   | Field         | I/O | Value                             |
   +===============+=====+===================================+
   | ``reset``     | in  | reset counter after query         |
   +---------------+-----+-----------------------------------+
   | ``hits_set``  | out | ``hits`` field is set             |
   +---------------+-----+-----------------------------------+
   | ``bytes_set`` | out | ``bytes`` field is set            |
   +---------------+-----+-----------------------------------+
   | ``hits``      | out | number of hits for this rule      |
   +---------------+-----+-----------------------------------+
   | ``bytes``     | out | number of bytes through this rule |
   +---------------+-----+-----------------------------------+

动作: ``DUP``
^^^^^^^^^^^^^^^

Duplicates packets to a given queue index.

This is normally combined with QUEUE, however when used alone, it is
actually similar to QUEUE + PASSTHRU.

- Non-terminating by default.

.. _table_rte_flow_action_dup:

.. table:: DUP

   +-----------+------------------------------------+
   | Field     | Value                              |
   +===========+====================================+
   | ``index`` | queue index to duplicate packet to |
   +-----------+------------------------------------+

动作: ``RSS``
^^^^^^^^^^^^^^^

Similar to QUEUE, except RSS is additionally performed on packets to spread
them among several queues according to the provided parameters.

Note: RSS hash result is stored in the ``hash.rss`` mbuf field which
overlaps ``hash.fdir.lo``. Since `Action: MARK`_ sets the ``hash.fdir.hi``
field only, both can be requested simultaneously.

- Terminating by default.

.. _table_rte_flow_action_rss:

.. table:: RSS

   +--------------+------------------------------+
   | Field        | Value                        |
   +==============+==============================+
   | ``rss_conf`` | RSS parameters               |
   +--------------+------------------------------+
   | ``num``      | number of entries in queue[] |
   +--------------+------------------------------+
   | ``queue[]``  | queue indices to use         |
   +--------------+------------------------------+

动作: ``PF``
^^^^^^^^^^^^^^

Redirects packets to the physical function (PF) of the current device.

- No configurable properties.
- Terminating by default.

.. _table_rte_flow_action_pf:

.. table:: PF

   +---------------+
   | Field         |
   +===============+
   | no properties |
   +---------------+

动作: ``VF``
^^^^^^^^^^^^^^

Redirects packets to a virtual function (VF) of the current device.

Packets matched by a VF pattern item can be redirected to their original VF
ID instead of the specified one. This parameter may not be available and is
not guaranteed to work properly if the VF part is matched by a prior flow
rule or if packets are not addressed to a VF in the first place.

- Terminating by default.

.. _table_rte_flow_action_vf:

.. table:: VF

   +--------------+--------------------------------+
   | Field        | Value                          |
   +==============+================================+
   | ``original`` | use original VF ID if possible |
   +--------------+--------------------------------+
   | ``vf``       | VF ID to redirect packets to   |
   +--------------+--------------------------------+

Negative types
~~~~~~~~~~~~~~

All specified pattern items (``enum rte_flow_item_type``) and actions
(``enum rte_flow_action_type``) use positive identifiers.

The negative space is reserved for dynamic types generated by PMDs during
run-time. PMDs may encounter them as a result but must not accept negative
identifiers they are not aware of.

A method to generate them remains to be defined.

Planned types
~~~~~~~~~~~~~

Pattern item types will be added as new protocols are implemented.

Variable headers support through dedicated pattern items, for example in
order to match specific IPv4 options and IPv6 extension headers would be
stacked after IPv4/IPv6 items.

Other action types are planned but are not defined yet. These include the
ability to alter packet data in several ways, such as performing
encapsulation/decapsulation of tunnel headers.

规则管理
----------------

A rather simple API with few functions is provided to fully manage flow
rules.

Each created flow rule is associated with an opaque, PMD-specific handle
pointer. The application is responsible for keeping it until the rule is
destroyed.

Flows rules are represented by ``struct rte_flow`` objects.

校验
~~~~~~~~~~

Given that expressing a definite set of device capabilities is not
practical, a dedicated function is provided to check if a flow rule is
supported and can be created.

.. code-block:: c

   int
   rte_flow_validate(uint8_t port_id,
                     const struct rte_flow_attr *attr,
                     const struct rte_flow_item pattern[],
                     const struct rte_flow_action actions[],
                     struct rte_flow_error *error);

The flow rule is validated for correctness and whether it could be accepted
by the device given sufficient resources. The rule is checked against the
current device mode and queue configuration. The flow rule may also
optionally be validated against existing flow rules and device resources.
This function has no effect on the target device.

The returned value is guaranteed to remain valid only as long as no
successful calls to ``rte_flow_create()`` or ``rte_flow_destroy()`` are made
in the meantime and no device parameter affecting flow rules in any way are
modified, due to possible collisions or resource limitations (although in
such cases ``EINVAL`` should not be returned).

Arguments:

- ``port_id``: port identifier of Ethernet device.
- ``attr``: flow rule attributes.
- ``pattern``: pattern specification (list terminated by the END pattern
  item).
- ``actions``: associated actions (list terminated by the END action).
- ``error``: perform verbose error reporting if not NULL. PMDs initialize
  this structure in case of error only.

Return values:

- 0 if flow rule is valid and can be created. A negative errno value
  otherwise (``rte_errno`` is also set), the following errors are defined.
- ``-ENOSYS``: underlying device does not support this functionality.
- ``-EINVAL``: unknown or invalid rule specification.
- ``-ENOTSUP``: valid but unsupported rule specification (e.g. partial
  bit-masks are unsupported).
- ``EEXIST``: collision with an existing rule. Only returned if device
  supports flow rule collision checking and there was a flow rule
  collision. Not receiving this return code is no guarantee that creating
  the rule will not fail due to a collision.
- ``ENOMEM``: not enough memory to execute the function, or if the device
  supports resource validation, resource limitation on the device.
- ``-EBUSY``: action cannot be performed due to busy device resources, may
  succeed if the affected queues or even the entire port are in a stopped
  state (see ``rte_eth_dev_rx_queue_stop()`` and ``rte_eth_dev_stop()``).

创建
~~~~~~~~

Creating a flow rule is similar to validating one, except the rule is
actually created and a handle returned.

.. code-block:: c

   struct rte_flow *
   rte_flow_create(uint8_t port_id,
                   const struct rte_flow_attr *attr,
                   const struct rte_flow_item pattern[],
                   const struct rte_flow_action *actions[],
                   struct rte_flow_error *error);

Arguments:

- ``port_id``: port identifier of Ethernet device.
- ``attr``: flow rule attributes.
- ``pattern``: pattern specification (list terminated by the END pattern
  item).
- ``actions``: associated actions (list terminated by the END action).
- ``error``: perform verbose error reporting if not NULL. PMDs initialize
  this structure in case of error only.

Return values:

A valid handle in case of success, NULL otherwise and ``rte_errno`` is set
to the positive version of one of the error codes defined for
``rte_flow_validate()``.

销毁
~~~~~~~~~~~

Flow rules destruction is not automatic, and a queue or a port should not be
released if any are still attached to them. Applications must take care of
performing this step before releasing resources.

.. code-block:: c

   int
   rte_flow_destroy(uint8_t port_id,
                    struct rte_flow *flow,
                    struct rte_flow_error *error);


Failure to destroy a flow rule handle may occur when other flow rules depend
on it, and destroying it would result in an inconsistent state.

This function is only guaranteed to succeed if handles are destroyed in
reverse order of their creation.

Arguments:

- ``port_id``: port identifier of Ethernet device.
- ``flow``: flow rule handle to destroy.
- ``error``: perform verbose error reporting if not NULL. PMDs initialize
  this structure in case of error only.

Return values:

- 0 on success, a negative errno value otherwise and ``rte_errno`` is set.

刷新
~~~~~

Convenience function to destroy all flow rule handles associated with a
port. They are released as with successive calls to ``rte_flow_destroy()``.

.. code-block:: c

   int
   rte_flow_flush(uint8_t port_id,
                  struct rte_flow_error *error);

In the unlikely event of failure, handles are still considered destroyed and
no longer valid but the port must be assumed to be in an inconsistent state.

Arguments:

- ``port_id``: port identifier of Ethernet device.
- ``error``: perform verbose error reporting if not NULL. PMDs initialize
  this structure in case of error only.

Return values:

- 0 on success, a negative errno value otherwise and ``rte_errno`` is set.

查询
~~~~~

Query an existing flow rule.

This function allows retrieving flow-specific data such as counters. Data
is gathered by special actions which must be present in the flow rule
definition.

.. code-block:: c

   int
   rte_flow_query(uint8_t port_id,
                  struct rte_flow *flow,
                  enum rte_flow_action_type action,
                  void *data,
                  struct rte_flow_error *error);

Arguments:

- ``port_id``: port identifier of Ethernet device.
- ``flow``: flow rule handle to query.
- ``action``: action type to query.
- ``data``: pointer to storage for the associated query data type.
- ``error``: perform verbose error reporting if not NULL. PMDs initialize
  this structure in case of error only.

Return values:

- 0 on success, a negative errno value otherwise and ``rte_errno`` is set.

详细错误报告
-----------------------

The defined *errno* values may not be accurate enough for users or
application developers who want to investigate issues related to flow rules
management. A dedicated error object is defined for this purpose:

.. code-block:: c

   enum rte_flow_error_type {
       RTE_FLOW_ERROR_TYPE_NONE, /**< No error. */
       RTE_FLOW_ERROR_TYPE_UNSPECIFIED, /**< Cause unspecified. */
       RTE_FLOW_ERROR_TYPE_HANDLE, /**< Flow rule (handle). */
       RTE_FLOW_ERROR_TYPE_ATTR_GROUP, /**< Group field. */
       RTE_FLOW_ERROR_TYPE_ATTR_PRIORITY, /**< Priority field. */
       RTE_FLOW_ERROR_TYPE_ATTR_INGRESS, /**< Ingress field. */
       RTE_FLOW_ERROR_TYPE_ATTR_EGRESS, /**< Egress field. */
       RTE_FLOW_ERROR_TYPE_ATTR, /**< Attributes structure. */
       RTE_FLOW_ERROR_TYPE_ITEM_NUM, /**< Pattern length. */
       RTE_FLOW_ERROR_TYPE_ITEM, /**< Specific pattern item. */
       RTE_FLOW_ERROR_TYPE_ACTION_NUM, /**< Number of actions. */
       RTE_FLOW_ERROR_TYPE_ACTION, /**< Specific action. */
   };

   struct rte_flow_error {
       enum rte_flow_error_type type; /**< Cause field and error types. */
       const void *cause; /**< Object responsible for the error. */
       const char *message; /**< Human-readable error message. */
   };

Error type ``RTE_FLOW_ERROR_TYPE_NONE`` stands for no error, in which case
remaining fields can be ignored. Other error types describe the type of the
object pointed by ``cause``.

If non-NULL, ``cause`` points to the object responsible for the error. For a
flow rule, this may be a pattern item or an individual action.

If non-NULL, ``message`` provides a human-readable error message.

This object is normally allocated by applications and set by PMDs in case of
error, the message points to a constant string which does not need to be
freed by the application, however its pointer can be considered valid only
as long as its associated DPDK port remains configured. Closing the
underlying device or unloading the PMD invalidates it.

Caveats
-------

- DPDK does not keep track of flow rules definitions or flow rule objects
  automatically. Applications may keep track of the former and must keep
  track of the latter. PMDs may also do it for internal needs, however this
  must not be relied on by applications.

- Flow rules are not maintained between successive port initializations. An
  application exiting without releasing them and restarting must re-create
  them from scratch.

- API operations are synchronous and blocking (``EAGAIN`` cannot be
  returned).

- There is no provision for reentrancy/multi-thread safety, although nothing
  should prevent different devices from being configured at the same
  time. PMDs may protect their control path functions accordingly.

- Stopping the data path (TX/RX) should not be necessary when managing flow
  rules. If this cannot be achieved naturally or with workarounds (such as
  temporarily replacing the burst function pointers), an appropriate error
  code must be returned (``EBUSY``).

- PMDs, not applications, are responsible for maintaining flow rules
  configuration when stopping and restarting a port or performing other
  actions which may affect them. They can only be destroyed explicitly by
  applications.

For devices exposing multiple ports sharing global settings affected by flow
rules:

- All ports under DPDK control must behave consistently, PMDs are
  responsible for making sure that existing flow rules on a port are not
  affected by other ports.

- Ports not under DPDK control (unaffected or handled by other applications)
  are user's responsibility. They may affect existing flow rules and cause
  undefined behavior. PMDs aware of this may prevent flow rules creation
  altogether in such cases.

PMD接口
-------------

The PMD interface is defined in ``rte_flow_driver.h``. It is not subject to
API/ABI versioning constraints as it is not exposed to applications and may
evolve independently.

It is currently implemented on top of the legacy filtering framework through
filter type *RTE_ETH_FILTER_GENERIC* that accepts the single operation
*RTE_ETH_FILTER_GET* to return PMD-specific *rte_flow* callbacks wrapped
inside ``struct rte_flow_ops``.

This overhead is temporarily necessary in order to keep compatibility with
the legacy filtering framework, which should eventually disappear.

- PMD callbacks implement exactly the interface described in `Rules
  management`_, except for the port ID argument which has already been
  converted to a pointer to the underlying ``struct rte_eth_dev``.

- Public API functions do not process flow rules definitions at all before
  calling PMD functions (no basic error checking, no validation
  whatsoever). They only make sure these callbacks are non-NULL or return
  the ``ENOSYS`` (function not supported) error.

This interface additionally defines the following helper functions:

- ``rte_flow_ops_get()``: get generic flow operations structure from a
  port.

- ``rte_flow_error_set()``: initialize generic flow error structure.

More will be added over time.

Device compatibility
--------------------

No known implementation supports all the described features.

Unsupported features or combinations are not expected to be fully emulated
in software by PMDs for performance reasons. Partially supported features
may be completed in software as long as hardware performs most of the work
(such as queue redirection and packet recognition).

However PMDs are expected to do their best to satisfy application requests
by working around hardware limitations as long as doing so does not affect
the behavior of existing flow rules.

The following sections provide a few examples of such cases and describe how
PMDs should handle them, they are based on limitations built into the
previous APIs.

全局位掩码
~~~~~~~~~~~~~~~~

Each flow rule comes with its own, per-layer bit-masks, while hardware may
support only a single, device-wide bit-mask for a given layer type, so that
two IPv4 rules cannot use different bit-masks.

The expected behavior in this case is that PMDs automatically configure
global bit-masks according to the needs of the first flow rule created.

Subsequent rules are allowed only if their bit-masks match those, the
``EEXIST`` error code should be returned otherwise.

Unsupported layer types
~~~~~~~~~~~~~~~~~~~~~~~

Many protocols can be simulated by crafting patterns with the `Item: RAW`_
type.

PMDs can rely on this capability to simulate support for protocols with
headers not directly recognized by hardware.

``ANY`` pattern item
~~~~~~~~~~~~~~~~~~~~

This pattern item stands for anything, which can be difficult to translate
to something hardware would understand, particularly if followed by more
specific types.

Consider the following pattern:

.. _table_rte_flow_unsupported_any:

.. table:: Pattern with ANY as L3

   +-------+-----------------------+
   | Index | Item                  |
   +=======+=======================+
   | 0     | ETHER                 |
   +-------+-----+---------+-------+
   | 1     | ANY | ``num`` | ``1`` |
   +-------+-----+---------+-------+
   | 2     | TCP                   |
   +-------+-----------------------+
   | 3     | END                   |
   +-------+-----------------------+

Knowing that TCP does not make sense with something other than IPv4 and IPv6
as L3, such a pattern may be translated to two flow rules instead:

.. _table_rte_flow_unsupported_any_ipv4:

.. table:: ANY replaced with IPV4

   +-------+--------------------+
   | Index | Item               |
   +=======+====================+
   | 0     | ETHER              |
   +-------+--------------------+
   | 1     | IPV4 (zeroed mask) |
   +-------+--------------------+
   | 2     | TCP                |
   +-------+--------------------+
   | 3     | END                |
   +-------+--------------------+

|

.. _table_rte_flow_unsupported_any_ipv6:

.. table:: ANY replaced with IPV6

   +-------+--------------------+
   | Index | Item               |
   +=======+====================+
   | 0     | ETHER              |
   +-------+--------------------+
   | 1     | IPV6 (zeroed mask) |
   +-------+--------------------+
   | 2     | TCP                |
   +-------+--------------------+
   | 3     | END                |
   +-------+--------------------+

Note that as soon as a ANY rule covers several layers, this approach may
yield a large number of hidden flow rules. It is thus suggested to only
support the most common scenarios (anything as L2 and/or L3).

Unsupported actions
~~~~~~~~~~~~~~~~~~~

- When combined with `Action: QUEUE`_, packet counting (`Action: COUNT`_)
  and tagging (`Action: MARK`_ or `Action: FLAG`_) may be implemented in
  software as long as the target queue is used by a single rule.

- A rule specifying both `Action: DUP`_ + `Action: QUEUE`_ may be translated
  to two hidden rules combining `Action: QUEUE`_ and `Action: PASSTHRU`_.

- When a single target queue is provided, `Action: RSS`_ can also be
  implemented through `Action: QUEUE`_.

Flow rules priority
~~~~~~~~~~~~~~~~~~~

While it would naturally make sense, flow rules cannot be assumed to be
processed by hardware in the same order as their creation for several
reasons:

- They may be managed internally as a tree or a hash table instead of a
  list.
- Removing a flow rule before adding another one can either put the new rule
  at the end of the list or reuse a freed entry.
- Duplication may occur when packets are matched by several rules.

For overlapping rules (particularly in order to use `Action: PASSTHRU`_)
predictable behavior is only guaranteed by using different priority levels.

Priority levels are not necessarily implemented in hardware, or may be
severely limited (e.g. a single priority bit).

For these reasons, priority levels may be implemented purely in software by
PMDs.

- For devices expecting flow rules to be added in the correct order, PMDs
  may destroy and re-create existing rules after adding a new one with
  a higher priority.

- A configurable number of dummy or empty rules can be created at
  initialization time to save high priority slots for later.

- In order to save priority levels, PMDs may evaluate whether rules are
  likely to collide and adjust their priority accordingly.

Future evolutions
-----------------

- A device profile selection function which could be used to force a
  permanent profile instead of relying on its automatic configuration based
  on existing flow rules.

- A method to optimize *rte_flow* rules with specific pattern items and
  action types generated on the fly by PMDs. DPDK should assign negative
  numbers to these in order to not collide with the existing types. See
  `Negative types`_.

- Adding specific egress pattern items and actions as described in
  `Attribute: Traffic direction`_.

- Optional software fallback when PMDs are unable to handle requested flow
  rules so applications do not have to implement their own.

API migration
-------------

Exhaustive list of deprecated filter types (normally prefixed with
*RTE_ETH_FILTER_*) found in ``rte_eth_ctrl.h`` and methods to convert them
to *rte_flow* rules.

``MACVLAN`` to ``ETH`` → ``VF``, ``PF``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*MACVLAN* can be translated to a basic `Item: ETH`_ flow rule with a
terminating `Action: VF`_ or `Action: PF`_.

.. _table_rte_flow_migration_macvlan:

.. table:: MACVLAN conversion

   +--------------------------+---------+
   | Pattern                  | Actions |
   +===+=====+==========+=====+=========+
   | 0 | ETH | ``spec`` | any | VF,     |
   |   |     +----------+-----+ PF      |
   |   |     | ``last`` | N/A |         |
   |   |     +----------+-----+         |
   |   |     | ``mask`` | any |         |
   +---+-----+----------+-----+---------+
   | 1 | END                  | END     |
   +---+----------------------+---------+

``ETHERTYPE`` to ``ETH`` → ``QUEUE``, ``DROP``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*ETHERTYPE* is basically an `Item: ETH`_ flow rule with a terminating
`Action: QUEUE`_ or `Action: DROP`_.

.. _table_rte_flow_migration_ethertype:

.. table:: ETHERTYPE conversion

   +--------------------------+---------+
   | Pattern                  | Actions |
   +===+=====+==========+=====+=========+
   | 0 | ETH | ``spec`` | any | QUEUE,  |
   |   |     +----------+-----+ DROP    |
   |   |     | ``last`` | N/A |         |
   |   |     +----------+-----+         |
   |   |     | ``mask`` | any |         |
   +---+-----+----------+-----+---------+
   | 1 | END                  | END     |
   +---+----------------------+---------+

``FLEXIBLE`` to ``RAW`` → ``QUEUE``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*FLEXIBLE* can be translated to one `Item: RAW`_ pattern with a terminating
`Action: QUEUE`_ and a defined priority level.

.. _table_rte_flow_migration_flexible:

.. table:: FLEXIBLE conversion

   +--------------------------+---------+
   | Pattern                  | Actions |
   +===+=====+==========+=====+=========+
   | 0 | RAW | ``spec`` | any | QUEUE   |
   |   |     +----------+-----+         |
   |   |     | ``last`` | N/A |         |
   |   |     +----------+-----+         |
   |   |     | ``mask`` | any |         |
   +---+-----+----------+-----+---------+
   | 1 | END                  | END     |
   +---+----------------------+---------+

``SYN`` to ``TCP`` → ``QUEUE``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*SYN* is a `Item: TCP`_ rule with only the ``syn`` bit enabled and masked,
and a terminating `Action: QUEUE`_.

Priority level can be set to simulate the high priority bit.

.. _table_rte_flow_migration_syn:

.. table:: SYN conversion

   +-----------------------------------+---------+
   | Pattern                           | Actions |
   +===+======+==========+=============+=========+
   | 0 | ETH  | ``spec`` | unset       | QUEUE   |
   |   |      +----------+-------------+         |
   |   |      | ``last`` | unset       |         |
   |   |      +----------+-------------+         |
   |   |      | ``mask`` | unset       |         |
   +---+------+----------+-------------+---------+
   | 1 | IPV4 | ``spec`` | unset       | END     |
   |   |      +----------+-------------+         |
   |   |      | ``mask`` | unset       |         |
   |   |      +----------+-------------+         |
   |   |      | ``mask`` | unset       |         |
   +---+------+----------+---------+---+         |
   | 2 | TCP  | ``spec`` | ``syn`` | 1 |         |
   |   |      +----------+---------+---+         |
   |   |      | ``mask`` | ``syn`` | 1 |         |
   +---+------+----------+---------+---+         |
   | 3 | END                           |         |
   +---+-------------------------------+---------+

``NTUPLE`` to ``IPV4``, ``TCP``, ``UDP`` → ``QUEUE``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*NTUPLE* is similar to specifying an empty L2, `Item: IPV4`_ as L3 with
`Item: TCP`_ or `Item: UDP`_ as L4 and a terminating `Action: QUEUE`_.

A priority level can be specified as well.

.. _table_rte_flow_migration_ntuple:

.. table:: NTUPLE conversion

   +-----------------------------+---------+
   | Pattern                     | Actions |
   +===+======+==========+=======+=========+
   | 0 | ETH  | ``spec`` | unset | QUEUE   |
   |   |      +----------+-------+         |
   |   |      | ``last`` | unset |         |
   |   |      +----------+-------+         |
   |   |      | ``mask`` | unset |         |
   +---+------+----------+-------+---------+
   | 1 | IPV4 | ``spec`` | any   | END     |
   |   |      +----------+-------+         |
   |   |      | ``last`` | unset |         |
   |   |      +----------+-------+         |
   |   |      | ``mask`` | any   |         |
   +---+------+----------+-------+         |
   | 2 | TCP, | ``spec`` | any   |         |
   |   | UDP  +----------+-------+         |
   |   |      | ``last`` | unset |         |
   |   |      +----------+-------+         |
   |   |      | ``mask`` | any   |         |
   +---+------+----------+-------+         |
   | 3 | END                     |         |
   +---+-------------------------+---------+

``TUNNEL`` to ``ETH``, ``IPV4``, ``IPV6``, ``VXLAN`` (or other) → ``QUEUE``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*TUNNEL* matches common IPv4 and IPv6 L3/L4-based tunnel types.

In the following table, `Item: ANY`_ is used to cover the optional L4.

.. _table_rte_flow_migration_tunnel:

.. table:: TUNNEL conversion

   +-------------------------------------------------------+---------+
   | Pattern                                               | Actions |
   +===+==========================+==========+=============+=========+
   | 0 | ETH                      | ``spec`` | any         | QUEUE   |
   |   |                          +----------+-------------+         |
   |   |                          | ``last`` | unset       |         |
   |   |                          +----------+-------------+         |
   |   |                          | ``mask`` | any         |         |
   +---+--------------------------+----------+-------------+---------+
   | 1 | IPV4, IPV6               | ``spec`` | any         | END     |
   |   |                          +----------+-------------+         |
   |   |                          | ``last`` | unset       |         |
   |   |                          +----------+-------------+         |
   |   |                          | ``mask`` | any         |         |
   +---+--------------------------+----------+-------------+         |
   | 2 | ANY                      | ``spec`` | any         |         |
   |   |                          +----------+-------------+         |
   |   |                          | ``last`` | unset       |         |
   |   |                          +----------+---------+---+         |
   |   |                          | ``mask`` | ``num`` | 0 |         |
   +---+--------------------------+----------+---------+---+         |
   | 3 | VXLAN, GENEVE, TEREDO,   | ``spec`` | any         |         |
   |   | NVGRE, GRE, ...          +----------+-------------+         |
   |   |                          | ``last`` | unset       |         |
   |   |                          +----------+-------------+         |
   |   |                          | ``mask`` | any         |         |
   +---+--------------------------+----------+-------------+         |
   | 4 | END                                               |         |
   +---+---------------------------------------------------+---------+

``FDIR`` to most item types → ``QUEUE``, ``DROP``, ``PASSTHRU``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*FDIR* is more complex than any other type, there are several methods to
emulate its functionality. It is summarized for the most part in the table
below.

A few features are intentionally not supported:

- The ability to configure the matching input set and masks for the entire
  device, PMDs should take care of it automatically according to the
  requested flow rules.

  For example if a device supports only one bit-mask per protocol type,
  source/address IPv4 bit-masks can be made immutable by the first created
  rule. Subsequent IPv4 or TCPv4 rules can only be created if they are
  compatible.

  Note that only protocol bit-masks affected by existing flow rules are
  immutable, others can be changed later. They become mutable again after
  the related flow rules are destroyed.

- Returning four or eight bytes of matched data when using flex bytes
  filtering. Although a specific action could implement it, it conflicts
  with the much more useful 32 bits tagging on devices that support it.

- Side effects on RSS processing of the entire device. Flow rules that
  conflict with the current device configuration should not be
  allowed. Similarly, device configuration should not be allowed when it
  affects existing flow rules.

- Device modes of operation. "none" is unsupported since filtering cannot be
  disabled as long as a flow rule is present.

- "MAC VLAN" or "tunnel" perfect matching modes should be automatically set
  according to the created flow rules.

- Signature mode of operation is not defined but could be handled through a
  specific item type if needed.

.. _table_rte_flow_migration_fdir:

.. table:: FDIR conversion

   +----------------------------------------+-----------------------+
   | Pattern                                | Actions               |
   +===+===================+==========+=====+=======================+
   | 0 | ETH, RAW          | ``spec`` | any | QUEUE, DROP, PASSTHRU |
   |   |                   +----------+-----+                       |
   |   |                   | ``last`` | N/A |                       |
   |   |                   +----------+-----+                       |
   |   |                   | ``mask`` | any |                       |
   +---+-------------------+----------+-----+-----------------------+
   | 1 | IPV4, IPv6        | ``spec`` | any | MARK                  |
   |   |                   +----------+-----+                       |
   |   |                   | ``last`` | N/A |                       |
   |   |                   +----------+-----+                       |
   |   |                   | ``mask`` | any |                       |
   +---+-------------------+----------+-----+-----------------------+
   | 2 | TCP, UDP, SCTP    | ``spec`` | any | END                   |
   |   |                   +----------+-----+                       |
   |   |                   | ``last`` | N/A |                       |
   |   |                   +----------+-----+                       |
   |   |                   | ``mask`` | any |                       |
   +---+-------------------+----------+-----+                       |
   | 3 | VF, PF (optional) | ``spec`` | any |                       |
   |   |                   +----------+-----+                       |
   |   |                   | ``last`` | N/A |                       |
   |   |                   +----------+-----+                       |
   |   |                   | ``mask`` | any |                       |
   +---+-------------------+----------+-----+                       |
   | 4 | END                                |                       |
   +---+------------------------------------+-----------------------+

``HASH``
~~~~~~~~

There is no counterpart to this filter type because it translates to a
global device setting instead of a pattern item. Device settings are
automatically set according to the created flow rules.

``L2_TUNNEL`` to ``VOID`` → ``VXLAN`` (or others)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All packets are matched. This type alters incoming packets to encapsulate
them in a chosen tunnel type, optionally redirect them to a VF as well.

The destination pool for tag based forwarding can be emulated with other
flow rules using `Action: DUP`_.

.. _table_rte_flow_migration_l2tunnel:

.. table:: L2_TUNNEL conversion

   +---------------------------+--------------------+
   | Pattern                   | Actions            |
   +===+======+==========+=====+====================+
   | 0 | VOID | ``spec`` | N/A | VXLAN, GENEVE, ... |
   |   |      |          |     |                    |
   |   |      |          |     |                    |
   |   |      +----------+-----+                    |
   |   |      | ``last`` | N/A |                    |
   |   |      +----------+-----+                    |
   |   |      | ``mask`` | N/A |                    |
   |   |      |          |     |                    |
   +---+------+----------+-----+--------------------+
   | 1 | END                   | VF (optional)      |
   +---+                       +--------------------+
   | 2 |                       | END                |
   +---+-----------------------+--------------------+
