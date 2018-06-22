---
title: elasticsearch源码分析(一)--整体架构
date: 2018-05-26 12:12:57
categories: 
- Elasticsearch源码分析专题
---

# 一、源码主要模块

我下载的Elasticsearch的源码版本为5.6.4

![es整体结构](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/es01.png)









从上图来看：Elasticsearch主要包含以下几个模块

distribution：elasticsearch的打包发行相关，将elasticsearch打成各种发行包（zip，deb，rpm，tar）的模块。具体用法如是，在相应的发行版本模块下执行publishToMavenLocal这个Task，如果执行成功的话就会在路径build/distributions下生成对应的发行包，这种打好的包就能在生产服务器上运行。如果自己修改了源码，打包时就需要用到该模块了。

![:\我的资料\ELK日志监控\es源码分析\distribution整体架构.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/es02-2.png)

![:\我的资料\ELK日志监控\es源码分析\distribution整体架构-打包zip.pn](G:\我的资料\ELK日志监控\es源码分析\distribution整体架构-打包zip.png)

![:\我的资料\ELK日志监控\es源码分析\distribution整体架构-打包zip成功.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/es02-3.png)

core：核心包，elasticsearch的源码主要在这个里面，Elasticsearch索引管理、集群管理、服务发现、查询、对Lucene操作的封装等都位于该模块



buildSrc：elasticsearch的构建相关的代码，gradle相关依赖配置都在改模块下

![:\我的资料\ELK日志监控\es源码分析\buildSrc整体架构.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/buildSrc%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84.png)

client：作为连接elasticsearch的客户端相关代码，它提供了Rest方式（基于Http）、transport （Java Netty内部的通信方式）等方式。

![:\我的资料\ELK日志监控\es源码分析\client整体架构.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/client%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84.png)

modules：作为elasticsearch除核心外的必备模块相关代码,比如对Netty的封装、父子类查询、重建索引

![:\我的资料\ELK日志监控\es源码分析\modules整体架构.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/modules%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84.png)

plugins：作为elasticsearch必备的插件的相关代码，丰富ES的相关功能，比如IK分词器插件、mapper-attachments/ingest-attachment文件处理插件。

![:\我的资料\ELK日志监控\es源码分析\plugings整体架构.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/plugings%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84.png)



# 二、Elasticsearch整体架构图

![:\我的资料\ELK日志监控\es源码分析\ES架构图.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/ES%E6%9E%B6%E6%9E%84%E5%9B%BE.png)



服务发现以及选主 ZenDiscovery

恢复以及容灾

搜索引擎 Search

ClusterState

网络层

Rest 和 RPC

线程池