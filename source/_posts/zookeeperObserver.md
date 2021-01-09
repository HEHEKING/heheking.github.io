---
title: Zookeeper 的 Observer（旁观者）模式
date: 2020/12/1 11:29:30
categories:
- 教程
tags:
- zookeeper
---

这两天参加了 数据分析 相关的比赛，碰到了 zookeeper 的 **旁观者** 模式。

不太理解，故作记录。
<!-- more -->

资料整理和参考：https://blog.csdn.net/damacheng/article/details/84610594

## 一、目的

旁观者模式的目的是为了 zookeeper 集群能够处理更多的读请求。通过增加 observer（旁观者）节点的数量，可以接受更多的读请求，且不会增加集群的写入负担。但增加 observer（旁观者）并不是完全没有损耗的。每增加一个新的 observer（旁观者）会在提交事务后增加一条接收到的 inform 消息，但这个损耗总的来说是比 follower 投票来的少。

使用 observer 的另一个原因是跨数据中心部署。把 participant 分散到多个数据中心可能会极大拖慢系统，因为数据中心之间的网络的延迟。使用 observer 的话，更新操作都在一个单独的数据中心来处理，并发送到其他数据中心，让其他数据中心的 client 消费数据。阿里开源的跨机房同步系统 Otter 就使用了 observer 模式，可以参考。

注意 observer 的使用并无法完全消除数据中心之间的网络延迟，因为 observer 不得不把更新请求转发到另一个数据中心的 leader，并处理 inform 消息，网络速度极慢的话也会有影响，它的优势是为本地读请求提供快速响应。

### Inform 消息

简而言之，follower 会得到两个消息，而 observer 只会得到一个。follower 通过广播得到 proposal 的内容，接下来获得一个简单 commit 消息，此消息只包含了 zxid 。相反，observer 得到一个包含了已被 commit 的 proposal 的 inform 消息。

因为 observer 不会接收 proposal 并参与投票，leader 不会发送 proposal 给 observer。leader 发送给 follower 的 commit 消息只包含zxid，并没有 proposal 本身。所以，只发送 commit 消息给 observer 则不会让 observer 得知已提交的 proposal 。这就是使用 inform 消息的原因，此消息本质上是一个包含了已被 commit 的 proposal 的 commit 消息。

参与了决定是否 commit 一个 proposal 的投票的 server 就称为 **PARTICIPANT** server（与 observer 相对应），leader和follower都属于这种server。

## 二、配置

配置集群内的 observer 需要更改 ./conf/zoo.cfg

对于 observer 来说，它需要在 server list 中被标识为 observer，并对自身的 **zoo.cfg** 中的 **peerType** 进行配置

```
# 设置
peerType=observer
# server list
server.1=master:2181:3181
server.2=slave1:2181:3181:observer
server.3=slave2:2181:3181
server.4=slave3:2181:3181
```

在这里，slave1作为旁观者在 server list 中被标识，所以对于 slave1 的 zoo.cfg，我们将 **peerType=observer** 加入配置