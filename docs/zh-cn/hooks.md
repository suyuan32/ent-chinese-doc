---
id: hooks
title: 事件钩子
---

`Hook` 选项允许在操作之前和之后添加自定义逻辑来改变图。

## 变更 (Mutation)

Mutation 操作是一个改变数据库的操作。 例如，将一个新节点添加到图表，删除两个节点之间的边或删除多个节点。

有五种mutation类型：
- `Create` - 在图中创建节点。
- `UpdateOne` - 在图中更新一个节点。 例如，增加字段。
- `Update` - 更新图表中匹配断言的多个节点。
- `DeleteOne` - 从图中删除一个节点。
- `Delete` - 删除所有符合预期的节点。

每个生成的节点类型都有自己的mutation类型。 例如，所有的 [`User` 构造器](crud.mdx#create-an-entity)， 共用同一个生成的 `UserMutation` 对象。

并且，所有构造器类型都实现了通用的 <a target="_blank" href="https://pkg.go.dev/entgo.io/ent?tab=doc#Mutation"> `ent.Mutation` </a> 接口。

## 事件钩子 (Hooks)

钩子函数可以得到 <a target="_blank" href="https://pkg.go.dev/entgo.io/ent?tab=doc#Mutator">`ent.Mutator`</a> 并返回一个mutator。 他们充当mutation之间的中间件作用。 它类似于流行的 HTTP 中间件模式。

```go
type (
    // Mutator is the interface that wraps the Mutate method.
    Mutator interface {
        // Mutate apply the given mutation on the graph.
        Mutate(context.Context, Mutation) (Value, error)
    }

    // Hook defines the "mutation middleware". A function that gets a Mutator
    // and returns a Mutator. For example:
    //
    //  hook := func(next ent.Mutator) ent.Mutator {
    //      return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
    //          fmt.Printf("Type: %s, Operation: %s, ConcreteType: %T\n", m.Type(), m.Op(), m)
    //          return next.Mutate(ctx, m)
    //      })
    //  }
    //
    Hook func(Mutator) Mutator
)
```

有两种类型的Mutation钩子 - **schema hooks** 和 **runtime hooks**。 **Schema Hooks** 主要用于定义模式中的自定义变更逻辑 而 **Runtime Hooks** 用于添加日志、计量、跟踪等。 让我们查看两种版本：

## 运行时钩子(Runtime hooks)

让我们用一个例子展示所有类型的变更操作：

```go
func main() {
    client, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
    if err != nil {
        log.Fatalf("failed opening connection to sqlite: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run the auto migration tool.
    if err := client.Schema.Create(ctx); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
    // Add a global hook that runs on all types and all operations.
    client.Use(func(next ent.Mutator) ent.Mutator {
        return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
            start := time.Now()
            defer func() {
                log.Printf("Op=%s\tType=%s\tTime=%s\tConcreteType=%T\n", m.Op(), m.Type(), time.Since(start), m)
            }()
            return next.Mutate(ctx, m)
        })
    })
    client.User.Create().SetName("a8m").SaveX(ctx)
    // Output:
    // 2020/03/21 10:59:10 Op=Create    Type=User   Time=46.23µs    ConcreteType=*ent.UserMutation
}
```

全局钩子有助于添加跟踪、计量、日志等等。 但有时用户想要更多的粒度：

```go
func main() {
    // <client was defined in the previous block>

    // Add a hook only on user mutations.
    client.User.Use(func(next ent.Mutator) ent.Mutator {
        // Use the "<project>/ent/hook" to get the concrete type of the mutation.
        return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
            return next.Mutate(ctx, m)
        })
    })

    // Add a hook only on update operations.
    client.Use(hook.On(Logger(), ent.OpUpdate|ent.OpUpdateOne))

    // Reject delete operations.
    client.Use(hook.Reject(ent.OpDelete|ent.OpDeleteOne))
}
```

假定您想要共享一个在多个类型之间改变字段的钩子(例如， `Group` and `User`)。 有两种方法可以做到这一点:

```go
/ 方法1：使用类型断言。
client.Use(func(next ent.Mutator) ent.Mutator {
    type NameSetter interface {
        SetName(value string)
    }
    return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
        // A schema with a "name" field must implement the NameSetter interface. 
        if ns, ok := m.(NameSetter); ok {
            ns.SetName("Ariel Mashraki")
        }
        return next.Mutate(ctx, m)
    })
})

// 方法2：使用通用ent.Mutation 接口。
client.Use(func(next ent.Mutator) ent.Mutator {
    return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
        if err := m.SetField("name", "Ariel Mashraki"); err != nil {
            // An error is returned, if the field is not defined in
            // the schema, or if the type mismatch the field type.
        }
        return next.Mutate(ctx, m)
    })
})
```

## 模式钩子(Schema hooks)

Schema钩子是在类型schema中定义的，仅适用于与 schema类型匹配的变更上。 在schema中定义钩子的动机是收集关于节点类型的所有逻辑 在一个地方，即schema。

```go
package schema

import (
    "context"
    "fmt"

    gen "<project>/ent"
    "<project>/ent/hook"

    "entgo.io/ent"
)

// Card holds the schema definition for the CreditCard entity.
type Card struct {
    ent.Schema
}

// Hooks of the Card.
func (Card) Hooks() []ent.Hook {
    return []ent.Hook{
        // First hook.
        hook.On(
            func(next ent.Mutator) ent.Mutator {
                return hook.CardFunc(func(ctx context.Context, m *gen.CardMutation) (ent.Value, error) {
                    if num, ok := m.Number(); ok && len(num) < 10 {
                        return nil, fmt.Errorf("card number is too short")
                    }
                    return next.Mutate(ctx, m)
                })
            },
            // Limit the hook only for these operations.
            ent.OpCreate|ent.OpUpdate|ent.OpUpdateOne,
        ),
        // Second hook.
        func(next ent.Mutator) ent.Mutator {
            return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
                if s, ok := m.(interface{ SetName(string) }); ok {
                    s.SetName("Boring")
                }
                v, err := next.Mutate(ctx, m)
                // Post mutation action.
                fmt.Println("new value:", v)
                return v, err
            })
        },
    }
}
```

## 钩子注册

When using [**schema hooks**](#schema-hooks), there's a chance of a cyclic import between the schema package, and the generated ent package. To avoid this scenario, ent generates an `ent/runtime` package which is responsible for registering the schema-hooks at runtime.

:::important Users **MUST** import the `ent/runtime` in order to register the schema hooks. The package can be imported in the `main` package (close to where the database driver is imported), or in the package that creates the `ent.Client`.

```go
import _ "<project>/ent/runtime"
```
:::

## 评估顺序

Hooks are called in the order they were registered to the client. Thus, `client.Use(f, g, h)` executes `f(g(h(...)))` on mutations.

Also note, that **runtime hooks** are called before **schema hooks**. That is, if `g`, and `h` were defined in the schema, and `f` was registered using `client.Use(...)`, they will be executed as follows: `f(g(h(...)))`.

## 钩子辅助工具

自动生成的钩子包提供了几个辅助工具，可以帮助您在钩子被执行 时实现控制。

```go
package schema

import (
    "context"
    "fmt"

    "<project>/ent/hook"

    "entgo.io/ent"
    "entgo.io/ent/schema/mixin"
)


type SomeMixin struct {
    mixin.Schema
}

func (SomeMixin) Hooks() []ent.Hook {
    return []ent.Hook{
        // Execute "HookA" only for the UpdateOne and DeleteOne operations.
        hook.On(HookA(), ent.OpUpdateOne|ent.OpDeleteOne),

        // Don't execute "HookB" on Create operation.
        hook.Unless(HookB(), ent.OpCreate),

        // Execute "HookC" only if the ent.Mutation is changing the "status" field,
        // and clearing the "dirty" field.
        hook.If(HookC(), hook.And(hook.HasFields("status"), hook.HasClearedFields("dirty"))),

        // Disallow changing the "password" field on Update (many) operation.
        hook.If(
            hook.FixedError(errors.New("password cannot be edited on update many")),
            hook.And(
                hook.HasOp(ent.OpUpdate),
                hook.Or(
                    hook.HasFields("password"),
                    hook.HasClearedFields("password"),
                ),
            ),
        ),
    }
}
```

## 事务钩子

钩子也可以注册在活动的事务中，并且将在 `Tx.Commit` 或 `Tx.Rollback` 上执行。 在 [事务](transactions.md#hooks) 中获取更多信息。

## 代码生成钩子

`entc` 软件包提供了一个将钩子(中间件) 添加到代码生成阶段的方法。 在 [代码生成](code-gen.md#code-generation-hooks) 中获取更多信息。 
