---
layout: post
title: 开启solr debug
tags:
- solr
categories: solr
description: 如何solr源码debug分析解决问题
---

本文主要介绍solr6.6.0版本下，如何源码导入idea，进行debug。

<!-- more -->

# 下载编译导入Idea
从 [solr历史版本](https://archive.apache.org/dist/lucene/solr/) 中选择6.6.0版本下载。

进入到解压后的源码文件夹中去，
```shell
-- 查看ant提示
ant 
-- 执行ant idea
ant idea
```
执行结束后，导入工程到idea中。

# 远程debug

>  bin/solr start -s "example/example-DIH/solr" -a "-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=4044"

`-s`后指定索引目录， `-a`后可以追加java参数。


idea开启debug

<img src="/assets/img/solr/solr1.png" width="650"/>


# 小试牛刀
以solr的`DataImportHandler`机制为例，它支持仅通过简单的sql配置，即可全量/增量同步DB数据到solr中，具体可参考[DataImportHandler](https://solr.apache.org/guide/6_6/uploading-structured-data-store-data-with-the-data-import-handler.html)

- debug启动

>  bin/solr start -s "example/example-DIH/solr" -a "-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=4044"

- 执行全量导入

> http://localhost:8983/solr/db/dataimport?command=full-import&jdbcurl=jdbc:hsqldb:./example-DIH/hsqldb/ex&jdbcuser=sa&jdbcpassword=secret

  <img src="/assets/img/solr/solr2.png" width="650"/>

  <img src="/assets/img/solr/solr3.png" width="650"/>

# 分析生产问题
之所以下载源码debug，是因为生产上遇到了一个问题，查询了相关资料，没有找到明确的答案，不得已只能进行源码debug分析。

问题是这样的，生产上商品表，每隔15分钟会定时做一次索引增量更新，也就是说，商品发布后，最迟15分钟即可被搜索到。
正常情况下，工作机制应该是类似这样的。

- 15.00时，`select * from item where gmt_modified >= ${dataimporter.last_index_time}`, 这里的dataimporter.last_index_time是文件`dataimport.properties`中记录的上次执行索引发起的时间(假设目前是14.45)。执行索引结束后，会将15.00 写到文件中。

- 15.15时，仍然执行上述逻辑，应该查询到15.00之后变化的商品数据并索引到solr中，再将15.15写到文件中，以供下一次读取。

但是有些时候，发现`增量索引有延迟`， 已经16.00了，可15.15之后的数据都还没被索引。

最终通过debug发现，同一时间，只允许一个线程去索引。延迟是因为，某次索引耗时太长(同事写了个低效的sql，无效批量更新了很多商品，导致15分钟索引无法完成)。 该影响具有后置性，后续执行增量索引时的查询sql时间条件可能不再是当前时间的15分钟前，可能更长，导致查询数据越来越多，使延迟越来越严重。



# 参考
1. [solr历史版本](https://archive.apache.org/dist/lucene/solr/)
2. [debugging-solr-5-in-intellij](https://opensourceconnections.com/blog/2015/04/30/debugging-solr-5-in-intellij/)
3. [solr常用命令](https://www.cnblogs.com/studyhs/p/5181808.html)
4. [在Idea下编译solr 6.1源码](https://blog.csdn.net/adalu1986/article/details/52293140)
5. [用Intellij idea搭建solr调试环境](https://www.cnblogs.com/jeniss/p/5995921.html)
6. [DataImportHandler](https://solr.apache.org/guide/6_6/uploading-structured-data-store-data-with-the-data-import-handler.html)