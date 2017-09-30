..  BSD LICENSE
    Copyright(c) 2016 Intel Corporation. All rights reserved.

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


加密设备库
===========================

cryptodev库提供了加密设备的框架，通过定义通用API(支持一组加密操作)进行管理和配置软硬件加密轮询模式驱动。
该框架当前仅支持密码、认证、链式密码/认证和AEAD对称加密操作。

设计原则
-----------------

cryptodev库遵守和DPDK以太网设备框架相同的基本设计原则。加密框架提供了通用的加密设备框架，
支持物理(硬件)和虚拟(软件)加密设备。通用加密API可以管理和配置加密设备，
并且支持由加密轮询模式驱动提供的加密操作。


设备管理
-----------------

设备创建
~~~~~~~~~~~~~~~

物理加密设备是在DPDK初始化阶段通过PCI探测发现的，
每个PCI设备都有唯一标识符(PCI BDF，bus/bridge, device, function)。
物理加密设备可以通过EAL命令行选项加入到DPDK的白/黑名单中。

虚拟设备可以通过两种机制创建，一种使用EAL命令行参数，另外一种在应用中直接调用EAL的API创建。

通过命令行参数 --vdev 创建

.. code-block:: console

   --vdev  'cryptodev_aesni_mb_pmd0,max_nb_queue_pairs=2,max_nb_sessions=1024,socket_id=0'

在应用中使用 rte_vdev_init 创建

.. code-block:: c

   rte_vdev_init("cryptodev_aesni_mb_pmd",
                     "max_nb_queue_pairs=2,max_nb_sessions=1024,socket_id=0")

所有虚拟加密设备都支持以下初始化参数:

* ``max_nb_queue_pairs`` - 设备支持的"队列对"数量
* ``max_nb_sessions`` - 设备支持的会话数量
* ``socket_id`` - 设备所在的socket


设备标识
~~~~~~~~~~~~~~~~~~~~~

每个设备无论是虚拟或者物理的都由两个标识符唯一标识:

- 设备的唯一索引，用于在cryptodev API暴露的所有函数中指定加密设备。

- 设备名称，用于管理或调试时在控制台消息中显示加密设备。为了便于使用，端口名中包含了端口索引。


设备配置
~~~~~~~~~~~~~~~~~~~~

加密设备的配置包含下面的操作:

- 资源申请，包括硬件资源(如果是物理设备)。
- 设备重置到默认状态。
- 统计计数器的初始化。

rte_cryptodev_configure 用于配置加密设备。

.. code-block:: c

   int rte_cryptodev_configure(uint8_t dev_id,
                               struct rte_cryptodev_config *config)

``rte_cryptodev_config`` 结构用于传递配置参数。
它包含所选择的socket、队列对的数量和会话内存池的配置。

.. code-block:: c

    struct rte_cryptodev_config {
        int socket_id;
        /**< Socket to allocate resources on */
        uint16_t nb_queue_pairs;
        /**< Number of queue pairs to configure on device */

        struct {
            uint32_t nb_objs;
            uint32_t cache_size;
        } session_mp;
        /**< Session mempool configuration */
    };


队列对的配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

每个加密设备队列对都是通过 ``rte_cryptodev_queue_pair_setup`` 独立配置的。

.. code-block:: c

    int rte_cryptodev_queue_pair_setup(uint8_t dev_id, uint16_t queue_pair_id,
                const struct rte_cryptodev_qp_conf *qp_conf,
                int socket_id)

    struct rte_cryptodev_qp_conf {
        uint32_t nb_descriptors; /**< Number of descriptors per queue pair */
    };


逻辑核，内存和队列对的关系
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于处理器的逻辑核和接口使用本地内存时，加密设备库和PMD库一样支持NUMA。
因此加密操作和对称加密操作所使用的会话和mbuf应该从本地内存池(从本地内存中创建的内存池)中申请。
如果可能的话，应该保持缓冲区在本地处理器上，这样可以获得最佳性能，
缓冲区描述符应该使用从本地内存池中申请的mbuf填充。

