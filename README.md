### 简介

一个简单的http系统，用于提供一种通过异步执行业务，提高服务器对于高频请求的承受能力的方法。

由于是个简单的demo，故一些运行环境信息及参数写死在代码里了

运行在本地8099端口

有  /aaa  和   /bbb  两个接口，通过POST方法传入消息。两个接口的业务都是time.sleep 传入的时间的秒数 之后返回。

其中  /aaa 是同步的   /bbb 是异步的

消息体为 

```
{
	"sleep_time" : 1  //此数字可以更改，类型为int64
}
```

### 注意

在dispatcher.go文件中，将dispatch方法更新为newdispatch方法。其改动主要在于：

之前的dispatch方法，每当缓冲队列中有一个任务，就会拿出来，然后开一个goroutine，阻塞等待worker的到来。这样的话缓冲队列没有起到缓冲的作用，本应在缓冲队列中的任务，以阻塞的goroutine的形式等待在了内存中。

而newdispatch方法，是每当有空闲的worker，就阻塞等待一个缓冲队列的任务。假如暂时没有worker空闲，则任务会在缓冲队列里等待。避免了上述问题。

### 关于两个初始参数

这个调度系统有两个重要参数。MaxWorker(最大工人数)，也就是最大并发处理任务的数量；MaxQueue(最大队列长度)，也就是缓冲队列的容量。关于这两个数值怎样设定呢？

根据实际情况，下面做了一些简单计算


假设一个request源，平均每秒发n个请求，每个请求在服务端处理时间为s秒。服务端有一个长度为L的缓冲队列，同时有w个worker并行处理缓冲队列里的请求。每个worker执行一个请求，执行完再从缓冲队列里拿新的请求来处理。求一个s、n、w、L的关系等式，使得缓冲队列永远不会满

这个问题其实又可以分为两个问题：

1、在网络状况稳定的情况下，请求发送速度、处理速度相对稳定，要保证队列不会溢出，也就是单位周期内，处理效率要大于请求效率

2、在网络状况出现抖动等不稳定的情况，要保证L作为缓冲队列，能够容纳这部分多出来的请求，之后在网络状况回复正常后，可以逐渐消化掉缓冲的任务

对于1、

```
设周期为t，则在该周期中：
n·t·s < w·t   （周期内所有生成的待处理时间  <  消费者在该周期内的工作时间）
故有 n·s < w 

eg. 每秒请求为100，每个请求处理时间平均2秒，则，消费者数量大于 2*100 = 200 时，处理能力大于生产能力

```

对于2、

这里把网络突发状况带来的处理能力下降近似反映为请求生成数量的增高，把请求方用乘法扩大m倍，消费方用加法引入L相关的一个因式，大约能算出来L的和m相关的值

```
设网络不稳定时，请求能力为原来的m倍，则有：
m·n·s < w + s·L  (请求量扩大后，需要小于worker的工作能力加上 缓冲队列L所带来的缓冲能力)
故有  L > （m·n·s - w）/ s

```

### 基于上面初始参数的一点应用

上面提到了两个初始参数之间的一些关系，下面增加一点使用上的讨论。

在上面的场景中，我们默认请求发送源是固定的，而处理请求的服务端服务能力是限制条件极小的，故而重点讨论缓冲队列和worker数量的关系。但是实际场景中，服务端的能力根据业务场景、服务器能力等是有限制的。在我应用到这个结构的一个例子里，服务程序会出现报错类似```too many open files```的错误。大约是系统的文件描述符到达上限，导致新的连接请求无法被重新建立，也就是我们遇到了服务资源瓶颈。

在这种情况下，服务端可以采取横向扩展多服务做负载均衡，而如果保持服务端不变，就需要对请求端做一个限流。

刚好这边的请求是同步的，也就是收到服务端的返回后才会发下一个请求。所以它其实是基于高频单线程的发送形成的伪并发场景，这是这部分讨论的一个基点，它使得我们可以通过控制请求响应速度来控制请求发送速度。而我们知道这份代码的处理方式，是将发来的请求放入工作缓冲队列然后立刻返回，之后异步对请求内容进行服务。

基于这样的前提，我们先设定worker的数量。在这个例子里，worker的作用主要是在业务逻辑较大的情况下，把串行高频到来的请求并行地处理执行。我们假设找到一个服务端并行执行业务保持系统能够承受的上限数目为n，将worker设置为低于这个n的值（低于是为了避免网络环境或系统环境抖动带来的波动影响而提供一些缓冲余地）。也就是无论是怎样频次的请求到来速度，都会在这里被整理以n的并发请求服务，这样保证了服务端和系统资源的安全。多余的请求缓冲到我们的jobqueue工作队列里，当工作队列满，则阻塞请求的返回。此时，请求发送源会发现发出请求的响应速度变慢，由于是同步单线程发送，则相当于放缓了请求发送的速度。

总的来说，从请求到处理，首先是一个
		同步->异步->同步的过程。（同步发送请求->异步处理事件->每个worker同步完成一个任务）
	同时也是一个
		串行->并行->串行的过程。（请求轮流到达并在缓冲队列排队->多worker同时执行多个任务->每个worker专心处理一个任务）

这两种变化的阶段相重叠，实现了一种可控的灵活性。

PS. 对于处理大量请求的需求，可以把多个服务系统放到多个服务器上，然后通过负载均衡的原理提升业务处理能力，而这里的代码，大约可以作为一个简单调度器的原型了。


