
# 4.1 理解 ip_tables
iptables 只提供了一个名为“iptables”的内存中的命名规则数组，以及关于每个钩子处包的遍历起点等信息。注册了表之后，在用户空间中，可以通过 getsockopt() 和 setsockopt() 来读取和替换其内容。

iptables 不与任何 netfilter 钩子注册：它依赖其他模块来完成这些工作，并在适当的时候向其传递数据；一个模块必须分别注册 netfilter 钩子和 ip_tables，并提供一种机制，在到达钩子时调用 ip_tables。

ip_tables 数据结构
为了方便起见，用户空间和内核中使用的表示规则的数据结构是相同的，尽管有一些字段仅在内核中使用。每个规则由以下部分组成：

每个规则包含以下部分：

- 一个 `struct ipt_entry`。
- 零个或多个 `struct ipt_entry_match` 结构，每个结构后附有可变数量（0 或更多字节）的数据。
- 一个 `struct ipt_entry_target` 结构，后附有可变数量（0 或更多字节）的数据。

规则数据结构的可变性为扩展提供了巨大的灵活性，正如我们将看到的，特别是每个匹配或目标都可以携带任意量的数据。
不过这样做确实也带来了一些陷阱：我们必须注意对齐问题。
我们通过确保 `ipt_entry`、`ipt_entry_match` 和 `ipt_entry_target` 结构是方便大小的，
并且所有数据都向上对齐到使用该宏 `IPT_ALIGN()` 的机器的最大对齐值来解决这个问题。

`struct ipt_entry` 包含以下字段：
- 一个用于匹配 IP 头部特性的 `struct ipt_ip` 部分。
- 一个 `nf_cache` 位域，显示规则检查了包中的哪些部分。
- 一个表示 `ipt_entry_target` 结构开始位置相对于本规则起始位置偏移的 `target_offset` 字段。这个字段应该始终正确对齐（使用 `IPT_ALIGN` 宏）。
- 一个表示规则总大小，包括匹配和目标字段的 `next_offset` 字段。同样也需要使用 IPT_ALIGN 宏进行对齐。
- 一个用于内核跟踪包遍历的 `comefrom` 字段。
- 一个包含匹配到此规则的数据包的包和字节计数器的 `struct ipt_counters` 字段。

`struct ipt_entry_match` 和 `struct ipt_entry_target` 非常相似，
因为它们都包含一个总长度字段（分别为 `match_size` 和 `target_size`），
以及一个用户空间持有匹配或目标名称的联合体和一个用于内核的指针。

## 辅助函数
由于规则数据结构的复杂性，提供了一些辅助函数：

`ipt_get_target()`
此内联函数返回规则目标的指针。

`IPT_MATCH_ITERATE()`
此宏对给定规则中的每个匹配调用给定函数。第一个参数是 `struct ipt_match_entry`，其他参数（如有）则由 IPT_MATCH_ITERATE() 宏提供。函数必须返回零继续迭代，或非零值停止。

`IPT_ENTRY_ITERATE()`
该函数接受一个指针指向一个条目、表中的条目总数和要调用的函数。第一个参数是 `struct ipt_entry`，其他参数（如有）由 IPT_ENTRY_ITERATE() 宏提供。函数必须返回零继续迭代，或非零值停止。

## ip_tables 从用户空间使用
用户空间有四个操作：可以读取当前表、读取信息（钩位置和表大小）、替换表并抓取旧计数器，以及添加新的计数器。

这允许任何原子操作在用户空间中模拟：这是通过 libiptc 库实现的，该库为程序提供了“添加/删除/替换”的便利语义。

由于这些表被转移到内核空间，对于类型不同的用户空间和内核空间规则（如 Sparc64 的 32-bit 用户空间），对齐成为问题。这些问题在 `libiptc.h` 中通过重定义 IPT_ALIGN 解决。

