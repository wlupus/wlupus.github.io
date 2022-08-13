# Mysql Connection


<!--more-->

## 1 问题描述
在使用`mysql-connector-python`库的时候，发现每当页面发起多次http请求的时候，就会导致mysql crash。

## 2 解决方案
经过查看相关代码逻辑，推测是多次请求同时调用了数据库，导致了这个问题。在网上搜索相关的内容，推测是因为mysql的connection不是线程安全的，将全局共用的connection修改为单个线程使用，即可解决这个问题。

## 3 反思
之所以会有全局共用一个connection的奇葩写法，是因为这个项目是在一个开源项目的基础上改的，那个旧的项目采用了这种奇葩写法，又因为那个项目一个页面只会有一个请求会查询sql，所以这个问题就隐藏了。在使用的变量涉及到多线程时(例如在处理http请求时)，一定要考虑是否是线程安全的。

{{<admonition info "潜在问题">}}
在Stack Overflow上有看到使用connection pool的解决方案，推测是反复创建并销毁connection开销比较大，可以留作后续的改进方案
{{</admonition>}}
