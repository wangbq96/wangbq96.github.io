---
title: Timeline的推(push)模式和拉(pull)模式
tags:
- 数据库
- 已完结
categories:
- 技术
- 软件设计
author: 汪博全
date: 2017-03-09 00:12:00
---

> 已完结

# Timeline的特点
虽然有着各种不同的 timelines，但它们大概都有这 3 个共同的特点：

* 每个人的 timeline 都是个人独有的。
* 由一系列按时间顺序排列的items组成。
* Timeline 中的条目有多种不同的来源。

<!-- more -->

其第一个特点正是它存在的价值：每个用户独有的 timeline，才是他最关心的 timeline。而这也意味着存储 timeline 的开销，是和用户量成正比的。

而第二个特点也就意味着它具备时效性，用户并不关心较老的条目，因此 timeline 的长度并不需要随时间无限增长。对大部分的互联网产品的 timeline 而言，几百条的 items 已经够用了(新浪微博目前是 450 条，其 API 只能获取 150 条；Twitter API 可以获取 3200 条)；另一种策略是按照时间截断（Google Reader 只展示一个月内的未读条目）。有限的条目量，更方便估算存储 feed 的开销，并且能带来更好的性能。

另外，因为用户只关心较新或者未读的条目，所以 timeline 的读写比例大致为 1:1。当活跃用户较少时，写操作甚至可能多于读操作。

# 推模式和拉模式
它的第三个特点则是它复杂的原因，也导致了它的实现分成了两类模型：

* Pull（拉）模型：计算 timeline 时，从各个来源获取最新的条目，然后聚合起来。
* Push（推）模型：条目产生时，推送到其关注者的 timeline 中。

拉模型实现起来很容易，存储开销也少（不需要为每个用户单独保存一份数据），但因为获取时要实时计算，因此读性能比较差。

推模型的存储开销则比较大，而且在关注者较多时，推送的开销也很大，不过读性能很好，并且更容易实现实时更新和消息推送。

当然也可以把二者结合起来，对来源进行分类，那些关注者较多的来源用拉模型来获取，其余仍推到关注者的 timeline。只要这些来源不多（通常它们还被缓存了），对读性能的影响也并不算大，但极大减少了推的开销。
出于实现的简洁性和效率的考虑，最常用的还是推模型。这也就意味着开发者其实是很讨厌「大 V」的。无奈限制被关注者人数是很不合理的，因此只能通过限制关注人数来限制推的频率了。加上更新太频繁也会让用户疲倦，于是大多数互联网产品都限制了关注人数上限（一般为数千）。

## 拉模式的实现
建立两张表
`item(id, poster_id, content, status, created_at)`
`friendship(id, follower_id, followee_id, created_at)`

一句sql即可实现拉模式

```sql
SELECT *
FROM item
WHERE poster_id IN (SELECT followee_id
                    FROM friendship
                    WHERE follower_id = user_id)
AND status = public
ORDER BY created_at
DESC LIMIT 0, 50;
```

不过缺点也蛮明显的，假如关注了 1000 个用户，第二条语句将查询 1000 次，然后把结果合并起来，再进行排序。这样做运行速度非常低，除非没什么用户。

而如果要改进，似乎也只能用缓存了。Friendship 表中的数据可以按 follower_id 缓存，于是获取关注的用户变成了 O(1) 的操作；Item 表中的数据可以按 poster_id 缓存，于是获取 timeline 中的数据就变成了 O(N) 的操作（如果限制了最多关注 1000 人，则 N 最大为 1000）；最后在内存中进行合并和排序。现在 MySQL 基本没有查询压力了，只要应用服务器足够多，这仍然是个可用的方案。考虑到 timeline 中只有较新的数据才经常被读取，所以缓存的量可以进行一些缩减。

而随着用户量的增长，item 和 friendship 表都需要进行分表。一个可能的方案是把 item 表按 poster_id 分表，同时也把近期的 items 按时间分表（当缓存失效时，可以加快查询）；friendship 表则可能需要分别对 follower_id 和 followee_id 分表，多存一份冗余的数据。

## 推模式的实现
建立三张表
`item(id, poster_id, content, status, created_at)`
`friendship(id, follower_id, followee_id, created_at)`
`timeline(id, user_id, item_id, created_at)`
当用户发布了一条微博后，从 friendship 中获取所有的关注者，然后批量给他们塞入这条微博即可。而这个操作其实允许延迟和失败，所以只要塞到任务队列里即可。

而获取 timeline 则可以直接按 user_id 查询了：

```sql
SELECT item_id FROM timeline
WHERE user_id = user_id ORDER BY created_at DESC LIMIT 0, 50;
```
再根据 item_id，从 item 表获取详情即可。
随着时间的增长，timeline 表也需要分表，似乎没什么理由不按 user_id 分表。

相较于拉模型，推模型还需要多做两件事：

* Item 被发布者删除或隐藏时，需要从他自己和所有关注者的 timeline 中删除。
* 取消关注时，需要把关注者的 timeline 中所有发自该被关注者的 items 删除。

第一件事其实不做也可以，展示时再判断一下 status 即可。如果非要删除的话，可以找到所有关注者，进行该删除操作：

```sql
DELETE FROM timeline WHERE user_id IN (user_ids) AND item_id = item_id;
```
或者给 item_id 增加一条索引，然后找到所有相关的表，都进行该删除操作：

```sql
DELETE FROM timeline WHERE item_id = item_id;
```

第二件事看上去并不太好做，如果可以忍受的话，遍历 timeline 中的所有 items 也是可行的，毕竟一个用户的 timeline 长度是有限的，只是一个 O(N) 的操作而已。
更好的做法是给 timeline 表增加一个冗余字段 poster_id，并增加一条 (user_id, poster_id) 的索引，就能这样删除了：

```sql
DELETE FROM timeline WHERE user_id = user_id AND poster_id = poster_id;
```

# 参考资料
[如何构建 timeline](https://www.keakon.net/2015/10/05/%E5%A6%82%E4%BD%95%E6%9E%84%E5%BB%BAtimeline)
[微博feed系统的推(push)模式和拉(pull)模式和时间分区拉模式架构探讨](http://www.cnblogs.com/sunli/archive/2010/08/24/twitter_feeds_push_pull.html)
[ Feed系统设计 资源整理](https://my.oschina.net/feichexia/blog/222877)
