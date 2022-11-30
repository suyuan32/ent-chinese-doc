此框架提供了一系列代码生成特性，可以自行选择使用。

## 用法

特性开关可以通过 CLI 标志或作为参数提供给 `gen` 包。

#### CLI

```console
go run -mod=mod entgo.io/ent/cmd/ent generate --feature privacy,entql ./ent/schema
```

#### Go

```go
// +build ignore

package main

import (
    "log"
    "text/template"

    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
)

func main() {
    err := entc.Generate("./schema", &gen.Config{
        Features: []gen.Feature{
            gen.FeaturePrivacy,
            gen.FeatureEntQL,
        },
        Templates: []*gen.Template{
            gen.MustParse(gen.NewTemplate("static").
                Funcs(template.FuncMap{"title": strings.ToTitle}).
                ParseFiles("template/static.tmpl")),
        },
    })
    if err != nil {
        log.Fatalf("running ent codegen: %v", err)
    }
}
```

## 特性列表

### 自动解决合并冲突

`schema/snapshot` 选项告诉 `entc` (ent codegen) 将最新模式的快照存储在内部包中，并在无法构建用户模式时使用它来自动解决合并冲突。

可以使用“--feature schema/snapshot”标志将此选项添加到项目中，但请参阅 [ent/ent/issues/852](https://github.com/ent/ent/issues/852) 以 获得更多关于它的背景信息。

### 隐私层

隐私层允许为数据库中实体的查询和变更配置隐私策略。

可以使用 `--feature privacy` 标志将此选项添加到项目中，您可以在 [privacy](privacy.md) 文档中了解更多信息。

### EntQL过滤

`entql` 选项在运行时为不同的查询构建器提供通用和动态过滤功能。

可以使用 `--feature entql` 标志将此选项添加到项目中，您可以在 [privacy](privacy.md#multi-tenancy) 文档中了解更多信息。

### 命名边（Named Edges）

`namedges` 选项提供了一个 API，用于使用自定义名称预加载边缘。

可以使用 `--feature namedges` 标志将此选项添加到项目中，您可以在 [Eager Loading](eager-load.mdx) 文档中了解更多信息。

### Schema 配置

`sql/schemaconfig` 选项允许您将备用 SQL 数据库名称传递给模型。 当您的模型并非都在一个数据库下并且分布在不同的schema中时，这很有用。

可以使用 --feature sql/schemaconfig 标志将此选项添加到项目中。 生成代码后，您现在可以使用一个新选项：

```go
c, err := ent.Open(dialect, conn, ent.AlternateSchema(ent.SchemaConfig{
    User: "usersdb",
    Car: "carsdb",
}))
c.User.Query().All(ctx) // SELECT * FROM `usersdb`.`users`
c.Car.Query().All(ctx)  // SELECT * FROM `carsdb`.`cars`
```

### 行级锁

`sql/lock` 选项允许使用 SQL `SELECT ... FOR {UPDATE | 分享}`语法。

可以使用 --feature sql/lock 标志将此选项添加到项目中。

```go
tx, err := client.Tx(ctx)
if err != nil {
    log.Fatal(err)
}

tx.Pet.Query().
    Where(pet.Name(name)).
    ForUpdate().
    Only(ctx)

tx.Pet.Query().
    Where(pet.ID(id)).
    ForShare(
        sql.WithLockTables(pet.Table),
        sql.WithLockAction(sql.NoWait),
    ).
    Only(ctx)
```

### 自定义 SQL 修饰符

`sql/modifier` 选项允许向构建器添加自定义 SQL 修饰符，并在语句执行之前对其进行修改。

可以使用 --feature sql/modifier 标志将此选项添加到项目中。

#### Modify Example 1

```go
client.Pet.
    Query().
    Modify(func(s *sql.Selector) {
        s.Select("SUM(LENGTH(name))")
    }).
    IntX(ctx)
```

上面的代码将产生以下 SQL 查询：

```sql
SELECT SUM(LENGTH(name)) FROM `pet`
```

#### 修改示例 2

```go
var p1 []struct {
    ent.Pet
    NameLength int `sql:"length"`
}

client.Pet.Query().
    Order(ent.Asc(pet.FieldID)).
    Modify(func(s *sql.Selector) {
        s.AppendSelect("LENGTH(name)")
    }).
    ScanX(ctx, &p1)
```

上面的代码将产生以下 SQL 查询：

```sql
SELECT `pet`.*, LENGTH(name) FROM `pet` ORDER BY `pet`.`id` ASC
```

#### 修改示例 3

```go
var v []struct {
    Count     int       `json:"count"`
    Price     int       `json:"price"`
    CreatedAt time.Time `json:"created_at"`
}

client.User.
    Query().
    Where(
        user.CreatedAtGT(x),
        user.CreatedAtLT(y),
    ).
    Modify(func(s *sql.Selector) {
        s.Select(
            sql.As(sql.Count("*"), "count"),
            sql.As(sql.Sum("price"), "price"),
            sql.As("DATE(created_at)", "created_at"),
        ).
        GroupBy("DATE(created_at)").
        OrderBy(sql.Desc("DATE(created_at)"))
    }).
    ScanX(ctx, &v)
```

上面的代码将产生以下 SQL 查询：

```sql
SELECT
    COUNT(*) AS `count`,
    SUM(`price`) AS `price`,
    DATE(created_at) AS `created_at`
FROM
    `users`
WHERE
    `created_at` > x AND `created_at` < y
GROUP BY
    DATE(created_at)
ORDER BY
    DATE(created_at) DESC
```

#### 修改示例 4

```go
var gs []struct {
    ent.Group
    UsersCount int `sql:"users_count"`
}

client.Group.Query().
    Order(ent.Asc(group.FieldID)).
    Modify(func(s *sql.Selector) {
        t := sql.Table(group.UsersTable)
        s.LeftJoin(t).
            On(
                s.C(group.FieldID),
                t.C(group.UsersPrimaryKey[1]),
            ).
            // Append the "users_count" column to the selected columns.
            AppendSelect(
                sql.As(sql.Count(t.C(group.UsersPrimaryKey[1])), "users_count"),
            ).
            GroupBy(s.C(group.FieldID))
    }).
    ScanX(ctx, &gs)
```

上面的代码将产生以下 SQL 查询：

```sql
SELECT
    `groups`.*,
    COUNT(`t1`.`group_id`) AS `users_count`
FROM
    `groups` LEFT JOIN `user_groups` AS `t1`
ON
    `groups`.`id` = `t1`.`group_id`
GROUP BY
    `groups`.`id`
ORDER BY
    `groups`.`id` ASC
```


#### 修改示例 5

```go
client.User.Update().
    Modify(func(s *sql.UpdateBuilder) {
        s.Set(user.FieldName, sql.Expr(fmt.Sprintf("UPPER(%s)", user.FieldName)))
    }).
    ExecX(ctx)
```

上面的代码将产生以下 SQL 查询：

```sql
UPDATE `users` SET `name` = UPPER(`name`)
```

#### 修改示例 6

```go
client.User.Update().
    Modify(func(u *sql.UpdateBuilder) {
        u.Set(user.FieldID, sql.ExprFunc(func(b *sql.Builder) {
            b.Ident(user.FieldID).WriteOp(sql.OpAdd).Arg(1)
        }))
        u.OrderBy(sql.Desc(user.FieldID))
    }).
    ExecX(ctx)
```

上面的代码将产生以下 SQL 查询：

```sql
UPDATE `users` SET `id` = `id` + 1 ORDER BY `id` DESC
```

#### 修改示例 7

上面的代码将产生以下 SQL 查询：

```go
client.User.Update().
    Modify(func(u *sql.UpdateBuilder) {
        sqljson.Append(u, user.FieldTags, []string{"tag1", "tag2"}, sqljson.Path("values"))
    }).
    ExecX(ctx)
```

上面的代码将产生以下 SQL 查询：

```sql
UPDATE `users` SET `tags` = CASE
    WHEN (JSON_TYPE(JSON_EXTRACT(`tags`, '$.values')) IS NULL OR JSON_TYPE(JSON_EXTRACT(`tags`, '$.values')) = 'NULL')
    THEN JSON_SET(`tags`, '$.values', JSON_ARRAY(?, ?))
    ELSE JSON_ARRAY_APPEND(`tags`, '$.values', ?, '$.values', ?) END
    WHERE `id` = ?
```

### SQL原始API

`sql/execquery` 选项允许使用底层驱动程序的 `ExecContext`/`QueryContext` 方法执行语句。 有关完整文档，请参阅：[DB.ExecContext](https://pkg.go.dev/database/sql#DB.ExecContext) 和 [DB.QueryContext](https://pkg.go.dev/database/ sql#DB.QueryContext）。

```go
// From ent.Client.
if _, err := client.ExecContext(ctx, "TRUNCATE t1"); err != nil {
    return err
}

// From ent.Tx.
tx, err := client.Tx(ctx)
if err != nil {
    return err
}
if err := tx.User.Create().Exec(ctx); err != nil {
    return err
}
if _, err := tx.ExecContext("SAVEPOINT user_created"); err != nil {
    return err
}
// ...
```

> 注意 使用 `ExecContext`/`QueryContext` 执行的语句不会通过 Ent，并且可能会跳过应用程序中的基础层，例如挂钩、隐私（授权）和验证器。 

### 更新插入(Upsert)

`sql/upsert` 选项允许使用 SQL `ON CONFLICT` / `ON DUPLICATE KEY` 语法配置 upsert 和 bulk-upsert 逻辑。 如需完整文档，请转到 [Upsert API](crud.mdx#upsert-one)。

可以使用 --feature sql/upsert 标志将此选项添加到项目中。

```go
// Use the new values that were set on create.
id, err := client.User.
    Create().
    SetAge(30).
    SetName("Ariel").
    OnConflict().
    UpdateNewValues().
    ID(ctx)

// In PostgreSQL, the conflict target is required.
err := client.User.
    Create().
    SetAge(30).
    SetName("Ariel").
    OnConflictColumns(user.FieldName).
    UpdateNewValues().
    Exec(ctx)

// Bulk upsert is also supported.
client.User.
    CreateBulk(builders...).
    OnConflict(
        sql.ConflictWhere(...),
        sql.UpdateWhere(...),
    ).
    UpdateNewValues().
    Exec(ctx)

// INSERT INTO "users" (...) VALUES ... ON CONFLICT WHERE ... DO UPDATE SET ... WHERE ...
```
