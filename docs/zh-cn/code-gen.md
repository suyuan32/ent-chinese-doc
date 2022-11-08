## 安装

本项目有一个叫做 `ent` 的代码工具。 若要安装 `ent` 运行以下命令：

```bash
go get -d entgo.io/ent/cmd/ent
```

## 初始化一个新的 Schema

为了生成一个或多个 schema 模板，运行 `ent init` 如下：

```bash
go run -mod=mod entgo.io/ent/cmd/ent init User Pet
```

`init` 将在 `ent/schema` 目录下创建 2个 schemas (`user.go` 和 `pet.go`)。 如果 `ent` 目录不存在，将自动创建。 一般约定将 `ent` 目录放在项目的根目录下。

## 生成资源文件

每次添加或修改 [fields](schema-fields.md) 和 [edges](schema-edges) 后, 你都需要生成新的实体. 在项目的根目录执行 `ent generate`或直接执行`go generate`命令重新生成资源文件:


```bash
go generate ./ent
```

`generate`将会按照schema模板生成生成以下资源:

- `Client` 和 `Tx` 对象用于与图的交互。
- 每个schema对应的增删改查， 查看[CRUD](crud.mdx)了解更多信息。
- 每个schema的实体对象(Go结构体)。
- 含常量和查询条件的包，用于与生成器交互。
- 用于数据迁移的`migrate`包。 查看 [迁移](migrate.md) 获取更多信息。
- 一个在变更前执行的 `hook` 包 查看 [钩子](hooks.md) 获取更多信息。

## `entc`和`ent`之间的版本兼容性

在项目中使用 `ent` CLI时，需要确保CLI使用的**版本**与项目使用的 `ent` 版本相同。

