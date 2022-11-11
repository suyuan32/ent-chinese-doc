正如 [介绍](code-gen.md) 部分所述，对 schemas 运行 `ent` 命令， 将生成以下资源：

- `Client` 和 `Tx` 对象用于与图的交互。
- 每个schema对应的增删改查生成器， 查看 [CRUD](crud.md) 了解更多信息。
- 每个schema的实体对象(Go结构体)。
- 含常量和查询条件的包，用于与生成器交互。
- 用于数据迁移的`migrate`包. 查看 [迁移](migrate.md) 获取更多信息。

## 创建新的客户端

**MySQL**

```go
package main

import (
    "log"

    "<project>/ent"

    _ "github.com/go-sql-driver/mysql"
)

func main() {
    client, err := ent.Open("mysql", "<user>:<pass>@tcp(<host>:<port>)/<database>?parseTime=True")
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
}
```

**PostgreSQL**

```go
package main

import (
    "log"

    "<project>/ent"

    _ "github.com/lib/pq"
)

func main() {
    client, err := ent.Open("postgres","host=<host> port=<port> user=<user> dbname=<database> password=<pass>")
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
}
```

**SQLite**

```go
package main

import (
    "log"

    "<project>/ent"

    _ "github.com/mattn/go-sqlite3"
)

func main() {
    client, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
}
```


**Gremlin (AWS Neptune)**

```go
package main

import (
    "log"

    "<project>/ent"
)

func main() {
    client, err := ent.Open("gremlin", "http://localhost:8182")
    if err != nil {
        log.Fatal(err)
    }
}
```

## 创建一个实体

通过 **Save** 保存一个用户.

```go
a8m, err := client.User.    // UserClient.
    Create().               // 用户创建构造器
    SetName("a8m").         // 设置字段的值
    SetNillableAge(age).    // 忽略nil检查
    AddGroups(g1, g2).      // 添加多个边
    SetSpouse(nati).        // 设置单个边
    Save(ctx)               // 创建并返回
```

通过 **SaveX** 保存一个宠物; 和 **Save** 不一样， **SaveX** 在出错时 panic。

```go
pedro := client.Pet.    // PetClient.
    Create().           // 宠物创建构造器
    SetName("pedro").   // 设置字段的值
    SetOwner(a8m).      // 设置主人 (唯一的边)
    SaveX(ctx)          // 创建并返回
```

## 批量创建

通过 **Save** 批量保存宠物。

```go
names := []string{"pedro", "xabi", "layla"}
bulk := make([]*ent.PetCreate, len(names))
for i, name := range names {
    bulk[i] = client.Pet.Create().SetName(name).SetOwner(a8m)
}
pets, err := client.Pet.CreateBulk(bulk...).Save(ctx)
```

## 更新单个实体

更新一个从数据库内的实体。

```go
a8m, err = a8m.Update().    // 用户更新构造器
    RemoveGroup(g2).        // 移除特定的边
    ClearCard().            // 清空唯一的边
    SetAge(30).             // 设置字段的值
    Save(ctx)               // 保存并返回
```


## 通过ID更新

```go
pedro, err := client.Pet.   // PetClient.
    UpdateOneID(id).        // 宠物更新构造器
    SetName("pedro").       // 设置名字字段
    SetOwnerID(owner).      // 通过ID设置唯一的边
    Save(ctx)               // 保存并返回
```

## 批量更新

通过断言筛选。

```go
n, err := client.User.          // UserClient.
    Update().                   // 宠物更新构造器
    Where(                      //
        user.Or(                // (age >= 30 OR name = "bar") 
            user.AgeGT(30),     //
            user.Name("bar"),   // AND
        ),                      //  
        user.HasFollowers(),    // UserHasFollowers()  
    ).                          //
    SetName("foo").             // 设置名字字段
    Save(ctx)                   // 执行并返回
```

通过边上的断言筛选。

```go
n, err := client.User.          // UserClient.
    Update().                   // 宠物更新构造器
    Where(                      // 
        user.HasFriendsWith(    // UserHasFriendsWith (
            user.Or(            //   age = 20
                user.Age(20),   //      OR
                user.Age(30),   //   age = 30
            )                   // )
        ),                      //
    ).                          //
    SetName("a8m").             // 设置名字字段
    Save(ctx)                   // 执行并返回
```

## 合并单个实体

