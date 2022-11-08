如果你想在单元测试中测试 `ent.Client`，你可以用已经生成好的 `enttest` 包来创建客户端和自动执行表结构迁移：

```go
package main

import (
    "testing"

    "<project>/ent/enttest"

    _ "github.com/mattn/go-sqlite3"
)

func TestXXX(t *testing.T) {
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
    defer client.Close()
    // ...
}
```

如果需要传递选项参数给 `Open`，请使用 `enttest.Option`：

```go
func TestXXX(t *testing.T) {
    opts := []enttest.Option{
        enttest.WithOptions(ent.Log(t.Log)),
        enttest.WithMigrateOptions(migrate.WithGlobalUniqueID(true)),
    }
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1", opts...)
    defer client.Close()
    // ...
}
```
