# 初步介绍 netfilter
## netfilter 的位置

协议栈通过调用NF_HOOK，将skb交给netfilter处理，netfilter处理完成后，可能将skb返还给协议栈，也可能消耗了skb。

用户通过iptables命令，和ip_tables模块交互，ip_tables模块调整netfilter模块的hook链，从而实现对skb处理逻辑的调整。

             ┌───────┐         ┌────────────┐ setsockopt
             │socket │         │iptables命令│ getsockopt
             └───┬───┘         └────────────┘
                 │ ▲              │ ▲
       ──────────┼─┼──────────────┼─┼───────────────►
                 ▼ │              ▼ │
            ┌──────────────────────────────────┐
            │           内核接口               │
            └───┬─────────────────┬────────────┘
                ▼ ▲               ▼ ▲
            ┌─────┴───┐        ┌────┴────────────┐
            │ 传输层  │        │ip_tables内核模块│
            └───┬─────┘        └──┬──────────────┘
                ▼ ▲               ▼ ▲
            ┌─────┴──┐  hook   ┌────┴─────────┐
            │ 网络层 │◄────────┤netfilter模块 │
            └───┬────┘───────► └──────────────┘
                ▼ ▲
            ┌─────┴─────┐
            │ 网络接口层│
            └───┬───────┘
                ▼ ▲
            ┌─────┴────┐
            │ 驱动层   │
            └──────────┘

## 网络层和netfilter

### hook点

网络层在5个位置调用了NF_HOOK，以将skb的所有权交给netfilter

                 ┌──────────────────────────────────────────────┐
                 │                   传输层                     │
                 └───────────────────────────────┬──────────────┘
                         ▲                       │
                         │                       ▼
                         │                    ┌──────┐
                         │                    │route │
                         │                    └──┬───┘
                         │                       ▼
                      ┌──┴──┐                 ┌───────┐
                      │INPUT│                 │OUTPUT │
                      └─────┘                 └──┬────┘
                         ▲                       ▼
      ┌───────────┐   ┌──┴──┐    ┌───────┐    ┌──────────┐    ┌───────────┐
      │ PREROUTING├─► │route├───►│FORWARD├───►│dst_output├──► │POSTROUTING│
      └───────────┘   └─────┘    └───────┘    └──────────┘    └─────┬─────┘
            ▲                                                       │
            │                                                       │
            │                                                       ▼
      ┌─────┴──────────────────────────────────────────────────────────────┐
      │                               接口层                               │
      └────────────────────────────────────────────────────────────────────┘

### 返回值
进入netfilter的skb可能被netfilter消耗掉，也可能返回协议栈，netfilter通过返回值告知协议栈skb的现状，具体返回值包括以下5种:
- NF_ACCEPT 继续正常传skb。这个返回值告诉 Netfilter：到目前为止，该数据包还是被接受的并且该数据包应当被递交到网络协议栈的下一个阶段。
- NF_DROP 丢弃该skb，不再传输。
- NF_STOLEN 模块接管该skb，告诉Netfilter“忘掉”该skb。该回调函数将从此开始对skb的处理，并且Netfilter应当放弃对该skb做任何的处理。但是，这并不意味着该数据包的资源已经被释放。这个数据包以及它独自的sk_buff数据结构仍然有效，只是回调函数从Netfilter 获取了该数据包的所有权。
- NF_QUEUE 对该数据报进行排队(通常用于将数据报给用户空间的进程进行处理)
- NF_REPEAT 再次调用该回调函数，应当谨慎使用这个值，以免造成死循环。

## 详解hook

### NF_HOOK : 协议栈到 netfilter
IP层通过调用NF_HOOK，来标记hook点，此处的skb交给netfilter处理
```c
// @ pf：协议族名，Netfilter架构同样可以用于IP层之外，因此这个变量还可以有诸如PF_INET6，PF_DECnet等名字。
// @ hook：HOOK点的名字，对于IP层，就是取上面的五个值；
// @ skb：不解释；
// @ indev：数据包进来的设备，以struct net_device结构表示；
// @ outdev：数据包出去的设备，以struct net_device结构表示；
// @ okfn:是个函数指针，当所有的该HOOK点的所有登记函数调用完后，转而走此流程
//
// 返回 1 表示数据包还回给网络层，其他值表示数据包已经被netfilter消费掉
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk, struct sk_buff *skb,
	struct net_device *in, struct net_device *out,
	int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	int ret = nf_hook(pf, hook, net, sk, skb, in, out, okfn);
	if (ret == 1)
		ret = okfn(net, sk, skb);
	return ret;
}
```

### nf_hook : netfilter处理数据包
netfilter获得数据包后，根据数据包的协议族找到对应的hooks链，根据hook点，找到具体的hook链，使用具体的hook链对skb进行处理
```c
/**
 * nf_hook - 调用 netfilter 捕获点
 * 返回 1 表示钩子已允许数据包通过。在这种情况下，调用者必须调用 okfn 函数。返回其他值表示钩子已经消耗了该数据包。
 */
static inline int nf_hook(u_int8_t pf, unsigned int hook, struct net *net,
			  struct sock *sk, struct sk_buff *skb,
			  struct net_device *indev, struct net_device *outdev,
			  int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	struct nf_hook_entries *hook_head = NULL;
	int ret = 1;

	rcu_read_lock();
    // 根据协议族找到hooks，再根据hook点找到对应hook链
	switch (pf) {
	case NFPROTO_IPV4:
		hook_head = rcu_dereference(net->nf.hooks_ipv4[hook]);
		break;
    // ...
	}

	if (hook_head) {
		struct nf_hook_state state;
        // 将参数封装到state
		nf_hook_state_init(&state, hook, pf, indev, outdev,
				   sk, net, okfn);
        // 遍历hook链规则，处理数据包
		ret = nf_hook_slow(skb, &state, hook_head, 0);
	}
	rcu_read_unlock();

	return ret;
}
```

