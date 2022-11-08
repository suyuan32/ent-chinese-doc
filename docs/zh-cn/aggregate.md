## 聚合

`聚合`允许添加一个或多个聚合函数。

```go
package main

import (
    "context"

    "<project>/ent"
    "<project>/ent/payment"
    "<project>/ent/pet"
)

func Do(ctx context.Context, client *ent.Client) {
    // 聚合一个字段
    sum, err := client.Payment.Query().
        Aggregate(
            ent.Sum(payment.Amount),
        ).
        Int(ctx)

    // 聚合多个字段。
    var v []struct {
        Sum, Min, Max, Count int
    }
    err := client.Pet.Query().
        Aggregate(
            ent.Sum(pet.FieldAge),
            ent.Min(pet.FieldAge),
            ent.Max(pet.FieldAge),
            ent.Count(),
        ).
        Scan(ctx, &v)
}
```

## 分组

对users按照`name` and `age`字段进行分组，并计算age字段的总和。

```go
package main

import (
    "context"

    "<project>/ent"
    "<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
    var v []struct {
        Name  string `json:"name"`
        Age   int    `json:"age"`
        Sum   int    `json:"sum"`
        Count int    `json:"count"`
    }
    err := client.User.Query().
        GroupBy(user.FieldName, user.FieldAge).
        Aggregate(ent.Count(), ent.Sum(user.FieldAge)).
        Scan(ctx, &v)
}
```

按单个字段进行分组。

```go
package main

import (
    "context"

    "<project>/ent"
    "<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
    names, err := client.User.
        Query().
        GroupBy(user.FieldName).
        Strings(ctx)
}
```

## 按Edge进行分组

自定义聚合函数对于编写你自己特定的存储逻辑非常有用。

下面展示了如何使用`id`和`name`对users进行分组并计算他们pets的平均`age`。

```go
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/pet"
    "<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
    var users []struct {
        ID      int
        Name    string
        Average float64
    }
    err := client.User.Query().
        GroupBy(user.FieldID, user.FieldName).
        Aggregate(func(s *sql.Selector) string {
            t := sql.Table(pet.Table)
            s.Join(t).On(s.C(user.FieldID), t.C(pet.OwnerColumn))
            return sql.As(sql.Avg(t.C(pet.FieldAge)), "average")
        }).
        Scan(ctx, &users)
}
```

## Having与Group By

[自定义SQL修饰符](https://entgo.io/docs/feature-flags/#custom-sql-modifiers)可以帮助你执行高级的SQL查询。 下面演示如何检索每个角色中年龄最大的用户。


```go
package main

import (
    "context"
    "log"

    "entgo.io/ent/dialect/sql"
    "<project>/ent"
    "<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
    var users []struct {
        Id      Int
        Age     Int
        Role    string
    }
    err := client.User.Query().
        Modify(func(s *sql.Selector) {
            s.GroupBy(user.Role)
            s.Having(
                sql.EQ(
                    user.FieldAge,
                    sql.Raw(sql.Max(user.FieldAge)),
                ),
            )
        }).
        ScanX(ctx, &users)
}

```

**注意：** `sql.Raw`非常重要。 它告诉查询条件`sql.Max`并不是一个参数，需要将其作为函数处理。

上面的代码实质上生成了以下 SQL 查询：

```sql
SELECT * FROM user GROUP BY user.role HAVING user.age = MAX(user.age)
```
