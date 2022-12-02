`ent`的迁移支持功能，可使数据库 schema 与你项目根目录下的 `ent/migrate/schema.go` 中定义的 schema 对象保持一致。

## 自动迁移

在应用程序初始化过程中运行自动迁移逻辑：

```go
if err := client.Schema.Create(ctx); err != nil {
    log.Fatalf("failed creating schema resources: %v", err)
}
```

`Create` 创建你项目 `ent` 部分所需的的数据库资源 。 默认情况下，`Create`以*"append-only"*模式工作；这意味着，它只创建新表和索引，将列追加到表或扩展列类型。 例如，将`int`改为`bigint`。

想要删除列或索引怎么办？

## 删除资源

`WithDropIndex` 和 `WithDropColumn` 是用于删除表列和索引的两个选项。

```go
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(
        ctx, 
        migrate.WithDropIndex(true),
        migrate.WithDropColumn(true), 
    )
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

为了在调试模式下运行迁移 (打印所有SQL查询)，请运行：

```go
err := client.Debug().Schema.Create(
    ctx, 
    migrate.WithDropIndex(true),
    migrate.WithDropColumn(true),
)
if err != nil {
    log.Fatalf("failed creating schema resources: %v", err)
}
```

## 全局唯一ID

默认情况下，每个表的SQL主键从1开始；这意味着不同类型的多个实体可以有相同的ID。 不像AWS Neptune，节点ID是UUID。

如果您使用 [GraphQL](https://graphql.org/learn/schema/#scalar-types), 它要求 对象ID是唯一的，会导致不能正常运行。

若要启用 Universal-IDs 支持您的项目，请在迁移时使用`WallUniqueID` 选项。

:::note 版本迁移用户应该参考 [的文档](versioned-migrations.mdx#a-word-on-global-unique-ids) 在 MySQL 5.*上使用 `WelcomalUnqueID` :::

```go
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    if err := client.Schema.Create(ctx, migrate.WithGlobalUniqueID(true)); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

**它是如何工作的？** `` 迁移分配给每个实体的 ID 1<<32 范围 (表格中)， 并将此信息存储在名为 `ent_types` 的表中。 例如, 类型 `A` 将有范围在 `[1,4294967296)` 的 IDs, 和类型 `B` 将有范围在 `[4294967296,8589934592)` 的 IDs 等.

请注意，如果启用此选项，表格的最大数量是 **65535**。

## 离线模式

**由于Atlas很快成为默认迁移引擎，离线迁移将被替换为[版本迁移](./zh-cn/versioned-migrations.md) 。**

离线模式允许您在数据库执行schema更改之前写到 `io.Writer` 中。 它有助于在数据库执行之前验证SQL命令，或获取一个 SQL 脚本 手动运行。

**打印变化**
```go
package main

import (
    "context"
    "log"
    "os"

    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Dump migration changes to stdout.
    if err := client.Schema.WriteTo(ctx, os.Stdout); err != nil {
        log.Fatalf("failed printing schema changes: %v", err)
    }
}
```

**Write changes to a file**
```go
package main

import (
    "context"
    "log"
    "os"

    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Dump migration changes to an SQL script.
    f, err := os.Create("migrate.sql")
    if err != nil {
        log.Fatalf("create migrate file: %v", err)
    }
    defer f.Close()
    if err := client.Schema.WriteTo(ctx, f); err != nil {
        log.Fatalf("failed printing schema changes: %v", err)
    }
}
```

## 外键

默认情况下， `ent` 在定义关系(edges) 时使用外键来保证数据库中使用正确性和一致性。

然而， `ent` 也提供了一个选项来禁用此功能，使用 `WithForeignKeys` 选项。 您应该注意到，将此选项设置为 `false`， 将会告诉迁移系统不要在schema DDL 中创建外键，edges 验证和清除必须由开发者手动处理。

我们期望在不久的将来提供一套钩子在申请级别限制外键。

```go
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(
        ctx,
        migrate.WithForeignKeys(false), // Disable foreign keys.
    )
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

## 迁移钩子

Entc 软件包提供了一个选项将钩子(中间件) 添加到代码生成阶段。 此选项对于修改或过滤迁移正在进行的表格或在数据库中创建自定义资源是理想的。

```go
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"

    "entgo.io/ent/dialect/sql/schema"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(
        ctx,
        schema.WithHooks(func(next schema.Creator) schema.Creator {
            return schema.CreateFunc(func(ctx context.Context, tables ...*schema.Table) error {
                // Run custom code here.
                return next.Create(ctx, tables...)
            })
        }),
    )
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

## Atlas集成