## ip_tables 使用和遍历
内核从指明的钩位置开始遍历。该规则被检查，如果 `struct ipt_ip` 元素匹配，则逐个检查每个 `struct ipt_entry_match`（关联于此匹配的匹配函数将被调用）。

如果匹配函数返回 0，则在该规则上停止迭代。如果它将 `hotdrop` 参数设置为 1，则包也将立即丢弃（这用于某些可疑的包，如 tcp 匹配函数中的包）。

如果迭代继续到末尾，计数器将增加，检查 `struct ipt_entry_target`：如果是标准目标，读取 `verdict` 字段（负值表示一个包决定，正值表示一个跳转偏移量）。

如果答案是正的且偏移不是下一个规则的，则设置变量 `back`，并将先前的 `back` 值放在该条目的 `comefrom` 字段中。

对于非标准目标，调用目标函数：它返回一个决定（非标准目标无法跳跃，这会破坏静态循环检测代码）。决定可以是 `IPT_CONTINUE`，继续到下一个规则。

# 4.2 扩展 iptables
因为我是懒的，所以iptables相当有扩展性。这是一个把工作推给别人的方法，这也是开源精神（参见自由软件，RMS所说的自由软件就是关于自由的，当我写这段话的时候我正在参加他的一个演讲）。

扩展 iptables 可能涉及到两个部分：通过编写一个新的模块来扩展内核，以及可能通过编写新的共享库来扩展用户空间程序iptables。

## 内核
自己编写内核模块本身相当简单，从示例中可以看出。需要注意的一点是你的代码必须是可重入的：在同一时间可以从用户空间进来一个数据包，
而另一个则在中断中进来。实际上，在SMP架构下，在2.3.4及以上的版本中每个CPU可以同时处理一条中断中的数据包。

### 涉及函数
你需要了解的一些函数包括：

`init_module()`
这是模块的入口点。它返回一个负数错误码，或者如果成功注册自身与 netfilter 通信则返回0。

`cleanup_module()`
这是模块的退出点；应从 netfilter 注销自身。

`ipt_register_match()`
用于注册新匹配类型。您将传递一个 `struct ipt_match` 结构，通常声明为静态（文件范围）变量。

`ipt_register_target()`
用于注册新类型。您将传递一个 `struct ipt_target` 结构，通常声明为静态（文件范围）变量。

`ipt_unregister_target()`
用于注销您的目标。

`ipt_unregister_match()`
用于注销您的匹配。

关于在新挑战或目标的额外空间中做复杂的事情（例如提供计数器）的一个警告。在SMP机器上，每个CPU都会将整个表格使用memcpy复制一遍：
如果你真的想保留中心信息，你应该看看`limit'匹配中使用的那种方法。

### 新匹配函数
通常，新的匹配函数会被编写为独立模块。这些模块可以被进一步扩展，尽管这种情况并不常见。
一种方法是使用netfilter框架的`nf_register_sockopt'函数来让用户直接与你的模块进行交互。
另一种方法是提供其他模块注册自己的符号的方式，就像netfilter和ip_tables所做的那样。

你新匹配函数的核心是 `ipt_match` 结构体，它传递给 `ipt_register_match()` 函数。该结构体有以下字段：

