## 限量

`Limit` 限制查询只返回 `n` 个结果.

```go
users, err := client.User.
    Query().
    Limit(n).
    All(ctx)
```


## 偏移

`Offset` 设置第一个返回结果在所有结果中的位置.

```go
users, err := client.User.
    Query().
    Offset(10).
    All(ctx)
```

## 排序

`Order` 设置按照一个或多个字段值来对返回结果排序。 注意，如果给定的字段不是有效的列或外键，将返回错误。

```go
users, err := client.User.Query().
    Order(ent.Asc(user.FieldName)).
    All(ctx)
```

## 按边排序

为了按边 (关系) 的字段排序，从此边开始遍历 (你想要进行排序的边) 并应用排序，然后转到目标类型。

下面展示了 如何根据用户 `"pets"` 的 `"name"` 来对用户进行排序。
```go
users, err := client.Pet.Query().
    Order(ent.Asc(pet.FieldName)).
    QueryOwner().
    All(ctx)
```

## 自定义排序

如果您想写自己的特定逻辑，可以使用自定义排序函数。

下面展示了 如何根据 宠物的名字 和 主人的名字 来对宠物进行升序排序。

```go
names, err := client.Pet.Query().
    Order(func(s *sql.Selector) {
        // 连接用户表，以通过 owner-name 和 pet-name 进行排序.
        t := sql.Table(user.Table)
        s.Join(t).On(s.C(pet.OwnerColumn), t.C(user.FieldID))
        s.OrderBy(t.C(user.FieldName), s.C(pet.FieldName))
    }).
    Select(pet.FieldName).
    Strings(ctx)
```