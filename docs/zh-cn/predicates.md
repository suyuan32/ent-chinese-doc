## 字段断言

- **布尔类型**：
  - =, !=
- **数字类型**：
  - =, !=, >, <, >=, <=,
  - IN, NOT IN
- **时间类型**：
  - =, !=, >, <, >=, <=
  - IN, NOT IN
- **字符类型**：
  - =, !=, >, <, >=, <=
  - IN, NOT IN
  - Contains, HasPrefix, HasSuffix
  - ContainsFold, EqualFold （只能用于 **SQL** 语句）
- **JSON类型：**
  - =, !=
  - =, !=, >, <, >=, <= on nested values (JSON path).
  - Contains （只能用于包含嵌套的 JSON path 表达式）
  - HasKey, Len&lt;P>
  - `null` checks for nested values (JSON path).
- **可选**字段:
  - IsNil, NotNil

## Edge 断言

- **HasEdge**. 例如，对于一个 `Pet` 类型的 edge `owner` 来说 ，可以使用:

  ```go
   client.Pet.
        Query().
        Where(pet.HasOwner()).
        All(ctx)
  ```

- **HasEdgeWith**. 也可以将断言表示为集合的形式

  ```go
   client.Pet.
        Query().
        Where(pet.HasOwnerWith(user.Name("a8m"))).
        All(ctx)
  ```


## 否定 (NOT)

```go
client.Pet.
    Query().
    Where(pet.Not(pet.NameHasPrefix("Ari"))).
    All(ctx)
```

## 析取 (OR)

```go
client.Pet.
    Query().
    Where(
        pet.Or(
            pet.HasOwner(),
            pet.Not(pet.HasFriends()),
        )
    ).
    All(ctx)
```

## 合取 (AND)

```go
client.Pet.
    Query().
    Where(
        pet.And(
            pet.HasOwner(),
            pet.Not(pet.HasFriends()),
        )
    ).
    All(ctx)
```

## 自定义断言

如果你想写自己的特定逻辑，可以使用自定义断言。

#### 获取用户1、2和3的所有宠物

```go
pets := client.Pet.
    Query().
    Where(func(s *sql.Selector) {
        s.Where(sql.InInts(pet.FieldOwnerID, 1, 2, 3))
    }).
    AllX(ctx)
```
上面的代码实质上生成了以下 SQL 查询：
```sql
SELECT DISTINCT `pets`.`id`, `pets`.`owner_id` FROM `pets` WHERE `owner_id` IN (1, 2, 3)
```

#### 统计名为 `URL` 的JSON字段包含 `Scheme` 的用户数

```go
count := client.User.
    Query().
    Where(func(s *sql.Selector) {
        s.Where(sqljson.HasKey(user.FieldURL, sqljson.Path("Scheme")))
    }).
    CountX(ctx)
```

The above code will produce the following SQL query:

```sql
-- PostgreSQL
SELECT COUNT(DISTINCT "users"."id") FROM "users" WHERE "url"->'Scheme' IS NOT NULL

-- SQLite and MySQL
SELECT COUNT(DISTINCT `users`.`id`) FROM `users` WHERE JSON_EXTRACT(`url`, "$.Scheme") IS NOT NULL
```

#### 获取所有拥有 `"Tesla"` 汽车的用户

请考虑使用一个 ent 的查询语句，如下：

```go
users := client.User.Query().
    Where(user.HasCarWith(car.Model("Tesla"))).
    AllX(ctx)
```

此查询可以用三种不同形式重写： `IN`, `EXISTS` 和 `JOIN`。

```go
// `IN` version.
users := client.User.Query().
    Where(func(s *sql.Selector) {
        t := sql.Table(car.Table)
        s.Where(
            sql.In(
                s.C(user.FieldID),
                sql.Select(t.C(user.FieldID)).From(t).Where(sql.EQ(t.C(car.FieldModel), "Tesla")),
            ),
        )
    }).
    AllX(ctx)

// `JOIN` version.
users := client.User.Query().
    Where(func(s *sql.Selector) {
        t := sql.Table(car.Table)
        s.Join(t).On(s.C(user.FieldID), t.C(car.FieldOwnerID))
        s.Where(sql.EQ(t.C(car.FieldModel), "Tesla"))
    }).
    AllX(ctx)

// `EXISTS` version.
users := client.User.Query().
    Where(func(s *sql.Selector) {
        t := sql.Table(car.Table)
        p := sql.And(
            sql.EQ(t.C(car.FieldModel), "Tesla"),
            sql.ColumnsEQ(s.C(user.FieldID), t.C(car.FieldOwnerID)),
        )
        s.Where(sql.Exists(sql.Select().From(t).Where(p)))
    }).
    AllX(ctx)
```

上面的代码实质上生成了以下 SQL 查询：

