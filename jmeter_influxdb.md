title: 使用InfluxDB && Grafana 对Jmeter实时数据持久化及展示
category: monitoring
author: cshuo
date: 2017-02-26
tags: monitoring,load testing

---
>Jmeter是常用的压测工具，使用Jmeter可以对Web应用进行性能测试，常用的性能指标有相应时间(RT)和吞吐率(TP)；Jmeter性能数据的持久化可以通过添加Backend listener来完成，但是这种方式会在InfluxDB里创建相当多的我并不care的measurements，且我所关心的metric并没有直接创建，所以就有了本文直接修改Jmeter的Apache core library的方法。

<!-- more -->

### 主要思路
使用Jmeter的Non-GUI模式运行一个测试，对应的输出如下所示：
![](https://raw.githubusercontent.com/cshuo/bpic/master/amqp.png)
我们想要持久化的就是图中那些metric:
* Avg, min, max Rt
* TP
* Total requests proessed
* Failed requests

思路就是对$jmeter/lib/ext/ApacheJmeter_core.jar$ 进行修改，每当它将结果输出到控制台时，同时也让它把数据发送到InfluxDB数据库；这样我们就不需要创建单独的Listener来获取对应的性能数据，也省掉了上述metric的计算过程。

### Jar包及Grafana模板配置
[TAG-Influx-Grafana.zip](http://www.testautomationguru.com/download/640/)
* ApacheJMeter_core.jar: 两个版本对应Jmeter的V2.13和V3.0，选取正确的版本替换lib/ext/里的官方jar包；此外需要对jemter的user.properties进行修改，见一下配置。
* 两个Grafana的 dashboard json 分别对应压测的sample数据和summary数据, 从Grafana的Dashboard面板导入。

```
#True to send data to influx db
summariser.influx.out.enabled=true
 
# influxdb server ip
summariser.influx.ip=10.11.12.13
 
# influxdb port to post the data via http
summariser.influx.port=8086
 
# influxdb database
summariser.influx.db=jmeter
 
# name of the project. might be useful when you use same DB for multiple projects.
# it is stored as tag for faster query
summariser.influx.project=taguru
 
# name of the test suite. might be useful when you run the same test for different conditions.
# it is stored as tag for faster query
summariser.influx.project.suite=taguru-100-users
 
# timeouts
summariser.influx.connection.timeout=5000
summariser.influx.socket.timeout=5000
```

注意：
* 使用了修改后的jar包和以上配置，不需要再添加backend listener；
* 默认不给InfluxDB设置用户名和密码；
* 修改只在Jmeter的Non-GUI模式下可用。
<br>
使用以上jar包和配置运行压测脚本后，可以在InfluxDB看到一下三个measurements:
* delta
* total
* samples
![](https://raw.githubusercontent.com/cshuo/bpic/master/jmeter_measurements.png)

### Grafana 配置
* 配置Data source指向安装的InfluxDB数据库;
* 从Grafana的dashboard导入上面下载的配置文件(json格式)；
配置正确的话可以看到如下的Dashboard:
![](https://raw.githubusercontent.com/cshuo/bpic/master/grafana.png)