### nf_hook_slow
使用某条hook链进行遍历处理skb
```c
/* 如果 okfn() 需要由调用者执行，
 * 返回 1 对于 NF_DROP，否则返回 0。 调用方必须持有 rcu_read_lock 锁。 */
int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state,
		 const struct nf_hook_entries *e, unsigned int s)
{
	unsigned int verdict;
	int ret;

	for (; s < e->num_hook_entries; s++) {
        // 由s索引到当前规则，对skb进行检查
		verdict = nf_hook_entry_hookfn(&e->hooks[s], skb, state);
            return entry->hook(entry->priv, skb, state);
        // 根据规则的结果执行内置目标
		switch (verdict & NF_VERDICT_MASK) {
		case NF_ACCEPT:
			break;
		case NF_DROP:
			kfree_skb(skb);
			ret = NF_DROP_GETERR(verdict);
			if (ret == 0)
				ret = -EPERM;
			return ret;
		case NF_QUEUE:
			ret = nf_queue(skb, state, e, s, verdict);
			if (ret == 1)
				continue;
			return ret;
		default:
            /* 对于 NF_STOLEN 的隐式处理，以及任何其他非常规判定结果。*/
			return 0;
		}
	}

	return 1;
}
```

### 二维数据hook链表

具体hook函数是通过二维索引找到，hook表示hook点，s表示顺序即优先级
`net->nf.hooks_ipv4[hook][s]`

相关数据结构:
```c
struct net {
	struct netns_nf		nf;
	struct netns_xt		xt;
    // ...
};

enum nf_inet_hooks {
	NF_INET_PRE_ROUTING,
	NF_INET_LOCAL_IN,
	NF_INET_FORWARD,
	NF_INET_LOCAL_OUT,
	NF_INET_POST_ROUTING,
	NF_INET_NUMHOOKS
};

struct netns_nf {
	struct nf_hook_entries __rcu *hooks_ipv4[NF_INET_NUMHOOKS];
	struct nf_hook_entries __rcu *hooks_ipv6[NF_INET_NUMHOOKS];
    // ...
};

struct nf_hook_entries {
	u16				num_hook_entries;
	/* padding */
	struct nf_hook_entry		hooks[];

/* 背景资料：指向每个钩子原始 orig_ops 的指针，
 * 随后是用于通过 call_rcu 函数释放结构的 RCU 头和空闲空间。
 *
 * 这不是 struct nf_hook_entry 结构的一部分，因为它在慢路径（钩子注册/注销）中唯一需要：
 * const struct nf_hook_ops     *orig_ops[]
 *
 * 同样出于这个原因，我们将这些信息存储在末尾——只有在删除钩子时才需要，
 * 而在处理数据包路径过程中不需要：struct nf_hook_entries_rcu_head     head
 */
};

struct nf_hook_ops {
	/* User fills in from here down. */
	nf_hookfn		*hook;
	struct net_device	*dev;
	void			*priv;
	u_int8_t		pf;
	unsigned int		hooknum;
	/* Hooks are ordered in ascending priority. */
	int			priority;
};

struct nf_hook_entry {
	nf_hookfn			*hook;
	void				*priv;
};
```
# 从netfilter 到 iptables
- netfilter只有链的概念
  - netfilter提供的`struct netns_nf`，实现了不同协议族的`hook[链][顺序]`
- iptables在其上增加了table的概念。
  - xt_table: 为实现不同协议族的不同功能的表提供支持, 表的核心内容是多个hook点的规则链集合

## 什么是表
表是相同功能hook的统一入口

表首先转换自己为 nf_hook_ops ，再加入nf的对应链

如filter表将自己加入nf 的 INPUT, OUTPUT, FORWARD三个位置，处理方法统一为 `iptable_filter_hook`

```c
#define FILTER_VALID_HOOKS ((1 << NF_INET_LOCAL_IN) | \
			    (1 << NF_INET_FORWARD) | \
			    (1 << NF_INET_LOCAL_OUT))

static const struct xt_table packet_filter = {
	.name		= "filter",
	.valid_hooks	= FILTER_VALID_HOOKS,
	.me		= THIS_MODULE,
	.af		= NFPROTO_IPV4,
	.priority	= NF_IP_PRI_FILTER,
	.table_init	= iptable_filter_table_init,
};

static int __init iptable_filter_init(void)
	filter_ops = xt_hook_ops_alloc(&packet_filter, iptable_filter_hook);

static unsigned int
iptable_filter_hook(void *priv, struct sk_buff *skb,
		    const struct nf_hook_state *state)
	return ipt_do_table(skb, state, state->net->ipv4.iptable_filter);
```

## xt_table
### xt_table 初始化
对init_net的 netns_xt的各个协议族链表初始化
```c
static int __init xt_init(void)
    xt_net_init(net)
        for (i = 0; i < NFPROTO_NUMPROTO; i++)
            INIT_LIST_HEAD(&net->xt.tables[i]);
```

xt_table除了 `net->xt.tables[]`还有iptables的各个类型的表

```c
struct net
	struct netns_ipv4	ipv4;
        struct xt_table		*iptable_filter;
        struct xt_table		*iptable_mangle;
        struct xt_table		*iptable_raw;
        struct xt_table		*arptable_filter;
        struct xt_table		*nat_table;
```

在ip_table中xt_table的private会被设置为 xt_table_info，
xt_table_info记录了表的所有hook点的规则链
```c
/* 表格本身 */
struct xt_table_info {
	/* 每个表的大小 */
	unsigned int size;
	/* 入口数量： FIXME. --RR */
	unsigned int number;
	/* 初始入口数量。用于模块使用计数 */
	unsigned int initial_entries;

    // 每个hook点在entry的偏移
	unsigned int hook_entry[NF_INET_NUMHOOKS];

	unsigned int underflow[NF_INET_NUMHOOKS];

	/*
	 * 用户链的数目。由于表中不能有循环，最多可能有 @stacksize 次跳转（用户链数）。
	 */
	unsigned int stacksize;
	void ***jumpstack;

	/* 大小对齐的字节数组，用于存储规则 */
	unsigned char entries[0] __aligned(8);
};

```

### xt_hook_ops_alloc : 从xt_table 到 nf_hook_ops

xt_table模块提供了 xt_hook_ops_alloc 方法，用于将表注册到netfilter，

具体来说是将 xt_table 转换成功 netns_nf 可用的 nf_hook_ops数组，nf_hook_ops数组可以用于加入netns_nf的对应hook链

