# MapReduce论文阅读笔记

## 摘要

### 1、MapReduce目的

处理大型数据

### 2、概述

map函数 -> 处理键值对，生成中间键值对

reduce函数 -> 合并同一中间键值对关联中间值

## 一、介绍

输入数据通常很大，计算需分布在上百、上千机器上，存在如何对计算并行化、如何分配数据、如何处理故障的问题。

由map、reduce语言启发，发现多数计算涉及对输入进行映射运算，计算出中间键值对，对共享同一键值对的所有值应用reduce运算，组合导出数据。用于简化大型计算，重新执行作为主要的容错机制。

## 二、编程模型

Map：获取输入对，生成中间键值对，将同一键相关的中间值分组放在一起，传给Reduce

Reduce函数：接受键值和该键值的一组值，将这些值合并在一起，形成一组更小的值，通常只生成0/1个输出值。可以处理太大的数据。

### 2.1 举例说明

伪代码：

```c++
map(stirng key, string value){
	//传入key:文件名
	//传入value:文件文本
	for(auto w: value){//对每个单词word在文件文本value
		mid(w,"1")//生成中间键值对:(w,"1") -> 每个单词+出现次数
	}
}

reduce(string key, iterator values){
    //传入key:一个单词
    //传入values:一个计数列表
    int res = 0;
    for(auto v: values){
        res+=atoi(v);
        return string(res);
    }
}
```

流程图：

![image-20230426102338333](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230426102338333.png)

### 2.2 类型

map ( k1, v1 ) -> list ( k2, v2 ) 

reduce ( k2, list ( v2 ) ) -> list ( v2 )

### 2.3 更多示例

## 三、接口实现

### 3.1 执行概述

![image-20230426110559418](C:\Users\菲菲\AppData\Roaming\Typora\typora-user-images\image-20230426110559418.png)

（1）、输入文件被MapReduce库拆分split为M个片段，每个片段16~64兆字节。【最左侧Input files】用户程序fork多个副本，其中一个特殊：Master，其余为worker【上方fork】

（2）、Master为空闲的worker分配任务：Map/Reduce【assign map/reduce】

（3）、负责map的worker读取已被拆分的内容，解析键值对，调用自定义Map函数，生成的键值对此时在**内存**中进行缓冲【read】

（4）、内存缓存中间键值对定期写入本地磁盘，进行分区，分区个数由Reduce数量决定，写入磁盘位置告知Master，Master转发给worker【local write】

（5）、负责reduce的worker收到通知后，远程调用读取，按中间键值对进行排序【remote read】

（6）、负责reduce的worker迭代排序后数据，调用Reduce函数，结果写入对应output文件【write】，output文件会写入GFS

（7）、所有Map、Reduce完成，Master通知用户程序

### 3.2 Master数据结构

对于每个Map和Reduce，状态有：idle空闲、in-process进行中、complete完成，对于非空闲状态又有标识。

### 3.3 容错

#### 3.3.1 Worker failure

1、Master每隔一段时间ping Worker【心跳检测】

2、Worker失联，标记为failed

3、Worker失效后：已完成map task标记idle，未完成reduce task无改变【map结果写在 local disk，reduce 储存在GFS】。map task A失效，交给 map task B，通知所有reduce task从B读数据。

#### 3.3.2 Master failure

周期备份Master数据checkpoints，Master宕机回滚最后一个checkpoints

### 3.6 落后Task

落后Task：集群中执行缓慢的任务，决定总体的速度

当MapReduce操作接近完成时，Master会对未完成In-process的余下任务启动backup执行，当primary或backup有一个执行完成，即被标记完成

## 四、模型改进

### 4.3 Combiner函数

中间键可能会显著重复，如单词计数会遵循Zipf分布(齐夫定律)，将产生数百或数千条<the,1>记录，需要用combiner进行压缩，一般情况下combiner使用代码与reducer相同。

## 五、性能表现

## 六、使用经验

## 七、相关工作

## 八、结论



