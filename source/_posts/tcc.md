---
title: 分布式事务：TCC模式
date: 2022-02-25 16:02:34
tags: Seata
---

**场景：** 1. 用户提交订单；2. 减少库存数量；
**Try：**资源的检测和预留；
**Confirm：**执行的业务操作提交；要求 Try 成功 Confirm 一定要能成功；
**Cancel：**预留资源释放；
**幂等性：**Confirm与Cancel必须实现幂等，因为如果执行失败，TC会不断重试。
**允许空回滚：**Try未执行过的情况下，也可以Cancel成功。
**防悬挂控制：**Cancel后Try被调用也不执行。

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
订单RM -> TC: 注册分支事务
订单RM -> 订单RM: Try
订单RM -> TC: Done
TC -> 订单TM: Done

group 成功的场景

== 库存事务 ==

订单TM -> 库存RM: 传递XID
库存RM -> TC: 注册分支事务
库存RM -> 库存RM: Try
库存RM -> TC: Done
TC -> 订单TM: Done

== 全局事务 ==

订单TM -> TC: 提交全局事务
TC -> 订单RM: Confirm
TC -> 库存RM: Confirm
TC --> 订单TM: 已提交

end


group 失败的场景

== 库存事务 ==

订单TM -> 库存RM: 传递XID
库存RM -> TC: 注册分支事务
库存RM -> 库存RM: Try
库存RM -> TC: Fail
TC -> 订单TM: Fail

== 全局事务 ==

订单TM -> TC: 回滚全局事务
TC -> 订单RM: Cancel
TC -> 库存RM: Cancel
TC --> 订单TM: 已回滚

end
{% endplantuml %}