通常一个表需要关联多个hook点，所以需要创建对应数量的nf_hook_ops，而hook的回调方法是一样的。

```c
/**
 * xt_hook_ops_alloc - 设置新表的钩子
 * @table: 需要设置钩子所需的原数据表
 * @fn: 钩子函数
 *
 * 此函数将创建 x_table 所需的 nf_hook_ops，以便将其传递给 xt_hook_link_net()。
 */
struct nf_hook_ops *
xt_hook_ops_alloc(const struct xt_table *table, nf_hookfn *fn)
	unsigned int hook_mask = table->valid_hooks;
	struct nf_hook_ops *ops;
	uint8_t hooknum;

    // 根据表插入链的数量，创建对应的多个nf_hook_ops
    // hweight32 计算32bit中置一位的个数
	uint8_t i, num_hooks = hweight32(hook_mask);

	ops = kcalloc(num_hooks, sizeof(*ops), GFP_KERNEL);

	for (i = 0, hooknum = 0; i < num_hooks && hook_mask != 0;
	     hook_mask >>= 1, ++hooknum) {
		if (!(hook_mask & 1))
			continue;
		ops[i].hook     = fn;
		ops[i].pf       = table->af;
		ops[i].hooknum  = hooknum;
		ops[i].priority = table->priority;
		++i;
	}

	return ops;
}
```
### packet_filter —— filter表
iptables 的 filter 模块通过定义 packet_filter 来定义自己的表。
之后需要将 packet_filter加入 xt_table 模块。
```c
// filter表需要注册到3个hook点
#define FILTER_VALID_HOOKS ((1 << NF_INET_LOCAL_IN) | \
			    (1 << NF_INET_FORWARD) | \
			    (1 << NF_INET_LOCAL_OUT))

static const struct xt_table packet_filter = {
	.name		= "filter",
	.valid_hooks	= FILTER_VALID_HOOKS,
	.me		= THIS_MODULE,
	.af		= NFPROTO_IPV4,
	.priority	= NF_IP_PRI_FILTER,
	.table_init	= iptable_filter_table_init, // 实现对nf和xt_table的注册
};
```
### filter_ops —— hook回调函数
iptables 的 filter 模块定义 filter_ops 用于插入 netfilter 的hook点
filter_ops 的初始化通过 xt_hook_ops_alloc 完成

```c
static struct nf_hook_ops *filter_ops __read_mostly; // nf_hook_ops数组
```

filter模组中利用 packet_filter 构造 filter_ops ，将 iptable_filter_hook设置为回调函数
```c

// 根据xt_table表packet_filter构造用于注册nf的ops
static int __init iptable_filter_init(void)
	filter_ops = xt_hook_ops_alloc(&packet_filter, iptable_filter_hook);

// ip_tables的nf回调函数
static unsigned int
iptable_filter_hook(void *priv, struct sk_buff *skb,
		    const struct nf_hook_state *state)
    // skb 数据包
    // state 数据包相关信息
    // iptable_filter filter表，其中有所有三种hook点的规则链
    // ipt_do_table会根据实际hook 点从iptable_filter表中找到对应规则链，
    // 完成对skb的处理
	return ipt_do_table(skb, state, state->net->ipv4.iptable_filter );
```

## netfilter 提供的核心数据结构
```c
typedef unsigned int nf_hookfn(void *priv,
			       struct sk_buff *skb,
			       const struct nf_hook_state *state);

struct nf_hook_ops {
	/* User fills in from here down. */
	nf_hookfn		*hook;   // 回调函数
	struct net_device	*dev; // 输出/输出设备
	void			*priv;  // 回调函数的参数
	u_int8_t		pf;   // 协议族
	unsigned int		hooknum; // hook编号，说明hook位置
    /* 挂钩按升序优先级排列。*/
	int			priority; // 优先级
};

// nf某个hook点的hook数组
struct nf_hook_entries {
	u16				num_hook_entries; // hook数量
	/* padding */
	struct nf_hook_entry		hooks[];
};

struct nf_hook_entry {
	nf_hookfn			*hook; // 回调函数
	void				*priv; // 回调函数的参数
};
```

## ipt_filter的初始化
```c
// 在xt_tables注册filter表
// 在nf注册filter的3个hook点
static int __init iptable_filter_init(void)
    // 创建 nf_hook_ops 数组
	filter_ops = xt_hook_ops_alloc(&packet_filter, iptable_filter_hook);
    // 向xt_table和netfilter进行注册
    iptable_filter_table_init(net);
        struct ipt_replace *repl;
        // 创建filter表的替换表
        repl = ipt_alloc_initial_table(&packet_filter);
        ((struct ipt_standard *)repl->entries)[1].target.verdict =
            forward ? -NF_ACCEPT - 1 : -NF_DROP - 1;
        err = ipt_register_table(net, &packet_filter, repl, filter_ops,
                     &net->ipv4.iptable_filter);

            struct xt_table_info *newinfo;
            struct xt_table_info bootstrap = {0}; // 空的规则表
            struct xt_table *new_table;
            void *loc_cpu_entry;
            // 创建记录表规则的实际对象 newinfo
            newinfo = xt_alloc_table_info(repl->size);
                info = kvmalloc(sz, GFP_KERNEL_ACCOUNT);
                memset(info, 0, sizeof(*info));
                info->size = size;
                return info;
            // 将规则拷贝到newinfo
            loc_cpu_entry = newinfo->entries;
            memcpy(loc_cpu_entry, repl->entries, repl->size);
            // 根据用户空间传入的 repl设置内核空间使用的newinfo
            // 并且确保设置的合法行
            ret = translate_table(net, newinfo, loc_cpu_entry, repl);
            // 进行filter表的注册
            new_table = xt_register_table(net, table, &bootstrap, newinfo);
                struct xt_table_info *private;
                struct xt_table *t, *table;
                // 拷贝filter表
                table = kmemdup(input_table, sizeof(struct xt_table), GFP_KERNEL);
                // 当前表的规则是空
                table->private = bootstrap;
                // 将newinfo设置为表的规则
                xt_replace_table(table, 0, newinfo, &ret);
                    table->private = newinfo;
                // 将表加入网络命令空间的xt
                list_add(&table->list, &net->xt.tables[table->af]);
                // 返回完成注册的表
                return table;
            // 设置网络命名空间的iptables表，如果是filter则为
            // net->ipv4.iptable_filter = new_table;
            WRITE_ONCE(*res, new_table);
            // 进行nf的注册
            // 表通常有多个hook点，所以注册多个hook点
            ret = nf_register_net_hooks(net, ops, hweight32(table->valid_hooks));
                for (i = 0; i < n; i++)
                    err = nf_register_net_hook(net, &reg[i]);
```