Ent 支持 [合并](https://en.wikipedia.org/wiki/Merge_(SQL)) 记录，使用 [`sql/upsert`](features.md#upsert) 功能标识。

```go
err := client.User.
    Create().
    SetAge(30).
    SetName("Ariel").
    OnConflict().
    // 使用新设定的新值。
    UpdateNewValues().
    Exec(ctx)

id, err := client.User。
    Create().
    SetAge(30).
    SetName("Ariel").
    OnConflict().
    // 使用创建时设定的“age”。
    UpdateAge().
    // 在发生冲突的情况下设置不同的“name”。
    SetName("Mashraki").
    ID(ctx)

// 自定义 UPDATE 子句。
err := client.User.
    Create().
    SetAge(30).
    SetName("Ariel").
    OnConflict().
    UpdateNewValues().
    // 使用自定义更新覆盖一些字段。
    Update(func(u *ent.UserUpsert) {
        u.SetAddress("localhost")
        u.AddCount(1)
        u.ClearPhone()
    }).
    Exec(ctx)
```

在 PostgreSQL 中，需要设置 [冲突目标](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)：

```go
// 使用 fluent API 设置列名。
err := client.User.
    Create().
    SetName("Ariel").
    OnConflictColumns(user.FieldName).
    UpdateNewValues().
    Exec(ctx)

// 使用 SQL API 设置列名。
err := client.User.
    Create().
    SetName("Ariel").
    OnConflict(
        sql.ConflictColumns(user.FieldName),    
    ).
    UpdateNewValues().
    Exec(ctx)

// 使用 SQL API 设置约束名称。
err := client.User.
    Create().
    SetName("Ariel").
    OnConflict(
        sql.ConflictConstraint(constraint), 
    ).
    UpdateNewValues().
    Exec(ctx)
```

使用 SQL API自定义执行的语句：

```go
id, err := client.User.
    Create().
    OnConflict(
        sql.ConflictColumns(...),
        sql.ConflictWhere(...),
        sql.UpdateWhere(...),
    ).
    Update(func(u *ent.UserUpsert) {
        u.SetAge(30)
        u.UpdateName()
    }).
    ID(ctx)

// INSERT INTO "users" (...) VALUES (...) ON CONFLICT WHERE ... DO UPDATE SET ... WHERE ...
```

:::info 由于 upsert API 是使用 `ON CONFLICT` 子句（和 MySQL 中的 `ON DUPLICATE KEY`）实现的， Ent 只对数据库执行一条语句，因此创建的 [hooks](hooks.md)应该只针对此类操作。 :::

## 合并多个实体

```go
err := client.User.             // UserClient
    CreateBulk(builders...).    // User bulk create.
    OnConflict().               // User bulk upsert.
    UpdateNewValues().          // Use the values that were set on create in case of conflict.
    Exec(ctx)                   // Execute the statement.
```

## 在图中查询

查询所有拥有关注者的用户。
```go
users, err := client.User.      // UserClient.
    Query().                    // User query builder.
    Where(user.HasFollowers()). // filter only users with followers.
    All(ctx)                    // query and return.
```

获取特定用户的所有关注者，并从图的某一个节点开始遍历
```go
users, err := a8m.
    QueryFollowers().
    All(ctx)
```

查询某用户的关注者的所有宠物。
```go
users, err := a8m.
    QueryFollowers().
    QueryPets().
    All(ctx)
```

没有评论的文章数量
```go
n, err := client.Post.
    Query().
    Where(
        post.Not(
            post.HasComments(), 
        )
    ).
    Count(ctx)
```

可以在 [下一节](traversals.md) 中找到更高级的遍历方式。

## 字段选择

获取所有宠物名字

```go
names, err := client.Pet.
    Query().
    Select(pet.FieldName).
    Strings(ctx)
```

获取所有名字唯一的宠物名字

```go
names, err := client.Pet.
    Query().
    Unique(true).
    Select(pet.FieldName).
    Strings(ctx)
```

统计名字唯一的宠物数量

```go
n, err := client.Pet.
    Query().
    Unique(true).
    Select(pet.FieldName).
    Count(ctx)
```

获取对象及关联对象的部分字段。获取宠物及其主人，但是只选择和使用 `ID` 和 `Name` 字段。

```go
pets, err := client.Pet.
    Query().
    Select(pet.FieldName).
    WithOwner(func (q *ent.UserQuery) {
        q.Select(user.FieldName)
    }).
    All(ctx)
```

扫描所有宠物的名字和年龄到自定义结构。

```go
var v []struct {
    Age  int    `json:"age"`
    Name string `json:"name"`
}
err := client.Pet.
    Query().
    Select(pet.FieldAge, pet.FieldName).
    Scan(ctx, &v)
if err != nil {
    log.Fatal(err)
}
```

更新实体但只返回部分内容。

```go
pedro, err := client.Pet.
    UpdateOneID(id).
    SetAge(9).
    SetName("pedro").
    // Select 允许选择返回实体的一个或多个字段 (列)
    // 默认选择所有 Schema 中定义的所有字段
    Select(pet.FieldName).
    Save(ctx)
```

## 删除一个

删除一个实体。

```go
err := client.User.
    DeleteOne(a8m).
    Exec(ctx)
```

根据ID删除。

```go
err := client.User.
    DeleteOneID(id).
    Exec(ctx)
```

## 批量删除

使用条件批量删除。

```go
_, err := client.File.
    Delete().
    Where(file.UpdatedAtLT(date)).
    Exec(ctx)
```

## 更新或创建 (Upsert)

每个生成的节点类型都有突变类型。 例如，所有的 [`User` 构造器](crud.md#create-an-entity)， 共用同一个生成的 `UserMutation` 对象。 并且，所有构造器类型都实现了通用的 <a target="_blank" href="https://pkg.go.dev/entgo.io/ent?tab=doc#Mutation"> `ent.Mutation` </a> 接口。

如果想要编写通用的代码，例如在 `ent.UserCreate` 和 `ent.UserUpdate` 上使用相同的操作，请使用 `UserMutation` 对象。

```go
func Do() {
    creator := client.User.Create()
    SetAgeName(creator.Mutation())
    updater := client.User.UpdateOneID(id)
    SetAgeName(updater.Mutation())
}

// SetAgeName 设置 age 和 name 对于任何突变
func SetAgeName(m *ent.UserMutation) {
    m.SetAge(32)
    m.SetName("Ariel")
}
```

有时候，你想在不同的类型上执行相同的操作。 对于这种情况，你可以使用通用的 `ent.Mutation` 接口，也可以创建你自己的接口。

```go
func Do() {
    creator1 := client.User.Create()
    SetName(creator1.Mutation(), "a8m")

    creator2 := client.Pet.Create()
    SetName(creator2.Mutation(), "pedro")
}

// SetNamer 包装了两种方法，
// 用于获取和设置突变中的 "name" 字段
type SetNamer interface {
    SetName(string)
    Name() (string, bool)
}

func SetName(m SetNamer, name string) {
    if _, exist := m.Name(); !exist {
        m.SetName(name)
    }
}
```