从V0.10开始, Ent 支持使用 [Atlas](https://atlasgo.io)运行迁移， 这是一个更强大的迁移框架，涵盖当前Ent 迁移包不支持的许多功能。 为了使用 Atlas 引擎执行迁移，请使用 `WiAtlas(true)` 选项。

```go {21}
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"

    "entgo.io/ent/dialect/sql/schema"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(ctx, schema.WithAtlas(true))
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

除了标准选项外(如： `WithDropColumn`, `WithGlobalUniqueID`), Atlas 提供额外的 选项绑定到schema 迁移步骤中。

![atlas-migration-process](https://entgo.io/images/assets/migrate-atlas-process.png)


#### Atlas `Diff` 和 `Apply` 钩子

这里有两个示例展示如何绑定到Atlas `Diff` 和 `Apply` 阶段。

```go
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"

    "ariga.io/atlas/sql/migrate"
    atlas "ariga.io/atlas/sql/schema"
    "entgo.io/ent/dialect"
    "entgo.io/ent/dialect/sql/schema"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err :=  client.Schema.Create(
        ctx,
        // Hook into Atlas Diff process.
        schema.WithDiffHook(func(next schema.Differ) schema.Differ {
            return schema.DiffFunc(func(current, desired *atlas.Schema) ([]atlas.Change, error) {
                // Before calculating changes.
                changes, err := next.Diff(current, desired)
                if err != nil {
                    return nil, err
                }
                // After diff, you can filter
                // changes or return new ones.
                return changes, nil
            })
        }),
        // Hook into Atlas Apply process.
        schema.WithApplyHook(func(next schema.Applier) schema.Applier {
            return schema.ApplyFunc(func(ctx context.Context, conn dialect.ExecQuerier, plan *migrate.Plan) error {
                // Example to hook into the apply process, or implement
                // a custom applier. For example, write to a file.
                //
                //  for _, c := range plan.Changes {
                //      fmt.Printf("%s: %s", c.Comment, c.Cmd)
                //      if err := conn.Exec(ctx, c.Cmd, c.Args, nil); err != nil {
                //          return err
                //      }
                //  }
                //
                return next.Apply(ctx, conn, plan)
            })
        }),
    )
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

#### `Diff` 钩子示例

如果 `ent/schema`的一个字段更改名称， Ent 不会检测到此更改，建议在diff 阶段执行 `DropColumn` 和 `AddColumn` 。 一个办法是在字段中使用 [StorageKey](schema-fields.md#storage-key) 选项，并在数据库表中保留旧列名称。 However, using Atlas `Diff` hooks allow replacing the `DropColumn` and `AddColumn` changes with a `RenameColumn` change.

```go
func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    // ...
    if err := client.Schema.Create(ctx, schema.WithDiffHook(renameColumnHook)); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}

func renameColumnHook(next schema.Differ) schema.Differ {
    return schema.DiffFunc(func(current, desired *atlas.Schema) ([]atlas.Change, error) {
        changes, err := next.Diff(current, desired)
        if err != nil {
            return nil, err
        }
        for _, c := range changes {
            m, ok := c.(*atlas.ModifyTable)
            // Skip if the change is not a ModifyTable,
            // or if the table is not the "users" table.
            if !ok || m.T.Name != user.Table {
                continue
            }
            changes := atlas.Changes(m.Changes)
            switch i, j := changes.IndexDropColumn("old_name"), changes.IndexAddColumn("new_name"); {
            case i != -1 && j != -1:
                // Append a new renaming change.
                changes = append(changes, &atlas.RenameColumn{
                    From: changes[i].(*atlas.DropColumn).C,
                    To: changes[j].(*atlas.AddColumn).C,
                })
                // Remove the drop and add changes.
                changes.RemoveIndex(i, j)
                m.Changes = changes
            case i != -1 || j != -1:
                return nil, errors.New("old_name and new_name must be present or absent")
            }
        }
        return changes, nil
    })
}
```

#### `Apply` 钩子示例

`Apply` 钩子允许访问和修改迁移计划及其原始更改（SQL 语句），但除此之外，它对于在应用计划之前或之后执行自定义 SQL 语句也很有用。例如，默认情况下不允许将可空列更改为不带默认值的不可空列，但是，我们可以使用“Apply”挂钩来解决此问题，该挂钩“Update”此列中包含“NULL”值的所有行：

```go
func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    // ...
    if err := client.Schema.Create(ctx, schema.WithApplyHook(fillNulls)); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}

func fillNulls(next schema.Applier) schema.Applier {
    return schema.ApplyFunc(func(ctx context.Context, conn dialect.ExecQuerier, plan *migrate.Plan) error {
        // There are three ways to UPDATE the NULL values to "Unknown" in this stage.
        // Append a custom migrate.Change to the plan, execute an SQL statement directly
        // on the dialect.ExecQuerier, or use the ent.Client used by the project.

        // Execute a custom SQL statement.
        query, args := sql.Dialect(dialect.MySQL).
            Update(user.Table).
            Set(user.FieldDropOptional, "Unknown").
            Where(sql.IsNull(user.FieldDropOptional)).
            Query()
        if err := conn.Exec(ctx, query, args, nil); err != nil {
            return err
        }

        // Append a custom statement to migrate.Plan.
        //
        //  plan.Changes = append([]*migrate.Change{
        //      {
        //          Cmd: fmt.Sprintf("UPDATE users SET %[1]s = '%[2]s' WHERE %[1]s IS NULL", user.FieldDropOptional, "Unknown"),
        //      },
        //  }, plan.Changes...)

        // Use the ent.Client used by the project.
        //
        //  drv := sql.NewDriver(dialect.MySQL, sql.Conn{ExecQuerier: conn.(*sql.Tx)})
        //  if err := ent.NewClient(ent.Driver(drv)).
        //      User.
        //      Update().
        //      SetDropOptional("Unknown").
        //      Where(/* Add predicate to filter only rows with NULL values */).
        //      Exec(ctx); err != nil {
        //      return fmt.Errorf("fix default values to uppercase: %w", err)
        //  }

        return next.Apply(ctx, conn, plan)
    })
}
```
