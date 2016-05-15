title: RRD浅析    
date: 2015-11-23 19:44:04    
category: 监控  
tags: RRD
---
> 本文是对开源监控工具Ganglia使用的RRD数据库的一个简单介绍，此外还有一些有关RRDTool的基本操作。
<!--more-->

## RRD数据库
RRD是Round Robin Database的缩写，是一种环形数据库，它是专门设计来存储时序数据的。每当有新的数据到来时，一般也会有一个时间戳伴随着被存储起来，这里的时间戳是epoch值。RRDTool是RRD数据库配套的一个工具，它可以安装在Unix或者Windows系统上，RRDTool提供了一系列的命令集来对RRD进行不同的操作。    

RRD数据库与其他数据库的不同:      

+ RRDTool既是后端工具，又是前端工具；之所以这么说，是因为作为数据库，首先RRDTool可以存储数据，此外，RRDTool也提供了一个根据数据库数据进行绘图的功能，这让它具备了前端的特性。    
![](/img/rrd_gra.jpg)
+ 对于线性数据库，一般新的数据都被插入到数据库表的最后，所以数据库的大小随着数据的插入而不断的增长；而对于RRD数据库，其大小在创建时就已经指定。简单的说，可以把RRD数据库想象成一个环，数据都被插入到环的周边，有一个指针时钟指向新的数据要插入到的位置，当指针达到起始点时，就会覆盖原来存在的数据，这样一来，数据库的大小就固定不变了，这也是“Round Robin”的由来。     
+ 其他数据库的数据都是被提供的，而RRD数据库可以通过配置，来计算旧的数据到新的数据之间的变化，并把这些信息存储起来。      
+ 其他数据库更新的驱动是新的数据被提供，而RRD数据库的更新是按照按照预先定义的时间间隔来进行的；如果在固定的事件间隔内没有获得新的数据，那么RRDTool将会存储一个UNKNOWN值，所以一般使用RRD数据库都会有一个脚本每隔一段时间就向RRD提供一个数据。   

### RRD的基本结构
因为RRD数据库设计目的是监控，所以它的结构比较简单，创建一个RRD数据库的方式如下：

```
rrdtool create target.rrd \     
         --start 1023654125 \     
         --step 300 \      
         DS:mem:GAUGE:600:0:671744 \     
         RRA:AVERAGE:0.5:12:24 \     
         RRA:AVERAGE:0.5:288:31     
```


    
下面是相关的一些基本概念：    

* DS: Data Source, 它指的是设备上被监控的变量，一个RRD数据库中可能有多个DS。其声明格式如下：
> DS:variable_name:DST:heartbeat:min:max      

	每隔一个step, DS的一个新的值就会到来，然后数据库就会被更新，这个值也被叫做	PDP(Primary Data Point)，PDP的范围由min,max来制定，如果在heartbeat时间内没有	收到数据，那么该PDP就会被置为UNKNOWN。在上面的例子中，每隔300s就会产生一个新的	PDP。

* DST: Data Source Type, 指的是DS的类型，可以是COUNTER, DERIVE, ABSOLUTE, GAUGE，详细解释如下表：      
![](/img/rrd_dst.png)
* CF: Consolidation Function, 是指合并数据的方式，可选的有AVERAGE, MAX, MIN, LAST。
* RRA: Round Robin Archive, 是对采集到的数据以某种方式(CF)的归档。声明格式如下
> RRA:CF:xff:step:rows    

	在这里又有一个新的概念CDP(consolidated data point), 一个CPD是把step个PDP按照指定的CF进行合并得到的一个数值，而这个RRA中包含rows个CDPs。这里的xff是指CDP合并时允许出现的UNKNOWN的最大比率，在本例中即是，如果step个PDP中有一半是UNKNOWN，那么该CDP是UKNOWN,否则就去不是UKNOWN值的平均值。如下图是RRA的结构以及数据更新的方式：    
![](/img/rra.png)    
如图所示，在RRD数据库中会为一个RRA分配一部分空间，n(step)个sample(PDP)合并成一个CDP然后存储到这块区域的开头位置，如果空间已满则旧的数据会被覆盖，这样的方式就保证了数据库的大小是不会增长的，同时将PDP合并成CDP的做法，又可以保证RRD可以存储很长一段时间的数据。

了解了这些基本概念之后，上面的这个例子就比较容易理解了,首先给这个数据库命名为target.rrd，数据的开始时间是epoch时间1023654125，每隔300s获取一个PDP，然后DS制定了实际被监控的变量及其类型和值域，最后定义了两个RRA，对于第一个RRA, 12(steps)个PDPs以平均(CF)的方式进行合并得到一个CDP，24个这样的CDP组成一个RRA归档。PDP之间的间隔是300s,所以CDP之间的间隔就是12*300，即1小时，24个这样的CDP就是1天，因此我们可以得知，第一个RRA就是mem的一天的监控统计。

## RRDTool常用命令
### rrdtool dump 

>rrdtool dump filename.rrd [filename.xml] [--header|-h {none,xsd,dtd}] [--no-header|-n] [--daemon|-d address] [> filename.xml]

含义：将一个rrd数据库导出为xml文件     
示例：     
```
rrdtool dump load_one.rrd test.xml
```

### rrdtool fetch 
>rrdtool fetch filename CF [--resolution|-r resolution] [--start|-s start] [--end|-e end] [--align-start|-a] [--daemon|-d address]

含义：从一个rrd数据库根据条件取出数据     
示例：    
```
rrdtool fetch load_one.rrd AVERAGE -r 12 -s e-1200 -e now
```

### rrdtool xport
>rrdtool xport [-s|--start seconds] [-e|--end seconds] [-m|--maxrows rows] [--step value] [--json] [--enumds] [--daemon|-d address] [DEF:vname=rrd:ds-name:CF] [CDEF:vname=rpn-expression] [XPORT:vname[:legend]] 

含义：可以从若干个RRD中得到XML或JSON格式的数据。
示例：
```
rrdtool xport --start end-1h --end now --step 10 
DEF:ds1=load_one.rrd:sum:AVERAGE DEF:ds2=load_one.rrd:num:AVERAGE XPORT:ds1:sum XPORT:ds2:num
```

### rrdtool graph
>rrdtool graph|graphv filename [option ...] [data definition ...] [data calculation ...] [variable definition ...] [graph element ...] [print element ...]

含义：根据RRD数据库中的数据绘制图形，生成图片。
示例：
```
rrdtool graph new.png --end now --start end-150 --title cpu_user --vertical-label % --width 250 --height 150 DEF:ds=cpu_user.rrd:sum:AVERAGE:step=3 AREA:ds#0000FF:cpu_user
```
生成的图片：    
![](/img/cpu_user.png)







  





