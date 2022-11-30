如果您使用的是 Atlas 迁移引擎，则可以使用它的版本化迁移功能。 它不会将计算的更改直接应用于数据库，而是生成一组迁移文件，其中包含迁移数据库所需的 SQL 语句。 然后可以根据您的需要编辑这些文件，并通过您喜欢的任何工具（如 golang-migrate、Flyway、liquibase）应用。

![atlas-versioned-migration-process](https://entgo.io/images/assets/migrate-atlas-versioned.png)

## 配置

为了让 Ent 对您的代码进行必要的更改，您必须使用以下两个选项之一启用此功能：

1. 如果您使用默认的 go generate 配置，只需添加 `--feature sql/versioned-migration` 到 `ent/generate.go` 文件中，如下:

```go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate --feature sql/versioned-migration ./schema
```

2. 如果您使用的是代码生成包（例如，如果您使用的是 Ent 扩展），请按如下方式添加功能标志:

```go
//go:build ignore

package main

import (
    "log"

    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
)

func main() {
    err := entc.Generate("./schema", &gen.Config{}, entc.FeatureNames("sql/versioned-migration"))
    if err != nil {
        log.Fatalf("running ent codegen: %v", err)
    }
}
```

## 生成版本化迁移文件

### 使用客户端(client)

重新生成项目后，Ent 客户端上将有一个额外的 `Diff` 方法，您可以使用它来检查连接的数据库，将其与Schema定义进行比较，并创建将数据库迁移到图形所需的 sql 语句。

```go
package main

import (
    "context"
    "log"

    "<project>/ent"

    "ariga.io/atlas/sql/migrate"
    "entgo.io/ent/dialect/sql/schema"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Create a local migration directory.
    dir, err := migrate.NewLocalDir("migrations")
    if err != nil {
        log.Fatalf("failed creating atlas migration directory: %v", err)
    }
    // Write migration diff.
    err = client.Schema.Diff(ctx, schema.WithDir(dir))
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

您只需调用 `go run -mod=mod main.go` 即可创建一组新的迁移文件

### 使用图 (Graph)

您还可以在没有实例化的 Ent 客户端的情况下生成新的迁移文件。 如果你想让迁移文件创建成为 go generate 工作流的一部分，这会很有用。

```go
package main

import (
    "context"
    "log"

    "ariga.io/atlas/sql/migrate"
    "entgo.io/ent/dialect/sql"
    "entgo.io/ent/dialect/sql/schema"
    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
)

func main() {
    // Load the graph.
    graph, err := entc.LoadGraph("/.schema", &gen.Config{})
    if err != nil {
        log.Fatalln(err)
    }
    tbls, err := graph.Tables()
    if err != nil {
        log.Fatalln(err)
    }
    // Create a local migration directory.
    d, err := migrate.NewLocalDir("migrations")
    if err != nil {
        log.Fatalln(err)
    }
    // Open connection to the database.
    dlct, err := sql.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalln(err)
    }
    // Inspect it and compare it with the graph.
    m, err := schema.NewMigrate(dlct, schema.WithDir(d))
    if err != nil {
        log.Fatalln(err)
    }
    if err := m.Diff(context.Background(), tbls...); err != nil {
        log.Fatalln(err)
    }
}
```

## 执行迁移

Atlas 迁移引擎还不支持将迁移文件应用到数据库中，因此要管理和执行生成的迁移文件，您必须依赖外部工具（或手动执行）。 默认情况下，Atlas 为计算出的差异生成一个“Up”和一个“Down”迁移文件。 这些文件与 [golang-migrate/migrate](https://github.com/golang-migrate/migrate) 包兼容, 您可以使用该工具来管理部署中的迁移.

```shell
migrate -source file://migrations -database mysql://root:pass@tcp(localhost:3306)/test up
```

## 从自动迁移到版本化迁移

如果您在生产环境中已经有一个 Ent 应用程序并且想要从自动迁移切换到新版本迁移，您需要采取一些额外的步骤.

1. 创建反映当前部署状态的初始迁移文件（或多个文件，如果需要）。

> 请确保您的Schema定义与您部署的版本一致。 然后启动一个空数据库执行 diff 命令一次。 这将生成创建当前状态Schema所需的语句。

2. 配置您用于管理迁移的工具以将此文件视为**已应用(Applied)**。

> 在 `golang-migrate` 的情况下，这可以通过强制您的数据库版本为 [点此查看](https://github.com/golang-migrate/migrate/blob/master/GETTING_STARTED.md#forcing-your-database-version)来完成操作.