`list`
这个字段设置为任何垃圾数据，例如 `{ NULL, NULL }`。
`name`
这个字段是用户空间中所称的匹配函数名称。名称应与模块名称匹配（即，如果名称为“mac”，那么模块必须是“ipt_mac.o”）以便自动加载工作能正常进行。
`match`
这个字段是一个指向一个匹配函数的指针，该函数接收 `skb`（数据包结构）、输入和输出设备指针（其中一个可能为空，取决于钩子），以及处理的那一规则中的匹配数据指针（在用户空间中准备的结构体）。它还需要提供 IP 偏移量（非零表示非头部碎片）、协议头指针（即 IP 头之后的位置）、数据长度（即包长度减去 IP 头部长度）和一个指向 `hotdrop` 变量的指针。如果返回值为非零，则说明该数据包匹配，且可以将 `hotdrop` 设置为 1 来指示如果函数返回 0 则应立即丢弃此包。
`checkentry`
这个字段是一个指向检查规则规范的函数指针。如果该函数返回 0，则规则不会从用户端被接受。例如，“tcp”匹配类型只接受 TCP 包，因此若 `struct ipt_ip` 部分中的规则没有指定协议必须为 TCP，那么将返回 0。tablename 参数允许你的匹配模块控制它可以使用的表，而 hook_mask 是一个掩码，表示该规则可以调用的钩子集：如果你的匹配函数从某些 netfilter 钩子中看起来不合理，你可以在这里避免这种情况。
`destroy`
这个字段是一个指向当使用该匹配时删除某个条目时要调用的函数指针。这允许你在检查条目时动态分配资源，并在此清理这些资源。
`me`
这个字段被设置为 `THIS_MODULE`，给出模块的指针，导致类型规则的数量在创建和销毁时上升下降。这样可以防止用户卸载该模块（且 cleanup_module() 被调用）而引发规则引用它的情况。

### 新表
如果你想创建一个特定目的的新表，可以调用 `ipt_register_table()` 函数，并提供一个 `struct ipt_table` 结构体。该结构体包含以下字段：
`list`
此字段设置为任意垃圾数据，例如 `{ NULL, NULL }`。
`name`
此字段是用户空间中引用的表功能名称。名称应与模块名称匹配（即，如果名称为 "nat"，那么模块必须为 "iptable_nat.o")，以便自动加载工作。
`table`
这是一个完全填充的 `struct ipt_replace` 结构体，用于用户空间替换表使用。需要将 `counters` 指针设置为 NULL。此数据结构可以声明为 `__initdata`，从而在引导后丢弃。
`valid_hooks`
这是 IPv4 netfilter 插口的掩码值；这些位掩码表示将进入表的 IPv4 netfilter 插口。这用于检查那些入口点是否有效，并计算 ipt_match 和 ipt_target 的 `checkentry()` 函数可能可用的插口。
`lock`
这是一个整张表的读写自旋锁；初始化为 RW_LOCK_UNLOCKED。
`private`
此字段内部由 ip_tables 代码使用。

## 用户空间工具
现在你已经编写了漂亮的内核模块，你可能希望从用户空间控制它的选项。而不是为每个扩展版本分支iptables，我使用的是最新的90年代技术：飞鼠（Furby）。抱歉，我是说共享库。

### 新表
新表通常不需要 iptables 的任何扩展：用户只需使用 `-t` 选项来让它使用新的表。

共享库应有一个 `_init()` 函数，在加载时会自动被调用：相当于内核模块的 `init_module()` 函数。这个函数应该调用 `register_match()` 或 `register_target()`，
具体取决于你的共享库是否提供了新的匹配或目标。

你需要提供一个共享库：它可以用来初始化部分结构，或者提供额外选项。我甚至在没有实际做的事情时现在也要求一个共享库，以减少由于共享库缺失而引发的问题报告。

`iptables.h` 头文件中描述了一些有用的函数，特别是：

`check_inverse()`
检查是否真的为 `!`，如果是，则设置 `invert` 标志如果尚未设置。如果返回 true，你应该像示例那样递增 optind。

`string_to_number()`
将字符串转换为给定范围内的数字，并在无效或超出范围时返回 -1。`string_to_number` 依赖于 `strtol`（参见手册页），这意味着前缀 "0x" 会使数字以十六进制基数表示，前缀 "0" 则会使它以八进制基数表示。

`exit_error()`
如果发现错误应被调用。通常第一个参数是 `PARAMETER_PROBLEM`，意味着用户没有正确使用命令行。

### 新匹配函数
您共享库的 `_init()` 函数将一个指向静态结构 `struct iptables_match` 的指针传递给 `register_match()` 函数，该结构具有以下字段:

`next`
此指针用于创建匹配项（如用于列出规则）的链表。应该设置为初始值 NULL。

