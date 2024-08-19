---
title: 深入 GORM
published: 2024-08-01 17:00:00
description: 'GORM deep dive.'
image: ''
tags: ['Microservice', 'Golang', 'Hertz']
category: 'Microservice'
draft: true 
---

>   GORM 是 Golang 中的 ORM 库，ORM 简单理解就是将高级编程语言中的对象（Golang 中的 struct）映射到数据库中的表结构。同时抽象了 SELECT、WHERE 等 SQL 语句为对象上的方法，避免了每次操作数据库手动拼接增删改查等 SQL 语句带来的复杂性与错误。

## Base

### struct 与 Table 的映射

GORM 默认使用结构体的 snake + 复数命名方式映射表名，例如对于 `type User struct`，表名为 `users`；`type UserProfile struct`，表名为 `user_profiles`。

 **自定义表名**：

通过在指定的 `struct` 上实现 `Tabler` 接口来更改默认的表名：

`Tabler` 接口 `gorm.io/gorm/schema/schema.go, line: 108`：

```go
type Tabler interface {
	TableName() string
}
```

例如:

```go
func (User) TableName() string {
	return "profiles"
}
```

:::note



:::
