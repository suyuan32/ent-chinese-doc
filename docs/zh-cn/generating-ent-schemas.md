## 概述

为了方便创建以编程方式生成 ent.Schema 的工具，ent 支持使用 entgo.io/contrib/schemast 包操作 schema/ 目录。

## API

### 载入

为了操作现有的架构目录，我们必须首先将其加载到 `schemast.Context` 对象中：

```go
package main

import (
    "fmt"
    "log"

    "entgo.io/contrib/schemast"
)

func main() {
    ctx, err := schemast.Load("./ent/schema")
    if err != nil {
        log.Fatalf("failed: %v", err)
    }
    if ctx.HasType("user") {
        fmt.Println("schema directory contains a schema named User!")
    }
}
```

### 输出

要将我们的上下文打印回目标目录，请使用 `schemast.Print`：

```go
package main

import (
    "log"

    "entgo.io/contrib/schemast"
)

func main() {
    ctx, err := schemast.Load("./ent/schema")
    if err != nil {
        log.Fatalf("failed: %v", err)
    }
    // A no-op since we did not manipulate the Context at all.
    if err := schemast.Print("./ent/schema"); err != nil {
        log.Fatalf("failed: %v", err)
    }
}
```

### 变更器

要改变“ent/schema”目录，我们可以使用“schemast.Mutate”，它需要一个“schemast.Mutator”列表来应用于上下文：

```go
package schemast

// Mutator changes a Context.
type Mutator interface {
    Mutate(ctx *Context) error
}
```

目前，只实现了一种类型的 `schemast.Mutator`，`UpsertSchema`：

```go
package schemast

// UpsertSchema implements Mutator. UpsertSchema will add to the Context the type named
// Name if not present and rewrite the type's Fields, Edges, Indexes and Annotations methods.
type UpsertSchema struct {
    Name        string
    Fields      []ent.Field
    Edges       []ent.Edge
    Indexes     []ent.Index
    Annotations []schema.Annotation
}
```

使用它：

```go
package main

import (
    "log"

    "entgo.io/contrib/schemast"
    "entgo.io/ent"
    "entgo.io/ent/schema/field"
)

func main() {
    ctx, err := schemast.Load("./ent/schema")
    if err != nil {
        log.Fatalf("failed: %v", err)
    }
    mutations := []schemast.Mutator{
        &schemast.UpsertSchema{
            Name: "User",
            Fields: []ent.Field{
                field.String("name"),
            },
        },
        &schemast.UpsertSchema{
            Name: "Team",
            Fields: []ent.Field{
                field.String("name"),
            },
        },
    }
    err = schemast.Mutate(ctx, mutations...)
    if err := ctx.Print("./ent/schema"); err != nil {
        log.Fatalf("failed: %v", err)
    }
}
```

运行这个程序后，可以看淡schema目录中存在两个新文件：`user.go` 和 `team.go`：

```go
// user.go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema"
    "entgo.io/ent/schema/field"
)

type User struct {
    ent.Schema
}

func (User) Fields() []ent.Field {
    return []ent.Field{field.String("name")}
}
func (User) Edges() []ent.Edge {
    return nil
}
func (User) Annotations() []schema.Annotation {
    return nil
}
```

```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema"
    "entgo.io/ent/schema/field"
)

type Team struct {
    ent.Schema
}

func (Team) Fields() []ent.Field {
    return []ent.Field{field.String("name")}
}
func (Team) Edges() []ent.Edge {
    return nil
}
func (Team) Annotations() []schema.Annotation {
    return nil
}
```

### 使用边(edge)

边在 ent 中这样定义：

```go
edge.To("edge_name", OtherSchema.Type)
```

这种语法依赖于这样一个事实，即当我们定义边缘时，`OtherSchema` 结构已经存在，因此我们可以引用它的 `Type` 方法。 当我们以编程方式生成schema时，显然我们需要在类型定义存在之前以某种方式向代码生成器描述边缘。 为此，您可以执行以下操作：
```go
type placeholder struct {
    ent.Schema
}

func withType(e ent.Edge, typeName string) ent.Edge {
    e.Descriptor().Type = typeName
    return e
}

func newEdgeTo(edgeName, otherType string) ent.Edge {
    // we pass a placeholder type to the edge constructor:
    e := edge.To(edgeName, placeholder.Type)
    // then we override the other type's name directly on the edge descriptor: 
    return withType(e, otherType)
}
```

## 例子

`protoc-gen-ent` ([doc](https://github.com/ent/contrib/tree/master/entproto/cmd/protoc-gen-ent)) 是一个以编程方式生成 `ent. Schema 来自 .proto 文件，它使用 schemast 来操作目标 schema 目录。 要查看如何，[阅读源代码](https://github.com/ent/contrib/blob/master/entproto/cmd/protoc-gen-ent/main.go#L34)。

## 注意事项

`schemast` 仍处于实验阶段，API 将来可能会发生变化。 此外，一小部分“ent.Field”定义 API 目前不受支持，要查看不受支持功能的完整列表，请参阅 [源代码](https://github.com/ent/contrib/blob/aed7a43a3e54550c1dd9a1a066ce1236b4bae56c/schemast/field.go#L158)。
