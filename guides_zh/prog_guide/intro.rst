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

介绍
============

这篇文档提供了DPDK的软件架构、开发环境信息和程序优化指导。

示例应用编码、编译和运行命令的详细信息请参考 *Sample Applications User Guide*。

编译和运行应用的详细方法请参考 *DPDK Getting Started Guide*。

文档导航
---------------------

DPDK文档建议阅读顺序:

*	**发布说明** : 提供了指定发布版本的信息，包括支持的特性、使用限制、修复的问题、已知问题等。
	同时也以FAQ格式提供常见问题解答。
	
*	**入门指南** : 描述了如何安装和配置DPDK软件，带领用户快速地使用DPDK。
	
*	**FreeBSD*入门指南** : FreeBSD*系统的DPDK使用指南。
	DPDK Release 1.6.0版本增加了对FreeBSD*的支持。根据这篇指导可以帮助你在FreeBSD*系统上快速入门DPDK。
	
*	**开发者指南** (当前文档): 描述:
	
	*   Linux环境下软件架构和使用方法 (通过示例介绍使用方法)
	
	*   构建系统(包括使用根目录Makefile命令构建开发套件和应用)和应用移植指导
	
	*   一些优化手段，包括已经在软件中使用的和新开发时应该考虑到的
	
	同时也提供了一组术语。
	
*	**API参考手册** : 提供了DPDK函数、数据结构和其他的程序结构的详细信息
	
*	**示例程序指导(Sample Applications User Guide)**: 
	介绍了一些示例程序。
	对每个示例程序提供了功能描述，编译和运行命令。

相关出版物
--------------------

下面的文档提供了使用DPDK开发应用的相关信息:

*   Intel® 64 and IA-32 Architectures Software Developer's Manual Volume 3A: System Programming Guide
