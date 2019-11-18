
# 1 并发、协程、信道

Go 通过协程实现并发，协程之间靠信道通信

## 1.1 并发、并行是什么？
- *并行*其实很好理解，就是同时执行的意思，在某一时间点能够执行多个任务。
想达到并行效果，最简单的方式就是借助多线程或多进程，这样才可在同一时刻执行多个任务。单线程是永远无法达到并行状态的。例，"合作并行开发某个项目"
- *并发*是在某一时间段内可以同时处理多个任务。我们通常会说程序是并发设计的，也就是说它允许多个任务同时执行，这个同时指的就是一段时间内。单线程中多个任务以间隔执行实现并发。例，我边听歌边写代码。

> 并行是指两个或者多个事件在同一时刻发生；而并发是指两个或多个事件在同一时间间隔内发生。

[多线程或多进程是并行的基础，但单线程也通过协程实现了并发。 ]()

## 1.2 go协程是什么？
Go 协程是与其他函数或方法一起并发运行的函数或方法。Go 协程可以看作是轻量级线程。与线程相比，创建一个 Go 协程的成本很小。
因此在 Go 应用中，常常会看到有数以千计的 Go 协程并发地运行。
Go 创建一个协程非常简单，只要在方法或函数调用之前加关键字 go 即可。

```go
// 匿名协程
go func(){
}
```

## 1.3 Go 协程相比于线程的优势
- 相比线程而言，Go 协程的成本极低。堆栈大小只有若干 kb，并且可以根据应用的需求进行增减。而线程必须指定堆栈的大小，其堆栈是固定不变的。
- Go 协程会复用（Multiplex）数量更少的 OS 线程。即使程序有数以千计的 Go 协程，也可能只有一个线程。如果该线程中的某一 Go 协程发生了阻塞（比如说等待用户输入），那么系统会再创建一个 OS 线程，并把其余 Go 协程都移动到这个新的 OS 线程。所有这一切都在运行时进行，作为程序员，我们没有直接面临这些复杂的细节，而是有一个简洁的 API 来处理并发。
- Go 协程使用信道（Channel）来进行通信。信道用于防止多个协程访问共享内存时发生竞态条件（Race Condition）。信道可以看作是 Go 协程之间通信的管道。