`name`
匹配函数的名字。这应与库名称匹配（例如 "tcp" 对于 `libipt_tcp.so`）。

`version`
通常使用 IPTABLES_VERSION 宏来设置：这是用来确保 iptables 程序不会误加载错误共享库的。

`size`
此匹配项的数据大小；您应该使用 IPT_ALIGN() 宏来确保其正确对齐。

`userspacesize`
对于一些匹配项，内核会改变某些字段（例如 `limit` 目标是一个实例）。这意味着简单的 `memcmp()` 不足以比较两个规则（这是删除匹配规则功能所需）。如果是这种情况，请将不更改的字段放在结构开头，并在此处提供这些不变字段的大小。通常情况下，这与 `size` 字段相同。

`help`
一个函数，用于打印选项摘要。

`init`
此可以用于初始化 ipt_entry_match 结构中的额外空间（如果有），并设置任何 nfcache 标志；如果您正在检查不能使用 `linux/include/netfilter_ipv4.h` 中的内容，则应 OR in NFC_UNKNOWN 位。它会在调用 `parse()` 前被调用。

`parse`
当在命令行中看到一个未识别选项时，此函数会被调用；如果选项确实是您的库，请返回非零值。`invert` 为 true 如果已看到 `!`。`flags` 指针专用于匹配库的使用，并通常用来存储已指定选项的掩码位。确保调整 nfcache 字段。您可能需要根据需要重新分配 `ipt_entry_match` 结构的大小，但必须确保该大小通过 IPT_ALIGN 宏传递。

`final_check`
在解析完命令行后被调用；它会将用于您的库的整数 `flags` 作为参数传递。这给您一个机会检查任何必需选项是否已指定，例如：如果这些是必需的，请调用 `exit_error()`。

`print`
此功能由链列表代码使用（如有必要），打印规则中的额外匹配信息（如果存在）。将数字标志设置为用户已指定 `-n` 标志。

`save`
这是与 parse 相反的功能；它是 `iptables-save` 用于重现创建规则时使用的选项的工具。

`extra_opts`
这是一个指向您库提供的额外选项的 NULL 结尾列表。此列表会被合并到当前选项，并传递给 getopt_long，参见手册页以获取详细信息。getopt_long 的返回代码将成为您的 parse() 函数的第一个参数 (`c`)。

在此结构的末尾还有用于 iptables 内部使用的其他元素：您不需要设置它们。


### 新目标
共享库中的 `_init()` 函数将 `register_target()` 传递一个指向静态 `struct iptables_target` 的指针，
该结构体具有与上述详细描述的 `iptables_match` 结构体相似的字段。


### 使用 `libiptc`
`libiptc` 是一个 iptables 控制库，专为在内核模块 `iptables` 中列出和操作规则而设计。尽管当前主要用于 `iptables` 程序，但它使得编写其他工具变得相对简单。
你需要以 root 权限来使用这些功能。

内核表本身只是一个包含规则的表格，以及一组表示入口点的数字。库通过提供链名（如 "INPUT"）这一抽象来处理链名。
用户定义的链则通过在用户定义链头之前插入一个错误节点来进行标记，该节点在目标的额外数据部分中包含了链名（内置链的位置由三个表条目点定义）。

支持的标准目标包括：ACCEPT、DROP、QUEUE（分别对应 `NF_ACCEPT`、`NF_DROP` 和 `NF_QUEUE`），RETURN（被翻译为特殊值 `IPT_RETURN`，由 `ip_tables` 处理），
以及 JUMP（从链名转换为目标在表中的实际偏移量）。

当调用 `iptc_init()` 时，包括计数器在内的表将被读取。这些表项通过以下函数进行操作：`iptc_insert_entry()`, `iptc_replace_entry()`, `iptc_append_entry()`, 
`iptc_delete_entry()`, `iptc_delete_num_entry()`, `iptc_flush_entries()`, `iptc_zero_entries()`，`iptc_create_chain()`, `iptc_delete_chain()`, 和 
`iptc_set_policy()`。

