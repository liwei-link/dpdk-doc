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

.. _Ring_Library:

Ring库
============

ring可用于队列管理，其底层原理不是无限大小的链表，rte_ring库有如下属性: 

*   FIFO（先进先出）

*   大小固定, 指针存储在表中

*   无锁化

*   多消费者/单消费者出队

*   多生产者/单生产者入队

*   Bulk dequeue - 从队列中取出指定数量对象，若可出队数量与指定数量不同操作失败，否则成功。

*   Bulk enqueue - 向队列中加入指定数量对象，若可入队数量与指定数量不同操作失败，否则成功。

*   Burst dequeue - 从队列中取出指定数量对象，若队列对象数量不足则取出所有可用对象。

*   Burst enqueue - 向队列中加入指定数量对象，若队列空间不足则加入可用空间数量的对象。

非链式队列数据结构的优势如下:

*   更加快速; 仅需要一次Compare-And-Swap指令sizeof(void \*)，而不是多次double-Compare-And-Swap指令。

*   比完全无锁队列还简单。

*   适合bulk enqueue/dequeue操作。
    因为指针存储在表中，多个对象出队不会产生像链式队列那么多的缓存未命中。
    多对象bulk dequeue操作也不会比单个对象更加耗费资源。

劣势:

*   大小固定

*   比链式队列更耗内存。一个空ring中至少包含N个指针。

一个Ring的简单表示，消费者和生产者的头尾指针指向该数据结构中对象。

.. _figure_ring1:

.. figure:: img/ring1.*

   Ring 结构


FreeBSD* 中Ring实现参考
----------------------------------------------

下面的代码在FreeBSD 8.0中加入，在一些网络设备驱动使用（至少Intel的驱动中有使用）:

    * `bufring.h in FreeBSD <http://svn.freebsd.org/viewvc/base/release/8.0.0/sys/sys/buf_ring.h?revision=199625&amp;view=markup>`_

    * `bufring.c in FreeBSD <http://svn.freebsd.org/viewvc/base/release/8.0.0/sys/kern/subr_bufring.c?revision=199625&amp;view=markup>`_

Linux* 中无锁环形缓冲区（Ring Buffer）
------------------------------

下面的链接描述了 `Linux无锁环形缓冲区（Ring Buffer）设计 <http://lwn.net/Articles/340400/>`_.

附加特性
-------------------

名称
~~~~

ring由名称唯一标识。无法创建名称相同的两个ring（第二个rte_ring_create()会返回NULL）

使用案例
---------

    *  DPDK应用间通信

    *  内存池分配器

Ring缓冲区剖析
------------------------

本节解释ring缓冲区如何运转。ring由两对头尾组成，一个用于生产者，另一个用于消费者。
即下列图示中的prod_head, prod_tail, cons_head和cons_tail.

每张图代表ring的一种简化状态，ring是环形缓冲区。函数局部变量放在图片顶部，ring结构放在图片底部。

单生产者入队
~~~~~~~~~~~~~~~~~~~~~~~

本节解释了一个生产者如何向ring中加入一个对象。
本例中只有一个生产者，仅有生产者头尾（prod_head和prod_tail）会被修改。

初始时prod_head和prod_tail指向同一位置。

入队第一步
^^^^^^^^^^^^^^^^^^

首先, 拷贝 *ring->prod_head* 和 ring->cons_tail 到局部变量。
局部变量prod_next指向prod_head的下一个元素，在bulk enqueue操作中则指向后面数个元素。

ring空间不足（通过cons_tail检测）返回错误。

.. _figure_ring-enqueue1:

.. figure:: img/ring-enqueue1.*

   入队第一步


入队第二步
^^^^^^^^^^^^^^^^^^^

修改 *ring->prod_head* 使其和prod_next指向同一位置。

新增对象（obj4）的指针被复制到ring中。

.. _figure_ring-enqueue2:

.. figure:: img/ring-enqueue2.*

   入队第二步


入队最后一步
^^^^^^^^^^^^^^^^^

一旦对象加入到ring中，就把ring->prod_tail修改成和 *ring->prod_head* 一样的指向。入队完成。


.. _figure_ring-enqueue3:

.. figure:: img/ring-enqueue3.*

   入队最后一步


单消费者出队
~~~~~~~~~~~~~~~~~~~~~~~

本节解释了一个消费者如何从ring中取出一个对象。
本例中只有一个消费者，仅有消费者头尾（cons_head和cons_tail）会被修改。

初始时cons_head和cons_tail指向同一位置。

出队第一步
^^^^^^^^^^^^^^^^^^

首先，拷贝 ring->cons_head 和 ring->prod_tail 到局部变量。
局部变量 cons_next 指向cons_head的下一个元素，在bulk dequeue操作中则指向后面数个元素。

ring空间不足（通过 prod_tail 检测）返回错误。


.. _figure_ring-dequeue1:

.. figure:: img/ring-dequeue1.*

   出队第一步


出队第二步
^^^^^^^^^^^^^^^^^^^

