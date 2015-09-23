---
title: Golang调度器源码分析
date: 2015-09-20 17:54:00
categories: golang
---

#1 为什么Golang需要调度器？

&#160; &#160; &#160; &#160;Goroutine的引入是为了方便高并发程序的编写。
一个Goroutine在进行阻塞操作（比如系统调用）时，会把当前线程中的其他Goroutine移交到其他线程中继续执行，
从而避免了整个程序的阻塞。

&#160; &#160; &#160; &#160;由于Golang引入了垃圾回收（gc），在执行gc时就要求Goroutine是停止的。通过自己实现调度器，就可以方便的实现该功能。
通过多个Goroutine来实现并发程序，既有异步IO的优势，又具有多线程、多进程编写程序的便利性。

&#160; &#160; &#160; &#160;引入Goroutine，也意味着引入了极大的复杂性。一个Goroutine既要包含要执行的代码，
又要包含用于执行该代码的栈和PC、SP指针。

#2 调度器解决了什么问题？
##2.1 栈管理

&#160; &#160; &#160; &#160;既然每个Goroutine都有自己的栈，那么在创建Goroutine时，就要同时创建对应的栈。
Goroutine在执行时，栈空间会不停增长。
栈通常是连续增长的，由于每个进程中的各个线程共享虚拟内存空间，当有多个线程时，就需要为每个线程分配不同起始地址的栈。
这就需要在分配栈之前先预估每个线程栈的大小。如果线程数量非常多，就很容易栈溢出。

&#160; &#160; &#160; &#160;为了解决这个问题，就有了[Split Stacks](https://gcc.gnu.org/wiki/SplitStacks)技术：
创建栈时，只分配一块比较小的内存，如果进行某次函数调用导致栈空间不足时，就会在其他地方分配一块新的栈空间。
新的空间不需要和老的栈空间连续。函数调用的参数会拷贝到新的栈空间中，接下来的函数执行都在新栈空间中进行。

&#160; &#160; &#160; &#160;Golang的栈管理方式与此类似，但是为了更高的效率，使用了连续栈
（[Golang连续栈](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub)）
实现方式也是先分配一块固定大小的栈，在栈空间不足时，分配一块更大的栈，并把旧的栈全部拷贝到新栈中。
这样避免了Split Stacks方法可能导致的频繁内存分配和释放。

##2.2 抢占式调度

&#160; &#160; &#160; &#160;Goroutine的执行是可以被抢占的。如果一个Goroutine一直占用CPU，长时间没有被调度过，
就会被runtime抢占掉，把CPU时间交给其他Goroutine。

#3 调度器的设计

&#160; &#160; &#160; &#160;Golang调度器引入了三个结构来对调度的过程建模：

* *G* 代表一个Goroutine；
* *M* 代表一个操作系统的线程；
* *P* 代表一个CPU处理器，通常P的数量等于CPU核数（*GOMAXPROCS*）。

&#160; &#160; &#160; &#160;三者都在runtime2.go中定义，他们之间的关系如下： 

* *G*需要绑定在*M*上才能运行； 
* *M*需要绑定*P*才能运行； 
* 程序中的多个*M*并不会同时都处于执行状态，最多只有*GOMAXPROCS*个*M*在执行。

&#160; &#160; &#160; &#160;早期版本的Golang是没有*P*的，调度是由*G*与*M*完成。
这样的问题在于每当创建、终止Goroutine或者需要调度时，需要一个全局的锁来保护调度的相关对象(sched)。
全局锁严重影响Goroutine的并发性能。
([Scalable Go Scheduler](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit))

&#160; &#160; &#160; &#160;通过引入*P*，实现了一种叫做*work-stealing*的调度算法：

* 每个*P*维护一个*G*队列； 
* 当一个*G*被创建出来，或者变为可执行状态时，就把他放到*P*的可执行队列中； 
* 当一个*G*执行结束时，*P*会从队列中把该*G*取出；如果此时*P*的队列为空，即没有其他*G*可以执行，
就随机选择另外一个*P*，从其可执行的*G*队列中偷取一半。 

&#160; &#160; &#160; &#160;该算法避免了在Goroutine调度时使用全局锁。

#4 调度器的实现

##4.1 schedule()与findrunnable()函数

&#160; &#160; &#160; &#160;Goroutine调度是在*P*中进行，每当runtime需要进行调度时，会调用schedule()函数，
该函数在proc1.go文件中定义。

&#160; &#160; &#160; &#160;schedule()函数首先调用runqget()从当前*P*的队列中取一个可以执行的*G*。
如果队列为空，继续调用findrunnable()函数。findrunnable()函数会按照以下顺序来取得*G*：

1. 调用runqget()从当前*P*的队列中取*G*（和schedule()中的调用相同）；
2. 调用globrunqget()从全局队列中取可执行的*G*；
3. 调用netpoll()取异步调用结束的*G*，该次调用为非阻塞调用，直接返回；
4. 调用runqsteal()从其他*P*的队列中“偷”。

&#160; &#160; &#160; &#160;如果以上四步都没能获取成功，就继续执行一些低优先级的工作：

5. 如果处于垃圾回收标记阶段，就进行垃圾回收的标记工作；
6. 再次调用globrunqget()从全局队列中取可执行的*G*；
7. 再次调用netpoll()取异步调用结束的*G*，该次调用为阻塞调用。

&#160; &#160; &#160; &#160;如果还没有获得*G*，就停止当前*M*的执行，返回findrunnable()函数开头重新执行。
如果findrunnable()正常返回一个*G*，shedule()函数会调用execute()函数执行该*G*。
execute()函数会调用gogo()函数（在汇编源文件asm_XXX.s中定义，XXX代表系统架构），gogo()
函数会从*G*.sched结构中恢复出*G*上次被调度器暂停时的寄存器现场（SP、PC等），然后继续执行。

##4.2 如何进行抢占?

&#160; &#160; &#160; &#160;runtime在程序启动时，会自动创建一个系统线程，运行sysmon()函数（在proc1.go中定义）。
sysmon()函数在整个程序生命周期中一直执行，负责监视各个Goroutine的状态、判断是否要进行垃圾回收等。

&#160; &#160; &#160; &#160;sysmon()会调用retake()函数，retake()函数会遍历所有的*P*，如果一个*P*处于执行状态，
且已经连续执行了较长时间，就会被抢占。retake()调用preemptone()将*P*的stackguard0设为stackPreempt(关于stackguard的详细内容，可以参考
[Split Stacks](https://gcc.gnu.org/wiki/SplitStacks))，这将导致该*P*中正在执行的*G*进行下一次函数调用时，
导致栈空间检查失败。进而触发morestack()（汇编代码，位于asm_XXX.s中）然后进行一连串的函数调用，主要的调用过程如下：

    morestack()（汇编代码）-> newstack() -> gopreempt_m() -> goschedImpl() -> schedule()
    
&#160; &#160; &#160; &#160;在goschedImpl()函数中，会通过调用dropg()将*G*与*M*解除绑定；再调用globrunqput()将*G*加入全局runnable队列中。最后调用schedule()
来用为当前*P*设置新的可执行的*G*。

&#160; &#160; &#160; &#160;关于Golang抢占式调度的进一步学习，可以参考
[Go Preemptive Scheduler Design Doc](https://docs.google.com/document/d/1ETuA2IOmnaQ4j81AtTGT40Y4_Jr6_IDASEKg0t0dBR8/edit)。
