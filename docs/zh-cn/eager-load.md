## 概览

`ent` 支持查询关联的实体 (通过其边)。 关联的实体会填充到返回对象的 `Edges` 字段中。

让我们通过以下例子，了解与 Schema 相关的 API：

![er-group-users](https://entgo.io/images/assets/er_user_pets_groups.png)



**查询所有用户及其宠物：**
```go
users, err := client.User.
    Query().
    WithPets().
    All(ctx)
if err != nil {
    return err
}
// The returned users look as follows:
//
//  [
//      User {
//          ID:   1,
//          Name: "a8m",
//          Edges: {
//              Pets: [Pet(...), ...]
//              ...
//          }
//      },
//      ...
//  ]
//
for _, u := range users {
    for _, p := range u.Edges.Pets {
        fmt.Printf("User(%v) -> Pet(%v)\n", u.ID, p.ID)
        // Output:
        // User(...) -> Pet(...)
    }
} 
```

预加载允许同时查询多个关联 (包括嵌套)，也可以对其结果进行筛选、排序或限制数量。 例如：

```go
admins, err := client.User.
    Query().
    Where(user.Admin(true)).
    // 填充与 `admins` 相关联的 `pets`
    WithPets().
    // 填充与 `admins` 相关联的前5个 `groups`
    WithGroups(func(q *ent.GroupQuery) {
        q.Limit(5)              // 限量5个
        q.WithUsers()           // 为 `groups` 填充它们的 `users`.
    }).
    All(ctx)
if err != nil {
    return err
}

// 返回的结果类似于:
//
//  [
//      User {
//          ID:   1,
//          Name: "admin1",
//          Edges: {
//              Pets:   [Pet(...), ...]
//              Groups: [
//                  Group {
//                      ID:   7,
//                      Name: "GitHub",
//                      Edges: {
//                          Users: [User(...), ...]
//                          ...
//                      }
//                  }
//              ]
//          }
//      },
//      ...
//  ]
//
for _, admin := range admins {
    for _, p := range admin.Edges.Pets {
        fmt.Printf("Admin(%v) -> Pet(%v)\n", u.ID, p.ID)
        // Output:
        // Admin(...) -> Pet(...)
    }
    for _, g := range admin.Edges.Groups {
        for _, u := range g.Edges.Users {
            fmt.Printf("Admin(%v) -> Group(%v) -> User(%v)\n", u.ID, g.ID, u.ID)
            // Output:
            // Admin(...) -> Group(...) -> User(...)
        }
    }
} 
```

## API

每个查询构造器，都会为每一条边生成形如 `With<E>(...func(<N>Query))` 的方法。 `<E>` 是边的名称 (如, `WithGroups`)，`<N>` 是边的类型 (如, `GroupQuery`).

请注意，只有 SQL后端 支持此功能。

## 实现

由于一个Ent查询可以加载多个边，无法在一个单一的`JOIN`操作中加载所有关联。 因此，Ent加载每个关联时会进行额外的操作。 在未来的版本中会逐步进行优化。
