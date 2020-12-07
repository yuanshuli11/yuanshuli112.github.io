---
layout: post
title:  "并发场景下，mysql实现字段自增。"
categories: mysql
tags:  mysql  
author: 碧海长天
---

* content
{:toc}



表结构:
user_account
id	自增id
user_id	用户id
user_name	用户名称
money	金额

假设有这样一张用户金额的表。每一次充值都会变更money。

我们很容易想到直接更新：
update user_account set money=money+100 where user_id = 1

这样就能实现金额的更新。
但是如果更新完后需要获取到准确的数值呢？

这时候就又需要一个

select money from user_account where user_id = 1;

但是在并发场景下，也许你执行select时，money已经被另一个进程给更新了。
可能出现： 原来只有100元。 又充值了100 这时候本该展示200的。 但是因为另一个进程也进行扣款100的操作。
这时候在充值成功的页面就会展示： 

原始金额：100
充值金额：100
剩余金额：100

这样的情况。虽然用户的总金额100的确是对的，但是会影响体验。（这个只是个例子，真实应用的话可以用redis来解决）
如果只用mysql的话，我们应该怎样解决这个问题呢？

这里提供两种解决方式:
方式1.事务+排他锁
BEGIN;
SELECT * FROM user_account WHERE user_id=1 FOR UPDATE;
//new_money = money+100
UPDATE user_account SET money=new_money WHERE user_id = 1;
COMMIT;
可以理解为：先查询了一下当前的金额，并加上了排他锁，这样其他进程读这行数据时，就会处于等待的状态，当然也改不了改行的数据。
然后再执行更新。这时候mysql中的数值就会正常的加上100 而不会被干扰。
但这时候需要在代码中处理，将查询到的money+100  感觉有点不靠谱。

方式2.只使用事务
BEGIN;
UPDATE user_account SET money=money+100 WHERE user_id = 1;
SELECT * FROM user_account WHERE user_id=1 FOR UPDATE;
COMMIT;
不同点：将money在执行sql的时候自增加1.因为mysql事务更新时会有行锁（划重点）
所以，在事务提交前，其他进程时读不到修改后的money的。更无法进行相同的update更新操作。
这时候获取到的money的值就是准确的值了～。



（完）
