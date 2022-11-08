`Mixin` 允许您创建可复用的 `ent.Schema` 代码，用与合成添加到其他 schemas

`ent.Mixin` 接口如下：

```go
type Mixin interface {
    // Fields 数组的返回值会被添加到 schema 中。
    Fields() []Field
    // Edges 数组返回值会被添加到 schema 中。
    Edges() []Edge
    // Indexes 数组的返回值会被添加到 schema 中。
    Indexes() []Index
    // Hooks 数组的返回值会被添加到 schema 中。
    // 请注意，mixin 的钩子会在 schema 的钩子之前被执行。
    Hooks() []Hook
    // Policy 数组的返回值会被添加到 schema 中。
    //请注意，mixin（混入）的 policy（策略）会在 schema 的 policy 之前被执行。
    Policy() Policy
    // Annotations 方法返回要添加到 Schema 中的注解列表。
    Annotations() []schema.Annotation
}
```

## 示例

`Mixin` 多用于：将你 Schema 中常用的字段抽象出来。

```go
package schema

import (
    "time"

    "entgo.io/ent"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/mixin"
)

// -------------------------------------------------
// Mixin definition

// TimeMixin implements the ent.Mixin for sharing
// time fields with package schemas.
type TimeMixin struct{
    // We embed the `mixin.Schema` to avoid
    // implementing the rest of the methods.
    mixin.Schema
}

func (TimeMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Time("created_at").
            Immutable().
            Default(time.Now),
        field.Time("updated_at").
            Default(time.Now).
            UpdateDefault(time.Now),
    }
}

// DetailsMixin implements the ent.Mixin for sharing
// entity details fields with package schemas.
type DetailsMixin struct{
    // We embed the `mixin.Schema` to avoid
    // implementing the rest of the methods.
    mixin.Schema
}

func (DetailsMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age").
            Positive(),
        field.String("name").
            NotEmpty(),
    }
}

// -------------------------------------------------
// Schema definition

// User schema mixed-in the TimeMixin and DetailsMixin fields and therefore
// has 5 fields: `created_at`, `updated_at`, `age`, `name` and `nickname`.
type User struct {
    ent.Schema
}

func (User) Mixin() []ent.Mixin {
    return []ent.Mixin{
        TimeMixin{},
        DetailsMixin{},
    }
}

func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("nickname").
            Unique(),
    }
}

// Pet schema mixed-in the DetailsMixin fields and therefore
// has 3 fields: `age`, `name` and `weight`.
type Pet struct {
    ent.Schema
}

func (Pet) Mixin() []ent.Mixin {
    return []ent.Mixin{
        DetailsMixin{},
    }
}

func (Pet) Fields() []ent.Field {
    return []ent.Field{
        field.Float("weight"),
    }
}
```

## 内置Mixin

`mixin`包提供了一些内置的mixin，它们可以用于在schema中添加`create_time`和`update_time` 字段。

若要使用它们，请将 `mixin.Time` mixin 添加到您的schema，如下：
```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/mixin"
)

type Pet struct {
    ent.Schema
}

func (Pet) Mixin() []ent.Mixin {
    return []ent.Mixin{
        mixin.Time{},
        // Or, mixin.CreateTime only for create_time
        // and mixin.UpdateTime only for update_time.
    }
}
```
