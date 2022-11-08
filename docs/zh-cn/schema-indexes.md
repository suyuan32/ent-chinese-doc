## 多个字段

索引可以在一个或多个字段上配置以提高数据检索速度，也可以定义其唯一性。

```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/index"
)

// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

func (User) Indexes() []ent.Index {
    return []ent.Index{
        // 非唯一约束索引
        index.Fields("field1", "field2"),
        // 唯一约束索引
        index.Fields("first_name", "last_name").
            Unique(),
    }
}
```

请注意，如果要为单个字段设置唯一约束，请在字段生成器上使用 `Unique` 方法，如下：

```go
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("phone").
            Unique(),
    }
}
```

## 边上的索引

索引可以为字段和边的组合进行配置。 主要用法是在特定关系下设置字段的唯一性。 让我们来看一个例子：

![er-city-streets](https://entgo.io/images/assets/er_city_streets.png)

在上面的示例中，我们有一个带了许多 `Street` 的 `City`，并且我们想设置每个城市的街道名称都是唯一的。

`ent/schema/city.go`
```go
// City holds the schema definition for the City entity.
type City struct {
    ent.Schema
}

// Fields of the City.
func (City) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
    }
}

// Edges of the City.
func (City) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("streets", Street.Type),
    }
}
```

`ent/schema/street.go`
```go
// Street holds the schema definition for the Street entity.
type Street struct {
    ent.Schema
}

// Fields of the Street.
func (Street) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
    }
}

// Edges of the Street.
func (Street) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("city", City.Type).
            Ref("streets").
            Unique(),
    }
}

// Indexes of the Street.
func (Street) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("name").
            Edges("city").
            Unique(),
    }
}
```

`example.go`
```go
func Do(ctx context.Context, client *ent.Client) error {
    // 和 `Save`不同，当出现错误是 `SaveX` 抛出panic。
    tlv := client.City.
        Create().
        SetName("TLV").
        SaveX(ctx)
    nyc := client.City.
        Create().
        SetName("NYC").
        SaveX(ctx)
    // Add a street "ST" to "TLV".
    client.Street.
        Create().
        SetName("ST").
        SetCity(tlv).
        SaveX(ctx)
    // This operation fails because "ST"
    // was already created under "TLV".
    if err := client.Street.
        Create().
        SetName("ST").
        SetCity(tlv).
        Exec(ctx); err == nil {
        return fmt.Errorf("expecting creation to fail")
    }
    // Add a street "ST" to "NYC".
    client.Street.
        Create().
        SetName("ST").
        SetCity(nyc).
        SaveX(ctx)
    return nil
}
```

完整示例请参阅 [GitHub](https://github.com/ent/ent/tree/master/examples/edgeindex).

## 边(Edge)上索引

目前 `Edges` 字段后面通常跟着 `Fields` 字段. 然而复合索引需要考虑顺序， 查看 [Edge Fields](schema-edges#edge-field)了解更多.

```go
// Card holds the schema definition for the Card entity.
type Card struct {
    ent.Schema
}
// Fields of the Card.
func (Card) Fields() []ent.Field {
    return []ent.Field{
        field.String("number").
            Optional(),
        field.Int("owner_id").
            Optional(),
    }
}
// Edges of the Card.
func (Card) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("owner", User.Type).
            Ref("card").
            Field("owner_id").
            Unique(),
    }
}
// Indexes of the Card.
func (Card) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("owner_id", "number"),
    }
}
```

## 方言支持 (Dialect Support)

Dialect 功能需要配合注解 [annotations](schema-annotations.md). 例如为了实现在 Mysql 中的 [索引前缀](https://dev.mysql.com/doc/refman/8.0/en/column-indexes.html#column-indexes-prefix)， 需要如下配置 :

```go
// Indexes of the User.
func (User) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("description").
            Annotations(entsql.Prefix(128)),
        index.Fields("c1", "c2", "c3").
            Annotation(
                entsql.PrefixColumn("c1", 100),
                entsql.PrefixColumn("c2", 200),
            )
    }
}
```

上面的代码将会生成如下 Sql 语句:

```sql
CREATE INDEX `users_description` ON `users`(`description`(128))

CREATE INDEX `users_c1_c2_c3` ON `users`(`c1`(100), `c2`(200), `c3`)
```

## Atlas 支持（Atlas Support）

从 v0.10 开始 Ent 使用 Atlas 进行迁移  [Atlas](https://github.com/ariga/atlas). 它将提供更多对索引的配置, 配置他们的类型或顺序.

```go
func (User) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("c1").
            Annotations(entsql.Desc()),
        index.Fields("c1", "c2", "c3").
            Annotation(entsql.DescColumns("c1", "c2")),
        index.Fields("c4").
            Annotations(entsql.IndexType("HASH")),
        // 在 MYSQL 启用 FULLTEXT 搜索 ,
        // 和 GIN 在 PostgreSQL 中.
        index.Fields("c5").
            Annotations(
                entsql.IndexTypes(map[string]string{
                    dialect.MySQL:    "FULLTEXT",
                    dialect.Postgres: "GIN",
                }),
            ),
        // 对于 PostgreSQL, 我们可以包含索引
        // non-key 字段.
        index.Fields("workplace").
            Annotations(
                entsql.IncludeColumns("address"),
            ),
        // 定义部分索引在 SQLite 和 PostgreSQL 中.
        index.Fields("nickname").
            Annotations(
                entsql.IndexWhere("active"),
            ),
    }
}
```

上面的代码将生成如下SQL语句:

```sql
CREATE INDEX `users_c1` ON `users` (`c1` DESC)

CREATE INDEX `users_c1_c2_c3` ON `users` (`c1` DESC, `c2` DESC, `c3`)

CREATE INDEX `users_c4` ON `users` USING HASH (`c4`)

-- MySQL only.
CREATE FULLTEXT INDEX `users_c5` ON `users` (`c5`)

-- PostgreSQL only.
CREATE INDEX "users_c5" ON "users" USING GIN ("c5")

-- include index-only scan on PostgreSQL.
CREATE INDEX "users_workplace" ON "users" ("workplace") INCLUDE ("address")

-- Define partial index on SQLite and PostgreSQL.
CREATE INDEX "user_nickname" ON "users" ("nickname") WHERE "active"
```


## 存储名称 （Storage Key）

类似 Fields, 自定义索引名称使用 `StorageKey` 方法. 它将会映射到数据库中.

```go
func (User) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("field1", "field2").
            StorageKey("custom_index"),
    }
}
```
