---
title: "Openshift中Route的HAProxy实现无缝重载原理分析"
date: 2020-05-30T14:24:26+08:00
draft: false
---

## 一、背景
在openshift集群中（以下简称OCP），对外部流量的转发是通过Router控制器控制Route对象中的路由规则来重载Infra节点中的HAProxy配置文件实现的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020053013102280.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70)
在上图中的第3步，Router重载Haproxy配置的过程中是会有有一小段时间Haproxy服务不可用，那么在Haproxy重载过程中是如何做到用户请求不丢失呢？
## 二、Haproxy在不同版本中处理无缝重载的策略
#### OpenShift 3.9及更高版本
OpenShift 3.9及更高版本随附了基于HAProxy 1.8 7的HAProxy路由器映像。HAProxy 1.8中已完成的一项功能是内置的无缝重载。
在通过以下方式将侦听socket的文件描述符传递给新的HAProxy进程：另一个socket。由于socket永远不会关闭，因此可以最大程度地减少重载配置的延迟！
#### OpenShift 3.7及更早版本
OpenShift 3.7和更早版本附带基于HAProxy 1.5或更早版本的HAProxy路由器映像。这些较旧的版本不附带上面详细介绍的无缝重载功能。缺少的无缝重载功能的替代解决方案（以下详细介绍）已放入OpenShift 随附的默认重新加载脚本中。
在OpenShift 3.7和更早版本中，Router的DROP_SYN_DURING_RESTART环境变量必须设置为true，以便启用此功能。默认情况下禁用。另外，该pod必须具有特权访问权限。
因为DROP_SYN_DURING_RESTART功能导致将iptables规则（这就是为什么需要特权访问！）添加到在路由器重装过程中丢弃 SYN数据包的应用程序节点。重新加载过程完成后，该iptables规则将被删除，从而允许SYN数据包返回。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530135918889.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE3MzA1MjQ5,size_16,color_FFFFFF,t_70)

参考资料：

 - [https://access.redhat.com/articles/3679861](https://access.redhat.com/articles/3679861)
 - [https://www.haproxy.com/blog/truly-seamless-reloads-with-haproxy-no-more-hacks/](https://www.haproxy.com/blog/truly-seamless-reloads-with-haproxy-no-more-hacks/)