## 1.4 信道
### 1.4.1 声明信道
```go
var c chan int          // 方式一，为nil，不能发送也不能接受数据
c := make(chan int)    // 方式二，可用
```
[底层实现参考简书](https://segmentfault.com/a/1190000020286676?utm_source=tag-newest) 是一个带缓冲，包含两个双向链表，分别是接受，发送消息。
## 1.4.2 信道使用[无缓冲信道]
```go
c := make(chan int) // 写数据
c <- data

// 读数据
variable <- c  // 方式一
<- c              // 方式二,读出来的数据丢弃不使用
```

> 信道操作默认是阻塞的，往信道里写数据之后当前协程便阻塞，直到其他协程将数据读出。
一个协程被信道操作阻塞后，Go 调度器会去调用其他可用的协程，这样程序就不会一直阻塞。

```go
func printHello(c chan bool) {
     fmt.Println("hello world goroutine")
     <- c    // 读取信道的数据
 }

 func main() {
     c := make(chan bool)
     go printHello(c)
     c <- true    // main 协程阻塞
    fmt.Println("main goroutine")
```

输出
```shell
hello world goroutine
main goroutine
```

### 1.4.3 死锁
前面提到过，读/写数据的时候信道会阻塞，调度器会去调度其他可用的协程。问题来了，如果没有其他可用的协程会发生什么情况？没错，就会发生著名的死锁。
最简单的情况就是，只往信道写数据。只读不写也会报同样的错误。
> fatal error: all goroutines are asleep - deadlock!

### 1.4.4 关闭信道与 for loop
> val, ok := <- channel

> val 是接收的值，ok 标识信道是否关闭。为 true 的话，该信道还可以进行读写操作；为 false 则标识信道关闭，数据不能传输。
使用内置函数 close() 关闭信道。

使用 for range 读取信道，信道关闭，for range 自动退出。
**使用 for range 一个信道，发送完毕之后必须 close() 信道，不然发生死锁。**

```go
func printNums(ch chan int) {
     for i := 0; i < 4; i++ {
         ch <- i
     }
     close(ch)
 }

 func main() {
     ch := make(chan int)
    go printNums(ch)

    for v := range ch {
        fmt.Println(v)
    }
}
```

输出
```bash
1   0
2   1
3   2
4   3
```
### 1.4.5 缓冲信道和信道容量
```go
func main() {
     ch := make(chan int,3)

     ch <- 7
     ch <- 8
     ch <- 9
     //ch <- 10
     // 注释打开的话，协程阻塞，发生死锁
     会发生死锁：信道已满且没有其他可用信道读取数据

    fmt.Println("main stopped")
}
```

### 1.4.6 单向信道
之前创建的都是双向信道，既能发送数据也能接收数据。我们还可以创建单向信道，只发送或者只接收数据。
语法：

rch 是只发送信道，sch 是只接受信道。

```go
rch := make(<-chan int)
sch := make(chan<- int)
```

> 这种单向信道有什么用呢？我们总不能只发不接或只接不发吧。这种信道主要用在信道作为参数传递的时候，Go 提供了自动转化，双向转单向

### 1.4.7 注意
[链接参见](https://www.jianshu.com/p/d7218db63f6d)
#### 读写nil通道都会死锁

```go
// 读nil通道:

var dataStream chan interface{}

<-dataStream

// 写nil通道:

var dataStream chan interface{}

dataStream <- struct{}{}

都会死锁:
```

> fatal error: all goroutines are asleep - deadlock!



#### close 值为nil的channel会panic:

```go
var dataStream chan interface{}

close(dataStream)

```

> panic: close of nil channel

### 1.4.8 Select

select 可以安全的将channels与诸如取消、超时、等待和默认值之类的概念结合在一起。

select看起来就像switch 包含了很多case,然而与switch不同的是：select块中的case语句没有顺序地进行测试，如果没有满足任何条件，执行不会自动失败,如果同时满足多个条件随机执行一个case。

类似witch
1、case分支中必须是一个IO操作：

2、当case分支不满足监听条件，阻塞当前case分支

3、如果同时有多个case分支满足，select随机选定一个执行（select底层实现，case对应一个Goroutine），

4、一次select监听，只能执行一个case分支，未执行的分支将被丢弃。通常将select放于for循环中。

5、default在所有case均不满足时，默认执行的分组，为了防止忙轮询，通常将for中select中的default省略。

**【结论】**使用select的Goroutine，与其他Goroutine间才用异步通信。


# 2.通信
# 2.1 线程间通信
线程间的通信目的主要是用于*线程同步*。
> 线程同步：即当有一个线程在对内存进行操作时，其他线程都不可以对这个内存地址进行操作，直到该线程完成操作， 其他线程才能对该内存地址进行操作，而其他线程又处于等待状态。
同步就是协同步调，按预定的先后次序进行运行，去同步这种状态。

# 2.1.1 线程同步的方式和机制:
- 临界区（Critical Section）、互斥对象（Mutex）：主要用于互斥控制；都具有拥有权的控制方法，只有拥有该对象的线程才能执行任务，所以拥有，执行完任务后一定要释放该对象。
- 信号量（Semaphore）、事件对象（Event）：主要用于同步控制；事件对象是以通知的方式进行控制

```shell
锁机制：包括互斥锁、条件变量、读写锁

    - 互斥锁提供了以排他方式防止数据结构被并发修改的方法。
    - 读写锁允许多个线程同时读共享数据，而对写操作是互斥的。
    - 条件变量可以以原子的方式阻塞进程，直到某个特定条件为真为止。对条件的测试是在互斥锁的保护下进行的。条件变量始终与互斥锁一起使用。

信号量机制(Semaphore)：包括无名线程信号量和命名线程信号量

信号机制(Signal)：类似进程间的信号处理
```

```shell
线程同步的方法：
(1)wait():使一个线程处于等待状态，并且释放所持有的对象的lock。

(2)sleep():使一个正在运行的线程处于睡眠状态，是一个静态方法，调用此方法要捕捉 InterruptedException异常。

(3)notify():唤醒一个处于等待状态的线程，注意的是在调用此方法的时候，并不能确切的 唤醒某一个等待状态的线程，而是由JVM确定唤醒哪个线程，而且不是按优先级。

(4)notityAll ():唤醒所有处入等待状态的线程，注意并不是给所有唤醒线程一个对象的锁， 而是让它们竞争
```

go的实现

```go
# 锁
1.在golang的官方文档上，作者明确指出，golang并不希望依靠共享内存的方式进行进程的协同操作。而是希望通过管道channel的方式进行。
# 当然，golang也提供了共享内存，锁，等机制进行协同操作的包。sync包就是为了这个目的而出现的。

# Mutex 和 RWMutex实现了接口Locker
# Mutex就是互斥锁，互斥锁代表着当数据被加锁了之后，除了加锁的程序，其他程序不能对数据进行读操作和写操作

这个当然能解决并发程序对资源的操作。但是，效率上是个问题。当加锁后，其他程序要读取操作数据，就只能进行等待了

# RWMutex 读写锁分为读锁和写锁，读数据的时候上读锁，写数据的时候上写锁。有写锁的时候，数据不可读不可写。有读锁的时候，数据可读，不可写

# 2.临时对象池
当多个goroutine都需要创建同一个对象的时候，如果goroutine过多，可能导致对象的创建数目剧增。 而对象又是占用内存的，进而导致的就是内存回收的GC压力徒增。造成“并发大－占用内存大－GC缓慢－处理并发能力降低－并发更大”这样的恶性循环。
sync.Pool的使用非常简单，提供两个方法:Get和Put 和一个初始化回调函数New。
看下面这个例子（取自gomemcache）：

// keyBufPool returns []byte buffers for use by PickServer's call to
// crc32.ChecksumIEEE to avoid allocations. (but doesn't avoid the
// copies, which at least are bounded in size and small)
var keyBufPool = sync.Pool{
    New: func() interface{} {
        b := make([]byte, 256)
        return &b
    },
}

func (ss *ServerList) PickServer(key string) (net.Addr, error) {
    ss.mu.RLock()
    defer ss.mu.RUnlock()
    if len(ss.addrs) == 0 {
        return nil, ErrNoServers
    }
    if len(ss.addrs) == 1 {
        return ss.addrs[0], nil
    }
    bufp := keyBufPool.Get().(*[]byte)
    n := copy(*bufp, key)
    cs := crc32.ChecksumIEEE((*bufp)[:n])
    keyBufPool.Put(bufp)

    return ss.addrs[cs%uint32(len(ss.addrs))], nil
}

sync.Pool的回收是有的，它是在系统自动GC的时候，触发pool.go中的poolCleanup函数。
sync.Pool其实不适合用来做持久保存的对象池（比如连接池）。它更适合用来做临时对象池，目的是为了降低GC的压力。
参见链接 https://docs.kilvn.com/The-Golang-Standard-Library-by-Example/chapter16/16.01.html

# sync.Once
# WaitGroup
一个goroutine需要等待一批goroutine执行完毕以后才继续执行，那么这种多线程等待的问题就可以使用WaitGroup了。
# Cond
sync.Cond是用来控制某个条件下，goroutine进入等待时期，等待信号到来，然后重新启动;
sync.Cond还有一个BroadCast方法，用来通知唤醒所有等待的gouroutine。

```




# 2.2 进程间通信(8种)
> 进程通信：每个进程各自有不同的用户地址空间,任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据必须通过内核,在内核中开辟一块缓冲区,

进程A把数据从用户空间拷到内核缓冲区,进程B再从内核缓冲区把数据读走,内核提供的这种机制称为进程间通信。

## 2.2.1匿名管道( pipe )：
管道是一种半双工的通信方式，数据只能单向流动，而且只能在具有亲缘关系的进程间使用。进程的亲缘关系通常是指父子进程关系。

通过匿名管道实现进程间通信的步骤如下：

```
// fd参数返回两个文件描述符
// fd[0]指向管道的读端，fd[1]指向管道的写端
// fd[1]的输出是fd[0]的输入。
```

- 父进程创建管道，得到两个⽂件描述符指向管道的两端
- 父进程fork出子进程，⼦进程也有两个⽂件描述符指向同⼀管道。
- 父进程关闭`fd[0]`,子进程关闭`fd[1]`，即⽗进程关闭管道读端,⼦进程关闭管道写端（因为管道只支持单向通信）。⽗进程可以往管道⾥写,⼦进程可以从管道⾥读,管道是⽤环形队列实现的,数据从写端流⼊从读端流出,这样就实现了进程间通信。

### 2.2.2 高级管道
高级管道(popen)：将另一个程序当做一个新的进程在当前程序进程中启动，则它算是当前程序的子进程，这种方式我们成为高级管道方式。

### 2.2.3 有名管道通信
有名管道 (named pipe) ： 有名管道也是半双工的通信方式，但是它允许无亲缘关系进程间的通信。

### 2.2.4 信号量通信
信号量( semophore ) ： 信号量是一个计数器，可以用来控制多个进程对共享资源的访问。

它常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。

### 2.2.5 消息队列通信
消息队列( message queue ) ： 消息队列是由消息的链表，存放在内核中并由消息队列标识符标识。

消息队列克服了信号传递信息少、管道只能承载无格式字节流以及缓冲区大小受限等缺点。

### 2.2.6 信号
信号 ( signal ) ： 信号是一种比较复杂的通信方式，用于通知接收进程某个事件已经发生。

### 2.2.7 共享内存通信
共享内存( shared memory ) ：共享内存就是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。

共享内存是最快的 IPC 方式，它是针对其他进程间通信方式运行效率低而专门设计的。它往往与其他通信机制，如信号量，配合使用，来实现进程间的同步和通信。

### 2.2.8 套接字通信
套接字( socket ) ： 与其他通信机制不同的是，它可用于不同机器间的进程通信
通信过程如下：
 - 命名socket
 - 绑定
 - 监听
 - 连接服务器
 - 相互发送接收数据
 - 断开连接


# 2. go 内存管理

## 2.1 内存分配中的堆和栈:

栈（操作系统）：由操作系统自动分配释放 ，存放函数的参数值，局部变量的值等。其操作方式类似于数据结构中的栈。

堆（操作系统）： 一般由程序员分配释放， 若程序员不释放，程序结束时可能由OS回收，分配方式倒是类似于链表。

## 2.2 堆栈缓存方式
栈使用的是一级缓存， 他们通常都是被调用时处于存储空间中，调用完毕立即释放。

堆则是存放在二级缓存中，生命周期由虚拟机的垃圾回收算法来决定（并不是一旦成为孤儿对象就能被回收）。所以调用这些对象的速度要相对来得低一些。

[Go 编译器自行决定变量分配在堆栈或堆上，以保证程序的正确性。]

# 3. golang CSP并发模型 (Communicating Sequential Processes)
go 用到
CSP通讯顺序同步机制：
当程序获取到cpu轮片后，执行完程序后，goroutine进入挂起态时并不会放弃cpu使用权，而是将cpu使用权交给其他goroutine。CPU内容执行时间轮片速度慢，效率低。而在Goroutine内部交换使用权能极大的提升切换效率。

[文章来源](https://www.jianshu.com/p/36e246c6153d)
> CSP模型是上个世纪七十年代提出的，用于描述两个独立的并发实体通过共享的通讯 channel(管道)进行通信的并发模型。 CSP中channel是第一类对象，它不关注发送消息的实体，而关注与发送消息时使用的channel。

## 3.1 Golang CSP
Golang 就是借用CSP模型的一些概念为之实现并发进行理论支持，其实从实际上出发，go语言并没有，完全实现了CSP模型的所有理论，仅仅是借用了 process和channel这两个概念。process是在go语言上的表现就是 goroutine 是实际并发执行的实体，每个实体之间是通过channel通讯来实现数据共享。

## 3.2 Channel
Golang中使用 CSP中 channel 这个概念。channel 是被单独创建并且可以在进程之间传递，它的通信模式类似于 boss-worker 模式的，一个实体通过将消息发送到channel 中，然后又监听这个 channel 的实体处理，两个实体之间是匿名的，这个就实现实体中间的解耦，其中 channel 是同步的一个消息被发送到 channel 中，最终是一定要被另外的实体消费掉的，在实现原理上其实是一个阻塞的消息队列。

## 3.3 Goroutine
Goroutine 是实际并发执行的实体，它底层是使用协程(coroutine)实现并发，coroutine是一种运行在用户态的用户线程，类似于 greenthread，go底层选择使用coroutine的出发点是因为，它具有以下特点：
- 用户空间 避免了内核态和用户态的切换导致的成本
- 可以由语言和框架层进行调度
- 更小的栈空间允许创建大量的实例
可以看到第二条 用户空间线程的调度不是由操作系统来完成的，像在java 1.3中使用的greenthread的是由JVM统一调度的(后java已经改为内核线程)，还有在ruby中的fiber(半协程) 是需要在重新中自己进行调度的，而goroutine是在golang层面提供了调度器，并且对网络IO库进行了封装，屏蔽了复杂的细节，对外提供统一的语法关键字支持，简化了并发程序编写的成本。

## 3.4 Goroutine 调度器
上节已经说了，golang使用goroutine做为最小的执行单位，但是这个执行单位还是在用户空间，实际上最后被处理器执行的还是内核中的线程，用户线程和内核线程的调度方法有：

M:N 用户线程和内核线程是多对多的对应关系
![avatar](https://upload-images.jianshu.io/upload_images/1767848-9c4b06362907280d.png?imageMogr2/auto-orient/strip|imageView2/2/w/350/format/webp)

![avatar](https://upload-images.jianshu.io/upload_images/1767848-fc23b15dc52e407f.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/307/format/webp)

图2中

- M：是内核线程, 准确说是runtime对操作系统内核线程的虚拟
- P : 是调度协调，用于协调M和G的执行，内核线程只有拿到了 P才能对goroutine继续调度执行，一般都是通过限定P的个数来控制golang的并发度
- G : 是待执行的goroutine，包含这个goroutine的栈空间
- Gn : 灰色背景的Gn 是已经挂起的goroutine，它们被添加到了执行队列中，然后需要等待网络IO的goroutine，当P通过 epoll查询到特定的fd的时候，会重新调度起对应的，正在挂起的goroutine。
- Golang为了调度的公平性，在调度器加入了steal working 算法 ，在一个P自己的执行队列，处理完之后，它会先到全局的执行队列中偷G进行处理，如果没有的话，再会到其他P的执行队列中抢G来进行处理。

总结
Golang实现了 CSP 并发模型做为并发基础，底层使用goroutine做为并发实体，goroutine非常轻量级可以创建几十万个实体。实体间通过 channel 继续匿名消息传递使之解耦，在语言层面实现了自动调度，这样屏蔽了很多内部细节，对外提供简单的语法关键字，大大简化了并发编程的思维转换和管理线程的复杂性。

```shell
首先GPM是golang runtime里面的东西，是语言层面的实现。也就是说go实现了自己的调度系统。 理解了这一点 再往下看
M（machine）是runtime对操作系统内核线程的虚拟， M与内核线程一般是一一映射的关系， 一个groutine最终是要放到M上执行的；
P管理着一组Goroutine队列，P里面一般会存当前goroutine运行的上下文环境（函数指针，堆栈地址及地址边界），P会对自己管理的goroutine队列做一些调度（比如把占用CPU时间较长的goroutine暂停 运行后续的goroutine等等。。）当自己的队列消耗完了 会去全局队列里取， 如果全局队列里也消费完了 会去其他P对立里取。
G 很好理解，就是个goroutine的，里面除了存放本goroutine信息外 还有与所在P的绑定等信息。

GPM协同工作 组成了runtime的调度器。

P与M一般也是一一对应的。他们关系是： P管理着一组G挂载在M上运行。当一个G长久阻塞在一个M上时，runtime会新建一个M，阻塞G所在的P会把其他的G 挂载在新建的M上。当旧的G阻塞完成或者认为其已经死掉时 回收旧的M。

P的个数是通过runtime.GOMAXPROCS设定的，现在一般不用自己手动设，默认物理线程数（比如我的6核12线程， 值会是12）。 在并发量大的时候会增加一些P和M，但不会太多，切换太频繁的话得不偿失。内核线程的数量一般大于12这个值， 不要错误的认为M与物理线程对应，M是与内核线程对应的。 如果服务器没有其他服务的话，M才近似的与物理线程一一对应。

说了这么多。初步了解了go的调度，我想大致也明白了， 单从线程调度讲，go比起其他语言的优势在哪里了？
go的线程模型是M：N的。 其一大特点是goroutine的调度是在用户态下完成的， 不涉及内核态与用户态之间的频繁切换，包括内存的分配与释放，都是在用户态维护着一块大的内存池， 不直接调用系统的malloc函数（除非内存池需要改变）。 另一方面充分利用了多核的硬件资源，近似的把若干goroutine均分在物理线程上， 再加上本身goroutine的超轻量，以上种种保证了go调度方面的性能。
```


# 4.Interface
## 4.1 什么是接口

> 在一些面向对象的编程语言中，例如 Java、PHP 等，接口定义了对象的行为，只指定了对象应该做什么。行为的具体实现取决于对象。

> 在 Go 语言中，接口是一组方法的集合，但不包含方法的实现、是抽象的，接口中也不能包含变量。当一个类型 T 提供了接口中所有方法的定义时，就说 T 实现了接口。接口指定类型应该有哪些方法，类型自己决定如何去实现这些方法。

## 4.2 实现接口类的类型和值
### 4.2.1 静态类型和动态类型
> 变量的类型在声明时指定、且不能改变，称为静态类型。接口类型的静态类型就是接口本身。接口没有静态值，它指向的是动态值。接口类型的变量存的是实现接口的类型的值。该值就是接口的动态值，实现接口的类型就是接口的动态类型。

```go
type Iname interface {
    Mname()
}

type St1 struct {}
func (St1) Mname() {}
type St2 struct {}
func (St2) Mname() {}

func main() {
    var i Iname = St1{}
    fmt.Printf("type is %T\n",i)
    fmt.Printf("value is %v\n",i)
    i = St2{}
    fmt.Printf("type is %T\n",i)
    fmt.Printf("value is %v\n",i)
}
```
变量 i 的静态类型是 Iname，是不能改变的。动态类型却是不固定的，第一次分配之后，i 的动态类型是 St1，第二次分配之后，i 的动态类型是 St2，动态值都是空结构体。

输出：
```shell
type is main.St1
value is {}
type is main.St2
value is {}
# 变量 i 的静态类型是 Iname，是不能改变的。动态类型却是不固定的，第一次分配之后，i 的动态类型是 St1，第二次分配之后，i 的动态类型是 St2，动态值都是空结构体。

# 有时候，接口的动态类型又称为具体类型，当我们访问接口类型的时候，返回的是底层动态值的类型。
```

### 4.2.2 nil 接口值
当且仅当动态值和动态类型都为 nil 时，接口类型值才为 nil。

```go
type Iname interface {
    Mname()
}
type St struct {}
func (St) Mname() {}
func main() {
    var t *St
    if t == nil {
        fmt.Println("t is nil")
    } else {
        fmt.Println("t is not nil")
    }
    var i Iname = t
    fmt.Printf("%T\n", i)
    if i == nil {
        fmt.Println("i is nil")
    } else {
        fmt.Println("i is not nil")
    }
    fmt.Printf("i is nil pointer:%v",i == (*St)(nil))
}
//输出：

t is nil
*main.St
i is not nil
i is nil pointer:true
```
类型是 (*St)(nil) 而非nil

# 4.2.3 实现接口
```go
type Shape interface {
    Area() float32
}

type Rect struct {
    width  float32
    height float32
}

func (r Rect) Area() float32 {
    return r.width * r.height
}

func main() {
    var s Shape
    s = Rect{5.0, 4.0}
    r := Rect{5.0, 4.0}
    fmt.Printf("type of s is %T\n", s)
    fmt.Printf("value of s is %v\n", s)
    fmt.Println("area of rectange s", s.Area())
    fmt.Println("s == r is", s == r)
}
// 输出：

type of s is main.Rect
value of s is {5 4}
area of rectange s 20
s == r is true
```
