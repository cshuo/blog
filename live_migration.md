title: VM 热迁移详解
date: 2016-09-10 22:53:00
category: VM
tags: vm

---
> 虚拟机热迁移是虚拟化技术中的一个研究热点, 所谓迁移就是讲源主机上的操作系统连同所有应用程序移动到目的主机上, 并且保证系统和应用程序的状态在迁移前后的一致性. 一般各个虚拟化工具都有自己的迁移组件, 如VMware的Vmotion, Xen的XenMotion和Microsoft的Hyper-V. 虚拟机的迁移可以分为两大类: 离线迁移(Offline Migration)和热迁移(Live Migration). 离线迁移是指在迁移之前, 将虚拟机暂停, 拷贝其状态到目的主机,在目的主机主机重新建立运行状态, 恢复执行. 这种迁移技术适用于对服务可用性要求不高的场合. 本文更多的关注的是热迁移, 即在保证虚拟机中服务正常运行的同时, 将其从一个物理主机拷贝到另一个物理主机. 整个迁移过程对用户是透明的, 即用户感觉不到虚拟机位置的变化. 热迁移中重点是内存拷贝技术, 本文将详细介绍几种流行的内存拷贝技术, 以及一些可行的优化措施.

<!--more-->

---

## 内存迁移技术
虚拟机热迁移主要目的是保证服务不中断的同时, 将一个虚拟机在物理机之间进行移动. 对于应用程序状态来说, 主要涉及到其CPU状态和内存中的运行状态. CPU状态的拷贝几乎可以瞬间完成, 而内存的拷贝通常是一个相对比较长的持续过程, 拷贝过程中, 已经拷贝的内存可能会被重写. 目前流行的内存拷贝技术预拷贝(Pre-Copy), 延迟拷贝(Post-Copy)和混合拷贝技术.
### 预拷贝
预拷贝策略是使用较多的策略, 迁移开始后, 在将内存状态拷贝到目的节点的同时, 源物理主机上的虚拟机仍在继续运行. 如果一个内存也在拷贝之后再次被修改了(dirtied), 那么那将会在下一轮拷贝中发送到目的节点. 整个内存拷贝是个迭代的过程. 迭代的终止条件有多种可能: 1) 剩余的内存脏页数比较少; 2) 迭代次数达到上限; 3) 内存脏页数产生速率大于拷贝速率.
下面结合图文详解拷贝过程:
- Init Copy: 将虚拟机的全部内存页拷贝到目的主机上, 虚拟机管理器继续监视虚拟机的内存变化;
![](https://raw.githubusercontent.com/cshuo/bpic/master/live_mrt1.png)
- Iterative Pre-Copy: 每次迭代中, 将被修改的内存页拷贝到目的主机上;
![](https://raw.githubusercontent.com/cshuo/bpic/master/live_mrt2.png)
- Stop-and-Copy: 将源物理主机上的虚拟机暂停, CPU状态和剩余的内存脏页拷贝到目的主机上, 在目的主机上启动虚拟机.
![](https://raw.githubusercontent.com/cshuo/bpic/master/live_mrt3.png)

### 延迟拷贝
与预拷贝技术不同, 延迟拷贝首先把CPU状态复制到目的主机上,在目的主机上开启虚拟机, 然后源主机主动将内存页推送到目的主机上的虚拟机中. 与此同时, 目的主机上的虚拟机运行时可能会访问到不存在页面(还未被推送), 虚拟机将请求源主机立刻将这些页面发送过来. 可以看出在延迟拷贝策略中, 每个内存页面最多只被传送一次, 降低了预拷贝中冗余拷贝的开销.

延迟拷贝策略的具体过程如下:
- 将源物理主机上的虚拟机停止
- 拷贝虚拟机的处理器状态至目标节点
- 在目标节点上恢复启动虚拟机
- 通过网络从源节点上获取内存页

### 混合拷贝技术
根据以上介绍, 我们知道Pre-Copy适用于内存read-intensive的虚拟机热迁移, 对于write-intensive的虚拟机, 在Pre-Copy的每轮迭代中都会有大量新的内存页被修改, 导致迁移的效率低下. 对于Post-Copy则相反, 对于read-intensive的虚拟机, Post-Copy将会产生大量的缺页错, 导致虚拟机内应用性能的降低. Post-Copy更适用于内存比较大, 内存write-intensive的虚拟机热迁移.

为了是热迁移更具有一般性, 通用性, 一种自然的想法是将Pre-Copy和Post-Copy结合起来.具体过程如下:
- 采用Pre-Copy的初次迭代, 将虚拟机的全部内存页拷贝到目的主机, 与此同时, 虚拟机在源主机上继续运行;
- 一轮拷贝结束后, 虚拟机被暂停, 其处理器状态和不可分页内存被拷贝到目的主机;
- 在目的主机上恢复虚拟机, 开始使用Post-Copy策略继续拷贝内存页面.


### 存在的问题及一些优化措施
1. 问题: 一些内存页更新的速度很快, 每次迭代都被更新, 从而每次都被重新拷贝, 造成不必要的开销.
优化: 一种比较朴素的方法是只传输那些在上轮迭代中更新,本轮迭代中没更新的内存页. 这样可以在一定程度上减少每轮迭代中拷贝的内存页数.
2. 问题: 预拷贝策略一般会持续一定时间, 在这段时间内, 必然会占用网络带宽资源. 可能会对对网络比较敏感的服务造成影响. 另一方面, 如果将内存拷贝的带宽限制到比较低的水平话, 将会导致最终Stop-and-Copy的停机时间(Downtime)比较长.
优化: 采用动态迁移带宽限制的方法, 每次迭代中, 统计需要被拷贝的内存页数, 除以上轮迭代持续的时间, 得到一个内存页被更新的速度(Dirtying rate), 然后通过给这个速度增加一个常量来确定下一轮内存页拷贝的带宽限制. 系统管理员可以设置一个内存迁移的带宽上限, 当Dirtying rate大于这个上限或者在一轮迭代中剩余的脏页数小于一个特定值, 此时就可以结束迭代, 进入Stop-and-Copy阶段.
3. 问题: 延迟拷贝策略中, 目标物理机上的虚拟机在获取处理状态后就被启动, 内存访问会导致缺页错(Page fault), 然后这些页面会被请求从源主机上通过网络发送过来, 这是一个被动请求的过程(Demand Paging). 单纯的使用Demand Paging会导致整个迁移过程比较缓慢, Page fault也会对应用的性能造成一定影响.
优化: 在Demand Paging的同时, 从源物理主机主动推送内存页到目标节点(Active pushing). 这些主动推送的内存页不包括由于缺页错而被被动要求传送的页面. 通过这样的方式, 既保证了每个内存页只被拷贝了一次, 也加快了迁移的速度. Active pushing过程中最简单的方式是按序推送或随机推送, 显然这种方式不能很好的反映内存访问模式. 一种可行的方法是根据之前缺页错的地址, 将其附近的内存页进行拷贝(内存访问一般呈现区域密集性), 这样可以提高主动推送的内存页近期被访问的概率, 从而有效减少缺页错.
4. 问题: 预拷贝和延迟拷贝都存在一个问题, 就是在这两种情况下, 迁移过程中都会拷贝空页(Free page), 延长了迁移时间, 造成了不必要的开销.
优化: 使用Ballooning技术, 动态的释放虚拟机的空页到虚拟机管理器(Hypervisor). [Ballooning](http://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/techpaper/perf-vsphere-memory_management.pdf)是动态管理虚拟机内存分配的一种技术, 通常需要在guest kernel上安装一个Balloon驱动. 在一些内存页对虚拟机没用的时候, Balloon驱动可以将其回收, 返还给Hypervisor (气球膨胀); 当内存不够时, 也可以向Hypervisor请求额外的内存页(气球收缩). 通过动态的Ballooning, 空白内存可以有效减少Pre-Copy或Post-Copy过程中需要传递的内存页数.


### 参考资料
- [虚拟机技术漫谈](http://www.ibm.com/developerworks/cn/linux/l-cn-mgrtvm1/)
- [Live Migration of Virtual Machines](http://dl.acm.org/citation.cfm?id=1251223)
- [Post-Copy Live Migration of Virtual Machines](http://dl.acm.org/citation.cfm?id=1618528)
- [Understanding Memory Resource Management in VMware® ESX™ Server](http://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/techpaper/perf-vsphere-memory_management.pdf)
