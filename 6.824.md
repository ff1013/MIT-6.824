[TOC]



# 6.824

## Lecture 1 Introdution

### 1. Why 

- parallelism 并行处理增加容量

- faults tolerance 容错

- physical 天然物理隔离

- security/isolated 安全性

### 2. Challenges

- concurrence 并发性
- partial failure 局部失效
- performance 实际性能

### 3. Lab

- Lab 1 - MapReduce 分布式大数据框架
- Lab 2 - Raft - fault tolercance
- Lab 3 K/V server
- Lab 4 sharded K/V service

### 4. Infrastructure - Abstractions 基础架构抽象化

- storage 
- communication
- computation
- a big goal：hide the complexity of distribution 隐藏分发复杂性

### 5. Implementation tool 实现工具

- RPC
- threads
- locks

### 6. Performance

- scalability 可扩展性 ------> 2x computers -> 2x throughtput

### 7. Fault Tolerance

- Availability
- Recoverability
- NV storage 非易失性
- Replication 副本

### 8.  Consistancy 一致性

- stong system
- weak system

### 9. MapReduce

![image-20230423164423111](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230423164423111.png)

![image-20230427164852508](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230427164852508.png)

GFS google file systerm

具体见论文

这里的emit对应原论文的存到map对应磁盘里面

shuffle：是将行存储变为列存储，MapReduce代价最大的一部分

#### Q&A

Q1：能不能连续执行多个MapReduce

A1：可以，将前一个MapReduce的output作为下一个MR的input，MapReduce pipeline是一种很常见的做法

Q2：框架重要还是MapReduce函数重要

A2：对于我们来说关心环境框架如何组织这些的，对于程序员来说，不需要考虑程序员分布式知识，只需要考虑map reduce function

Q3：what is emit()

A3：（1）Where the job runs? worker

​		（2）map emit：map on worker machine 将output写到本地local disk，MR将map的output告知reduce，reduce自动拉取

​		（3）reduce emit：output写到GFS

​		（4）GFS自动备份&split file into chunks

Q4：->是什么

A4：->是网络通信，MR去某个网络地址获取正确的file

Q5：为什么不一部分机器只当GFS，一部分只当MR

A5：因为网络太慢，2004MR瓶颈在network

​		一台机器既是worker又是GFS，避免从网络中取数据，也就是就近读取

​		有网络瓶颈时，master会根据距离调整worker，从最近disk调取数据，实际上是将map task assign给含有该份数据的machine（理论是移动数据，实际上是移动计算）

随着网络速度瓶颈的解决，MR优势开始消失，MR操作是批处理，现实中更多是流处理streaming

Q6：是否可以通过Steaming的方式加速Reduce的读取

A6：较复杂，涉及现在的一些东西

## Lecture 2 RPC and Threads

<img src="C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230427173715267.png" alt="image-20230427173715267" style="zoom:30%;" />

每个线程都有它自己的栈，严格来说都在同一地址空间中。

### 1.Why choose threads

- I/O concurrency I/O并发

- Parallelism 并行 多核处理

- Convenience 方便 想要在后台周期性执行某些功能

Q：didn't want to use threads：asynchronous program异步编程

A：asynchronous program/ event-driven programming事件驱动编程

single thread of control that keeps state about many different activites

### 2.Thread challenges

- race 竞态 ----> use mutex使用锁解决竞态

Q：individual instructions are atomic？汇编指令是原子的吗

A：some are and some aren't不一定

Q：how goes go know which variable we're locking？

A：go has no idea. lock只关心加锁释放，对于之间对哪些变量进行操作并不关心，由程序员关注

Q：is it better to have the lock be the private bussiness of the data structure? 锁私有是不是更好

A：reasonble：the user , the data structure may never know 

​	potential problem：隐藏了锁的细节可能会出现1. the programmer knew that data was never shared but pay the lock overhead锁开销，有些地方不会出现race，但是对其上锁产生开销 2.dead lock死锁，隐藏细节不易恢复

- coordination 进程间通信协作

​	生产者-消费者模型

​	一些go相关内置结构：

​		1.channel

​		2.sync.Cond 条件变量

​		3.WaitGrop

- dead lock

### 3.crawl example 爬虫

### 4.RPC

-- 2021网课版部分 --

#### 1.概述

RPC：remote procedure calls -->远程过程调用

RPC ~ PC ：类本地在栈上调用

Client		server

z=fn(x,y)	fn(x,y){return x+y}

客户端发送x，y，服务器返回i计算结果

![image-20230430134114829](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230430134114829.png)

#### 2.go实现

- 声明结构：putArgs、putReply、GetArgs、GetReply

- client：

  func get() 作用：连接到客户端   --- Call()类上图的stub

  ```go
  func get(key string) string {
      clint := connect()
      args := GetReply{"subject"}
      reply := GetReply{}
      err := client.Call("KV.Get", &args, &reply)//传入要调用的方法，参数，响应
      //Call内部会发送序列化的参数，通过连接发送到服务器，等待回复
      if err != nil{
          log.Fatal("error:", err)
      }
      client.Close()
      return reply.Value
  }
  ```

  func put() 类get()

- server：

​	结构：KV struct

​	func server() 作用：分配一个新的服务器对象  ---  rpcs.Register(kv)注册所有在RPC服务器上实现在KV结构上的方法

​	go使用大写字母来表示公共方法，小写表示私有方法，rpcs.Register()只导出大写的方法

```go
func server() {
    kv := new(KV)
    kv.data = map[string]string{}
    rpcs := rpc.NewServer()
    rpcs.Register(kv) //注册方法
    l, e := net.Listen("tcp", "1234")//监听listen
    if e !=nil {
        log.Fatal("listen error:", e)
    }
    go func() {
        for {
            conn, err := l.Accpet()//接收
            if err == nil {
                go rpcs.ServerConn(conn)//ServerConn提供TCP连接
            } else {
                break
            }
        }
        l.close()//关闭
    }()
}
```

​	func Get() 作用：客户端调用Get，服务器实现Get

```go
func (kv *KV) Get(args *GetArgs, reply *GetReply) error {
	kv.mu.Lock()//首先调用锁，因为可能多个客户端会调用服务器，多个goroutine会调用Get和Put
    defer kv.mu.Unlock()
    
    val, ok := kv.data[args.Key] //Get函数查看映射中的键，返回值
    if ok {
        reply.Err = OK
        reply.Value = val
    } else { //如果映射中无条目
        reply.Err = ErrNokey
        reply.Value = ""
    }
    return nil
}
```

#### 3.PRC semantics under failures RPC失败语义

- at-least-once 服务器发生故障，客户端将自动重试并继续 

​	downside缺点：同一操作会被执行多次

- at-most-once 请求执行零次或一次，不超过一次  ----Go RPC

​	实现方式：过滤重复

- exactly-once 正常执行只有一次 hard --- lab3

## Lecture 3 GFS

### 1.Big Storage 大型存储

- Why is hard
  - Performance -> sharding 分片
  - Faults -> tolerance 容错
  - Tolerance -> replication 复制
  - Replication -> inconsistency不一致
  - Consistency -> low performance



















