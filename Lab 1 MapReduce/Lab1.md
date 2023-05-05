[TOC]



# Lab1：MapReduce

## 一、实验要求解读

### 1、实验概述

构建MapReduce系统，实现一个worker进程，该进程调用Map和Reduce，处理读写文件，以及实现一个master进程，分发任务给某一个worker进程，处理失败的进程，参考论文内容

### 2、配置环境

go 1.13版本

Linux系统

### 3、入门

1、拉取初始实验环境

```shell
$git clone git://g.csail.mit.edu/6.824-golabs-2022 6.824
$cd 6.824
$ls
Makefile src
```

mrsequential.go：顺序实现mapreduce【mr-mapreduce,sequential连续】

wc.go：word-count

indexer.go：文本索引

输入：pg-xxx.txx

输出：保留在mr-out-0中

```shell
$cd 6.824
$cd src/main
#将wc.go编译成插件形式,生成wc.so
$go build -race -buildmode=plugin ../mrapps/wc.go 
$rm mr-out*
#进行并发检测，将编译后生成的wc.so插件，以参数形式加mrsequential.go运行
$go run -race mrsequential.go wc.so pg*.txt
# 查看生成的文件
$more mr-out-0
```

### 4、实现

MapReduce：一个 **master** , 一个或多个 **worker**

worker ， master之间使用RPC通信

**worker**：向master请求一个任务，从一个或多个文件进行读取输入，输出写入一个或多个文件

**master**：需要注意worker是否在10秒内完成任务，如果没有将同一任务交给另一个worker

 main程序位于main/mrmaster.go和main/mrwork.go（运行代码）不要更改，实验要编写的代码实现在mr/master.go，mr/worker.go，mr/rpc.go中。

运行代码方法：

```shell
#确保单词计数插件是全新构建的
$go build -race -buildmode=plugin ../mrapps/wc.go 
$rm mr-out*
#pg-*.txt参数是输入文件，每个文件对应一个拆分，是一个Map的任务的输入
$go run mrmaster.go pg-*.txt
#在一个或多个窗口中，运行一些工作程序
$go run mrworker.go wc.so
#当master、worker完成后，查看mr-out- *中的输出，完成后，输出文件的序列排序和前文git下来的顺序输出mrsequential.go匹配
$cat mr-out- * | sort | more
```

测试：

测试脚本：main/test-mr.sh，检查输出是否正确，是否并行Map和Reduce任务，是否从崩溃程序中恢复

```shell
$cd main
$bash test-mr.sh
```

![image-20230429194838673](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230429194838673.png)

![image-20230429194852854](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230429194852854.png)

### 5、一些规则

- worker实现将第X个reduce任务输出文件mr-out-X

- 一个mr-out-X文件每个reduce输出应包含一行，以Go“%v %v"格式生成，并使用键值进行调用，查看main/mrsequential.go中的示例

  ![image-20230429195542093](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230429195542093.png)

- mr/coordinator.go实现Done()方法，在MapReduce作业完全完成时返回true，届时main/mrcoordinator.go将退出
- 全部完成后，工作进程应退出，一种简单的实现方法是使用`call()`的返回值：如果worker程序无法与master服务器联系，可以假定master由于作业完成而退出，worker也终止。master有给worker的”请退出“伪任务会很有用。

### 6、一些提示

- 一种入门方法：修改mr/worker.go的`Worker()`：发送`RPC`给master来请求任务，修改master：响应未启动map任务的文件名，修改worker读取该文件并调用map任务，参考`mrsequential.go`。
- mr/目录下更改，记得重构所有MapReduce插件，即：

```
$go build -race -buildmode=plugin ../mrapps/wc.go 
```

- 中间文件命名约定：`mr-X-Y`，X是Map任务号，Y是reduce任务号
- 存储中间键值对方法：Go的`encoding/json`包

![image-20230429201317715](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230429201317715.png)

- worker的map可以使用`ihash(key)`函数来为给定的key选择reduce任务
- 参考`mrsequential.go`：读取map输入、中间键值对排序、reduce输出
- 并发要锁定共享数据
- reduce需等到最后一个map执行完成才能开始。一种实现是worker定时向master寻求任务，`time.Sleep()`等待，请求之间需要睡眠。另一种实现是master服务器相关RPC处理程序具有一个循环，等待时间为`time.Sleep()`或`sync.Cond`
- mster无法区分崩溃worker、活着但无法工作的worker、速度太慢而无法工作的worker。最好的方案是master等待一段时间后放弃，重新发布任务给其他worker，本实验等待10秒，10秒后假定worker已经死亡
- 测试崩溃恢复用：`mrapps/crash.go`，它在map、reduce中随机退出
- 确保崩溃时未写完的文件不被读入，MapReduce论文提到了使用临时文件并在完全写入后自动对其重命名的技巧：使用`ioutil.TempFile`创建临时文件，`os.Rename`原子地对其重命名
- `test-mr.sh`运行子目录中`mr-tmp`中所有进程，若出错并想查看中间文件和输出文件可以在这里查看

### 7、提交

- 提交前，最后一次运行test-mr.sh
- `make lab1`命令打包

## 二、自行实现lab

### debug记录：

1、测试前要记得重新运行插件！

2、出现cannot load plugin wc.so![image-20230504000952863](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504000952863.png)

原因：插件编译wc.so的参数需要和启动参数一致，否则无法正常加载，我在编译插件时bulid语句：

```
go build -race -buildmode=plugin ../mrapps/wc.go
```

而编译worker时启动插件语句为：

```
go run mrworker.go wc.go
```

解决方法：

把编译worker的语句变为：

```
go run -race mrworker.go wc.go
```

3、编写map后测试一直处于循环中，未能结束退出：

![image-20230504004910505](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504004910505.png)

![image-20230504001646686](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504001646686.png)

解决思路：通过fmt打印步骤信息，调试出

<img src="C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504005047516.png" alt="image-20230504005047516" style="zoom:67%;" />

发现Map任务：1，调用的callDone中是successfel[0]，说明没有正常退出Map任务1

检查callDone()代码，发现自己在调用callDone()时未传入task参数，导致每次callDone都是退出Map任务0，修改代码：

```
callDone(&task)
```

![image-20230504091421925](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504091421925.png)

![image-20230504091344659](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504091344659.png)



### Test记录：

编写Map部分，尚未编写Reduce部分测试：

coordinator：

![image-20230504010305733](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504010305733.png)

worker：

![image-20230504010351474](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504010351474.png)

查看生成的中间文件，符合mr-X-Y的格式：

![image-20230504010641246](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504010641246.png)

编写完毕Map部分和Reduce部分：

![image-20230504021539877](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504021539877.png)

![image-20230504021841041](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504021841041.png)

![image-20230504021919802](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504021919802.png)













![image-20230504092251572](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504092251572.png)

![image-20230504092326858](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504092326858.png)

![image-20230504092349229](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504092349229.png)

![image-20230504092411687](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504092411687.png)

![image-20230504092437629](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504092437629.png)



![image-20230504092942849](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504092942849.png)

![image-20230504093048331](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504093048331.png)



![image-20230504093647690](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504093647690.png)

![image-20230504093838139](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504093838139.png)

![image-20230504093905005](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504093905005.png)

![image-20230504093949204](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504093949204.png)

![image-20230504094026464](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504094026464.png)

![image-20230504094050554](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504094050554.png)

![image-20230504094140835](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504094140835.png)

![image-20230504095100474](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504095100474.png)

![image-20230504095120030](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504095120030.png)

![image-20230504095250447](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504095250447.png)

![image-20230504095515131](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230504095515131.png)
