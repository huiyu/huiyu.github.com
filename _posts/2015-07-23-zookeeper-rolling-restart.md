---
layout: post
title: "Zookeeper Rolling Restart"
categories: 分布式系统
tags: [Zookeeper, 滚动更新]
---

截止目前的版本3.4.6，Zookeeper没有官方的动态增加节点的功能。因此只能通过滚动更新的方式来增加节点。

假设我们有一个三节点的zookeeper集群，要增加两个节点，步骤如下：

## 增加两个节点

新增两个节点：

    server.4=host:port:electionport
    server.5=host:port:electionport

启动这两个节点，这两个节点会作为follower加入现有集群。

## 修改配置

修改1-3节点配置：

    server.4=host:port:electionport
    server.5=host:port:electionport

保存文件。

## 重启Follower

依次重启所有follower，在重启下个follower节点之前，保证前一个follow节点已经加入集群。

## 重启Leader

在保证所有四个节点都加入集群后，重启leader节点。

## 参考资料

* [Rolling Restart in Apache ZooKeeper to dynamically add servers to the Ensemble](http://www.benhallbenhall.com/2011/07/rolling-restart-in-apache-zookeeper-to-dynamically-add-servers-to-the-ensemble/)
* [Adding 2 nodes to an existing 3-node ZooKeeper ensemble without losing the Quorum](https://gist.github.com/miketheman/6057930)