保证此的一个方法是通过 `go generate` 来使用 `go.mod` 内所定义的 `ent` CLI版本。 如果您的项目没有使用 [Go modules](https://github.com/golang/go/wiki/Modules#quick-start), 请设置一个：

```console
go mod init <project>
```

然后执行以下命令，以便将 `ent` 添加到您的 `go.mod` 文件：

```console
go get -d entgo.io/ent/cmd/ent
```

将 `generate.go` 文件添加到你项目的`<project>/ent` 目录中：

```go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

最后，你可以执行 `go generate ./ent` ，以便在您的项目方案中执行 `ent` 代码生成。

## 代码生成选项

关于代码生成选项的更多信息，执行 `ent generate -h`：

```console
generate go code for the schema directory

Usage:
  ent generate [flags] path

Examples:
  ent generate ./ent/schema
  ent generate github.com/a8m/x

Flags:
      --feature strings       extend codegen with additional features
      --header string         override codegen header
  -h, --help                  help for generate
      --idtype [int string]   type of the id field (default int)
      --storage string        storage driver to support in codegen (default "sql")
      --target string         target directory for codegen
      --template strings      external templates to execute
```

## Storage选项

`ent` 可以为 SQL 和 Gremlin 生成资源。 默认是 SQL

## 外部模板

`ent` 接受执行外部 Go 模板文件。 如果模板名称已由 `ent`定义，它将覆盖现有的模板。 否则，它将把执行后的输出写入到 与模板相同名称的文件。 支持参数  `file`, `dir` 和 `glob` 如下所示：

```console
go run -mod=mod entgo.io/ent/cmd/ent generate --template <dir-path> --template glob="path/to/*.tmpl" ./ent/schema
```

更多信息和示例可在 [外部模板](templates.md) 中找到。

## 把 `entc` 作为库使用

运行 `ent` 代码生成的另一个选项是创建一个名为 `ent/entc.go`的文件，文件包含以下内容， 然后由 `ent/generate.go` 文件来执行它：

```go title="ent/entc.go"
// +build ignore

package main

import (
    "log"

    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
    "entgo.io/ent/schema/field"
)

func main() {
    if err := entc.Generate("./schema", &gen.Config{}); err != nil {
        log.Fatal("running ent codegen:", err)
    }
}
```

```go title="ent/generate.go"
package ent

//go:generate go run -mod=mod entc.go
```

完整示例请参阅 [GitHub](https://github.com/ent/ent/tree/master/examples/entcpkg).

## Schema描述

要获取您Schema的描述，请执行：

```bash
go run -mod=mod entgo.io/ent/cmd/ent describe ./ent/schema
```

示例如下：

```console
Pet:
    +-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
    | Field |  Type   | Unique | Optional | Nillable | Default | UpdateDefault | Immutable |       StructTag       | Validators |
    +-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
    | id    | int     | false  | false    | false    | false   | false         | false     | json:"id,omitempty"   |          0 |
    | name  | string  | false  | false    | false    | false   | false         | false     | json:"name,omitempty" |          0 |
    +-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
    +-------+------+---------+---------+----------+--------+----------+
    | Edge  | Type | Inverse | BackRef | Relation | Unique | Optional |
    +-------+------+---------+---------+----------+--------+----------+
    | owner | User | true    | pets    | M2O      | true   | true     |
    +-------+------+---------+---------+----------+--------+----------+

User:
    +-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
    | Field |  Type   | Unique | Optional | Nillable | Default | UpdateDefault | Immutable |       StructTag       | Validators |
    +-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
    | id    | int     | false  | false    | false    | false   | false         | false     | json:"id,omitempty"   |          0 |
    | age   | int     | false  | false    | false    | false   | false         | false     | json:"age,omitempty"  |          0 |
    | name  | string  | false  | false    | false    | false   | false         | false     | json:"name,omitempty" |          0 |
    +-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
    +------+------+---------+---------+----------+--------+----------+
    | Edge | Type | Inverse | BackRef | Relation | Unique | Optional |
    +------+------+---------+---------+----------+--------+----------+
    | pets | Pet  | false   |         | O2M      | false  | true     |
    +------+------+---------+---------+----------+--------+----------+
```

## 代码生成Hooks (钩子)

`entc` 软件包提供了一个将钩子(中间件) 添加到代码生成阶段的方法。 这个方法适合在你需要对graph schema进行自定义验证或者生成附加资源时使用.

```go
// +build ignore

package main

import (
    "fmt"
    "log"
    "reflect"

    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
)

func main() {
    err := entc.Generate("./schema", &gen.Config{
        Hooks: []gen.Hook{
            EnsureStructTag("json"),
        },
    })
    if err != nil {
        log.Fatalf("running ent codegen: %v", err)
    }
}

// EnsureStructTag 字段确保所有的字段都有标签
func EnsureStructTag(name string) gen.Hook {
    return func(next gen.Generator) gen.Generator {
        return gen.GenerateFunc(func(g *gen.Graph) error {
            for _, node := range g.Nodes {
                for _, field := range node.Fields {
                    tag := reflect.StructTag(field.StructTag)
                    if _, ok := tag.Lookup(name); !ok {
                        return fmt.Errorf("struct tag %q is missing for field %s.%s", name, node.Name, field.Name)
                    }
                }
            }
            return next.Generate(g)
        })
    }
}
```

## 外部依赖

为了在 `ent` 软件包下扩展生成的客户端和生成器,，并自动注入它们结构字段的外部依赖关系，请在 [`ent/entc.go`](#use-entc-as-a-package)文件中使用 `entc.Dependency` 选项：

```go title="ent/entc.go" {3-12}
func main() {
    opts := []entc.Option{
        entc.Dependency(
            entc.DependencyType(&http.Client{}),
        ),
        entc.Dependency(
            entc.DependencyName("Writer"),
            entc.DependencyTypeInfo(&field.TypeInfo{
                Ident:   "io.Writer",
                PkgPath: "io",
            }),
        ),
    }
    if err := entc.Generate("./schema", &gen.Config{}, opts...); err != nil {
        log.Fatalf("running ent codegen: %v", err)
    }
}
```

然后，在您的应用程序中使用：

```go title="example_test.go" {5-6,15-16}
func Example_Deps() {
    client, err := ent.Open(
        "sqlite3",
        "file:ent?mode=memory&cache=shared&_fk=1",
        ent.Writer(os.Stdout),
        ent.HTTPClient(http.DefaultClient),
    )
    if err != nil {
        log.Fatalf("failed opening connection to sqlite: %v", err)
    }
    defer client.Close()
    // 在生成构建器中使用注入依赖项的示例。
    client.User.Use(func(next ent.Mutator) ent.Mutator {
        return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
            _ = m.HTTPClient
            _ = m.Writer
            return next.Mutate(ctx, m)
        })
    })
    // ...
}
```

完整的示例在 [GitHub](https://github.com/ent/ent/tree/master/examples/entcpkg).

## 特性开关

`entc` 软件包提供了一系列代码生成特性，可以自行选择使用。

欲了解更多信息，请查看 [特性开关](features.md)。
