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

.. _Timer_Library:

定时器
=============

定时器库为DPDK执行单元提供定时器服务，使回调函数可以异步执行。
特性:

*   定时器可以是周期的(重复)或者单次(只异步地执行一次)。

*   定时器可以在一个核上加载，在另一个核上执行。必须在调用rte_timer_reset()时指定。

*   精度高 (依赖本地核心定时器超时检查函数rte_timer_manage()的调用频率)。

*   如应用中不需要定时器，可以在编译时关闭，这样就不会调用rte_timer_manage()，从而提升性能。

定时器库使用rte_get_timer_cycles()函数(该函数使用高精度事件定时器(HPET)或CPU时间
戳计数器(TSC))提供可靠的时间基准。

该库提供定时器的增加，删除和重启的接口。该API基于BSD callout()，但有少许不同。
参考 `callout 手册 <http://www.daemon-systems.org/man/callout.9.html>`_.

实现细节
----------------------

定时器以每核为基础，一个核心上所有未决定时器以超时时间为序存放在一个跳跃表数据结构中。
使用的跳跃表有10级，表中的每一项在每一级上出现的概率是¼^level。这意味着表中所有项都会
出现在等级0上，每4个有1个出现在等级1上，每16个有1个出现在等级2上，以此类推到等级9。
因此从表中增加和删除定时器将在log(n)时间内完成，每个表可以有多达4^10个项，也就是每个
核心可以有大约1,000,000个定时器。

定时器结构中有一个status字段，它是一个定时器状态(stopped, pending, running, config)
和所有者(lcore id)的联合体。根据定时器状态，我们可以知道定时器是否出现在表中:

*   STOPPED: 无所有者，不在表中

*   CONFIG: 有所有者，不能被其他核修改，可能在表中(依赖前一个状态)

*   PENDING: 有所有者，在表中

*   RUNNING: 有所有者，不能被其他核修改，在表中

当定时器处于CONFIG或RUNNING状态时不允许重置或停止。修改定时器状态时，
应该使用Compare And Swap指令以确保状态(所有者+状态)可以被原子地修改。

在rte_timer_manage()函数中，跳跃表被作为一个普通列表(等级0列表)遍历，直到遇到未超时的定时器。
在表中没有超时定时器时，为了提高性能，表中第一个定时器的超时时间被存放在定时器表结构本身中。
在64位平台上，该值无需获取这个结构的锁就可以检查。(因为超时时间是64位值，在32位平台上既不使用
CAS指令也不使用锁该值是无法通过一次检查完成的，因此一次带锁的检查可以让额外的检查跳过)，
在32和64平台上，如果本地核的定时器表是空的，rte_timer_manage()函数不带锁的调用会直接返回。

使用案例
---------

定时器库用于周期性调用，比如垃圾回收，或者一些状态机(ARP，桥接等)。

参考
----------

*   `callout 手册 <http://www.daemon-systems.org/man/callout.9.html>`_
    - 在给定时间内执行一个函数的机制。

*   `HPET <http://en.wikipedia.org/wiki/HPET>`_
    - 高精度事件定时器(HPET)的信息。
	