表项的更改不会写回内核系统，直到调用 `iptc_commit()` 函数为止。这意味着两个库用户在同一链上竞争是可能的；需要使用锁定机制来防止这种情况的发生，
但目前尚未实现。

计数器没有与之相关的竞态条件，因为通过将计数器添加回内核的方式，在读取和写入表时的计数器增量仍然会在新的表中显示出来。

还有一些辅助函数：

`iptc_first_chain()`
此函数返回表中的第一个链名：空链则返回 `NULL`。

`iptc_next_chain()`
此函数返回下一个链名：`NULL` 表示没有更多链了。

`iptc_builtin()`
返回布尔值，如果给定的链名为内置链名，则返回 `true`。

`iptc_first_rule()`
返回指定链名中的第一个规则指针：空链则返回 `NULL`。

`iptc_next_rule()`
返回链中的下一个规则指针：表示链尾则返回 `NULL`。

`iptc_get_target()`
获取给定规则的目标。如果该目标为扩展目标，将返回其目标名称。如果该目标是跳转到另一个链，则返回该链名。如果它是判决（如 DROP），则返回该名。如果没有目标（如计数器式规则），则返回空字符串。

注意应使用此函数来代替直接使用 `ipt_entry` 结构体中的 `verdict` 字段，因为它提供了对标准判决进一步的解释功能。

`iptc_get_policy()`
获取内置链的目标策略，并将统计信息填充到 `counters` 参数中。

`iptc_strerror()`
此函数返回 `libiptc` 库失败代码的有意义解释。如果一个函数失败，它总是会设置 `errno`：可以使用这个值来调用 `iptc_strerror()` 获取错误消息。


# 4.3 理解NAT

欢迎来到内核中的网络地址转换。请注意，提供的基础设施旨在实现完整性而非纯粹的效率，并且未来可能对性能进行大幅优化。目前，我很高兴它能正常工作。

NAT被分为连接跟踪（不会修改任何数据包）和实际的NAT代码部分。连接跟踪的设计还允许其被iptables模块使用，因此它可以区分某些状态信息，
而这些信息是NAT并不关心的。

连接跟踪
连接跟踪挂钩到高优先级的`NF_IP_LOCAL_OUT`和`NF_IP_PRE_ROUTING`钩子中，以便在数据包进入系统之前就能看到它们。

`skb`结构体中的`nfct`字段是一个指向`ip_conntrack`内部`infos[]`数组之一位置的指针。因此，我们可以通过它所指向的`array`元素来确定该`skb`的状态：
这个指针同时包含了状态结构和此`skb`与该状态的关系。

这种区分状态的方式使得连接跟踪能够有效地管理网络流量中的连接行为。例如，它能识别出已建立（ESTABLISHED）或半打开（SYN_RECV）的连接，并将它们分别处理。
这有助于提高网络效率并优化资源使用。

提取 `nfct` 字段的最佳方法是调用 `ip_conntrack_get()`，它如果未设置则返回 NULL，否则返回连接指针，并填充 `ctinfo`，描述数据包与该连接之间的关系。
此枚举类型有几种值：

`IP_CT_ESTABLISHED`
数据包属于已建立的连接，在原始方向。

`IP_CT_RELATED`
数据包与连接相关联，并且在原始方向通过。

`IP_CT_NEW`
数据包试图创建一个新的连接（显而易见，它处于原始方向）。

`IP_CT_ESTABLISHED + IP_CT_IS_REPLY`
数据包属于已建立的连接，在回复方向。

`IP_CT_RELATED + IP_CT_IS_REPLY`
数据包与连接相关联，并且在回复方向通过。

因此可以通过检查 `>= IP_CT_IS_REPLY` 来识别回复数据包。

# 4.4 扩展连接跟踪/NAT
这些框架设计成能够容纳任何数量的协议和不同的映射类型。其中一些映射类型可能非常具体，比如负载均衡/故障切换类型的映射。