在加密操作、加密会话和数据缓冲区都在本地内存时，run-to-completion模型可以表现的更好，
尤其对于虚拟加密设备。当工作的逻辑核都在同一个处理器上时，pipe-line模型也是这样的。

在同一个加密设备上千万不要让多个逻辑核对同一个队列对进行出或入队操作，
因为这需要全局锁会导致性能下降。但是，可以让一个逻辑核从"由另外一个逻辑核入队的队列"中出队一个操作。
这意味着在包处理流水线中，加密的批量(crypto burst)出入队API是一个逻辑核间信息传递的逻辑场所。

设备特性和能力
---------------------------------

加密设备通过两种机制定义功能，全局设备特性和算法能力。全局设备特性是设备等级的特性，
适用于整个设备，比如硬件加速或者支持对称加密操作。

算法能力机制定义设备支持的独立算法/功能，比如特定对称加密密码或认证操作。


设备特性
~~~~~~~~~~~~~~~

当前定义的加密设备特性:

* 对称加密操作
* 非对称加密操作
* 对称加密操作链
* SSE 加速的 SIMD 向量操作
* AVX 加速的 SIMD 向量操作
* AVX2 加速的 SIMD 向量操作
* AESNI 加速的指令
* 硬件卸载处理


设备操作能力
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

特殊算法(加密PMD支持)的加密功能由操作类型，操作转换，转换标识符和转换细节定义。
完整的加密能力结构体定义参考 *DPDK API Reference*。

.. code-block:: c

   struct rte_cryptodev_capabilities;

每个加密轮询模式驱动都为支持的操作定义了一组私有的能力。下面是支持SHA1_HMAC认证算法和AES_CBC密码算法的PMD的能力示例。

.. code-block:: c

    static const struct rte_cryptodev_capabilities pmd_capabilities[] = {
        {    /* SHA1 HMAC */
            .op = RTE_CRYPTO_OP_TYPE_SYMMETRIC,
            .sym = {
                .xform_type = RTE_CRYPTO_SYM_XFORM_AUTH,
                .auth = {
                    .algo = RTE_CRYPTO_AUTH_SHA1_HMAC,
                    .block_size = 64,
                    .key_size = {
                        .min = 64,
                        .max = 64,
                        .increment = 0
                    },
                    .digest_size = {
                        .min = 12,
                        .max = 12,
                        .increment = 0
                    },
                    .aad_size = { 0 }
                }
            }
        },
        {    /* AES CBC */
            .op = RTE_CRYPTO_OP_TYPE_SYMMETRIC,
            .sym = {
                .xform_type = RTE_CRYPTO_SYM_XFORM_CIPHER,
                .cipher = {
                    .algo = RTE_CRYPTO_CIPHER_AES_CBC,
                    .block_size = 16,
                    .key_size = {
                        .min = 16,
                        .max = 32,
                        .increment = 8
                    },
                    .iv_size = {
                        .min = 16,
                        .max = 16,
                        .increment = 0
                    }
                }
            }
        }
    }


功能发现
~~~~~~~~~~~~~~~~~~~~~~

加密设备的轮询模式驱动通过 ``rte_cryptodev_info_get`` 函数发现特性和能力。

.. code-block:: c

   void rte_cryptodev_info_get(uint8_t dev_id,
                               struct rte_cryptodev_info *dev_info);

用户可以调用该函数查询指定加密PMD并获取所有设备特性和能力。
``rte_cryptodev_info`` 结构体包含所有与设备相关信息。

.. code-block:: c

    struct rte_cryptodev_info {
        const char *driver_name;
        enum rte_cryptodev_type dev_type;
        struct rte_pci_device *pci_dev;

        uint64_t feature_flags;

        const struct rte_cryptodev_capabilities *capabilities;

        unsigned max_nb_queue_pairs;

        struct {
            unsigned max_nb_sessions;
        } sym;
    };


操作处理
--------------------