修改 ring->cons_head 使其和 cons_next 指向同一位置。

出队对象（obj1）的指针被复制用户指针中，从而返回给用户。

.. _figure_ring-dequeue2:

.. figure:: img/ring-dequeue2.*

   出队第二步


出队最后一步
^^^^^^^^^^^^^^^^^

一旦对象加入到ring中，就把ring->prod_tail修改成和 *ring->prod_head* 一样的指向。入队完成。
最后，把 ring->cons_tail 修改成和 ring->cons_head 一样的指向。出队完成。


.. _figure_ring-dequeue3:

.. figure:: img/ring-dequeue3.*

   出队最后一步


多生产者入队
~~~~~~~~~~~~~~~~~~~~~~~~~~

本节解释两个生产者如何同时向队列中加入对象。
本例中仅有生产者头尾（prod_head和prod_tail）会被修改。

初始时prod_head和prod_tail指向同一位置。

多生产者入队第一步
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在两个核上，拷贝 *ring->prod_head* 和 ring->cons_tail 到局部变量。
局部变量 prod_next 指向 prod_head 的下一个元素。在bulk enqueue操作中则指向后面数个元素。

ring空间不足（通过 cons_tail 检测）返回错误。

.. _figure_ring-mp-enqueue1:

.. figure:: img/ring-mp-enqueue1.*

   多生产者入队第一步


多生产者入队第二步
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

修改 ring->prod_head 使其和 prod_next 指向同一位置。
该操作使用Compare-And-Swap (CAS) 指令完成，可以原子地完成以下操作: 

*   如果 ring->prod_head 和局部变量 prod_head 不同，CAS操作失败，返回第一步重新开始。

*   否则，ring->prod_head 被设置成局部的 prod_next，CAS操作成功，处理继续往下进行。

图中，核1操作成功，核2操作失败，核2从第一步重新开始。

.. _figure_ring-mp-enqueue2:

.. figure:: img/ring-mp-enqueue2.*

   多生产者入队第二步


多生产者入队第三步
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

核2上CAS操作重试成功。

核1更新了一个ring上的元素（obj4），核2更新另外一个（obj5）。

.. _figure_ring-mp-enqueue3:

.. figure:: img/ring-mp-enqueue3.*

   多生产者入队第三步


多生产者入队第四步
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

现在每个核都想更新 ring->prod_tail。但是，仅在 ring->prod_tail 和局部变量 prod_head相等时，
该核才能更新它。这只在核1上满足，因此 ring->prod_tail 由核1更新，核1完成操作。

.. _figure_ring-mp-enqueue4:

.. figure:: img/ring-mp-enqueue4.*

   多生产者入队第四步


多生产者入队最后一步
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

一旦 ring->prod_tail 由核1更新完成，核2也可以更新 ring->prod_tail，
因此核2上也完成了操作。

.. _figure_ring-mp-enqueue5:

.. figure:: img/ring-mp-enqueue5.*

   多生产者入队最后一步


32位模索引
~~~~~~~~~~~~~~~~~~~~~

在前面的图中， prod_head, prod_tail, cons_head 和 cons_tail 的索引用箭头表示。
在实际的实现当中，这些索引值并是不如我们所假设的从0到size(ring)-1。实际的索引是从0到2^32 -1，
当访问指针表时mask索引值。32位模也意味着如果索引操作（如，加/减）结果溢出，则会自动地做2^32模操作。

下面的两个例子会帮助你理解ring中索引的使用。

.. note::

    为了便于解释，使用16位代替32位模操作。另外，这四个索引被定义成16位无符号整数，现实中使用的是32位无符号整数。


.. _figure_ring-modulo1:

.. figure:: img/ring-modulo1.*

   32位模索引 - 实例1


ring包含11000个实例


.. _figure_ring-modulo2:

.. figure:: img/ring-modulo2.*

      32位模索引 - 实例2

ring包含12536个实例

.. note::

    为了便于理解，上例中我们使用了65536模操作。在真实环境中，这是低效率的，但是能够自动处理结果溢出问题。

代码始终会使生产者和消费者保持一定距离（0到size(ring)-1）。由于这个特性，我们能够在2个基于32位模的索引值上做减法:
这就是为什么索引溢出不是问题。

At any time, entries and free_entries are between 0 and size(ring)-1,
even if only the first term of subtraction has overflowed:

.. code-block:: c

    uint32_t entries = (prod_tail - cons_head);
    uint32_t free_entries = (mask + cons_tail -prod_head);

参考
----------

    *   `bufring.h in FreeBSD <http://svn.freebsd.org/viewvc/base/release/8.0.0/sys/sys/buf_ring.h?revision=199625&amp;view=markup>`_ (version 8)

    *   `bufring.c in FreeBSD <http://svn.freebsd.org/viewvc/base/release/8.0.0/sys/kern/subr_bufring.c?revision=199625&amp;view=markup>`_ (version 8)

    *   `Linux Lockless Ring Buffer Design <http://lwn.net/Articles/340400/>`_
