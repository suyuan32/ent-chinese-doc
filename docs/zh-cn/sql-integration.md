下面的示例将展示，如何使用`sql.DB`对象对`ent.Client`进行配置。

## 配置`sql.DB`

第一种形式:

```go
package main

import (
    "time"

    "<your_project>/ent"
    "entgo.io/ent/dialect/sql"
)

func Open() (*ent.Client, error) {
    drv, err := sql.Open("mysql", "<mysql-dsn>")
    if err != nil {
        return nil, err
    }
    // 获取数据库驱动中的sql.DB对象。
    db := drv.DB()
    db.SetMaxIdleConns(10)
    db.SetMaxOpenConns(100)
    db.SetConnMaxLifetime(time.Hour)
    return ent.NewClient(ent.Driver(drv)), nil
}
```

第二种形式:

```go
package main

import (
    "database/sql"
    "time"

    "<your_project>/ent"
    entsql "entgo.io/ent/dialect/sql"
)

func Open() (*ent.Client, error) {
    db, err := sql.Open("mysql", "<mysql-dsn>")
    if err != nil {
        return nil, err
    }
    db.SetMaxIdleConns(10)
    db.SetMaxOpenConns(100)
    db.SetConnMaxLifetime(time.Hour)
    // 从db变量中构造一个ent.Driver对象。
    drv := entsql.OpenDB("mysql", db)
    return ent.NewClient(ent.Driver(drv)), nil
}
```

## 通过MySQL驱动使用Opencensus

```go
package main

import (
    "context"
    "database/sql"
    "database/sql/driver"

    "<project>/ent"

    "contrib.go.opencensus.io/integrations/ocsql"
    "entgo.io/ent/dialect"
    entsql "entgo.io/ent/dialect/sql"
    "github.com/go-sql-driver/mysql"
)

type connector struct {
    dsn string
}

func (c connector) Connect(context.Context) (driver.Conn, error) {
    return c.Driver().Open(c.dsn)
}

func (connector) Driver() driver.Driver {
    return ocsql.Wrap(
        mysql.MySQLDriver{},
        ocsql.WithAllTraceOptions(),
        ocsql.WithRowsClose(false),
        ocsql.WithRowsNext(false),
        ocsql.WithDisableErrSkip(true),
    )
}

// 打开一个新的数据库连接并对连接进行统计记录。
func Open(dsn string) *ent.Client {
    db := sql.OpenDB(connector{dsn})
    // 从db变量中构造一个ent.Driver对象。
    drv := entsql.OpenDB(dialect.MySQL, db)
    return ent.NewClient(ent.Driver(drv))
}
```


## 使用pgx驱动PostgreSQL

```go
package main

import (
    "context"
    "database/sql"
    "log"

    "<project>/ent"

    "entgo.io/ent/dialect"
    entsql "entgo.io/ent/dialect/sql"
    _ "github.com/jackc/pgx/v4/stdlib"
)

// 打开新的数据库连接
func Open(databaseUrl string) *ent.Client {
    db, err := sql.Open("pgx", databaseUrl)
    if err != nil {
        log.Fatal(err)
    }

    // 从db变量中构造一个ent.Driver对象。
    drv := entsql.OpenDB(dialect.Postgres, db)
    return ent.NewClient(ent.Driver(drv))
}

func main() {
    client := Open("postgresql://user:password@127.0.0.1/database")

    // 将你的代码写在这里。 比如这一段示例代码:
    ctx := context.Background()
    if err := client.Schema.Create(ctx); err != nil {
        log.Fatal(err)
    }
    users, err := client.User.Query().All(ctx)
    if err != nil {
        log.Fatal(err)
    }
    log.Println(users)
}
```