内部，连接跟踪将一个数据包转换为“元组”，在搜索与之匹配的绑定或规则之前表示出有趣的部分。这个元组包含可操作的部分和不可操作的部分；称为“源”和“目的地”，因为这是来源NAT世界中第一个数据包的视图（而在目的地NAT世界则是回复数据包）。同一方向内相同流中的每个数据包的元组都是相同的。

例如，TCP数据包的元组包含可操作部分：源IP地址和源端口号，不可操作部分：目标IP地址和目的端口号。然而，可操作部分和不可操作部分并不一定必须是同一种类型；比如ICMP数据包的元组包含了可操作部分：源IP地址和ICMP ID号，不可操作部分：目标IP地址和ICMP类型及代码。

每一个元组都有一个逆元组，即该流中回复数据包的数据。例如，ICMP ping数据包（从192.168.1.1到1.2.3.4，ID 12345）的逆元组是一个回复ping数据包（从1.2.3.4到192.168.1.1，ID 12345）。

这些元组通过`struct ip_conntrack_tuple`广泛使用。事实上，与包来自的那个hook（对期待的类型化操作有影响）和涉及的设备一起，它们提供了关于包的全部信息。

大多数元组包含在`struct ip_conntrack_tuple_hash`中，该结构增加了双链表条目，并指向属于哪个连接的元组指针。

一个连接由`struct ip_conntrack`表示；它有两个`struct ip_conntrack_tuple_hash`字段：一个是原始数据包方向的引用（tuplehash[IP_CT_DIR_ORIGINAL]），另一个是回复方向的数据包引用（tuplehash[IP_CT_DIR_REPLY]）。

无论如何，NAT代码首先会查看连接跟踪代码是否已经提取了元组并找到了现有的连接。这通过检查skbuff中的nfct字段来实现，该字段告诉我们这是新连接的尝试，还是未定义的方向；在后者情况下，则执行先前确定的操作。

如果这是一个新的连接开始，我们将使用标准iptables遍历机制在`nat`表中查找该元组的规则。如果找到匹配的规则，将用于初始化对该方向和回复的数据包操作，并告知连接跟踪代码，期望的回复类型已经改变。然后按照上述方式进行处理。

如果没有找到相应的规则，会创建一个“空”绑定：通常不会映射数据包，而是存在以确保我们不会在现有的流上重映射另一条流。有时，由于已经在该流上进行了映射，因此不能创建空绑定；此时，根据协议的特定操作可能会尝试重新映射它，尽管它是名义上的空绑定。

## 标准NAT目标
NAT目标与其他iptables扩展目标类似，不同之处在于它们只在“nat”表中使用。SNAT和DNAT目标都会用到一个额外的数据`struct ip_nat_multi_range`；这是用来指定映射允许绑定的地址范围。一个范围元素`struct ip_nat_range`包含了一个最小和最大IP地址（都是包括的），以及一个特定协议（例如TCP端口）的最大和最小值（也是包括在内的）。还有一个字段用于标志，以说明是否可以将IP地址进行映射（有时我们只想映射元组中特定协议的部分，而不需要整个IP地址），还有另一个用来说明指定协议范围部分的有效性。

一个多范围是一个这样的元素数组`struct ip_nat_range`；这意味着一个范围可能是“1.1.1.1-1.1.1.2 ports 50-55 AND 1.1.1.3 port 80”。每个范围元素都会扩展这个范围（对于那些喜欢集合论的人来说，这是一个联合）。


# 4.5 理解 Netfilter

Netfilter 很简单，上一部分已经详细描述了它。不过有时需要超越 NAT 或 ip_tables 架构所提供的功能，或者完全替代它们。

