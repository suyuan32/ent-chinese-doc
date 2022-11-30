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

> 注意有些Schema名称 (如 `Client`) 是不可用的，详情查看 [internal use](https://pkg.go.dev/entgo.io/ent/entc/gen#ValidSchemaName). 您可以使用上述注释来规避保留名称 [点此查看](schema-annotations.md#custom-table-name). 

## 它只是又一个 ORM

如果你习惯用边来定义关系，Ent可以很好的满足你的需求。 查看例子快速开始 [Edges](./zh-cn/schema-edges.md).
