---
layout:     post
title: etcd源码分析（二）
category: blog
description: etcd
---

# 首先我们可以看到 progress的两个重要元素有三种状态
+ progress中的两个重要元素 （match next）
+ match是leader知道的对于自己的更新状态
+ next将要复制的最新的元素序号
+ probe
+ replicate
+ snapshot
  
## follower处于 probe状态
+ leader每次只发送一个replication message
+ 直到收到msgHeartbeatResp 或者收到 msgAppResp


## follower处于replicate状态
+  当follower给leader发送rejct为false时 切换到replicate状态
+  leader这是会以流方式向follower发送信息
+  当follower发送的reject状态为true时 重新置为probe状态
# 当follower的数据和leader的数据相差太多时
+ leader把follower的状态变为snapshit
+ leader发送snapshot
+ follower接受完毕snapshot数据后 回到probe状态

---

新当选的leader，会把所有的follower置为 probe，把match置为0，把next置为自身log entry set的最大值

---

# 下面几种可能示例 可能导致raft集群不可用
+ 新加入一个follower时，leader发送snapshot，时会占用大量网速，导致不能发送给别的节点heartbeat，导致集群不可用


# 下面看这几个函数
+ 成为探测状态


```
func (pr *Progress) becomeProbe() {
	if pr.State == ProgressStateSnapshot {
		pendingSnapshot := pr.PendingSnapshot
		pr.resetState(ProgressStateProbe)
		pr.Next = max(pr.Match+1, pendingSnapshot+1)
	} else {
		pr.resetState(ProgressStateProbe)
		pr.Next = pr.Match + 1
	}
}
```
如果一开始的状态是snapshot，那么就把next改为snapshot和match最大的值加一


# pendingsnapshot
+ 被设置成snapshot的索引，如果这个参数被设置，那么复制过程就会被停止主节点不会再发送消息直到收到失败消息

# recentActive（bool）
+ 接受到任何消息从相应的follower说明这个progress是active状态的
+ 直到一次选举失败后 才会被重置为 false

# ins inflights 
+ 这是一个传输信息的滑动窗口
+ 当窗口填满时 不应该再加入信息
+ 当一个leader发送信息，里面一定存在最后的index信息
+ 当一个leader收到回复后，先前的窗口应该free掉

```
func (pr *Progress) becomeProbe() {
	// If the original state is ProgressStateSnapshot, progress knows that
	// the pending snapshot has been sent to this peer successfully, then
	// probes from pendingSnapshot + 1.
	if pr.State == ProgressStateSnapshot {
		pendingSnapshot := pr.PendingSnapshot
		pr.resetState(ProgressStateProbe)
		pr.Next = max(pr.Match+1, pendingSnapshot+1)
	} else {
		pr.resetState(ProgressStateProbe)
		pr.Next = pr.Match + 1
	}
}
```

从上面这段代码可以看出，在接受快照后，应该把next置为match或pendingSnapshot中大的一方加一
这样做可以避免数据的重新传输

---

总结 progress中的代码比较少 其余的函数都是关于窗口的，此处略过