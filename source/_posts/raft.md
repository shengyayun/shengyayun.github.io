---
title: Raft算法分析
date: 2019-04-22 22:28:26
tags: raft
---

## 导言
Raft算法用于保证分布式系统的强一致性，被目前知名的etcd所采用。

## 一. 心跳
集群中每个节点都需要定期向其他节点发送心跳包，如果超过一半的节点超过一定时间(阈值)未收到该节点的心跳包，那么认定该节点已下线。

{% plantuml %}
节点A <-> 节点B: 心跳
note right: 基于http长连接的stream类型通道
节点A <-> 节点C: 心跳
节点A <-> 节点D: 心跳
节点A <-> 节点E: 心跳
节点B <-> 节点C: 心跳
节点B <-> 节点D: 心跳
节点B <-> 节点E: 心跳
节点C <-> 节点D: 心跳
节点C <-> 节点D: 心跳
节点C <-> 节点E: 心跳
节点D <-> 节点E: 心跳
{% endplantuml %}


## 二. 选举
集群中节点角色可为Leader、Follower或Candidate其中一个。当Follower在一定时间内没有收到来自Leader的心跳，会将自己角色变更为Candidate，然后等待一段随机的时间后，发起选举。

{% plantuml %}
hnote over 节点A : Leader(任期:n)
hnote over 节点B : Follower
alt 来自Leader的心跳包超时
	节点A ->x 节点B : 心跳包超时
	hnote over 节点B : Candidate
	loop
		节点B -> 节点B : 等待随机时间
		note right : 避免多个Candidate同时发起选举
		节点B -> 节点B : 发起竞选，任期:++n
		note right : 每次发起选举，都会让任期+1
		alt 收到超过半数的节点(包括自己)赞成，选举成功
			节点B -> 节点C : 发起选举请求
			note right : 选举请求主要包含本次选举的任期与上个有效任期的最新日志的标号
			节点C --> 节点B : 该选举的任期不是最新，拒绝
			note right : 其他节点在本选举未结束的情况下发起了新的选举，也就是所谓的"竞选冲突"
			节点B -> 节点D : 发起选举请求
			节点D x--> 节点B : 回复超时
			节点B -> 节点E : 发起选举请求
			节点E --> 节点B : 候选保存的数据不完整，拒绝
			note right : 候选节点会向其他节点提供自己的新任期与上个有效任期的\n最新日志的编号，其他节点以此判断候选节点的数据是否完整。
			hnote over 节点B : Candidate
		else 未收到超过半数的节点(包括自己)赞成，选举失败
			节点B -> 节点C : 发起选举请求
			节点C --> 节点B : 赞成
			节点B -> 节点D : 发起选举请求
			节点D --> 节点B : 赞成
			节点B -> 节点E : 发起选举请求
			节点E --> 节点B : 选举冲突，但对方任期更新且数据完整，赞成
			note right : 节点E正在选举中，但节点B发起了新一轮的选举。\n如果节点B的任期是最新的，且保存了完整的数据，节点E会投赞成票。
			hnote over 节点B : Leader(任期:n)
			节点B ->
			note right : 选举成功，跳出loop
		else 选举过程中收到来自Leader的心跳包，终止选举
			节点B -> 节点B : 收到来自Leader的心跳包
			hnote over 节点B : Follower
			note right : 可能是来自原Leader的心跳包(原Leader任期不变)，也可能是其他节点选举成功。
		end
	end
end

alt 原Leader节点从宕机中恢复
	节点A -> 节点B : 心跳包
	alt 其他节点选举完成
		节点B --> 节点A : 任期已结束
		hnote over 节点A : Follower
	else 其他节点选举中
		节点B -> 节点B : 终止选举
		节点B -> 节点A : 正常
		hnote over 节点B : Follower
	end
end
{% endplantuml %}


## 三. 强一致
假设集群中最多n台机器发生故障，那么最少需要2n+1个节点。比起在拜占庭问题中最少需要3n+1个节点，raft协议的优势在于节点不存在欺骗问题。

{% plantuml %}
用户 -> "Follower节点(们)" : 更新操作
"Follower节点(们)" -> Leader节点 : 转发操作
Leader节点 -> Leader节点 : WAL日志
Leader节点 -> "Follower节点(们)" : 同步日志
note right : 确保超过一半的节点同步日志成功
"Follower节点(们)" --> Leader节点 : 同步日志成功
note right : "Follower节点(们)"会判断同步过来的日志的任期是否过期，\n并且根据本次提交的日志编号判断如何追加日志
Leader节点 -> Leader节点 : 更新状态机
Leader节点 -> "Follower节点(们)" : 更新状态机
"Follower节点(们)" --> Leader节点 : 更新状态机成功
Leader节点 -> "Follower节点(们)" : 更新操作完成
"Follower节点(们)" --> 用户 : 更新操作完成
{% endplantuml %}
