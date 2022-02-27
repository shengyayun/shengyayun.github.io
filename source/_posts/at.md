---
title: 分布式事务：AT模式
date: 2022-02-25 14:27:13
tags: Seata
---
#### 场景
1. 用户提交订单；
2. 减少库存数量；


#### 术语
TC (Transaction Coordinator) - 事务协调者
维护全局和分支事务的状态，驱动全局事务提交或回滚。

TM (Transaction Manager) - 事务管理器
定义全局事务的范围：开始全局事务、提交或回滚全局事务。

RM (Resource Manager) - 资源管理器
管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。


#### 时序图
{% plantuml %}
skinparam ParticipantPadding 20
skinparam BoxPadding 20

box 订单服务
participant 订单TM
participant 订单RM
endbox
box 库存服务
participant 库存RM
endbox
participant TC

订单TM -> TC: 开启全局事务
TC --> 订单TM: 返回XID

== 订单事务 ==

订单TM -> 订单RM: 传递XID
订单RM -> TC: 注册事务
订单RM -> 订单RM: 执行事务
订单RM -> TC: 执行成功
TC -> 订单TM: 订单事务执行成功

group 成功的场景

== 库存事务 ==

订单TM -> 库存RM: 传递XID
库存RM -> TC: 注册事务
库存RM -> 库存RM: 执行事务
库存RM -> TC: 执行成功
TC -> 订单TM: 库存事务执行成功

== 全局事务 ==

订单TM -> TC: 提交全局事务
TC -> 订单RM: 提交事务
TC -> 库存RM: 提交事务
TC --> 订单TM: 已提交

end

group 失败的场景

== 库存事务 ==

订单TM -> 库存RM: 传递XID
库存RM -> TC: 注册事务
库存RM -> 库存RM: 执行事务
库存RM -> TC: 执行失败
TC -> 订单TM: 库存事务执行失败

== 全局事务 ==

订单TM -> TC: 回滚全局事务
TC -> 订单RM: 回滚事务
TC -> 库存RM: 回滚事务
TC --> 订单TM: 已回滚

end
{% endplantuml %}
