
版权声明：本文为本文为博主原创文章，转载请注明出处 [https://github.com/wsg1100。如有错误，欢迎指正。](https://github.com)


目录* [一、前言](https://github.com)
	+ [PREEMPT\-RT（RT Throttling）](https://github.com)
* [一、xenomai watchdog介绍](https://github.com):[milou加速器](https://xinminxuehui.org)
* [二、xenomai watchdog工作原理](https://github.com)
* [三、使用场景](https://github.com)

本文介绍xenomai watchdog，有什么用？它是如何工作的？


## 一、前言


介绍xenomai watchdog之前，有必要先介绍操作系统对实时任务的调度，实时任务的调度是指在满足实时任务时间约束的情况下，对任务进行排队和执行的策略。两种常见的实时任务调度算法是RR调度（Round Robin，轮转调度）和FIFO调度（First In First Out，先进先出调度）。


正常情况下，高优先级实时任务对CPU时间绝对的优先权。如果此时最高优先级任务存在bug，出错或进入一个不存在主动和被动让出CPU资源的逻辑时，系统中的鼠标、键盘、屏幕等非实时任务将会因为得不到CPU运行时间饿死，导致系统失去响应。


为此PREEMPT\-RT和xenomai给出了不同的解决方案。


### PREEMPT\-RT（RT Throttling）


对于PREEMPT\-RT，PREEMPT\-RT提供了一个机制，确保非实时任务能在某个时间点执行，该机制也被称为RT限流（RT Throttling），它由两个值决定:


* `/proc/sys/kernel/sched_rt_period_us` 定义了微秒级别的窗口，在这个窗口里调度器会在实时和非实时任务之间共享资源，默认1 s。
* `/proc/sys/kernel/sched_rt_runtime_us` 则规定了在上述窗口中为实时任务分配的时长比例。默认值950000us，即95%。意味着实时任务在每 1 秒内最多可以使用 950 毫秒的 CPU 时间，剩余的 50 毫秒留给其他非实时任务。


可以通过以下方式修改这些值：



```
echo 950000 > /proc/sys/kernel/sched_rt_runtime_us
echo 1000000 > /proc/sys/kernel/sched_rt_period_us

```


> 需要注意的是，修改这些值需要超级用户（root）权限。


RT Throttling保证了即使实时任务出现错误或者无限循环，也会为非实时任务预留一定的CPU运行时间，方便我们定位和debug。


xenomai也有实时任务的限制措施xenomai watchdog，但与PREEMPT\-RT的RT Throttling不同。


## 一、xenomai watchdog介绍


xenomai watchdog是xenomai内核提供的一个**检测xenomai实时任务是否长期占用CPU机制**，内核编译时通过以下配置启用该功能。



```
[*] Xenomai/cobalt  ---> 
     [*]   Debug support  --->
    		[*]   Watchdog support
    		(4)     Watchdog timeout 

```

其中`Watchdog timeout`是看门狗动作的超时时间，时间单位是秒，允许配置的默认最大时间为60秒。内核启用后，看门狗超时时间还可通过内核参数`watchdog_timeout`在启动时修改，单位：秒，值不受限制。


当xenomai watchdog触发时，watchdog会向**当前cpu运行的线程**发送SIGDEBUG signal，该信号会使实时任务结束，同时内核会输出信息，实时任务结束后系统恢复响应，通过`demsg`命令可以看到。



```
[Xenomai] watchdog triggered on CPU #0 -- runaway thread 'RT_Thread' signaled

```

那xenomai watchdog是如何工作的？有什么局限？不使用会发生什么？


## 二、xenomai watchdog工作原理


我们知道Xenomai 是一个双调度核操作系统，它在内核态添加了一个高优先级的实时调度核 Cobalt 来管理实时任务。Cobalt 调度核与 Linux 调度核共存，通过 Ipipeline 机制将两个调度上下文分为实时域和非实时域，Ipipeline 确保了 Cobalt 内核（实时域）的优先级高于 Linux 内核（非实时域,也称root domain），linux内核退化为成为 Cobalt 内核的idle任务，从而保障实时任务的实时性；（有关该部分，请查阅本博客其他文章）。


实时域和非实时域会随着任务的运行情况而来回切换。当没有实时任务需要运行释放CPU资源给linux非实时任务，或者实时任务调用了linux提供的系统资源的实时，会切换到非实时域。
![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/xenomai-arc.png)


看门狗的触发逻辑是这样的，**当进入实时任务调度上下文的时候，看门狗启动开始定时，离开实时上下文（实时任务调用了非实时服务或者主动睡眠让出 cpu） 停止，**只要看门狗超时说明实时任务在这段时间内一直在运行，**看门狗看管的是整个实时任务集合，不是某个特定任务，看门狗超时触发的时候会把当前 cpu 运行的任务 kill 掉，任何一个实时任务都有可能在watchdog触发这个时间点上，存在误伤**。


![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/image-20220424091221790.png)


具体代码如下：



```
static inline void enter_root(struct xnthread *root)
{
	struct xnarchtcb *rootcb __maybe_unused = xnthread_archtcb(root);

#ifdef CONFIG_XENO_OPT_WATCHDOG
	xntimer_stop(&root->sched->wdtimer);
#endif
	/*...*/
}

static inline void leave_root(struct xnthread *root)
{
	struct xnarchtcb *rootcb = xnthread_archtcb(root);
	struct task_struct *p = current;

	/*...*/

#ifdef CONFIG_XENO_OPT_WATCHDOG
	xntimer_start(&root->sched->wdtimer, get_watchdog_timeout(),
		      XN_INFINITE, XN_RELATIVE);
#endif
}

```

而看门狗处理逻辑也很简单，如果当前处于是root域，不处理；若当前是用户态实时任务，则直接发送信号；若当前运行的内核态实时任务，则将当前任务状态设置为XNKICKED并取消运行。



```
static void watchdog_handler(struct xntimer *timer)
{
	struct xnsched *sched = xnsched_current();
	struct xnthread *curr = sched->curr;

	if (likely(xnthread_test_state(curr, XNROOT))) {/*当前处于root域*/
		xnsched_reset_watchdog(sched);
		return;
	}

	if (likely(++sched->wdcount < wd_timeout_arg))
		return;

	trace_cobalt_watchdog_signal(curr);

	if (xnthread_test_state(curr, XNUSER)) {	/*用户态实时任务*/
		printk(XENO_WARNING "watchdog triggered on CPU #%d -- runaway thread "
		       "'%s' signaled\n", xnsched_cpu(sched), curr->name);
		xnthread_call_mayday(curr, SIGDEBUG_WATCHDOG);
	} else {								/*内核态实时任务*/
		printk(XENO_WARNING "watchdog triggered on CPU #%d -- runaway thread "
		       "'%s' canceled\n", xnsched_cpu(sched), curr->name);
		/*
		 * On behalf on an IRQ handler, xnthread_cancel()
		 * would go half way cancelling the preempted
		 * thread. Therefore we manually raise XNKICKED to
		 * cause the next call to xnthread_suspend() to return
		 * early in XNBREAK condition, and XNCANCELD so that
		 * @thread exits next time it invokes
		 * xnthread_test_cancel().
		 */
		xnthread_set_info(curr, XNKICKED|XNCANCELD);
	}

	xnsched_reset_watchdog(sched);
}

```

## 三、使用场景


xenomai watchdog会导致出问题的实时任务退出，所以一般在实时软件开发阶段，开启watchdog可以尽早暴露实时应用潜在的出错或无限循环问题，避免软件发布后产生严重后果。


如果实时应用发布后，在特定场景下出现系统无响应问题，可用启用watchdog来排查定位。


下一篇文章，我将给大家介绍一个真实生产环境中遇到的问题，一个外部条件触发低优先级实时任务进入无限循环逻辑后，导致整个系统实时任务调度异常的问题，敬请期待。