### 向xt_table注册
```c
// input_table[input] -> static const struct xt_table packet_filter
// boostrap[output] -> 
// newinfo[input] -> 根据 static const struct xt_table packet_filter 转换的 xt_table_info
// 复制 input_table 到new_table, 将newinfo加入 net->xt.tables[table->af] 链表头部
struct xt_table *xt_register_table(struct net *net,
                   const struct xt_table *input_table,
                   struct xt_table_info *bootstrap,
                   struct xt_table_info *newinfo)
	struct xt_table_info *private;
	struct xt_table *t, *table;
    // 把filter表复制一份
	table = kmemdup(input_table, sizeof(struct xt_table), GFP_KERNEL);
    // 如果filter已经注册，则退出
	list_for_each_entry(t, &net->xt.tables[table->af], list) {
		if (strcmp(t->name, table->name) == 0) {
			ret = -EEXIST;
			goto unlock;
		}
	}
    // newinfo中会具体的规则，将其绑定到table
	xt_replace_table(table, 0, newinfo, &ret);
        ret = xt_jumpstack_alloc(newinfo);
        table->private = newinfo;

	private = table->private;
	private->initial_entries = private->number;
    // filter表加入xt
	list_add(&table->list, &net->xt.tables[table->af]);
```

### 向netfilter注册
```c
// reg 需要注册的hook操作
// n : 注册数量，比如filter有三个hook点，则需要三次
int nf_register_net_hooks(struct net *net, const struct nf_hook_ops *reg,
			  unsigned int n)
	for (i = 0; i < n; i++) 
        // 对具体hook点进行注册
		nf_register_net_hook(net, &reg[i]);

int nf_register_net_hook(struct net *net, const struct nf_hook_ops *reg)
		__nf_register_net_hook(net, NFPROTO_IPV4, reg);

static int __nf_register_net_hook(struct net *net, int pf,
				  const struct nf_hook_ops *reg)
    // 根据协议族pf找到为 nf.hooks_ipv4
    // 根据hook位置找到对应动态数组 nf_hook_entries
	pp = nf_hook_entry_head(net, pf, reg->hooknum, reg->dev);
        return net->nf.hooks_ipv4 + hooknum;
	p = nf_entry_dereference(*pp);
    // 将hook操作加入数组
	new_hooks = nf_hook_entries_grow(p, reg);
	if (!IS_ERR(new_hooks))
		rcu_assign_pointer(*pp, new_hooks);
```

```c
// 完成hook方法的注册
// nf_hook_entries是动态数据，并且是两个动态数组拼接而成(nf_hook_entry 和 nf_hook_ops)
static struct nf_hook_entries *
nf_hook_entries_grow(const struct nf_hook_entries *old,
		     const struct nf_hook_ops *reg)
	if (old) {
        // 找到原有数据的 ops数组 和 entry数量
		orig_ops = nf_hook_entries_get_hook_ops(old);
            unsigned int n = e->num_hook_entries;
            const void *hook_end;
            hook_end = &e->hooks[n]; /* this is *past* ->hooks[]! */
            return (struct nf_hook_ops **)hook_end;

		for (i = 0; i < old_entries; i++)
			if (orig_ops[i] != &dummy_ops)
				alloc_entries++;
    }

    // 创建新的第二维度数组
	new = allocate_hook_entries_size(alloc_entries);
        struct nf_hook_entries *e;
        // 注意数组分为三部分
        size_t alloc = sizeof(*e) + // num_hook_entries
                   sizeof(struct nf_hook_entry) * num +
                   sizeof(struct nf_hook_ops *) * num +
                   sizeof(struct nf_hook_entries_rcu_head);
        e = kvzalloc(alloc, GFP_KERNEL);
        if (e)
            e->num_hook_entries = num;
        return e;

    // 找到新第二维数据中ops起点
	new_ops = nf_hook_entries_get_hook_ops(new);
        unsigned int n = e->num_hook_entries;
        const void *hook_end;
        hook_end = &e->hooks[n]; /* this is *past* ->hooks[]! */
        return (struct nf_hook_ops **)hook_end;

    // 根据旧数据和新节点填充new数组
	i = 0;
	nhooks = 0;
	while (i < old_entries) {
		if (orig_ops[i] == &dummy_ops) {
			++i;
			continue;
		}

		if (inserted || reg->priority > orig_ops[i]->priority) {
			new_ops[nhooks] = (void *)orig_ops[i];
			new->hooks[nhooks] = old->hooks[i];
			i++;
		} else {
			new_ops[nhooks] = (void *)reg;
			new->hooks[nhooks].hook = reg->hook;
			new->hooks[nhooks].priv = reg->priv;
			inserted = true;
		}
		nhooks++;
	}

	if (!inserted) {
		new_ops[nhooks] = (void *)reg;
		new->hooks[nhooks].hook = reg->hook;
		new->hooks[nhooks].priv = reg->priv;
	}

	return new;
```

## 总结
### xt_table 部分
```c
new_table = xt_register_table(net, table, &bootstrap, newinfo);

    net
	   struct netns_xt xt;
	       struct list_head tables[NFPROTO_NUMPROTO];    packet_filter
                NFPROTO_UNSPEC =  0,                        │ 复制
                NFPROTO_INET   =  1,                        ▼
                NFPROTO_IPV4   =  2,──────────────────►  table ──────────┐
                NFPROTO_ARP    =  3,      加入链表          ▲            ▼
                NFPROTO_NETDEV =  5,                        │           xt_table_info
                NFPROTO_BRIDGE =  7,                        │               entries 规则 
                NFPROTO_IPV6   = 10,                        │
                NFPROTO_DECNET = 12,                        │
        struct xt_table		*iptable_filter;────────────────┘
        struct xt_table		*iptable_mangle;     更新iptable_filter表
        struct xt_table		*iptable_raw;
        struct xt_table		*arptable_filter;
        struct xt_table		*nat_table;



```
`struct xt_table packet_filter` 加入 `net->xt.tables[table->af]` 链表