DPDK数据路径上加密操作的调度是通过一组猝发式异步API执行的。加密设备的队列对通过批量入队API接收一批加密操作。在物理加密设备上，
批量入队API会把操作放到设备的硬件输入队列中。而对虚拟设备，加密操作的处理通常在调用入队操作期间就已经完成。
批量出队API会从加密设备的队列对中获取任何已处理过的可用操作，对于物理设备，
通常是直接从设备的已处理队列中获取，对于虚拟设备则是从存放已处理(入队时)操作的 ``rte_ring`` 中获取。

批量入队/出队 API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

批量入队API使用加密设备标识符和队列对标识符指定要调度处理的加密设备队列对。
``ops`` 是待处理操作数组，数组元素是 ``rte_crypto_op`` 结构体。``nb_ops`` 参数是 ``ops`` 数组的大小。
函数返回实际入队处理操作的数量，返回值等于 ``nb_ops`` 意味着所有的包都已入队。

.. code-block:: c

   uint16_t rte_cryptodev_enqueue_burst(uint8_t dev_id, uint16_t qp_id,
                                        struct rte_crypto_op **ops, uint16_t nb_ops)


出队API和入队API的格式一样，只是 ``nb_ops`` 和 ``ops`` 
参数用于指定用户希望获取已处理操作的最大数量和存放具体操作的数组。
返回值是实际返回操作的数量，该值一定不会大于 ``nb_ops``。

.. code-block:: c

   uint16_t rte_cryptodev_dequeue_burst(uint8_t dev_id, uint16_t qp_id,
                                        struct rte_crypto_op **ops, uint16_t nb_ops)


操作的表示
~~~~~~~~~~~~~~~~~~~~~~~~

加密操作由 rte_crypto_op 结构体表示，该结构中存储了所有加密操作需要的信息。

.. figure:: img/crypto_op.*

操作结构体中包含操作类型和操作状态，操作特有数据的引用，该数据的大小和内容依赖提供的操作而变化。
如果操作是从内存池中申请的，结构体中也包含源内存池。最后还有一个指向用户特定数据的不透明的指针。

如果加密操作是从加密操作内存池中申请的(参加下一节)，也可以通过该操作为应用申请私有内存。

加密PMD处理操作所需要的操作必须要由应用软件指定到 ``rte_crypto_op`` 结构体中。


操作管理和申请
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

加密操作利用Mempool库申请操作缓冲区，cryptodev库提供给了一组API用于管理这些加密操作。
因此，可以确保加密操作在channel和rank间交叉分布，进而获取最近的处理性能。
``rte_crypto_op`` 中包含了一个指明内存池的字段。调用 ``rte_crypto_op_free(op)`` 时，
操作被释放回原始的操作池中。

.. code-block:: c

   extern struct rte_mempool *
   rte_crypto_op_pool_create(const char *name, enum rte_crypto_op_type type,
                             unsigned nb_elts, unsigned cache_size, uint16_t priv_size,
                             int socket_id);


在资源池创建时，会调用 ``rte_crypto_op_init()`` 初始化每一个加密操作，然后调用 
``__rte_crypto_op_reset()`` 根据指定的类型参数配置操作的指定字段。

``rte_crypto_op_alloc()`` 和 ``rte_crypto_op_bulk_alloc()`` 用于从给定的加密操作内存池中申请指定类型的加密操作。
在每个操作返回给申请的用户前会调用 ``__rte_crypto_op_reset()`` ，因此在应用使用前操作总是处于可知的状态。

.. code-block:: c

   struct rte_crypto_op *rte_crypto_op_alloc(struct rte_mempool *mempool,
                                             enum rte_crypto_op_type type)

   unsigned rte_crypto_op_bulk_alloc(struct rte_mempool *mempool,
                                     enum rte_crypto_op_type type,
                                     struct rte_crypto_op **ops, uint16_t nb_ops)

``rte_crypto_op_free()`` 用于应用把操作释放会操作池中。

.. code-block:: c

   void rte_crypto_op_free(struct rte_crypto_op *op)


对称加密的支持
------------------------------

当前cryptodev库支持以下对称加密操作；密码，认证，包括这些操作链，也支持AEAD操作。


会话和会话管理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

