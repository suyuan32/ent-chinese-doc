结构注解(Schema annotations) 允许附加元数据到结构对象(例如字段和边) 上面，并且将元数据注入到外部模板中。 注解是一种Go类型，它能进行JSON序列化(例如 struct, map 或 slice)，并且需要实现[Annotation](https://pkg.go.dev/entgo.io/ent/schema?tab=doc#Annotation)接口。

内置注解能够配置不同的存储驱动(例如 SQL)，控制代码生成输出。

## 自定义表名

使用 `entsql` 注解的类型, 可以自定义表名，如下所示：

```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/dialect/entsql"
    "entgo.io/ent/schema"
    "entgo.io/ent/schema/field"
)

// User类型持有用户实体的结构(schema)定义
type User struct {
    ent.Schema
}

// 用户实体的注解
func (User) Annotations() []schema.Annotation {
    return []schema.Annotation{
        entsql.Annotation{Table: "Users"},
    }
}

// 用户实体的字段
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age"),
        field.String("name"),
    }
}
```

## 外键配置

Ent允许对外键的创建进行定制，并且为`ON DELETE`子句提供[referential action](https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html#foreign-key-referential-actions)。

```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/dialect/entsql"
    "entgo.io/ent/schema/edge"
    "entgo.io/ent/schema/field"
)

// User类型持有用户实体的结构(schema)定义
type User struct {
    ent.Schema
}

// 用户实体的字段
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            Default("Unknown"),
    }
}

// 用户实体的关系
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("posts", Post.Type).
            Annotations(entsql.Annotation{
                OnDelete: entsql.Cascade,
            }),
    }
}
```

上面的示例配置了外键，将父表的删除操作关联到子表，对子表中匹配的数据也进行删除。