### netfilter部分

```c

                                   struct xt_table packet_filter 
                                              │
                                              ▼
	       struct nf_hook_ops * filter_ops = xt_hook_ops_alloc(&packet_filter, iptable_filter_hook);
                                              │
                                              ▼
                                         filter_ops
                                              │
                                              └─────────────────────────────────────────────────┐
                                                                                                │
   init_net                                                                                     │
       nf                                                                                       │
         struct nf_hook_entries *hooks_ipv4[]                                                   │
            NF_INET_PRE_ROUTING                                                                 │
            NF_INET_LOCAL_IN                                                                    │
            NF_INET_FORWARD      ─────► struct nf_hook_entries                                  │
            NF_INET_LOCAL_OUT                 num_hook_entries                         ◄────────┤
            NF_INET_POST_ROUTING              struct nf_hook_entry[num_hook_entries]   ◄────────┤
                                              struct nf_hook_ops[num_hook_entries]     ◄────────┘
                                              struct nf_hook_entries_rcu_head


```
利用xt_table提供的xt_hook_ops_alloc，将xt_table类型的 packet_filter构造出 nf_hook_ops 类型的filter_ops，
再将filter_ops加入netfilter 对应的数组,
并且回调函数为 iptable_filter_hook 



# iptables的规则实现
## 数据结构
内核中iptables的规则的实现

         iptables规则数组
        ┌────────────┬─────────────────────┬─────────────────────┐
        │  ipt_entry │ ipt_entry_match(es) │  ipt_entry_target   │
        ├────────────┼─────────────────────┼─────────────────────┤
        │  ipt_entry │ ipt_entry_match(es) │  ipt_entry_target   │
        ├────────────┴─────────────────────┴─────────────────────┤
        │                     ...                                │
        └────────────────────────────────────────────────────────┘


- ipt_entry：标准匹配结构，主要包含数据包的源、目的IP，出、入接口和掩码等；
- ipt_entry_match：扩展匹配。一条rule规则可能有零个或多个ipt_entry_match结构；
- ipt_entry_target：一条rule规则有且仅有一个target动作。就是当所有的标准匹配和扩展匹配都符合之后才来执行该target。


### ipt_entry
```c
/* 
这个结构定义了每个防火墙规则。由三部分组成：
1）通用的IP头信息 
2）匹配特定的信息 
3）如果规则匹配则执行的目标操作 
*/
struct ipt_entry {
    // 用于通用匹配的IP关键匹配项
	struct ipt_ip ip;

    /* 标记我们需要关注的字段。*/
	unsigned int nfcache;

    /*ipt_entry 和 匹配 的大小之和*/
    // 用于根据ipt_entry找到他的ipt_entry_target
	__u16 target_offset;
    /* ipt_entry、matches 和 target 的总大小 */
    // 用于找到下一条ipt_entry
	__u16 next_offset;

    /* 返回指针 */
	unsigned int comefrom;

    /* 包和字节计数器。 */
	struct xt_counters counters;

    // 动态数组，内容为 ipt_entry_match(s) 和 ipt_entry_target
	unsigned char elems[0];
};

// 必须初始化为零
 struct ipt_ip {
    /* 源IP和目的IP地址 */
    struct in_addr src, dst;
    /* 源和目的IP地址的掩码 */
    struct in_addr smsk, dmsk;
    char iniface[IFNAMSIZ], outiface[IFNAMSIZ];
    unsigned char iniface_mask[IFNAMSIZ], outiface_mask[IFNAMSIZ];

    /* 协议，0 = ANY */
    __u16 proto;

    /* 标志字节 */
    __u8 flags;
    /* 反向标志字节 */
    __u8 invflags;
};
```

### ipt_entry_match
匹配分为标准匹配和扩展匹配.
标准匹配只会检查IP地址，端口等通用信息，不会用到xt_match, 所以ipt_entry_match的match为NULL
扩展匹配就需要自己是实现 xt_match对象和相关回调函数
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
            // 扩展match
			struct xt_match *match;
		} kernel;

		/* Total length */
		__u16 match_size;
	} u;

	unsigned char data[0];
};


struct xt_match {
	struct list_head list;

	const char name[XT_EXTENSION_MAXNAMELEN];
	u_int8_t revision;

    /* 返回 true 或 false：返回 FALSE 并设置 *hotdrop 为 1 来强制立即丢弃数据包。 */
    /* 参数在 2.6.9 版本后发生变化，因为现在必须处理非线性 skb，并使用 skb_header_pointer 和 skb_ip_make_writable。 */
    // 扩展match的具体match方法
	bool (*match)(const struct sk_buff *skb,
		      struct xt_action_param *);

    /* 当用户尝试插入这种类型的条目时被调用。*/
	int (*checkentry)(const struct xt_mtchk_param *);

    /* 当这种类型的条目被删除时调用。*/
	void (*destroy)(const struct xt_mtdtor_param *);
#ifdef CONFIG_COMPAT
    /* 当用户空间的对齐方式与内核空间的对齐方式不同调用时 */
	void (*compat_from_user)(void *dst, const void *src);
	int (*compat_to_user)(void __user *dst, const void *src);
#endif
    /* 如果你是模块，请将其设置为 THIS_MODULE，否则设置为 NULL */
	struct module *me;

	const char *table;
	unsigned int matchsize;
	unsigned int usersize;
#ifdef CONFIG_COMPAT
	unsigned int compatsize;
#endif
	unsigned int hooks;
	unsigned short proto;

