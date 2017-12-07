---
layout: post
title: nodeJs直连Java服务化dubbo协议长连接实现
description: "使用node直接采用dubbo协议长连接java服务化"
tags: [Nodejs, dubbo, 长连接, node-zoodubbo]
image:
background: hangzhou-1.png
comments: true
share: true
---

## 需求背景

这个需求的出现来源于我们的某业务接口优化中遇到的一个瓶颈：

初期接口在调用php提供的第三方接口时，遇到访问高峰的情况会出现该接口timeout的情况(如下图 平常时接口时延75ms，高峰时到达368ms )，同时服务器上会出现及大量的TIME_WAIT连接数(高峰时 2W左右)。<!--more-->

![时延表现][1]

分析是由于大流量时, 每次接口调用都会调用一次rest api，继而产生大量的tcp短连接导致的。这种不稳定的因素对我们的集群产生了极大的压力，同时接口时延也不稳定的上升。当务之急是减少短连接数，取而代之使用长连接的方式去请求。

同时我也了解到，异构系统间采用http长连接的做法在公司内部php集群并不支持(现阶段公司内部只有少数业务有成熟的长连接方案)，而又了解到java服务化方可以提供dubbo协议的接口请求。

因此就决定尝试使用dubbo协议跨过php这一层直接和java服务化做直连，这样既可以减少短连接数，同时也将调用链路变短了一层。

## nodeJs直连dubbo的做法

调用dubbo服务的过程主要分以下几步：

首先需要连接上dubbo的zk配置中心(从java服务化那边得到地址)，获取你需要的服务的dubbo节点连接信息（包括dubbo服务的host:port,服务名等等）

然后 使用sockets-pool(我们维护的)库 与获得的dubbo节点建立起起socket长连接，并维护在一个连接池队列中
在需要查询对应服务时，从连接池中获取可用连接，执行查询，然后释放该次连接到连接池。

demo代码如下：

```js
        var ZD = require('node-zoodubbo');
        var zd = new ZD({
                // config the addresses of zookeeper
                conn: '127.0.0.1:2181',
                dubbo: '2.8.0.2',
                root: 'dubbo'
        });

        // connect to zookeeperzd.connect();
        //zd.getProvider('com.member.query', '1.0.0', function(res, data){//      console.log(res, data);//});// get a invoker with a service path
        var invoker = zd.getInvoker('com.member.query', {
                version: '1.0.0',
                timeout: 2000,
                pool: {
                        max: 100,
                        min: 10,
                        maxWaitingClients: 500       
                }   
        });

        // excute a method with parametersvar method = 'memberInfo';
        var arg1 = {
                $class: 'com.member.memberRequestTO',
                $: {
                        uid: {
                                $class: 'java.lang.String',
                                $: '34625433'
                        }
                }
        };

        invoker.excute(method, [arg1])
        .then(res => console.log(res))
        .catch(e => console.log(e));
```

## node-zoodubbo 连接包实现

![流程][4]

查看[包地址](https://www.npmjs.com/package/node-zoodubbo)

关于整个包及连接实现[点击这里](http://7xq7m2.com1.z0.glb.clouddn.com/%5B%E9%BB%84%E5%A4%A9%E7%AB%8B%5Dnodejs%E7%9B%B4%E8%BF%9Ejava%E6%9C%8D%E5%8A%A1%E5%8C%96%E6%96%B9%E6%A1%88%20-%20%E5%89%AF%E6%9C%AC.pptx)

## 带来的效果

采用restapi方式调用黑名单接口时，该接口平常访问时延在27ms左右，高峰时接口时延159ms。

采用dubbo长连接后，黑名单接口请求时延平均在3ms左右，高峰时任然不变。同时由于我们把ip的接口请求也自己本地化实现了一套，接口时延从（平常时75ms，高峰时368ms）降到如下情况

![时延表现][2]

平常时 25ms 提升 300%，高峰时 56ms 提升 660%

## 结合某业务接口优化谈一谈QPS的提升

优化前对某接口做了差不多5组压测，每波大概20min，集群当时是9台机器，每台机器4个接口进程，集群qps测到 1200就上不去了（单机 130）。当时与运维讨论了很久未果， 甚至怀疑是测试工具的问题。

在压测过程中，我的服务打点时间始终是正常的且只维持在30ms左右，开始怀疑是不是我们node框架koa2的问题，第二天我在单机使用ab测试了空koa框架发现单机可以到达3000qps，因此框架的qps是不成问题的。

然后对接口的ab结果最终单机真实qps为 240.

再次与运维讨论怀疑是负载均衡而非框架的问题。然后又去了解了我们的进程管理工具PM2其实也承担了对请求的负载分发的工作，虽然其底层采用RollIng算法并不是问题，但是 pm2对请求队列的消耗过程并非是 epoll 类似的做法。

之后为了验证这个猜想使用了nginx反向代理到本机的4个端口，每个端口只跑一个进程，这样等于绕开了pm2分发，只由nginx来做负载均衡，发现QPS可以到达

![压测结果][3]
（最终单机真实qps为 437）


至此我们初步确定QPS的部分瓶颈在pm2的请求转发，采用直接走nginx的方式 直接带来了性能 182%的提升


[1]: http://7xq7m2.com1.z0.glb.clouddn.com/Image.png
[2]: http://7xq7m2.com1.z0.glb.clouddn.com/Image%20%282%29.png
[3]: http://7xq7m2.com1.z0.glb.clouddn.com/Image%20%283%29.png
[4]: http://7xq7m2.com1.z0.glb.clouddn.com/node%E7%9B%B4%E8%BF%9Edubbo%E6%9C%8D%E5%8A%A1%20%281%29.png

