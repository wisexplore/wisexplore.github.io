---
title: "Go map 的 key：为什么可以用结构体指针"
date: 2026-06-09T22:35:00+08:00
draft: false
description: "从链表判环里的 map[*ListNode]bool 出发，记录 Go map 使用指针作为 key 时到底比较的是什么。"
ShowToc: true
TocOpen: false
---

## 问题来源

刷环形链表 II 时，会看到一种直观做法：用哈希表记录访问过的节点。

在 Go 里通常会写成：

```go
visited := map[*ListNode]bool{}
```

这里容易疑惑的是：key 是什么？存的是节点值，还是节点地址？

## 结论

`*ListNode` 是指向链表节点的指针。

当 `map` 的 key 类型是 `*ListNode` 时，Go 比较的是指针值，也就是它指向的地址。

所以：

```go
visited[node] = true
```

表达的是：这个节点对象已经访问过。

如果后面再次走到同一个节点地址：

```go
if visited[node] {
	return node
}
```

就说明链表开始重复，当前节点就是环入口。

## 为什么不能只存节点值

链表里不同节点的 `Val` 可能相同。

例如两个节点的值都等于 `3`，它们仍然是两个不同节点。判断环时，关心的是“是不是同一个节点对象”，不是“值是不是一样”。

所以哈希表要记录节点指针，而不是记录 `node.Val`。

## bool 的含义

`map[*ListNode]bool` 里的 `bool` 只是标记是否访问过。

也可以用空结构体节省一点空间：

```go
visited := map[*ListNode]struct{}{}
```

不过刚开始刷题时，`bool` 更直观。

## 复盘结论

- `*ListNode` 作为 key 时，比较的是节点地址。
- 链表判环关心的是节点对象是否重复，不是节点值是否重复。
- `map[*ListNode]bool` 是“访问过这个节点了吗”的直接表达。
