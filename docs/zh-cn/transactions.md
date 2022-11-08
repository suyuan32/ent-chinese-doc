## 启动一个事务

```go
// GenTx 在一次事务中生成一系列实体。
func GenTx(ctx context.Context, client *ent.Client) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return fmt.Errorf("starting a transaction: %w", err)
    }
    hub, err := tx.Group.
        Create().
        SetName("Github").
        Save(ctx)
    if err != nil {
        return rollback(tx, fmt.Errorf("failed creating the group: %w", err))
    }
    // 创建 admin 组
    dan, err := tx.User.
        Create().
        SetAge(29).
        SetName("Dan").
        AddManage(hub).
        Save(ctx)
    if err != nil {
        return rollback(tx, err)
    }
    // 创建 "Ariel" 用户。
    a8m, err := tx.User.
        Create().
        SetAge(30).
        SetName("Ariel").
        AddGroups(hub).
        AddFriends(dan).
        Save(ctx)
    if err != nil {
        return rollback(tx, err)
    }
    fmt.Println(a8m)
    // 输出:
    // User(id=2, age=30, name=Ariel)

    // 提交事务
    return tx.Commit()
}

// 如果发生错误则调用 tx.Rollback 回滚并返回嵌套的错误信息
func rollback(tx *ent.Tx, err error) error {
    if rerr := tx.Rollback(); rerr != nil {
        err = fmt.Errorf("%w: %v", err, rerr)
    }
    return err
}
```

如果您在事务成功后查询已创建实体的边，则必须调用 `Unwrap()`（例如：`a8m.QueryGroups()`）。 Unwrap 将嵌入在实体中的底层客户端的状态恢复为不可回滚。

完整示例可参阅 [GitHub](https://github.com/ent/ent/tree/master/examples/traversal).

## 事务化客户端

你可能已经有代码是用了 `*ent.Client` 的，然后你想将它修改成（或封装成）在事务中执行。 对于这种情况，你可以编写一个事务化客户端。 你可以从已有事务中获取一个 `*ent.Client`

```go
// WrapGen 从现有事务中封装了的 “Gen” 函数。
func WrapGen(ctx context.Context, client *ent.Client) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    txClient := tx.Client()
    // 使用下面的 "Gen" 方法，只需要传递一个事务化客户端，不需要更改 "Gen" 的代码
    if err := Gen(ctx, txClient); err != nil {
        return rollback(tx, err)
    }
    return tx.Commit()
}

// Gen 生成一组实体。
func Gen(ctx context.Context, client *ent.Client) error {
    // ...
    return nil
}
```

完整示例可参阅 [GitHub](https://github.com/ent/ent/tree/master/examples/traversal).

## 最佳实践

在事务中使用回调函数来实现代码复用：

```go
func WithTx(ctx context.Context, client *ent.Client, fn func(tx *ent.Tx) error) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    defer func() {
        if v := recover(); v != nil {
            tx.Rollback()
            panic(v)
        }
    }()
    if err := fn(tx); err != nil {
        if rerr := tx.Rollback(); rerr != nil {
            err = fmt.Errorf("%w: rolling back transaction: %v", err, rerr)
        }
        return err
    }
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("committing transaction: %w", err)
    }
    return nil
}
```

使用：

```go
func Do(ctx context.Context, client *ent.Client) {
    // WithTx helper.
    if err := WithTx(ctx, client, func(tx *ent.Tx) error {
        return Gen(ctx, tx.Client())
    }); err != nil {
        log.Fatal(err)
    }
}
```

## 钩子

与[结构钩子](hooks.md#schema-hooks)和[运行时钩子](hooks.md#runtime-hooks)一样，钩子也可以注册在活跃的事务中，将会在`Tx.Commit`或者是`Tx.Rollback`时执行：

```go
func Do(ctx context.Context, client *ent.Client) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    // 添加一个钩子在 Tx.Commit 时
    tx.OnCommit(func(next ent.Committer) ent.Committer {
        return ent.CommitFunc(func(ctx context.Context, tx *ent.Tx) error {
            // 事务提交之前的代码写在这里
            err := next.Commit(ctx, tx)
            // 事务提交之后的代码写在这里
            return err
        })
    })
    // 添加一个钩子在 Tx.Rollback 时
    tx.OnRollback(func(next ent.Rollbacker) ent.Rollbacker {
        return ent.RollbackFunc(func(ctx context.Context, tx *ent.Tx) error {
            // 事务回滚之前的的代码写在这里
            err := next.Rollback(ctx, tx)
            // 事务回滚之后的代码写在这里
            return err
        })
    })
    //
    // <Code goes here>
    //
    return err
}
```

## Isolation Levels

Some drivers support tweaking a transaction's isolation level. For example, with the [sql](sql-integration.md) driver, you can do so with the `BeginTx` method.

```go
tx, err := client.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelRepeatableRead})
```