	unsigned short family;
};
```

### ipt_entry_target
target分为标准target和扩展target，
标准target就是 ACCEPT DROP REJECT等
扩展target就是 DNAT SNAT等以模块形式存在的target.
对于标准target不需要xt_target对象，即ipt_entry_target的target设置为NULL,
对于扩展target需要自己实现xt_target对象。
对于xt_target的target回调函数的编写需要注意一点：
该函数必须向netfilter框架返回IPT_CONTINUE, NF_ACCEPT, NF_DROP等值。
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

/*目标的注册钩子。*/
struct xt_target {
	struct list_head list;

	const char name[XT_EXTENSION_MAXNAMELEN];
	u_int8_t revision;

    /* 返回判决。自版本 2.6.9 以来，由于现在必须处理非线性 skbs，
       即使用 skb_copy_bits 和 skb_ip_make_writable，
       这个函数的参数顺序已经改变。 */
	unsigned int (*target)(struct sk_buff *skb,
			       const struct xt_action_param *);

    /*
     当用户尝试插入此类条目时被调用：
     hook_mask 是可以调用的钩子掩码。
     */
    /* 应返回成功值（0）或其他错误代码（-Exxx）。*/
	int (*checkentry)(const struct xt_tgchk_param *);

    /* 当这种类型的条目被删除时调用。*/
	void (*destroy)(const struct xt_tgdtor_param *);
#ifdef CONFIG_COMPAT
    /* 当用户空间对齐方式与内核空间的对齐方式不同调用时 */
	void (*compat_from_user)(void *dst, const void *src);
	int (*compat_to_user)(void __user *dst, const void *src);
#endif
	/* Set this to THIS_MODULE if you are a module, otherwise NULL */
    /* 如果你是模块，请设置此值为 THIS_MODULE，否则设置为 NULL */
	struct module *me;

	const char *table;
	unsigned int targetsize;
	unsigned int usersize;
#ifdef CONFIG_COMPAT
	unsigned int compatsize;
#endif
	unsigned int hooks;
	unsigned short proto;

	unsigned short family;
};

```

### 标准目标和扩展目标

xt_entry_target是基类target
xt_standard_target是继承xt_entry_target，添加verdict属性
```c
struct xt_standard_target {
	struct xt_entry_target target;
	int verdict; // 判决
};
```

### ipt_replace
用于iptables整张表替换的数据结构，主要见于用户空间配置规则时进行表的替换。
```c
struct ipt_replace {
    char name[XT_TABLE_MAXNAMELEN];       // 表名（如 "filter"）
    unsigned int valid_hooks;             // 有效的钩子位掩码（如 NF_INET_LOCAL_IN）
    unsigned int num_entries;             // 新规则的总数
    unsigned int size;                    // 新规则数据的总大小
    unsigned int hook_entry[NF_INET_NUMHOOKS]; // 各钩子点的规则入口偏移
    unsigned int underflow[NF_INET_NUMHOOKS];  // 各钩子点的规则下溢偏移
    unsigned int num_counters;            // 计数器数量
    struct xt_counters __user *counters;  // 用户空间传递的计数器指针
    unsigned char entries[];              // 实际规则数据（ipt_entry 数组）
};
```

### xt_table_info
iptables中记录表规则的实际结构
```c
struct xt_table_info {
    unsigned int size;       // 规则数据总大小（字节）
    unsigned int number;     // 当前规则条目总数
    unsigned int initial_entries;  // 初始规则数（用于优化）

    // 每个钩子点的规则入口偏移和下溢偏移
    unsigned int hook_entry[NF_INET_NUMHOOKS];
    unsigned int underflow[NF_INET_NUMHOOKS]; // 防止规则链越界，确保规则链结束在正确的位置

    void *entries[];         // 规则数据（存储实际规则链 ipt_entry 的数组）
};
```



## 防火墙算法

### ipt_do_table
iptables的所有表的入口方法都是ipt_do_table，如filter表

```c
static unsigned int
iptable_filter_hook(void *priv, struct sk_buff *skb,
		    const struct nf_hook_state *state)
	return ipt_do_table(skb, state, state->net->ipv4.iptable_filter);
```

```c
/*
 * skb : 被处理的数据
 * state : 包含输入输出设备，hook点等信息
 * table : filter/mangle/nat/raw 之一，包含hooks链
 */
unsigned int
ipt_do_table(struct sk_buff *skb,
	     const struct nf_hook_state *state,
	     struct xt_table *table)
	unsigned int hook = state->hook;
	const struct xt_table_info *private;
	const void *table_base;
	struct ipt_entry *e, **jumpstack;

	private = READ_ONCE(table->private); /* Address dependency. */
	table_base = private->entries;
	jumpstack  = (struct ipt_entry **)private->jumpstack[cpu];

    // 根据hook点从表中取得第一条规则
	e = get_entry(table_base, private->hook_entry[hook]);
        return (struct ipt_entry *)(base + offset);

	do {
		const struct xt_entry_target *t;
		const struct xt_entry_match *ematch;

        // skb必须符合标准匹配
		if (!ip_packet_match(ip, indev, outdev,
		    &e->ip, acpar.fragoff)) {
 no_match:
            // 如果标准匹配或扩展匹配失败，则使用下一条规则
			e = ipt_next_entry(e);
			continue;
		}

        // 如果通过标准匹配再进行扩展匹配
		xt_ematch_foreach(ematch, e) {
			acpar.match     = ematch->u.kernel.match;
			acpar.matchinfo = ematch->data;
			if (!acpar.match->match(skb, &acpar))
				goto no_match;
		}

        // 匹配成功，增加当前规则的计数器
		counter = xt_get_this_cpu_counter(&e->counters);
		ADD_COUNTER(*counter, skb->len, 1);

        // 获得规则的target
		t = ipt_get_target_c(e);

        // 如果没有定义扩展target，则使用标准target
		if (!t->u.kernel.target->target) {
			int v; // 表示判决

            // v <  0 为最终判决
            // v >= 0 为下条规则的偏移
            // 标准target没有逻辑操作，所以直接获得判决
			v = ((struct xt_standard_target *)t)->verdict;
			if (v < 0) {
				if (v != XT_RETURN) { // 如果是ACCEPT , REJECT，则得到最终判决
					verdict = (unsigned int)(-v) - 1;
					break;
				}
                // 判决为 RETURN，则需要考虑自定义链
				if (stackidx == 0) {
                    // policy ?
					e = get_entry(table_base,
					    private->underflow[hook]);
				} else {
                    // 自定义链返回到上层规则
					e = jumpstack[--stackidx];
					e = ipt_next_entry(e);
				}
				continue;
			}

            // v > 0 则为跳转到自定义规则或下一条规则
			if (table_base + v != ipt_next_entry(e) &&
			    !(e->ip.flags & IPT_F_GOTO)) {
                // 若不是跳转到下一条规则，则跳转到自定义规则链
				if (unlikely(stackidx >= private->stacksize)) {
					verdict = NF_DROP;
					break;
				}
                // 需要记录返回原有链的地址
				jumpstack[stackidx++] = e;
			}

            // 遍历下条规则
			e = get_entry(table_base, v);
			continue;
		}

        // 使用扩展target
		acpar.target   = t->u.kernel.target;
		acpar.targinfo = t->data;
		verdict = t->u.kernel.target->target(skb, &acpar);
		if (verdict == XT_CONTINUE) {
            /* 目标可能已经改变了某些东西。*/
			ip = ip_hdr(skb);
			e = ipt_next_entry(e);
		} else {
            // 完成判决
			break;
		}
	} while (!acpar.hotdrop);

	if (acpar.hotdrop)
		return NF_DROP;
	else return verdict;
```


