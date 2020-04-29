# Linux kernel interrupt（中断）
- 作者：Match

## 中断的作用
- easy 一张图就能解释这个问题
![interrupt.png](D:/images/interrupt.png)

## Linux kernel是怎么来使用interrupt的？
- 简单来说linux kernel维护了一张中断向量表，当有中断发生的时候，内核会根据中断号去中断向量表中找到对应的中断处理代码，然后保护现场并跳转进入中断处理程序。处理完以后再跳出来并回复现场。这样一次完整的中断的OK了。一般来说中断处理程序要求是能快速处理的程序，不允许添加各种锁，或者延时，或者会引起调度(schedule)的代码。为什么不允许？因为中断打断了原来的程序，然后中断处理程序又因为某种原因耗时很长，或者死锁了，那就无法回到原来的程序了，也就是卡住了。

### 万一非要在中断中延时或者影响中断处理效率的操作怎么办？
- 早有骨灰级大神给我们想好办法了，那就是把中断分成上半部和下半部，上半部负责处理快速的必要的处理，下半部则在可以做一些稍微复杂的操作。

- 啥？听说中断下半部分能做稍微复杂一点的操作？？？那么是怎么实现的呢？一脸的？？？

###  中断下半部分实现？

中断下半部机制 | 现在的使用情况  
-|-
~~BH~~ | 已经被淘汰了（凉凉）
~~任务队列（Task queues）~~ | 已经被淘汰了（凉凉）
软中断（Softirq） | 有用
tasklet | 有用
工作队列（Work queues） | 有用

1. 由上表可得，我们只要看下面3中实现方式就可以了，是不是很开心，哈哈哈。
2. 由于软中断（Softirq)和tasklet的实现方式很像，但是tasklet得接口更好用，所以又把软中断淘汰掉了，哇偶，又少了一个，开心！！！

###  中断下半部分有什么软用？

- 不好意思，它真的有点软用，哈哈哈。江湖规矩具体例子来说明。
- 现在我有一块**网卡**（又是你，怎么又是你**网卡**，不好意思，小编今天就想拿你开刀），我呢想打游戏，家里的网速的100M很快，网卡会在接受到网络数据的时候产生一个中断来告诉CPU，嗨CPU，我有网络数据了，你快来处理一下。这时候CPU就进入了这个中断来处理网络数据了，但是在处理的时候又有新的网络数据来了，这个时候网卡再发出中断，由于CPU还在处理上一个网络数据，所以网卡只能把数据放在自带的缓冲中，但是缓冲有限，存满了以后要是还有新的网络数据来就直接丢弃了，这样就导致了网络丢包，以至于我团战还没开始就结束了，很难受。这个时候，中断被 分成两部分的思想就派上用场了，首先网络中断产生，由中断上半部分负责把网卡缓存中的数据copy到内存中，然后网卡又可以把新的网络数据放到缓存中，到了适当的时候，中断 下半部分负责把原来copy到内存中的网络数据解析。从游戏画面上来说，哇，团战打起来了，我也看到了团战的过程。（爽歪歪）。当然知道了这个强大的作用以后就要知道一下实现原理了，这里用tasklet来做讲解。

#### tasklet是怎么实现的？

- tasklet由`tasklet_struct`这个结构体表示，一个结构体对象表示一个tasklet，多个tasklet组成一个链表，然后内核会在何时的时间去遍历这个链表，并找出符合条件的tasklet来执行，在执行的时候顺便把状态改成正在执行状态，这样在多核系统中，其它的处理器就不会再去执行了，执行完以后再把正在执行状态清除。这样一次完整的处理就OK了。同学们看到这里肯定觉得so easy是不是？你说不是我弄死你，哈哈哈。下面就从代码角度来看看吧，嘻嘻。

```C
struct tasklet_struct{
	//next这个指针用来指向链表中的下一个tasklet_struct
	struct tasklet_struct *next;
	//state表示当前这个tasklet的状态
	unsigned long state;
	//count是一个当前tasklet的引用计数器，用atomic_t定义表示改变量需要用原子操作
	atomic_t count;
	//tasklet的实际处理函数
	void (*func)(unsigned long);
	//传递给func函数的参数，用unsigned long做参数的话就可以传递指针了，内核的骨灰级大佬早就想好了的。
	unsigned long data;
};
```
> **说明一下：**
> 只有当*count*为0的时候tasklet对象才有可能被执行。

```C
//这个宏是创建一个tasklet
DECLARE_TASKLET(name, func, data);

//这个宏是删除一个tasklet
DECLARE_TASKLET_DISABLED(name, func, data);
```
- 万事具备，只欠自己动手写代码了

```C
...
struct cdev tasklet_cdev;

//tasklet的具体执行函数，就是每次进来打印一下第几次进来，打印一下参数
static void fun_tasklet(unsigned long arg)
{
	static int num = 0;
	printk("num=%d,fun tasklet arg=%lld\n",num,arg);
	num++;
}

//创建一个tasklet结构，传一个参数8888很吉利
DECLARE_TASKLET(Tasklet,fun_tasklet,(unsigned long)8888);

...

//中断发生的时候会调用的函数
static irqreturn_t tasklet_handler(int irq,void *dev_id)
{
	//调度tasklet
	tasklet_schedule(&Tasklet);

	return IRQ_HANDLED;
}

...

static int __init my_tasklet_init(void)
{
...

	//注册中断，我这里用网卡来模拟，我的电脑的网卡中断号是19，可以通过`cat /proc/interrupts`命令来查看linux的中断信息**IRQF_TRIGGER_HIGH**表示高电平出发，**IRQF_SHARED**表示共享中断
	ret = request_irq(19,tasklet_handler,IRQF_TRIGGER_HIGH | IRQF_SHARED,TASKLET_DEV_NAME,&tasklet_cdev);
	
...
}

static void __exit my_tasklet_exit(void)
{
	dev_t dev;

	printk("tasklet exit...");

	dev = MKDEV(TASKLET_MAJOR,TASKLET_MINOR);
	//释放中断
	free_irq(19,&tasklet_cdev);
	cdev_del(&tasklet_cdev);
	unregister_chrdev_region(dev,TASKLET_DEV_CNT);
}

...
```
- 由于是驱动程序，所以需要把代码编译成.ko文件，然后insmod上去（当然先要更具代码注册设备节点）最后你在启动浏览器，随便打开一个网页，就会产生很多网络中断。通过dmesg命令可以看到我们的网络中断程序被执行了，perfect。
![asklet_effect.jpg](D:/images/tasklet_effect.jpg)

#### 工作队列（Work queues）是啥玩意儿？
- 这玩意儿就是把中断下半部放到一个队列中，然后由一个内核线程来遍历这个队列，找到需要执行代码时就执行，没有找到就休眠，由于是在线程中执行，所以支持线程的特性（比如线程调度机制）。从代码上的实现来看和tasklet差不多，这里就允许我偷个懒不写代码了。（我没说我很勤快啊，啊哈哈）

## 结尾

- 完整源码在github中的地址 [https://github.com/520Matches/CoderStudy](https://github.com/520Matches/CoderStudy)
- 意见反馈邮箱 **HengDi_Sun@163.com**
- 喜欢的朋友关注微信订阅号 **CoderPark**，点击右下角的 **再看**
- 读者永远比小编长的好看~~~~

