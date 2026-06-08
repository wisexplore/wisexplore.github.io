---
title: "Redis Cluster 跨槽问题：key 命名不规范引发的坑"
date: 2026-06-08T12:00:00+08:00
draft: false
description: "记录一次因为 Redis key 命名不规范导致 multi-key 操作跨 slot 的问题，以及用 hash tag 统一槽位的修复思路。"
ShowToc: true
TocOpen: false
---

## 问题现象

Redis Cluster 中执行涉及多个 key 的命令时，出现类似跨槽错误：

```text
CROSSSLOT Keys in request don't hash to the same slot
```

这类问题通常出现在：

- `MGET`、`MSET`
- `DEL key1 key2`
- Lua 脚本同时操作多个 key
- 事务或批量命令里涉及多个 key

## 根因

Redis Cluster 会把 key 按哈希结果分配到不同 slot。

如果一个命令需要同时操作多个 key，Redis 要求这些 key 必须在同一个 slot。否则集群不知道该把这个命令交给哪个节点执行，于是报跨槽错误。

这次问题的根因是：key 命名不规范，导致业务上相关的一组 key 被分配到了不同 slot。

例如：

```text
user:1001:profile
user:1001:settings
```

看起来都属于用户 `1001`，但 Redis Cluster 默认会对整个 key 做 hash，它们不一定落在同一个 slot。

## 修复方式

使用 Redis Cluster 的 hash tag。

Redis 会优先对 key 中 `{}` 里的内容计算 slot。只要 `{}` 里的内容相同，这些 key 就会落到同一个 slot。

推荐写法：

```text
user:{1001}:profile
user:{1001}:settings
```

这两个 key 都会使用 `1001` 计算 slot，所以可以在同一个 multi-key 操作中使用。

## 命名规范

如果多个 key 需要被同一个命令、同一个 Lua 脚本或同一段事务逻辑一起操作，就要提前设计统一 hash tag。

建议：

```text
业务域:{聚合维度}:具体资源
```

示例：

```text
cart:{userId}:items
cart:{userId}:meta

order:{orderId}:detail
order:{orderId}:lock

rate:{userId}:counter
rate:{userId}:window
```

## 易错点

- 不是所有 Redis 命令都会跨槽，只有涉及多个 key 且这些 key 不在同一 slot 时才会触发。
- hash tag 只认 `{}` 中的内容，不是看 key 的前缀像不像。
- 不要随手把不同业务维度塞进同一个 hash tag，否则会造成某个 slot 过热。
- key 命名不是随便起字符串，它会影响 Redis Cluster 的路由和可用命令范围。

## 复盘结论

Redis Cluster 下，key 设计就是数据分布设计。

以后只要有多 key 操作，先问一句：

```text
这些 key 是否必须落在同一个 slot？
```

如果答案是是，就用统一 hash tag；如果答案是否，就避免使用要求同槽的命令或重新拆分操作。
