---
layout:     post
title: raft源码分析（一）
category: blog
description: etcd
---

# raft结构分析
+ proposeC 接受信息
+ confChangeC 接受集群变化
+ commitC 提交信息
+ errorC 报错
+ id node的id
+ peers 每个成员的url
+ join 是否在集群中
+ waldir wal的地址
+ snapdir snap的地址
+ snapCount
+ getSnapshot 一个初始化定义的函数 使用了锁和json解析
+ snapCount 全局定义的 最大的snapshot的数量
+ stopc 接受propose 信号管道
+ httpstopc 通过http信号关闭管道
+ httpdonec 完成关闭后发送管道

## main函数入口操作
+ 初始化cluster id kvport join proposeC ConfChangeC这几个变量


```
cluster := flag.String("cluster", "http://127.0.0.1:9021", "comma separated cluster peers")
	id := flag.Int("id", 1, "node ID")
	kvport := flag.Int("port", 9121, "key-value server port")
	join := flag.Bool("join", false, "join an existing cluster")
	flag.Parse()

	proposeC := make(chan string)
	defer close(proposeC)
	confChangeC := make(chan raftpb.ConfChange)
	defer close(confChangeC)
```

+ 定义一个getSnapshot函数（这个函数是将keyvalue通过json编码）这里使用了闭包，不用传递kvs即可调用方法


```
getSnapshot := func() ([]byte, error) { return kvs.getSnapshot() }
```

+ 新建一个raft节点


```
commitC, errorC, snapshotterReady := newRaftNode(*id, strings.Split(*cluster, ","), *join, getSnapshot, proposeC, confChangeC)
```

+ 定义好新的kv-Store  初始化kvstore snapshoter 并开启协程读取提交


```
s := &kvstore{proposeC: proposeC, kvStore: make(map[string]string), snapshotter: snapshotter}
	// replay log into key-value map
	s.readCommits(commitC, errorC)
	// read commits from raft into kvStore map until error
	go s.readCommits(commitC, errorC)
	return s
```