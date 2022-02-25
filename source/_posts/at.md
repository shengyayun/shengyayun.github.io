---
title: Automatic Transaction
date: 2022-02-25 14:27:13
tags: Seata
---

> 通过AT方案实现该分布式业务场景：
> 1. 用户提交订单；
> 2. 减少库存数量；
> 3. 扣减账户金额；

{% plantuml %}
autonumber
skinparam ParticipantPadding 20
skinparam BoxPadding 20

box 订单服务
participant 订单TM
participant 订单RM
endbox
box 库存服务
participant 库存RM
endbox
box 账户服务
participant 账户RM
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

== 库存事务 ==

订单TM -> 库存RM: 传递XID
库存RM -> TC: 注册事务
库存RM -> 库存RM: 执行事务
库存RM -> TC: 执行成功
TC -> 订单TM: 库存事务执行成功

group 成功的场景

== 账户事务 ==

订单TM -> 账户RM: 传递XID
账户RM -> TC: 注册事务
账户RM -> 账户RM: 执行事务
账户RM -> TC: 执行成功
TC -> 订单TM: 账户事务执行成功

== 全局事务 ==

订单TM -> TC: 提交全局事务
TC -> 订单RM: 提交事务
TC -> 库存RM: 提交事务
TC -> 账户RM: 提交事务
TC --> 订单TM: 全局事务已经提交

end

group 失败的场景

== 账户事务 ==

订单TM -> 账户RM: 传递XID
账户RM -> TC: 注册事务
账户RM -> 账户RM: 执行事务
账户RM -> TC: 执行失败
TC -> 订单TM: 账户事务执行失败

== 全局事务 ==

订单TM -> TC: 回滚全局事务
TC -> 订单RM: 回滚事务
TC -> 库存RM: 回滚事务
TC -> 账户RM: 回滚事务
TC --> 订单TM: 全局事务已经回滚

end
{% endplantuml %}