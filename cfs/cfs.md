# CFS分析

## CFS计算方法
- 什么是CFS：CFS是linux内核中的一种调度算法，它的全称是 Complete Fair Scheduler（完全公平调度算法）。
- CFS是怎么工作的：CFS是从就绪状态的进程中获取一个vruntime（虚拟时间）值最小的一个进程来执行，那么vruntime怎么来的呢？在回答这个问题前要先来说一说“权重”。“权重”是用来衡量进程优先级的一个值，权重越大，优先级越高。在linux内核中会有一个权重的数组，新创建的进程的权重默认值是1024。每个进程会有一个nice值，内核可以根据nice值来从数组中得到进程的权重值。
```C
//linux内核中的权重数组
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
//前面的-20，-15...这些值表示的是nice值，nice的范围是（-20～19）。从这个数组可以知道，当nice=0的时候，权重值是1024，当nice=1的时候，权重值是820....
```
```C
//数组sched_prio_to_wmult里面的值就是（2^32/权重）
const u32 sched_prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
};
```
```C
# define SCHED_FIXEDPOINT_SHIFT		10

# define NICE_0_LOAD_SHIFT	 (SCHED_FIXEDPOINT_SHIFT)

#define NICE_0_LOAD		(1L << NICE_0_LOAD_SHIFT)

#define WMULT_SHIFT	32

# define scale_load_down(w) \
({ \
	unsigned long __w = (w); \
	if (__w) \
		__w = max(2UL, __w >> SCHED_FIXEDPOINT_SHIFT); \
	__w; \
})

static void __update_inv_weight(struct load_weight *lw)
{
	unsigned long w;

	if (likely(lw->inv_weight))
		return;

	w = scale_load_down(lw->weight);

	if (BITS_PER_LONG > 32 && unlikely(w >= WMULT_CONST))
		lw->inv_weight = 1;
	else if (unlikely(!w))
		lw->inv_weight = WMULT_CONST;
	else
		lw->inv_weight = WMULT_CONST / w;
}

static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
	if (unlikely(se->load.weight != NICE_0_LOAD))
	//delta表示实际运行的时间.
		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);

	return delta;
}

//vruntime的计算函数
/*
 * delta_exec * weight / lw.weight
 *   OR
 * (delta_exec * (weight * lw->inv_weight)) >> WMULT_SHIFT
 *
 * Either weight := NICE_0_LOAD and lw \e sched_prio_to_wmult[], in which case
 * we're guaranteed shift stays positive because inv_weight is guaranteed to
 * fit 32 bits, and NICE_0_LOAD gives another 10 bits; therefore shift >= 22.
 *
 * Or, weight =< lw.weight (because lw.weight is the runqueue weight), thus
 * weight/lw.weight <= 1, and therefore our shift will also be positive.
 */
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
	u64 fact = scale_load_down(weight);
	u32 fact_hi = (u32)(fact >> 32);
	int shift = WMULT_SHIFT;
	int fs;

	__update_inv_weight(lw);

	if (unlikely(fact_hi)) {
		fs = fls(fact_hi);
		shift -= fs;
		fact >>= fs;
	}

//lw->inv_weight等于sched_prio_to_wmult数组中nice对应的值，比如nice=0，那么lw->inv_weight=4194304
	fact = mul_u32_u32(fact, lw->inv_weight);

	fact_hi = (u32)(fact >> 32);
	if (fact_hi) {
		fs = fls(fact_hi);
		shift -= fs;
		fact >>= fs;
	}

	return mul_u64_u32_shr(delta_exec, fact, shift);
}
...
...
...
static inline u64 mul_u32_u32(u32 a, u32 b)
{
	return (u64)a * b;
}

static inline u64 mul_u64_u32_shr(u64 a, u32 mul, unsigned int shift)
{
	u32 ah, al;
	u64 ret;

	al = a;
	ah = a >> 32;

	ret = mul_u32_u32(al, mul) >> shift;
	if (ah)
		ret += mul_u32_u32(ah, mul) << (32 - shift);

	return ret;
}
```
- CFS计算公式：根据```__calc_delta```函数的解析可以得出vruntime的计算公式，{vruntime = delta_exec * NICE_0_LOAD / weight = (delta_exec * NICE_0_LOAD * 2^32 / weight) >> 32}=>{vruntime = (delta_exec * NICE_0_LOAD * inv_weight) >> 32; inv_weight = 2^32 / weight;}(delta_exec：实际运行时间；NICE_0_LOAD：根据代码可以得到值为2^10就是1024；weight：权重)

## CFS举例
- 小试牛刀：现在有A、B、C三个进程，他们的nice值分别为-1、0、1，其中delta_exec=100ms。查```sched_prio_to_weight```数组可得对应的权重值为1277，1024，820。根据公式可得，A进程的vruntime = 100 * 1024 / 1277 = 80.18，同理可得B进程的vruntime = 100，C进程的vruntime = 124.87；由此可以得出进程A的vruntime的值最小，下次优先执行A进程。

## 总结
- 本文是以单核32位CPU的情况下来分析的，多核64位可能会有点不太一样，如有问题欢迎纠正。
- Email：HengDi_Sun@163.com
- 读者永远比小编长的好看～～～。
