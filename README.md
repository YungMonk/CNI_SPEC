# CNI_SPEC
对kubernetes中CNI官方规范的翻译

# 容器网络接口（CNI）说明
- [容器网络接口（CNI）说明](#容器网络接口（CNI）说明)
  - [版本](#版本)
      - [己发布的版本](#己发布的版本)
  - [概述](#概述)
  - [概要](#概要)
  - [第1节: 网络配置格式](#第1节:-网络配置格式)
    - [配置格式](#配置格式)
      - [插件配置对象:](#插件配置对象)
      - [配置举例](#配置举例)
  - [第2节: 执行协议](#第2节:-执行协议)
    - [概述](#描述)
    - [参数](#参数)
    - [错误](#错误)
    - [CNI 函数](#CNI-函数)
      - [`ADD`: 向网络中添加容器或者应用修改](#add-向网络中添加容器或者应用修改)
      - [`DEL`: 从网络中移除容器或者撤销所有的修改](#del-从网络中移除容器或者撤销所有的修改)
      - [`CHECK`: 检查容器的网络是否符合预期](#check-检查容器的网络是否符合预期)
      - [`VERSION`: 探测插件版本支持](#version-探测插件版本支持)
  - [第3节: 运行网络配置](#第3节:-运行网络配置)
    - [生命周期和调用顺序](#生命周期和调用顺序)
    - [Attachment 的参数](#attachment-的参数)
    - [添加 attachment](#添加-attachment)
    - [删除 attachment](#删除-attachment)
    - [校验 attachment](#校验-attachment)
    - [从插件配置派生出执行配置](#从插件配置派生出执行配置)
      - [派生出 `runtimeConfig`](#派生出-`runtimeConfig`)
  - [第4节: 插件委托](#第4节:-插件委托)
    - [委托插件协议](#委托插件协议)
    - [委托插件执行过程](#委托插件执行过程)
  - [第5节: 结果类型](#第5节:-结果类型)
    - [成功](#成功)
      - [委托插件 (IPAM)](#委托插件-(IPAM))
    - [返回错误](#返回错误)
    - [版本](#返回类型的版本)
  - [附录: 示例](#附录:-示例)
    - [Add 操作示例](#Add-操作示例)
    - [Check 操作示例](#Check-操作示例)
    - [Delete 操作示例](#Delete-操作示例)

## 版本

这是CNI **规范** 的 **1.0.0 预发布版** 并在积极开发中.

注意： 规范的版本是 **独立于CNI library、CNI Plugin版本的**  (例如：CNI[发行版](https://github.com/containernetworking/cni/releases)).

#### 己发布的版本

规范发行版的 Git tags 如下.

| tag                                                                                  | 链接                                                                       |重大修改                     |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- | --------------------------------- |
| [`spec-v0.4.0`](https://github.com/containernetworking/cni/releases/tag/spec-v0.4.0) | [spec at v0.4.0](https://github.com/containernetworking/cni/blob/spec-v0.4.0/SPEC.md) | 在 `DEL` 中引入 `CHECK` 命令并将其传递到 `prevResult` |
| [`spec-v0.3.1`](https://github.com/containernetworking/cni/releases/tag/spec-v0.3.1) | [spec at v0.3.1](https://github.com/containernetworking/cni/blob/spec-v0.3.1/SPEC.md) | 无 (仅限于排版修改)              |
| [`spec-v0.3.0`](https://github.com/containernetworking/cni/releases/tag/spec-v0.3.0) | [spec at v0.3.0](https://github.com/containernetworking/cni/blob/spec-v0.3.0/SPEC.md) | 丰富了结果类型，插件链接 |
| [`spec-v0.2.0`](https://github.com/containernetworking/cni/releases/tag/spec-v0.2.0) | [spec at v0.2.0](https://github.com/containernetworking/cni/blob/spec-v0.2.0/SPEC.md) | `VERSION` 命令                   |
| [`spec-v0.1.0`](https://github.com/containernetworking/cni/releases/tag/spec-v0.1.0) | [spec at v0.1.0](https://github.com/containernetworking/cni/blob/spec-v0.1.0/SPEC.md) | 初始版                   |

*不要依赖所有的稳定版. 未来, 可能会对过去正确的标记做出改变.*


## 概述

本文为Linux上的应用程序容器(_容器网络接口_ 或 _CNI_)给出了一个通用的基于插件的网络解决方案.

对于这个方案, 我们明确的提出3点要求或者说是定义:
- _container_ 是一个网络隔离域, 尽管规范中没有定义具体的隔离技术. 但是这些技术可以是 [网络命名空间][namespaces] 可以是虚拟机, 等等
- _network_ 指的是一组能唯一寻址并可以通信的端点. 端点可以是一个单独的容器（如上所述）、一台机器或一些其他网络设备（例如路由器）。容器可以从概念上添加到一个或多个网络或者从中删除.
- _runtime_ 是负责执行CNI插件的程序.
- _plugin_ 是给应用指定网络配置的程序.

本文档旨在指定 “runtimes” 和 “plugins” 之间的接口. 使用[RFC 2119][rfc-2119]中所述的关键字 "must", "must not", "required", "shall", "shall not", "should", "should not", "recommended", "may", "optional".

[namespaces]: http://man7.org/linux/man-pages/man7/namespaces.7.html
[rfc-2119]: https://www.ietf.org/rfc/rfc2119.txt



## 概要

CNI规范定义:

1. 管理员定义网络配置的格式.
2. 容器运行时向网络插件发出请求的协议.
3. 根据提供的配置执行插件的过程.
4. 插件将功能委托给其他插件的过程.
5. 用于插件向运行时返回结果类型数据.

## 第1节: 网络配置格式

CNI为管理员定义了一种网络配置格式。 它包含了容器运行时和插件使用的指令。在插件执行时，运行时解释这种配置格式，并将其转换为传递给插件的形式.

通常，网络配置是静态的。从概念上讲，它可以被认为是“在磁盘上”，尽管CNI规范实际上并不需要这样做。

### 配置格式

一个网络配置由一个具有以下键的JSON对象组成:

- `cniVersion` (string): CNI规范的[语义化版本 2.0.0](https://semver.org)，该配置列表和所有单独的配置都符合该版本,目前“1.0.0”。
- `name` (string): 网络名称。这在主机(或其他管理域)上的所有网络配置中应该是唯一的。必须以字母数字字符开头，后跟一个或多个字母数字字符、下划线、点(.)或连字符(-)的任意组合。
- `disableCheck` (boolean): 值为 `true` 或 `false`.  如果 `disableCheck` 是 `true`, 运行时不能为网络配置列表调用 `CHECK` 函数.  允许管理员防止 `检查` 一些由已知插件组合而导致的虚假错误。
- `plugins` (list): CNI插件及其配置的列表，即插件配置对象的列表。

#### 插件配置对象:
插件配置对象可能包含比这里定义的字段更多的字段。

运行时必须像第3节中定义的那样，将这些字段原封不动地传递给插件。

**必须的字段:**
- `type` (string): 匹配磁盘上CNI插件二进制文件的名称。不能包含系统文件路径中不允许的字符(例如/或\\).

**可选字段，协议中使用:**
- `capabilities` (dictionary): [第3节](#Deriving-runtimeConfig)定义

**保留字段，协议中使用:**
这些字段是运行时在执行时生成的，因此不应该在配置中使用。
- `runtimeConfig`
- `args`
- 以 `cni.dev/` 开头的任何字段

**可选的字段，公共的:**
这些字段没有在协议中使用，但是对插件有一定的标准含义。
插件使用这些配置中的字段，应该符合它们的预期语义。

- `ipMasq` (boolean): 如果插件支持，请在此网络的主机上设置IP伪装，如果主机将作为子网的网关，这些子网不能路由到分配给容器的IP，那么这个选项是需要的
- `ipam` (dictionary): IPAM (IP地址管理)对象中特定的值:
    - `type` (string): 指定IPAM插件可执行文件的文件名。不能包含系统文件路径中不允许的字符（例如/或\\）
- `dns` (dictionary, 可选): NDS对象中特定的值:
    - `nameservers` (list of strings, 可选): 该网络所知的DNS域名服务器的优先级排序列表。列表中的每个条目都是包含IPv4或IPv6地址的字符串。
    - `domain` (string, 可选): 用于短主机名查找的本地域。
    - `search` (list of strings, 可选): 用于短主机名查找的按优先级排序的搜索域列表。大多数解析程序将优先于 `域`。
    - `options` (list of strings, 可选): 可以传递给解析器的选项列表。

**其它字段:**
插件可以定义它们接受的附加字段，如果使用未知字段调用，可能会产生错误。在执行转换时，运行时必须在插件配置对象中保留未知字段。

#### 配置举例
```jsonc
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "plugins": [
    {
      "type": "bridge",
      // 插件的具体参数
      "bridge": "cni0",
      "keyA": ["some more", "plugin specific", "configuration"],
      
      "ipam": {
        "type": "host-local",
        // ipam specific
        "subnet": "10.1.0.0/16",
        "gateway": "10.1.0.1",
        "routes": [
            {"dst": "0.0.0.0/0"}
        ]
      },
      "dns": {
        "nameservers": [ "10.1.0.1" ]
      }
    },
    {
      "type": "tuning",
      "capabilities": {
        "mac": true
      },
      "sysctl": {
        "net.core.somaxconn": "500"
      }
    },
    {
        "type": "portmap",
        "capabilities": {"portMappings": true}
    }
  ]
}
```

## 第2节: 执行协议

### 描述

CNI协议基于容器运行时调用的二进制文件的执行。CNI定义了插件二进制文件和运行时之间的协议。

CNI插件负责以某种方式配置容器的网络接口。插件分为两大类：
* 接口类插件, 在容器内创建一个网络接口并确保它具有连接性
* 链式插件, 调整已创建接口的配置（但可能需要创建更多接口）

运行时通过环境变量和配置将参数传递给插件。它通过标准输入提供配置。如果操作成功，插件将在stdout上返回[结果](#Section-5-Result-Types)，如果操作失败，插件将在stderr上返回错误。配置和结果用JSON编码

参数定义特定于调用的设置，而配置(除了一些例外)对于任何给定的网络都是相同的。

运行时必须在运行时的网络域中执行插件。(在大多数情况下，这意味着根网络名称空间/ `dom0`)。

### 参数

协议参数通过OS环境变量传递给插件.

- `CNI_COMMAND`: 表示所需的操作; `ADD`, `DEL`, `CHECK`, or `VERSION`.
- `CNI_CONTAINERID`: Container ID. 容器的唯一明文标识符，由运行时分配。一定不能空。必须以字母数字字符开头，后跟一个或多个字母数字字符、下划线()、点(.)或连字符(-)的任意组合.
- `CNI_NETNS`: 对容器“隔离域”的引用。如果使用网络名称空间，则指向网络命名空间的路径（例如`/run/netns/[nsname]`）
- `CNI_IFNAME`: 要在容器内创建的接口的名称;如果插件不能使用这个接口名，它必须返回一个错误。
- `CNI_ARGS`: 用户在调用时传入的额外参数。以分号分隔的字母数字键值对;如："FOO=BAR;ABC=123"
- `CNI_PATH`: 搜索CNI插件可执行文件的路径列表。路径由特定于操作系统的列表分隔符分隔;例如Linux上的':'和Windows上的';'

### 错误

插件成功时必须返回0，失败时返回非0。如果插件遇到错误，它应该输出["error" result structure](#-error)(见下面)



### CNI 函数

CNI接口中定义了4个函数: `ADD`, `DEL`, `CHECK`, `VERSION`. 它们通过 `CNI_COMMAND` 环境变量传递给插件.

#### `ADD`: 向网络中添加容器或者应用修改

一个CNI插件，在接收到 `ADD` 命令时，也应该
- 在容器的 `CNI_NETNS` 中创建 `CNI_IFNAME` 定义的接口，或者
- 在容器 `CNI_NETNS` 中调整由 `CNI_IFNAME` 定义的接口的配置

如果CNI插件创建或者调整成功，它必须在标准输出上输出一个[result structure](#Success)(见下文)。如果插件提供了一个 `prevResult` 作为其输入配置的一部分，它必须通过传递它或适当修改它来处理 `prevResult` 。

如果容器中已经存在请求名称的接口，CNI插件必须返回一个错误。

运行时不应该对相同的 `(CNI_CONTAINERID, CNI_IFNAME)` 元组调用两次 `ADD` (中间没有调用过DEL)。这意味着，只有在每次添加都使用不同的接口名时，才能将给定的容器ID多次添加到特定的网络中。


**输入:**

运行时通过标准输入给插件的配置对象提供一个 `JSON-serialized`

必须的环境参数:
- `CNI_COMMAND`
- `CNI_CONTAINERID`
- `CNI_NETNS`
- `CNI_IFNAME`

可选的环境参数:
- `CNI_ARGS`
- `CNI_PATH`

#### `DEL`: 从网络中移除容器或者撤销所有的修改

一个CNI插件，在接收到 `DEL` 命令时，也应该
- 在容器的 `CNI_NETNS` 中删除 `CNI_IFNAME` 定义的接口
- 撤销插件在 `ADD` 功能中应用的任何修改

插件通常应该无误地完成 `DEL` 操作，即使缺少一些资源。例如，IPAM插件通常应该释放IP分配并返回success，即使容器网络命名空间不再存在，除非该网络命名空间对于IPAM管理至关重要。DHCP通常会在容器的网络接口上发送一个“释放”消息，因为DHCP租约有一个生命周期，这个释放动作不会被认为是关键的，如果这个动作失败了也不会返回错误。再举一个例子， `bridge` 插件应该将DEL操作委托给IPAM插件，并清理自己的资源，即使容器网络命名空间和容器网络接口都不再存在。

插件必须接受同一组（`CNI_CONTAINERID`、`CNI_IFNAME`）对 `DEL` 函数的多次调用，并返回成功，即使缺少相关接口或添加任何修改。

**输入:**

运行时通过标准输入给插件的配置对象提供一个 `JSON-serialized`

必须的环境参数:
- `CNI_COMMAND`
- `CNI_CONTAINERID`
- `CNI_IFNAME`

可选的环境参数:
- `CNI_NETNS`
- `CNI_ARGS`
- `CNI_PATH`


#### `CHECK`: 检查容器的网络是否符合预期
`CHECK` 是运行时探测现有容器状态的一种方法。

插件注意事项:
- 插件必须参考 `prevResult` 来确定预期的接口和地址。
- 该插件必须允许后边链接的插件对网络资源的修改，如：路由，使用 `ADD` 函数
- 如果CNI结果类型(接口、地址或路由)中包含的资源是由插件创建的，并且在 `prevResult` 中列出，但是缺少或处于无效状态，插件应该返回一个错误
- 如果其他未在Result类型中跟踪的资源(如以下所示)缺失或处于无效状态，插件应该返回一个错误
  - 防火墙策略
  - 流量控制
  - IP 预留
  - 外部依赖项，如连接所需的守护进程
  - 等等
- 如果插件发现到容器通常不可达的情况，则应该返回一个错误。
- 插件必须在处理ADD之后立即调用的 `CHECK` ，因此应该允许对任何异步资源有合理的收敛延迟。
- 插件应该对任何委托(例如IPAM)插件调用 `CHECK` ，并将所有错误传递给调用者。


运行时注意事项:

- 运行时不能对未被添加或在最后一次添加之后被删除的容器调用 `CHECK` 

- 如果在[配置](#配置格式)中 `disableCheck` 被设置为true，运行时不能调用 `CHECK`

- 运行时必须在网络配置中包含一个 `prevResult` 字段，该字段包含紧接前面的容器 `ADD` 的 `Result` 。运行时可能希望使用 libcni 对缓存结果的支持。

- 当插件返回错误时，运行时可能会选择停止执行链的 `CHECK`

- 运行时可以在成功的 `ADD` 之后立即执行 `CHECK` ，直到从网络上删除容器为止。

- 运行时可能认为 `CHECK` 失败意味着容器永远处于配置错误的状态。


**输入:**

运行时通过标准输入给插件的配置对象提供一个 `JSON-serialized`

必须的环境参数:
- `CNI_COMMAND`
- `CNI_CONTAINERID`
- `CNI_NETNS`
- `CNI_IFNAME`

可选的环境参数:
- `CNI_ARGS`
- `CNI_PATH`

除了 `CNI_PATH` 之外，所有的参数都必须与容器对应的 `ADD` 相同

#### `VERSION`: 探测插件版本支持
插件应该通过标准输出一个json序列化的版本结果对象(见下文)。

**输入:**

一个json序列化的对象，带有以下键:
- `cniVersion`: 使用的协议版本

必须的环境参数:
- `CNI_COMMAND`


## 第3节: 运行网络配置

本节描述容器运行时如何解释网络配置(如第1节中定义的)并相应地执行插件。运行时可能希望在容器中添加、删除或检查网络配置。这导致了一系列插件 `ADD` 、 `DELETE` 或 `CHECK` 的相应执行。本节还定义了如何转换网络配置并提供给插件。

在容器上对网络配置的操作称为 `Attachment`。Attachment 可以由( `CNI_CONTAINERID` , `CNI_IFNAME` )元组唯一标识。

### 生命周期和调用顺序

- 运行中的容器必须在调用任何插件之前为容器创建一个新的命名空间
- 运行中的容器不能为同一个容器进行并行调用操作，但是可以为不同的容器进行并行调用操作，因为这中间会涉及多个 `Attachment`
- 插件必须具有处理跨不同容器并发执行的事务。如有必要，他们必须对共享资源（如：IPAM数据库）进行加锁
- 容器运行时必须确保 _Add_ 后面紧跟着相应的 _Delete_。唯一的例外是发生灾难性故障时，例如节点丢失。即使 _Add_ 失败，也必须执行 _Delete_
- _Delete_ 调用之后可能会有额外的 _Deletes_ 调用
- 网络配置不应该在 _Add_ 和 _Delete_ 之间更改
- 网络配置不应在 _Attachment_ 之间更改

### Attachment 的参数

尽管 Attachment 之间的网络配置不应更改，但是容器运行时提供的某些参数是每个 attachment 的，这些参数是:

- **Container ID**: 容器的唯一明文标识符，由运行时分配。一定不能空。必须以字母数字字符开头，后跟一个或多个字母数字字符、下划线()、点(.)或连字符(-)的任意组合。在执行期间，始终将其设置为 `CNI_CONTAINERID` 参数。
- **Namespace**: 对容器“隔离域”的引用。如果使用网络命名空间，则为网络命名空间的路径(例如:`/run/netns/[nsname]`)。在执行过程中，始终设置为 `CNI_NETNS` 参数。
- **Container interface name**: 要在容器内创建的接口的名称。在执行期间，始终设置为 `CNI_IFNAME` 参数。
- **Generic Arguments**: 与特定 Attachment 相关的键-值字符串形式的额外参数。在执行期间，始终设置为 `CNI_ARGS` 参数。
- **Capability Arguments**: 这些也是键值对。键是字符串，而值是任何json可序列化的类型。键和值是按照[约定](https://github.com/containernetworking/cni/blob/master/CONVENTIONS.md)定义的

此外，必须向运行时提供一个搜索CNI插件的 _路径列表_。这也必须通过 `CNI_PATH` 环境变量在执行期间提供给插件

### 添加 attachment
网络配置文件中 `plugins` 字段下的所有配置的含义，
1. `type` 字段中的值是可执行文件，如果可执行文件不存在那么这是一个错误的值
2. 使用以下参数从插件配置中获得请求配置:
    - 如果这是列表中的第一个插件，则不提供前一个结果，
    - 对于所有其他插件，前一个结果是前一个插件的结果
3. 使用 `CNI_COMMAND=ADD` 执行插件二进制文件。提供上面定义的参数作为环境变量。通过标准输入提供派生的配置
4. 如果插件返回错误，停止执行并将错误返回给调用者

运行时必须持久化地存储最后一个插件返回的结果，因为在检查和删除操作中还需要使用这些结果。

### 删除 attachment
删除网络 attachment 与添加网络 attachment 基本相同，只是有一些关键区别:
- 插件列表以**相反的顺序**执行
- 前面提供的结果始终是 _add_ 操作的最终结果

对于在网络配置的 `plugins` 字段中定义的每个插件，*按相反的顺序*
1. `type` 字段中的值是可执行文件，如果可执行文件不存在那么这是一个错误的值
2. 从插件配置派生请求配置，使用初始 _add_ 操作的前一个结果。
3. 使用 `CNI_COMMAND=DEL` 执行插件二进制文件。提供上面定义的参数作为环境变量。通过标准输入提供派生的配置
4. 如果插件返回错误，停止执行并将错误返回给调用者

如果所有插件都返回成功，则向调用者返回成功

### 校验 attachment

运行时还可能要求确认每个插件给定的附件是否仍然有效。运行时必须使用与 _add_ 操作相同的 attachment 参数。

校验和添加相似，有两点不同:
- 前面提供的结果始终是 _add_ 操作的最终结果
- 如果网络配置定义了 `disableCheck`，那么总是向调用者返回成功

网络配置文件中 `plugins` 字段下的所有配置的含义，
1. `type` 字段中的值是可执行文件，如果可执行文件不存在那么这是一个错误的值
2. 从插件配置派生请求配置，使用初始 _add_ 操作的前一个结果。
3. 使用 `CNI_COMMAND=CHECK` 执行插件二进制文件。提供上面定义的参数作为环境变量。通过标准输入提供派生的配置
4. 如果插件返回错误，停止执行并将错误返回给调用者

如果所有插件都返回成功，则向调用者返回成功

### 从插件配置派生出执行配置
网络配置格式(即要执行的插件配置列表)必须转换为插件能够理解的格式(即单个插件配置)。本节描述这种转换。

单个插件调用的执行配置也是JSON。它由插件配置组成，除了指定的添加和删除外，基本上没有改变。

运行时必须将以下字段插入到执行配置中:
- `cniVersion`: 从网络配置的 `cniVersion` 字段中获取
- `name`: 从网络配置的 `name` 字段中获取
- `runtimeConfig`: 一个JSON对象，由插件提供的功能和运行时请求的功能组成(更多细节在下面)
- `prevResult`: 一个JSON对象，由 "previous" 插件返回的结果类型组成。"previous" 的含义由具体的操作(_add_, _delete_, _check_)来定义。

以下字段必须被运行时**删除**:
- `capabilities`

所有其他字段都应该原样传递.

#### 派生出 `runtimeConfig`
尽管 CNI_ARGS 被提供给所有插件，但没有明确的指出它们是否将被使用， _Capability_ 参数需要在配置中显式声明。因此，运行时可以确定给定的网络配置是否支持特定的 _capability_。_capability_ 不是由规范定义的——相反，它们是有文档记录的[约定](https://github.com/containernetworking/cni/blob/master/CONVENTIONS.md)。

正如第1节中定义的，插件配置包括一个可选的键， `capabilities` 。这个例子展示了一个支持 `portMapping` 功能的插件:

```json
{
  "type": "myPlugin",
  "capabilities": {
    "portMappings": true
  }
}
```

`runtimeConfig` 参数来自网络配置中的 `capabilities` 字段和运行时生成的 _capability arguments_。具体来说，插件配置支持并由运行时提供的任何功能都应该插入到 `runtimeConfig` 中。

因此，上面的例子可能会导致下面的内容作为执行配置的一部分传递给插件:

```json
{
  "type": "myPlugin",
  "runtimeConfig": {
    "portMappings": [ { "hostPort": 8080, "containerPort": 80, "protocol": "tcp" } ]
  }
  ...
}
```

## 第4节: 插件委托

有一些操作，无论出于什么原因，不能合理地实现为一个独立的链式插件。相反，一个CNI插件可能希望将一些功能委托给另一个插件。一个常见的例子是IP地址管理。

作为其操作的一部分，CNI插件需要为该接口分配(和维护)一个IP地址，并安装与该接口相关的任何必要路由。这给了CNI插件很大的灵活性，但也给它带来了很大的负担。许多CNI插件需要相同的代码来支持用户可能需要的几种IP管理方案(例如dhcp, host-local)。CNI插件可以选择将IP管理委托给其他插件。

为了减轻负担并使IP管理策略与CNI插件类型正交，我们定义了第三种类型的插件——IP地址管理插件(IPAM插件)，以及一个用于插件将功能委托给其他插件的协议。

然而，在IPAM插件执行的适当时刻调用它是CNI插件的责任，而不是运行时的责任。IPAM插件必须确定接口IP/子网、网关和路由，并将此信息返回到“主”插件进行应用。IPAM插件可以通过协议(例如dhcp)、存储在本地文件系统中的数据、网络配置文件中的“IPAM”部分等获取信息。


### 委托插件协议

与CNI插件一样，委托插件是通过运行可执行文件来调用的。在预定义的路径列表中搜索可执行文件，通过 `CNI_PATH` 指定给CNI插件。被委托的插件必须接收所有传递给CNI插件的相同的环境变量。与CNI插件一样，委托插件通过`stdin`接收网络配置，并通过`stdout`输出结果。

委托插件给“上层”插件提供了*完整的网络配置信息*，换句话说，在 IPAM 插件中，不全是和 IPAM 有关的配置信息

成功由返回码0和输出到stdout的 _Success_ 结果类型表示。

### 委托插件执行过程

当一个插件执行一个委托插件时，它应该这样做:
- 通过搜索 `CNI_PATH` 环境变量中提供的目录来查找插件二进制文件
- 使用接收到的相同环境和配置执行该插件
- 确保委托插件的标准错误输出到调用插件的标准错误

如果一个插件使用 `CNI_COMMAND=CHECK` 或 `DEL` 执行，那么它也必须执行任何委托的插件。如果任何一个被委托的插件返回错误，错误应该由上面的插件返回

如果在 `ADD` 中，委托的插件失败，“上层”插件应该在返回失败之前再次使用DEL执行。

## 第5节: 结果类型

插件可以返回三种结果类型之一:

- _Success_ (or _Abbreviated Success_)
- _Error_
- _Version_

### 成功

插件提供了一个 `prevResult` 键作为其请求配置的一部分，必须将其输出为其结果，包括插件所做的任何可能的修改。如果插件没有做任何将反映在 _Success_ 结果类型中的更改，那么它必须输出与提供的 `prevResult` 等价的结果

成功执行ADD操作后，插件必须输出一个具有以下键的JSON对象:

- `cniVersion`: 与输入时提供的版本相同——字符串"1.0.0"
- `interfaces`: 由附件创建的所有接口的数组，包括任何主机级接口:
    - `name`: 接口名
    - `mac`: 接口的硬件地址(如果适用)
    - `sandbox`: 接口的隔离域引用(例如到网络命名空间的路径)，如果在主机上，则为空。对于容器内部创建的接口，这应该是通过 `CNI_NETNS` 传递的值
- `ips`: 由这个附件分配的ip。插件可能包括分配给外部容器的ip
    - `address` (string): CIDR表示法的IP地址 (如: "192.168.1.3/24").
    - `gateway` (string): 此子网的默认网关(如果存在的话)
    - `interface` (uint): [CNI插件结果](#result)的接口列表索引，指示该IP配置应该应用于哪个接口
- `routes`: 由这个附件创建的路由:
    - `dst`: 路由的目的地，用CIDR表示
    - `gw`: 下一跳地址。如果未设置，则可能使用`ips`阵列中`gateway`中的值
- `dns`: 由DNS配置信息组成的字典
    - `nameservers` (list of strings): 该网络所知的DNS域名服务器的优先级排序列表。列表中的每个条目都是包含IPv4或IPv6地址的字符串。
    - `domain` (string): 用于短主机名查找的本地域
    - `search` (list of strings): 用于短主机名查找的优先级排序搜索域列表。大多数解析器会优先选择 `域名`
    - `options` (list of strings): 可以传递给解析器的选项列表

#### 委托插件 (IPAM)
被委托的插件可能会省略不相关的部分

Delegated IPAM plugins must return an abbreviated _Success_ object. Specifically, it is missing the `interfaces` array, as well as the `interface` entry in `ips`.
委托的IPAM插件必须返回一个简略的_Success_对象。具体来说，在`ips`中缺少`接口`数组和`接口`入口。


### 返回错误

如果遇到错误，插件应该输出一个带有以下字段的JSON对象:

- `cniVersion`: 与配置提供的值相同
- `code`: 数字错误码，请参阅下面的保留码
- `msg`: 描述错误的短消息
- `details`: 描述错误的较长的消息

例如:

```json
{
  "cniVersion": "1.0.0",
  "code": 7,
  "msg": "Invalid Configuration",
  "details": "Network 192.168.0.0/31 too small to allocate from."
}
```

错误码0-99用于已知错误。100+可以自由用于插件特定的错误


错误码|错误描述
---|---
 `1`|不兼容的 CNI 版本
 `2`|网络配置中不支持的字段。错误消息必须包含不支持的字段的键和值
 `3`|容器未知或不存在。此错误意味着运行时不需要执行任何容器网络清理(例如，调用容器上的`DEL`操作).
 `4`|无效的必要环境变量，如`CNI_COMMAND`, `CNI_CONTAINERID`等。错误消息必须包含无效变量的名称
 `5`|I/O 失败。例如，未能从 *stdin* 读取网络配置字节
 `6`|解码失败。例如，未能从bytes解封送网络配置或未能从字符串解码版本信息
 `7`|网络配置无效。如果网络配置上的某些验证不通过，将引发此错误
 `11`|请稍候重试。如果插件检测到一些应该被清除的临时条件，它可以使用此代码通知运行时它应该稍后重新尝试该操作

此外，stderr可用于非结构化输出，如日志

### 返回类型的版本

在`VERSION`操作时，插件必须输出一个具有以下键的JSON对象:

- `cniVersion`: 输入时指定的`cniVersion`的值
- `supportedVersions`: 支持的规范版本列表

例如:
```json
{
    "cniVersion": "1.0.0",
    "supportedVersions": [ "0.1.0", "0.2.0", "0.3.0", "0.3.1", "0.4.0", "1.0.0" ]
}
```


## 附录: 示例

我们假设前面第1节中所示的[网络配置](#配置举例)。对于这个附件，运行时将生成`portmap`和`mac`功能参数，以及通用参数“argA=foo”。示例使用`CNI_IFNAME=eth0`

### Add 操作示例

容器运行时将为`add`操作执行以下步骤。


1) 用下面的JSON调用 `bridge` 插件, `CNI_COMMAND=ADD`:

```json
{
    "cniVersion": "1.0.0",
    "name": "dbnet",
    "type": "bridge",
    "bridge": "cni0",
    "keyA": ["some more", "plugin specific", "configuration"],
    "ipam": {
        "type": "host-local",
        "subnet": "10.1.0.0/16",
        "gateway": "10.1.0.1"
    },
    "dns": {
        "nameservers": [ "10.1.0.1" ]
    }
}
```

`bridge` 插件将*IPAM*委托给 `host-local` 插件时，将使用完全相同的输入 `CNI_COMMAND=ADD` 执行 `host-local` 二进制文件。

`host-local` 插件返回以下结果:

```json
{
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1"
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
}
```

`bridge` 插件根据委托的 IPAM 配置返回以下结果:

```json
{
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "99:88:77:66:55:44",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
}
```

2) 接下来，使用 `CNI_COMMAND=ADD` 调用 `tuning` 插件。注意，提供了 `prevResult` 和 `mac` 功能参数。传递的请求配置是:

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "tuning",
  "sysctl": {
    "net.core.somaxconn": "500"
  },
  "runtimeConfig": {
    "mac": "00:11:22:33:44:66"
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "99:88:77:66:55:44",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```

插件返回以下结果。请注意，**mac** 已经改变了

```json
{
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
}
```

3) 最后，调用 `portmap` 插件，使用 `CNI_COMMAND=ADD`。注意，`prevResult` 匹配通过 `tuning` 返回的结果:

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "portmap",
  "runtimeConfig": {
    "portMappings" : [
      { "hostPort": 8080, "containerPort": 80, "protocol": "tcp" }
    ]
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```

`portmap` 插件输出的结果与 `bridge` 返回的结果完全相同，因为插件没有修改任何会改变结果的东西(即它只创建了iptables规则)


### Check 操作示例

结合前面的 _Add_ 操作，容器运行时将为 _Check_ 操作执行以下步骤:

1) 首先，通过下面的请求配置调用 `bridge` 插件，包括 _Add_ 操作中最后JSON响应结果中所包含的prevResult字段，**包括更改后的mac**. `CNI_COMMAND=CHECK`

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "bridge",
  "bridge": "cni0",
  "keyA": ["some more", "plugin specific", "configuration"],
  "ipam": {
    "type": "host-local",
    "subnet": "10.1.0.0/16",
    "gateway": "10.1.0.1"
  },
  "dns": {
    "nameservers": [ "10.1.0.1" ]
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```

`bridge` 插件在委托 IPAM 调用 `host-local` 时，`CNI_Command=CHECK`。它不会返回错误。

假设 `bridge` 插件满足要求，它在标准输出时不会产生输出，并以0返回码退出。

2) 接下来用下面的请求配置调用 `tuning` 插件

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "tuning",
  "sysctl": {
    "net.core.somaxconn": "500"
  },
  "runtimeConfig": {
    "mac": "00:11:22:33:44:66"
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```

同样，`tuning` 插件退出，表示成功

3) 最后，使用以下请求配置调用 `portmap` :

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "portmap",
  "runtimeConfig": {
    "portMappings" : [
      { "hostPort": 8080, "containerPort": 80, "protocol": "tcp" }
    ]
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```


### Delete 操作示例

结合之前相同的网络配置JSON列表，容器运行时将为 _Delete_ 操作执行以下步骤。请注意，插件的执行顺序与 _Add_ 和 _Check_ 操作相反。

1) 首先，通过以下请求配置调用 `portmap` 插件, `CNI_COMMAND=DEL`:

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "portmap",
  "runtimeConfig": {
    "portMappings" : [
      { "hostPort": 8080, "containerPort": 80, "protocol": "tcp" }
    ]
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```


2) 然后, 通过以下请求配置调用 `tuning` 插件, `CNI_COMMAND=DEL`:

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "tuning",
  "sysctl": {
    "net.core.somaxconn": "500"
  },
  "runtimeConfig": {
    "mac": "00:11:22:33:44:66"
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```

3) 最后调用 `bridge` 插件:

```json
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "type": "bridge",
  "bridge": "cni0",
  "keyA": ["some more", "plugin specific", "configuration"],
  "ipam": {
    "type": "host-local",
    "subnet": "10.1.0.0/16",
    "gateway": "10.1.0.1"
  },
  "dns": {
    "nameservers": [ "10.1.0.1" ]
  },
  "prevResult": {
    "ips": [
        {
          "address": "10.1.0.5/16",
          "gateway": "10.1.0.1",
          "interface": 2
        }
    ],
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ],
    "interfaces": [
        {
            "name": "cni0",
            "mac": "00:11:22:33:44:55"
        },
        {
            "name": "veth3243",
            "mac": "55:44:33:22:11:11"
        },
        {
            "name": "eth0",
            "mac": "00:11:22:33:44:66",
            "sandbox": "/var/run/netns/blue"
        }
    ],
    "dns": {
      "nameservers": [ "10.1.0.1" ]
    }
  }
}
```


`bridge` 插件在返回之前使用 `CNI_COMMAND=DEL` 执行 `host-local` 委托插件。