# nf hook
## 内核使用 nf hook 
协议栈在多处使用 NF_HOOK 留下hook点
```c
/*
 * pf : 协议族对于IPv4 NFPROTO_IPV4, IPv6 NFPROTO_IPV6
 * hook : 挂载点 
 * okfn : 当通过nf hook后调用的函数
 */
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk, struct sk_buff *skb,
	struct net_device *in, struct net_device *out,
	int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	int ret = nf_hook(pf, hook, net, sk, skb, in, out, okfn);
	if (ret == 1)
		ret = okfn(net, sk, skb);
	return ret;
}
```
使用示例
```c
	return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN,
		       net, NULL, skb, skb->dev, NULL,
		       ip_local_deliver_finish);

```
可使用的挂载点
```c
enum nf_inet_hooks {
	NF_INET_PRE_ROUTING,
	NF_INET_LOCAL_IN,
	NF_INET_FORWARD,
	NF_INET_LOCAL_OUT,
	NF_INET_POST_ROUTING,
	NF_INET_NUMHOOKS,
	NF_INET_INGRESS = NF_INET_NUMHOOKS,
};
```

# 注册nf hook
模块使用如下方法注册hook

```c
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *reg);
int nf_register_net_hooks(struct net *net, const struct nf_hook_ops *reg,
			  unsigned int n);
```


```c
typedef unsigned int nf_hookfn(void *priv,
			       struct sk_buff *skb,
			       const struct nf_hook_state *state);
struct nf_hook_ops {
	/* User fills in from here down. */
	nf_hookfn		*hook;     // hook回调函数
	struct net_device	*dev;
	void			*priv;
	u_int8_t		pf;        // 协议族
	unsigned int		hooknum; // 挂载点
	/* Hooks are ordered in ascending priority. */
	int			priority;       // 优先级，升序排队
};
```

每个hook的返回值
```c
#define NF_DROP 0
#define NF_ACCEPT 1
#define NF_STOLEN 2   // 数据包不传输，由hook方法处理
#define NF_QUEUE 3    // 将数据包排序，供用户空间使用
#define NF_REPEAT 4   // 再次调用hook
#define NF_STOP 5	/* Deprecated, for userspace nf_queue compatibility. */
#define NF_MAX_VERDICT NF_STOP
```
hook 的优先级
```c
enum nf_ip_hook_priorities {
	NF_IP_PRI_FIRST = INT_MIN,
	NF_IP_PRI_RAW_BEFORE_DEFRAG = -450,
	NF_IP_PRI_CONNTRACK_DEFRAG = -400,
	NF_IP_PRI_RAW = -300,              // raw
	NF_IP_PRI_SELINUX_FIRST = -225,
	NF_IP_PRI_CONNTRACK = -200,        // conntrack
	NF_IP_PRI_MANGLE = -150,           // mangle
	NF_IP_PRI_NAT_DST = -100,          // DNAT
	NF_IP_PRI_FILTER = 0,              // filter
	NF_IP_PRI_SECURITY = 50,
	NF_IP_PRI_NAT_SRC = 100,           // SNAT
	NF_IP_PRI_SELINUX_LAST = 225,
	NF_IP_PRI_CONNTRACK_HELPER = 300,
	NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,
	NF_IP_PRI_LAST = INT_MAX,
};


```

## nf_register_net_hook


```c
struct nf_hook_ops a = 
	{
		.hook		= ipv4_conntrack_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK,   // -200
	};
```

```c
// nf_register_net_hook ->  __nf_register_net_hook

static int __nf_register_net_hook(struct net *net, int pf,
				  const struct nf_hook_ops *reg)
{
	struct nf_hook_entries *p, *new_hooks;
	struct nf_hook_entries __rcu **pp;
	int err;

	switch (pf) {
	case NFPROTO_NETDEV:
		err = nf_ingress_check(net, reg, NF_NETDEV_INGRESS);
		if (err < 0)
			return err;
		break;
	case NFPROTO_INET:
		if (reg->hooknum != NF_INET_INGRESS)
			break;

		err = nf_ingress_check(net, reg, NF_INET_INGRESS);
		if (err < 0)
			return err;
		break;
	}

	// 根据 pf hooknum 获得旧的 插入点, 注意返回的二级指针
	pp = nf_hook_entry_head(net, pf, reg->hooknum, reg->dev);
		switch (pf) {
		case NFPROTO_IPV4:
			if (WARN_ON_ONCE(ARRAY_SIZE(net->nf.hooks_ipv4) <= hooknum))
				return NULL;
			return net->nf.hooks_ipv4 + hooknum;

	mutex_lock(&nf_hook_mutex);

	// 获得旧的插入点
	p = nf_entry_dereference(*pp);

	// 将新的hook信息加入插入点
	new_hooks = nf_hook_entries_grow(p, reg);

	...

	// 释放旧的插入点
	nf_hook_entries_free(p);
	return 0;
}
```

