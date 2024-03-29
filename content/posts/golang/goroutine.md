---
title: 'Go 调度器'
date: 2023-11-14T17:31:00+08:00
# weight: 1
# aliases: ["/alias"]
tags: ["golang","调度器","goroutine"]
author: "sweetpear"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
summary: "介绍了goroutine和线程的区别，包括创建销毁成本、内存占用和切换成本。讲述了Go程序的执行过程和调度器的结构及调度过程。"
# canonicalURL: "https://canonical.url/to/page"
---
# goroutine

## 参考
[操作系统 | 进程/线程切换问题](https://blog.csdn.net/weixin_47187147/article/details/124474016)

[Draveness 调度器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#m)

[Go 程序员面试笔试宝典 调度器](https://golang.design/go-questions/sched/what-is/)

[Go 特殊的goroutine](https://blog.haohtml.com/archives/22353/)

## 线程和进程和goroutine (进程->线程->goroutine)

### goroutine和线程区别
主要**三方面**：创建和销毁成本、内存占用、切换成本

|     | 进程 | 线程 | goroutine |
| --- | --- | --- |  --- |
| 创建和销毁 | 内核级（内存空间、文件描述符等） | 内核级 （优化：线程池） | 用户级 |
| 切换 | 切换页表刷新TLB（主要影响）、PCB切换， 慢 | 保存各种寄存器，较慢 | 只保存三个：PC，SP，BP，快|
| 内存占用 | 独立的虚拟内存 | 1MB+“guard page” | 初始2KB |

* 进程切换 = 指令序列切换 + 资源切换
* 线程切换 = 指令序列切换 + 共享资源

### 为什么线程轻量
同一个进程下，线程之间的共享内存空间（*内存占用*），所以创建和销毁时只需要关心自己独有的数据，还可以利用线程池进一步减小成本（*创建和销毁*），进行切换时不需要切换页表（*切换*）

### 为什么goroutine更轻量
同样共享进程内存空间，且栈空间会动态变化，可以减少内存浪费（*内存占用*），创建和销毁由 Go 运行时（ `Go runtime` ）负责管理，成本很小是用户级的。（*创建和销毁*）只需要保存和恢复 `Goroutine` 的上下文信息，而不需要切换整个线程的上下文（M：N模型），减少了频繁切换的内存开销（*切换*）

* 其实golang做的优化就是：原本每个任务都要对应一个线程，对于一个CPU来说遇到多任务并发执行的情况就要频繁切换线程（解决：等量M），并且需要从全局队列取任务，涉及到加解锁（解决：加P保存本地队列），造成资源的浪费。而 go 中只使用与机器 CPU 数量相等（默认）的线程，将相比线程更轻量的 `goroutine` 作为具体任务的执行载体，大部分情况下只需要通过处理器 P 和调度器在用户态进行调度，减少了线程切换带来的资源损耗。

## go程序执行
![go_execution](https://golang.design/go-questions/sched/assets/5.png)

* go program：用户程序
* runtime：`runtime` 是 Go 的运行时系统，它是一个在程序执行期间负责管理和支持 Go 语言特性的底层系统组件，用于支持 *Goroutine 的调度*、内存管理、垃圾回收、并发原语等关键功能。

Goroutine 的调度功能就是通过运行时中的 `scheduler` 实现的。

## 调度器 scheduler
在程序运行过程中，`runtime.schedt` 对象只有**一份**实体，它负责协调和管理 goroutine 的执行。

### 数据结构
`runtime.schedt` 结构体维护了**空闲** M 列表、**空闲** P 列表，全局 G 队列、dead G 缓存等等。
* 所有 M 所有 P 所有 G 保存在全局变量中（`allm`, `allp`, `allgs`）， 调度器中只是空闲的列表

### 初始化
1. 一些配置的初始化
2. 获取 `g0`（ g0 在运行时系统启动时创建）
3. 初始化 `m0`（初始化id以及gsignal等等）
4. 【加锁】 初始化 `GOMAXPROCS` 个 `P` 【解锁】

* 关于 `m0` 和 `g0`
![m0&g0](https://blogstatic.haohtml.com/uploads/2021/03/8e229b7806870bf4f17da207665b8a43.jpg)

## GMP
M -> P -> G
* M：Machine，内核线程，即传统意义上的线程
* P：Processor，处理器，运行在M上负责本地调度，**减缓了全局锁的调用频率**
* G：Goroutine，待执行任务

### G
只存在于 Go 语言的运行时，是**用户态**线程

#### 数据结构
`runtime.g` 结构体中包括使用的栈信息、当前绑定的M（可能为空）、抢占相关信息、调度相关信息等等

#### 状态
一个 `G` 最主要的生命周期可以概括为三个状态：
* 阻塞中  例如：正在执行系统调用 `_Gsyscall`，运行时阻塞（channel之类的）`_Gwaiting`，被抢占 `_Gpreempted`
* 可运行  例如：存储在运行队列中 `_Grunnable`
* 运行中  例如：被赋予了M和P `_Grunnable`

此外，还有 GC 相关状态、初始化状态（`_Gidle`、`_Gdead`）等。
将空闲的G置为 `_Gdead` 状态可以避免 GC 扫描未初始化的栈（不知道啥意思，总之是和避免GC干啥有关）

![G_status](https://golang.design/go-questions/sched/assets/15.png)

#### 初始化
1. 获取调用方 `G` 以及 `PC`；

2. 【锁住m0 避免抢占】用 `g0` 系统栈（`systemstack`）创建新的 `G`【解锁m0】（使用系统栈目的是减少栈帧的复制和管理开销，从而提高大型函数调用的性能）：
    1. 获取 `g0`（启动时创建的）、`p0`（[schedt初始化时创建](#调度器-scheduler)）
    2. 尝试从本地或者全局 dead G 列表中获取空闲的 Goroutine (`runtime.p.gFree`, `runtime.schedt.gFree`)，如果没有空闲的则创建一个新的 G 设置其状态为 dead 并加入全局 G 列表
    3. 初始化 G 的运行现场以及需要的参数

3. 放到运行队列中（本地p/全局schedt），根据传入的参数 `next`：
    * next 为 true：将 G 放到处理器的 `p.runnext` 上作为下一个执行的任务，如果 `p.runnext` 上原来有 G ，则把他踢到运行队列中；
    * next 为 false：如果本地队列有空间则放本地，否则，就会把本地队列中的一部分 Goroutine 和待加入的 Goroutine 添加到调度器持有的全局运行队列上；

4. 检查空闲的 P，如果有，将其唤醒（提高调度和执行效率）


### M
操作系统线程，调度器最多可以创建 `10000` 个线程，但是最多只会有当前机器内核数 `GOMAXPROCS` 个活跃线程能够正常**运行**。
[运行时启动时不会一下创建内核数个M](https://stackoverflow.com/a/39246575)

#### 数据结构
`runtime.m` 结构体中包括了调度线程 `g0` ，当前运行的goroutine `curg` 以及一些处理器相关字段、状态信息、调度相关信息等等 

#### 状态
* 两种：自旋和非自旋
自旋的时候，会努力找工作（检查全局队列，查看 network poller，试图执行 gc 任务，或者“偷”工作）；找不到的时候会进入非自旋状态，之后会休眠，直到有工作需要处理时，被其他工作线程唤醒，又进入自旋状态。

![spinning](https://golang.design/go-questions/sched/assets/17.png)

#### 初始化
创建时机：运行时系统启动时（创建m0、创建sysmon线程...）、调度M去运行P时（没有空闲M则创建新的）

`runtime.newm` 为创建新的M的顶层函数，执行了分配内存、初始化部分变量、调用底层函数新建操作系统线程等。

由汇编语言调用 `runtime.mstart` 启动线程，初始化PC、SP和signal等运行现场，进入调度循环。

### P
P 只是处理器的**抽象**，而非处理器本身。负责调度本地队列中的goroutine。

#### 数据结构
`runtime.p` 结构体反向存储着线程 `m` ，本地G队列（环形链表）、线程缓存（`mcache`）以及一些调试、GC 辅助的字段

#### 状态
一共有五个状态：
* _Pidle：P 处于空闲
* _Prunning：被 M 持有，正在执行
* _Psyscall：G 陷入系统调用
* _Pgcstop：被 M 持有，由于 GC 被停止
* _Pdead：由于 `GOMAXPROCS shrank`，当前 P 不再被使用

![processor_state](https://golang.design/go-questions/sched/assets/16.png)

#### 初始化
时机：运行时启动时，内核数变化时（很少见）
`runtime.procresize` 负责p的初始化，调用时进入 `world stopped` 状态；
1. 根据新的内核数扩容或缩减 `allp` 全局p列表
    * 扩容：申请新的P对象、做一些初始化操作
    * 缩减：释放被缩减的P所持有的资源，并置其为 `_Pdead` 状态
2. 判断当前G使用的P是否应该被释放（P在被缩减的范围内），如果是，则将当前P更换为 `allp[0]`；
3. 遍历全局P列表
    * 如果本地队列为空，则置为 `_Pidle` 状态，放入 `schedt` 调度器的空闲P列表
    * 如果本地队列不为空，则绑定一个M，加入可运行的P链表
4. 返回可运行的P链表

## 调度
### 触发调度的条件
根据 `runtime.schedule` 函数的调用方，可分为以下几类：
* 线程启动 `runtime.mstart` 和 Goroutine 执行结束时
* 内存同步访问时（mutex，channel等阻塞时），主动挂起
* 系统调用时，G 对应的 M 和 P 也阻塞在系统调用，并不会立刻发生抢占，只有当这个阻塞持续时间过长（10 ms）时，才会将 P（及之上的其他 G）抢占并分配到空闲的 M 上
* 协作式调度，主动让出P，发生在G时间片耗尽
* 系统监控、GC等

### 调度循环
调度函数 `runtime.schedule` 是一个永不返回的函数，是由每个系统线程M的g0调用的，不断**寻找**待运行的G并**执行**，在G结束后会**返回**调度函数开头开始下一轮循环；

#### 如何寻找可运行的goroutine（序号为寻找优先级）
1. 为保证公平，每61次调度将**首先从调度器（`runtime.sched`）全局队列中获取G**；取**1**个直接返回。
2. **从p本地队列获取**；取runnext，不行再取队首。
3. **从全局队列获取**；偷取的G数量最多为本地队列容量的**一半**，最少为全局队列中的G被每个P平分到的数量。
4. **从netpoll获取**；
5. **从其他P偷取**；
6. 如果找不到，则将m切换至**休眠**状态，在切换期间还会一步三回头地再确认一遍确实没有可运行的G了；

* 步骤1和3都是从全局队列获取，但是步骤3会获取多个G，返回全局队列的队首G，剩下的放到P的本地队列中。

#### 怎么偷
* `偷取` 特指从其他P中获取G。

总共尝试四次，每次都会以随机的顺序遍历所有P，直到偷到。

前三次会选择**不在**空闲状态的P（从空闲的P肯定获取不到G），如果队列中有G的话，会将**一半**的G偷到自己的本地队列里来；

如果前三次都没偷到，最后一次会先看runnext位置上有没有G，再看本地队列，有就偷**一个**，还可能窃取处理器的计时器；


#### 怎么执行
偷到G以后，调用 `runtime.execute` 让G在M上跑起来

1. 绑定M和G，将G置为 `_Grunning` 状态
2. 通过 `runtime.gogo` 将G调度到当前线程上，是由汇编指令实现的
3. G执行完成后，转换G状态为 `_Gdead`，解绑G/M，将G加入gFree列表，切换回g0开始新一轮的调度

### 总结一下
触发调度会将当前G切换回g0，由g0进行调度操作，待找到待运行的G后，将G切换到当前M上

## TODO
信号处理机制，协作与抢占