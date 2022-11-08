## 概述

Schema 描述了图中一个实体类型的定义，如 `User` 或 `Group`， 并可以包含以下配置：
- 实体的字段 (或属性)，如：`User` 的姓名或年龄。
- 实体的边 (或关系)。如：`User` 所属用户组，或 `User` 的朋友。
- 数据库相关的配置，如：索引或唯一索引。

下面是 Schema 的一个示例：

```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/edge"
    "entgo.io/ent/schema/index"
)

type User struct {
    ent.Schema
}

func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age"),
        field.String("name"),
        field.String("nickname").
            Unique(),
    }
}

func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("groups", Group.Type),
        edge.To("friends", User.Type),
    }
}

func (User) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("age", "name").
            Unique(),
    }
}
```

实体 Schema 通常存储在你项目根目录的 `ent/schema` 目录内， 且可以通过以下的 `entc` 命令生成：

```console
go run -mod=mod entgo.io/ent/cmd/ent init User Group
```

:::note Please note, that some schema names (like `Client`) are not available due to [internal use](https://pkg.go.dev/entgo.io/ent/entc/gen#ValidSchemaName). You can circumvent reserved names by using an annotation as mentioned [here](schema-annotations.md#custom-table-name). :::

## 它只是又一个 ORM

If you are used to the definition of relations over edges, that's fine. The modeling is the same. You can model with `ent` whatever you can model with other traditional ORMs. There are many examples in this website that can help you get started in the [Edges](schema-edges) section.