会话用于对称加密中存储固定不变的数据，这些数据定义在密码转换中，用于包流的操作处理。
会话中管理了诸如扩展密码索引、HMAC IPADs 和 OPAD这类信息，这些信息需要为特定加密操作计算，
但对于一个流的包基础来说这些信息却是固定不变的。加密会话以最佳方法为底层PMD缓存固定不变的数据，
进而加速了加密操作。

.. figure:: img/cryptodev_sym_sess.*

加密设备框架提供一组利用Mempool库的会话池管理API用于创建和释放会话。

框架也提供了钩子函数(因此PMD可以传入私有会话参数所需的内存数量)用于会话参数的配置和释放功能，
因此PMD可以在销毁会话时管理内存。

**注意**: 在某个特定设备上创建的会话只能用在相同类型的加密设备上，如果尝试在不同类型的设备上使用，
加密操作会失败。

``rte_cryptodev_sym_session_create()`` 用于在加密设备上创建对称会话。
对称的转换链用于指定特定操作和它的参数。转换的相关信息参加下面的章节。

.. code-block:: c

   struct rte_cryptodev_sym_session * rte_cryptodev_sym_session_create(
          uint8_t dev_id, struct rte_crypto_sym_xform *xform);

**注意**: 对于AEAD操作，所选的认证和加密算法必须是对齐的(aligned)，比如AES_GCM。


转换和转换链
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对称加密转换(``rte_crypto_sym_xform``)是一种用于指定加密操作细节的机制。
对于对称操作链，比如密码加密和认证生成，``next`` 指针把转换链接在一起。
支持链的加密设备必须设置对称加密操作链特性的标志。

当前有两种转换类型(密码和认证)指定给AEAD操作。AEAD操作需要把密码和认证转换链接在一起。
同时也很重要的是，转换传递的顺序就是链的顺序。

.. code-block:: c

    struct rte_crypto_sym_xform {
        struct rte_crypto_sym_xform *next;
        /**< next xform in chain */
        enum rte_crypto_sym_xform_type type;
        /**< xform type */
        union {
            struct rte_crypto_auth_xform auth;
            /**< Authentication / hash xform */
            struct rte_crypto_cipher_xform cipher;
            /**< Cipher xform */
        };
    };

API不会限制链在一起的转换数，但是处理该操作的底层的加密设备PMD会限制。

.. figure:: img/crypto_xform_chain.*


对称操作
~~~~~~~~~~~~~~~~~~~~

对称加密操作的结构体包含所有和对称加密操作相关的可变数据。该结构体用于加密，认证，AEAD或者链式操作。

对称操作必须至少有一个源数据缓冲区(``m_src``)、会话类型(session-based/less)、
一个合法的会话(或者session-less模式下的转换)和
根据在会话或转换链中指定的操作类型所需的认证/加密参数。

.. code-block:: c

    struct rte_crypto_sym_op {
        struct rte_mbuf *m_src;
        struct rte_mbuf *m_dst;

        enum rte_crypto_sym_op_sess_type type;

        union {
            struct rte_cryptodev_sym_session *session;
            /**< Handle for the initialised session context */
            struct rte_crypto_sym_xform *xform;
            /**< Session-less API Crypto operation parameters */
        };

        struct {
            struct {
                uint32_t offset;
                uint32_t length;
            } data;   /**< Data offsets and length for ciphering */

            struct {
                uint8_t *data;
                phys_addr_t phys_addr;
                uint16_t length;
            } iv;     /**< Initialisation vector parameters */
        } cipher;

        struct {
            struct {
                uint32_t offset;
                uint32_t length;
            } data;   /**< Data offsets and length for authentication */

            struct {
                uint8_t *data;
                phys_addr_t phys_addr;
                uint16_t length;
            } digest; /**< Digest parameters */

            struct {
                uint8_t *data;
                phys_addr_t phys_addr;
                uint16_t length;
            } aad;    /**< Additional authentication parameters */
        } auth;
    }


非对称加密
-----------------------

cryptodev API当前还不支持非对称功能。

加密设备API
~~~~~~~~~~~~~~~~~

cryptodev库API在文档 *DPDK API Reference* 中有所描述。