对于 netfilter（尤其是未来）来说，一个重要的问题是缓存机制。每个 skb 都有一个 `nfcache` 字段：一个掩码，表示头部分别检查过哪些字段，以及包是否被修改了。
其思想是，每个 netfilter 的钩子将与自己相关的位或进来，以便我们能够设计一套聪明的缓存系统，在某些情况下不需要通过 netfilter 将包转发。

最重要的是 NFC_ALTERED 位，表示包已经被修改（这已经在 IPv4 的 NF_IP_LOCAL_OUT 钩子中使用了来重路由被修改的包），以及 NFC_UNKNOWN 位，
表示因为检查了一些不能表达的属性，所以不应进行缓存。如果不清楚应该怎么做，可以将 skb 的 nfcache 字段中的 NFC_UNKNOWN 标志设置为 true。

# 4.6 编写新 Netfilter 模块
插入手钩到 Netfilter 中
要接收/修改内核中的数据包，你可以编写一个模块，并注册一个“Netfilter 插口”。这基本上是对某一点的一种兴趣表达；具体的点是基于协议的，
由特定协议的 Netfilter 头文件定义，例如 `netfilter_ipv4.h`。

要注册和注销 Netfilter 插口，你使用 `nf_register_hook` 和 `nf_unregister_hook` 函数。这些函数各接受一个指向 `struct nf_hook_ops` 的指针，并将其填充如下：

- **list**: 用于将你插入到链表中：设置为 `{ NULL, NULL }`
- **hook**: 当数据包到达该插口点时要调用的函数。你的函数必须返回 NF_ACCEPT、NF_DROP 或 NF_QUEUE。如果返回 NF_ACCEPT，则下一个与该点关联的插口函数会被调用。如果返回 NF_DROP，那么该数据包会被丢弃。如果返回 NF_QUEUE，则会将数据包排队。你接收一个skb指针的指针，因此你可以完全替换这个skb，如果你愿意的话。
- **flush**: 目前未使用：设计为在缓存被清除时传递数据包命中。可能永远不会实现：将其设置为 NULL。
- **pf**: 协议族，例如 `PF_INET` 用于 IPv4。
- **hooknum**: 你感兴趣的插口编号，例如 `NF_IP_LOCAL_OUT`。


## 处理排队的包
当前，这个接口由 ip_queue 使用；你可以注册来处理给定协议下的排队包。这与注册挂钩有类似的行为，不过你可以阻塞对包的处理，
并且只有在某个挂钩回复为 `NF_QUEUE` 的情况下你才能看到这样的包。

用来注册感兴趣排队包的两个函数是 `nf_register_queue_handler()` 和 `nf_unregister_queue_handler()`。
你注册的函数将以 `void *` 指针为参数传递给 `nf_register_queue_handler()`，这个指针是你在处理队列包时使用的。

如果没有其他注册处理某个协议的程序，则返回 `NF_QUEUE` 等效于返回 `NF_DROP`。

一旦你对排队包表达了兴趣，它们就开始排队。你可以对他们做任何你想做的事，但必须调用 `nf_reinject()` 来处理完他们（不要简单地调用 kfree_skb() 删除他们）。
当你重新注入一个 skb 时，你需要传给它skb、队列处理器所得到的 `struct nf_info` 结构以及判断：NF_DROP 导致他们被丢弃，NF_ACCEPT 导致他们继续遍历挂钩，
NF_QUEUE 导致他们再次排队，而 NF_REPEAT 导致由包所用挂钩进行咨询（小心避免无限循环）。

你可以查看 `struct nf_info` 来获取有关包的辅助信息，例如它位于哪个接口和哪个挂钩上。

## 从用户空间接收命令
通常，netfilter 组件会想要与用户空间进行交互。实现这一功能的方法是使用 setsockopt 机制。请注意，
每个协议都需要被修改以调用 nf_setsockopt() 来处理它不理解的 setsockopt 数字（以及 nf_getsockopt() 来处理 getsockopt 数字），
目前为止只有 IPv4、IPv6 和 DECnet 被进行了这些修改。