```sql
-- `IN` version.
SELECT DISTINCT `users`.`id`, `users`.`age`, `users`.`name` FROM `users` WHERE `users`.`id` IN (SELECT `cars`.`id` FROM `cars` WHERE `cars`.`model` = 'Tesla')

-- `JOIN` version.
SELECT DISTINCT `users`.`id`, `users`.`age`, `users`.`name` FROM `users` JOIN `cars` ON `users`.`id` = `cars`.`owner_id` WHERE `cars`.`model` = 'Tesla'

-- `EXISTS` version.
SELECT DISTINCT `users`.`id`, `users`.`age`, `users`.`name` FROM `users` WHERE EXISTS (SELECT * FROM `cars` WHERE `cars`.`model` = 'Tesla' AND `users`.`id` = `cars`.`owner_id`)
```

#### 获取宠物名称中包含特定表达式的所有宠物

生成的代码提供 `HasPrefix`, `HasSuffix`, `Contains`, 和 `ContainsFold` 断言用于模式匹配。 然而，如果要使用 `LIKE` 操作自定义表达式，请参考下面的例子。

```go
pets := client.Pet.Query().
    Where(func(s *sql.Selector){
        s.Where(sql.Like(pet.Name,"_B%"))
    }).
    AllX(ctx)
```

The above code will produce the following SQL query:

```sql
SELECT DISTINCT `pets`.`id`, `pets`.`owner_id`, `pets`.`name`, `pets`.`age`, `pets`.`species` FROM `pets` WHERE `name` LIKE '_B%'
```

#### 自定义SQL函数

若要使用内置的 SQL 函数，例如 `DATE()`，请使用以下选项之一：

1\. 允许一个 dialect-aware 断言方法可以使用 `sql.P` 选项：

```go
users := client.User.Query().
    Select(user.FieldID).
    Where(sql.P(func(b *sql.Builder) {
        b.WriteString("DATE(").Ident("last_login_at").WriteByte(')').WriteOp(OpGTE).Arg(value)
    })).
    AllX(ctx)
```

上面的代码实质上生成了以下 SQL 查询：

```sql
SELECT `id` FROM `users` WHERE DATE(`last_login_at`) >= ?
```

2\. 使用 `ExprP()` 选项内嵌到断言中：

```go
users := client.User.Query().
    Select(user.FieldID).
    Where(func(s *sql.Selector) {
        s.Where(sql.ExprP("DATE(last_login_at) >= ?", value))
    }).
    AllX(ctx)
```

上面的代码实质上生成了以下 SQL 查询：

```sql
SELECT `id` FROM `users` WHERE DATE(`last_login_at`) >= ?
```

## JSON 断言

JSON断言不是默认作为代码生成的一部分。 然而， ent提供官方的包 [`sqljson`](https://pkg.go.dev/entgo.io/ent/dialect/sql/sqljson) 用于[自定义断言选项](#custom-predicates) 应用断言在JSON字段中

#### 比较JSON值

```go
sqljson.ValueEQ(user.FieldData, data)

sqljson.ValueEQ(user.FieldURL, "https", sqljson.Path("Scheme"))

sqljson.ValueNEQ(user.FieldData, content, sqljson.DotPath("attributes[1].body.content"))

sqljson.ValueGTE(user.FieldData, status.StatusBadRequest, sqljson.Path("response", "status"))
```

#### 检查JSON键是否存在

```go
sqljson.HasKey(user.FieldData, sqljson.Path("attributes", "[1]", "body"))

sqljson.HasKey(user.FieldData, sqljson.DotPath("attributes[1].body"))
```

请注意，一个带有 `null` 值的字段也与此操作相匹配。

#### 检查 JSON `null` 文本

```go
sqljson.ValueIsNull(user.FieldData)

sqljson.ValueIsNull(user.FieldData, sqljson.Path("attributes"))

sqljson.ValueIsNull(user.FieldData, sqljson.DotPath("attributes[1].body"))
```

请注意， `ValueIsNull`字段如果值为 JSON `null`, 但不是数据库的 `NULL` 会返回 true

#### 比较JSON数组的长度

```go
sqljson.LenEQ(user.FieldAttrs, 2)

sql.Or(
    sqljson.LenGT(user.FieldData, 10, sqljson.Path("attributes")),
    sqljson.LenLT(user.FieldData, 20, sqljson.Path("attributes")),
)
```

#### 检查 JSON 值是否包含另一个值

```go
sqljson.ValueContains(user.FieldData, data)

sqljson.ValueContains(user.FieldData, attrs, sqljson.Path("attributes"))

sqljson.ValueContains(user.FieldData, code, sqljson.DotPath("attributes[0].status_code"))
```

#### 检查JSON字符串值是否包含给定的子字符串或有给定的后缀或前缀

```go
sqljson.StringContains(user.FieldURL, "github", sqljson.Path("host"))

sqljson.StringHasSuffix(user.FieldURL, ".com", sqljson.Path("host"))

sqljson.StringHasPrefix(user.FieldData, "20", sqljson.DotPath("attributes[0].status_code"))
```

#### 检查JSON值是否等于列表中的任何值

```go
sqljson.ValueIn(user.FieldURL, []any{"https", "ftp"}, sqljson.Path("Scheme"))

sqljson.ValueNotIn(user.FieldURL, []any{"github", "gitlab"}, sqljson.Path("Host"))
```

