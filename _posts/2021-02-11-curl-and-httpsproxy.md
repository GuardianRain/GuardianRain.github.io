---
layout: post
title: curl命令能通但代码httpclient请求不通的问题
categories: curl
description: 记录一个curl命令能通但代码httpclient请求不通的问题
keywords: curl，httpclient，httpsproxy
---

有一次同事pxe装好openstack环境以后，调试代码过程中发现httpclient访问不到某个IP，但在后台直接调用curl命令却可以访问，
定位了以后，发现是设置了https_proxy导致的差异，在此记录一下。

## 遇到的现象

在go编写的组件进程中，使用http.Client向某ip地址（设备）请求资源，报了如下错误：
```
No route to host
```
于是 route -n 查看路由表：
![curl-route](/images/posts/curl/curl-route.png)

发现默认网关已经设置了，于是ping目标IP，看看网络是否可达：
```
//直接ping
ping 目标IP
//从网关ping目标IP
ping -I 网关IP 目标IP
```
果然两个都ping不通，因此怀疑是否网关和目标设备之间的交换机上，没有放通对应的vlan号，导致网关与目标设备不可达。
经排查交换机，确实没放通vlan号，该问题解决。（但是本次要说的主要不是这个问题 哈哈 >o<）

在定位该问题过程中，有同事尝试了一下在后台用curl命令请求资源，结果竟然请求到了！
```
curl https://目标IP:目标端口/资源路径 -gki
```
于是开始定位为啥能curl通。

## 原因分析
定位过程中，发现如果curl命令使用 `--interface 网关IP` 指定源IP，就请求不到资源，这与前面从网关ping目标IP不通的现象保持一致。
由于环境中还配置了别的网络平面，因此尝试指定源IP为其它网络平面的地址：
```
curl --interface B网络平面的IP https://目标IP:目标端口/资源路径 -gki
```
发现竟然通了！
curl命令使用`-v`参数可以输出详细的通信过程，于是加了该参数调用不指定源IP的curl命令：
```
curl https://目标IP:目标端口/资源路径 -gki -v
```
输出如下：
```
* Uses proxy env variable https_proxy == 'http://172.28.10.100:60080'
*   Trying 172.28.10.100:60080...
* Connected to 172.28.10.100 (172.28.10.100) port 60080 (#0)
* allocate connect buffer!
* Establish HTTP proxy tunnel to 10.10.1.12:443
> CONNECT 10.10.1.12:443 HTTP/1.1
```
发现请求过程是：先连接到B网络平面的某IP(172.28.10.100)，再去连接目标IP。
查阅资料得知如果设置了环境变量https_proxy，curl命令会自动使用该环境变量，
`echo $https_proxy`查看结果果然设置了环境变量。
再查看go的http.client代码，https代理配置默认为空，这就解释了为何代码调用不通，但curl命令能通。

## 小结

经过以上的分析，真相已经大白，环境变量https_proxy的设置，导致了curl命令与代码中http.client的调用有差别。
附上linux设置http/https proxy的网络资料：https://www.cnblogs.com/marklove/p/10805432.html