title: Fuzzy Control - 模糊控制
category: Control Theory
author: cshuo
date: 2017-02-26
tags: Fuzzy

---
> 模糊逻辑广泛应用于在机器控制领域。“模糊”指的是这些逻辑可以解决单纯使用“True”或“False”不能解决的问题，比如”Partially True“。尽管其它的很多方法比如遗传算法或者神经网络在很多情境下都可以代替模糊控制，但模糊控制也有它不可代替的优点，比如它针对问题的解决方案很直观，易理解，模糊逻辑的设计往往是基于系统操作人员的经验。

<!-- more -->

模糊控制器概念上非常简单，它由输入阶段，处理阶段和输出阶段组成。输入阶段是将输入的数据(传感器、开关等)映射到合适的隶属函数以及对应的真值(true value) -- 模糊化；处理阶段设计到调用相关的模糊规则，产生各自的结果，然后利用特定方法将结果进行综合；最后输出阶段将综合结果反模糊化，得到一个具体的输出值。

### 模糊化
模糊控制器的输入值通常被映射到若干个隶属函数，称作”模糊集“，该过程被称作“模糊化”。
```
IF brake temperature IS warm AND speed IS not very fast
    THEN brake pressure IS slightly decreased.
```
比如这里“brake temperature”和"speed"都有自己对应的模糊集，如{cold, warm, hot}。输出值"brake pressure"也有自己模糊集{static, slightly increased, ...}。最常用的隶属函数是三角形隶属函数。

### 处理阶段
将输入值映射到对应的隶属函数以及真值之后，模糊控制器将基于模糊规则推理，判断采取什么动作。模糊规则是一些列的 IF-THEN语句，如：
```
IF (temperature is "cold") THEN (heater is "high")
```
IF部分被称作"antecedent"，THEN部分是"consequent"。如上例规则，它根据temperature的输入值映射到"cold"的真值，得到heater设置为”high“对应的真值。显然，"temperature"为cold的真值越大，heater设置为”high“的真值也越大。 通常一个模糊控制系统包含很多个模糊规则。在得到输入之后，往往若干个规则会被触发，得到若干个输出值和相应真值，下一步即使用特定方法进行综合，常用的是质心法：对于模糊控制的输出O，由对应的模糊集{r1, r2, r3, ..., rn},通过规则推理得到结果及对应的真值分别是&lt;o1, mu(r1)&gt;, &lt;o2, mu(r2)&gt; ... &lt;on, mu(rn)&gt;，使用质心法获取综合结果方式如下：

$$O = \frac{\sum\_{i=1}^n o\_i \times mu(r\_i)}{\sum\_{i=1}^nmu(r\_i)}$$

### 反模糊化
反模糊化是模糊化的逆过程。

如上所示的模糊规则给人一种疑惑，看起来好像不用模糊逻辑一样也可以达到目的，当然对于简单的系统确实可以不用模糊逻辑，但是通常复杂的系统的决策控制是基于一些列的模糊规则：
* 所有相关的规则都被调用，使用输入值的隶属函数和对应的真值来决定规则的输出值。
* 输出值然后同样被映射到一个隶属函数和真值来控制输出结果。
* 规则的输出结果被综合程一个值，即反模糊化。


### 模糊控制实例
考虑构建一个简单的反馈控制器，输入是error和delta, 输出值是delta, 即error的变化率。

![](https://raw.githubusercontent.com/cshuo/bpic/master/fuzzy.JPG)

输入值error和输出值对应的模糊集如下：
```
  LP:  large positive
  SP:  small positive
  ZE:  zero
  SN:  small negative
  LN:  large negative
```
输入值error，和delta对应的隶属函数如下：
```
__________________________________________________________________________
             -1    -0.75    -0.5   -0.25    0     0.25   0.5   0.75     1
__________________________________________________________________________

  mu(LP)      0      0        0      0      0      0     0.3    0.7     1
  mu(SP)      0      0        0      0     0.3    0.7     1     0.7    0.3
  mu(ZE)      0      0       0.3    0.7     1     0.7    0.3     0      0
  mu(SN)     0.3    0.7       1     0.7    0.3     0      0      0      0
  mu(LN)      1     0.7      0.3     0      0      0      0      0      0
__________________________________________________________________________
```

输出值的反模糊化隶属函数如下：
```
         LN           SN           ZE           SP           LP
__________________________________________________________________________
-1.0  |  XXXXXXXXXX   XXX          :            :            :           |
-0.75 |  XXXXXXX      XXXXXXX      :            :            :           |
-0.5  |  XXX          XXXXXXXXXX   XXX          :            :           |
-0.25 |  :            XXXXXXX      XXXXXXX      :            :           |
 0.0  |  :            XXX          XXXXXXXXXX   XXX          :           |
 0.25 |  :            :            XXXXXXX      XXXXXXX      :           |
 0.5  |  :            :            XXX          XXXXXXXXXX   XXX         |
 0.75 |  :            :            :            XXXXXXX      XXXXXXX     |
 1.0  |  :            :            :            XXX          XXXXXXXXXX  |
__________________________________________________________________________
```
模糊规则如下：
```
 rule 1:  IF e = ZE AND delta = ZE THEN output = ZE
 rule 2:  IF e = ZE AND delta = SP THEN output = SN
 rule 3:  IF e = SN AND delta = SN THEN output = LP
 rule 4:  IF e = LP OR  delta = LP THEN output = LN
```
反模糊化使用的是离散质心法：
```
 SUM( I = 1 TO 4 OF ( mu(I) * output(I) ) ) / SUM( I = 1 TO 4 OF mu(I) )
```
现在考虑当前得到的输入是：
```
  e     = 0.25
  delta = 0.5
```
我们可以得到如下的模糊化：
```
  ________________________
           e     delta
  ________________________

  mu(LP)      0      0.3
  mu(SP)     0.7      1
  mu(ZE)     0.7     0.3
  mu(SN)      0       0
  mu(LN)      0       0
  ________________________
```
将之应用到规则1可以得到：
```
rule 1: IF e = ZE AND delta = ZE THEN output = ZE
mu(1) = MIN( 0.7, 0.3 ) = 0.3
output(1) = 0
```
这里，mu(1)是规则1结果隶属函数的真值，如果使用质心法，就是规则1结果的权重。
output(1)是在输出值模糊集上结果隶属函数最大的值(ZE)。若使用质心法，对于单个规则的结果来说，就是质心的位置，如表2所示。该值独立于其真值。</br>
同样对于其它的规则，由如下结果：
```
  rule 2:  IF e = ZE AND delta = SP THEN output = SN
     mu(2)     = MIN( 0.7, 1 ) = 0.7   
     output(2) = -0.5

  rule 3: IF e = SN AND delta = SN THEN output = LP
     mu(3)     = MIN( 0.0, 0.0 ) = 0
     output(3) = 1

  rule 4: IF e = LP OR  delta = LP THEN output = LN
     mu(4)     = MAX( 0.0, 0.3 ) = 0.3
     output(4) = -1
```

使用质心法得到最终结果是：
$$\frac{mu(1)\times output(1)+mu(2)\times output(2)+mu(3)\times output(3)+mu(4)\times output(4)}{mu(1)+mu(2)+mu(3)+mu(4)}
=-0.5$$
根据反模糊化隶属度表，可以得出最终的输出结果是SN。