采用一种现在熟悉的技巧，我们通过使用 nf_register_sockopt() 命令来注册一个 `struct nf_sockopt_ops`。这个结构体的字段如下：

`list`
用于将其缝入链表中：设置为 `{ NULL, NULL }`。

`pf`
处理的协议族，例如 PF_INET。

`set_optmin`
和
`set_optmax`
这些指定了被处理的 setsockopt 数字的（不包括）范围。因此使用 0 和 0 意味着你没有定义任何 setsockopt 数字。

`set`
这是当用户调用你的 setsockopts 的时候，会被调用的函数。在该函数中你应该检查是否具有 NET_ADMIN 权限。

`get_optmin`
和
`get_optmax`
这些指定了被处理的 getsockopt 数字的（不包括）范围。因此使用 0 和 0 意味着你没有定义任何 getsockopt 数字。

`get`
这是当用户调用你的 getsockopts 的时候，会被调用的函数。在该函数中你应该检查是否具有 NET_ADMIN 权限。

最后两个字段用于内部使用。

# 4.7 用户空间的数据包处理

使用 libipq 库和 `ip_queue` 模块，现在几乎可以在用户空间中完成所有在内核中可以完成的操作。这意味着你甚至可以把你的代码完全开发在用户空间。
除非你尝试过滤大带宽的数据包，否则这种方法相对于内核中的数据包重排处理来说会更好。

Netfilter 的早期版本中，我通过将 iptables 的雏形移植到用户空间来证明这一点。Netfilter 开放了让更多人编写自己的高效网络重排模块的大门，
这些模块可以使用他们想用的语言编写。

# 附录
## ipt_entry
```c
/* 这个结构定义了每个防火墙规则。由三部分组成：1）通用的IP头信息，
   2）匹配特定的信息，3）如果规则匹配则执行的目标操作 */
struct ipt_entry {
	struct ipt_ip ip;

    /* 标记我们需要关注的字段。*/
	unsigned int nfcache;

    /* ipt_entry 和 匹配 的大小之和 */
	__u16 target_offset;
    /* Ipt_entry、matches和target的大小之和 */
	__u16 next_offset;

    /* 返回指针 */
	unsigned int comefrom;

    /* 包和字节计数器。 */
	struct xt_counters counters;

    /* 如果有匹配项，则为目标。 */
	unsigned char elems[0];
};
```

##  ipt_entry_match xt_entry_match
```c
#define ipt_entry_match xt_entry_match

struct xt_entry_match {
	union {
		struct {
			__u16 match_size;

			/* Used by userspace */
			char name[XT_EXTENSION_MAXNAMELEN];
			__u8 revision;
		} user;
		struct {
			__u16 match_size;

			/* Used inside the kernel */
			struct xt_match *match;
		} kernel;

		/* Total length */
		__u16 match_size;
	} u;

	unsigned char data[0];
};
```
## ipt_entry_target xt_entry_target
```c
#define ipt_entry_target xt_entry_target

struct xt_entry_target {
	union {
		struct {
			__u16 target_size;

			/* Used by userspace */
			char name[XT_EXTENSION_MAXNAMELEN];
			__u8 revision;
		} user;
		struct {
			__u16 target_size;

			/* Used inside the kernel */
			struct xt_target *target;
		} kernel;

		/* Total length */
		__u16 target_size;
	} u;

	unsigned char data[0];
};
```

## ipt_ip
```c
/* Yes, Virginia, you have to zero the padding. */
struct ipt_ip {
	/* Source and destination IP addr */
	struct in_addr src, dst;
	/* Mask for src and dest IP addr */
	struct in_addr smsk, dmsk;
	char iniface[IFNAMSIZ], outiface[IFNAMSIZ];
	unsigned char iniface_mask[IFNAMSIZ], outiface_mask[IFNAMSIZ];

	/* Protocol, 0 = ANY */
	__u16 proto;

	/* Flags word */
	__u8 flags;
	/* Inverse flags */
	__u8 invflags;
};
```