### nf_hook_entries_grow
```c
static struct nf_hook_entries *
nf_hook_entries_grow(const struct nf_hook_entries *old,
		     const struct nf_hook_ops *reg)
{
	unsigned int i, alloc_entries, nhooks, old_entries;
	struct nf_hook_ops **orig_ops = NULL;
	struct nf_hook_ops **new_ops;
	struct nf_hook_entries *new;
	bool inserted = false;

	alloc_entries = 1;
	old_entries = old ? old->num_hook_entries : 0;

	// 如果有旧的插入点，得到 旧的ops 数组
	// 并计算需要拷贝的 ops 项
	if (old) {
		orig_ops = nf_hook_entries_get_hook_ops(old);

		for (i = 0; i < old_entries; i++) {
			if (orig_ops[i] != &dummy_ops)
				alloc_entries++;
		}
	}

	if (alloc_entries > MAX_HOOK_COUNT)
		return ERR_PTR(-E2BIG);

	// 分配空间给新的插入点
	new = allocate_hook_entries_size(alloc_entries);
		struct nf_hook_entries *e;
		size_t alloc = sizeof(*e) +
				   sizeof(struct nf_hook_entry) * num +
				   sizeof(struct nf_hook_ops *) * num +
				   sizeof(struct nf_hook_entries_rcu_head);
		e = kvzalloc(alloc, GFP_KERNEL);
		if (e)
			e->num_hook_entries = num;
		return e;

	// 获得新的ops数组
	new_ops = nf_hook_entries_get_hook_ops(new);
		unsigned int n = e->num_hook_entries;
		const void *hook_end;
		hook_end = &e->hooks[n]; /* this is *past* ->hooks[]! */
		return (struct nf_hook_ops **)hook_end;


	// 按照优先级将新的reg插入到新插入点，并拷贝旧的项
	i = 0;
	nhooks = 0;
	while (i < old_entries) {
		if (orig_ops[i] == &dummy_ops) {
			++i;
			continue;
		}

		if (inserted || reg->priority > orig_ops[i]->priority) {
			new_ops[nhooks] = (void *)orig_ops[i];
			new->hooks[nhooks] = old->hooks[i];
			i++;
		} else {
			new_ops[nhooks] = (void *)reg;
			new->hooks[nhooks].hook = reg->hook;
			new->hooks[nhooks].priv = reg->priv;
			inserted = true;
		}
		nhooks++;
	}

	if (!inserted) {
		new_ops[nhooks] = (void *)reg;
		new->hooks[nhooks].hook = reg->hook;
		new->hooks[nhooks].priv = reg->priv;
	}

	// 返回新的插入点
	return new;
}
```

数据结构图

![](./pic/63.jpg)

## nf_hook 的调用
```c
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk, struct sk_buff *skb,
	struct net_device *in, struct net_device *out,
	int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	int ret = nf_hook(pf, hook, net, sk, skb, in, out, okfn);

		// 根据 pf 和 hooknum 找到 hook_entries
		switch (pf) {
		case NFPROTO_IPV4:
			hook_head = rcu_dereference(net->nf.hooks_ipv4[hook]);
			break;
			...
		}

		if (hook_head) {
			struct nf_hook_state state;

			nf_hook_state_init(&state, hook, pf, indev, outdev,
					   sk, net, okfn);
				pstate->hook = hook;
				pstate->pf = pf;
				pstate->in = indev;
				pstate->out = outdev;
				pstate->sk = sk;
				pstate->net = net;
				pstate->okfn = okfn;

			// 从0开始经过 hook entries
			ret = nf_hook_slow(skb, &state, hook_head, 0);
		}
		rcu_read_unlock();

		return ret;

	if (ret == 1)
		ret = okfn(net, sk, skb);
	return ret;
}
```

```c
int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state,
		 const struct nf_hook_entries *e, unsigned int s)
{
	unsigned int verdict;
	int ret;

	for (; s < e->num_hook_entries; s++) {
		
		// 执行 hook entry 的 hook方法
		verdict = nf_hook_entry_hookfn(&e->hooks[s], skb, state);
				return entry->hook(entry->priv, skb, state);

		switch (verdict & NF_VERDICT_MASK) {
		case NF_ACCEPT:
			break;
		case NF_DROP:
			kfree_skb(skb);
			ret = NF_DROP_GETERR(verdict);
			if (ret == 0)
				ret = -EPERM;
			return ret;
		case NF_QUEUE:
			ret = nf_queue(skb, state, s, verdict);
			if (ret == 1)
				continue;
			return ret;
		default:
			/* Implicit handling for NF_STOLEN, as well as any other
			 * non conventional verdicts.
			 */
			return 0;
		}
	}

	return 1;
}
```

# conntrack
conntrack 是基于 nf hook 实现，只要使用了conntrack模块，则一定会挂载 conntrack 的 hook
```c
static const struct nf_hook_ops ipv4_conntrack_ops[] = {
	{
		.hook		= ipv4_conntrack_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK,   // -200
	},
	{
		.hook		= ipv4_conntrack_local,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_CONNTRACK, // -200
	},
	{
		.hook		= ipv4_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM, // INT_MAX
	},
	{
		.hook		= ipv4_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM, // INT_MAX
	},
};
```
其中最重要的是 ipv4_conntrack_in  和 ipv4_conntrack_local

他们的优先级高于 nat hook，他们为使用NAT提供必要帮助。

![](./pic/62.jpg)

## 注册conntrack hook
```c
int nf_ct_netns_get(struct net *net, u8 nfproto)
	case NFPROTO_INET:
		nf_ct_netns_inet_get(net);
			nf_ct_netns_do_get(struct net *net, u8 nfproto)
					nf_register_net_hooks(net, ipv4_conntrack_ops,
									ARRAY_SIZE(ipv4_conntrack_ops));
```
## 连接跟踪 nf_conntrack_tuple
```c
```

## ipv4_conntrack_in
```c
static unsigned int ipv4_conntrack_in(void *priv,
				      struct sk_buff *skb,
				      const struct nf_hook_state *state)
	return nf_conntrack_in(skb, state);
```
