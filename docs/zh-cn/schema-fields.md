## 概述

Schema 中的字段（或属性）是节点的属性。 例如：`User` 有 `age`， `name`, `username` 和 `created_at` 4个字段。

![re-fields-properties](https://entgo.io/images/assets/er_fields_properties.png)

用 schema 的 `Fields ` 方法可返回这些字段。 如：

```go
package schema

import (
    "time"

    "entgo.io/ent"
    "entgo.io/ent/schema/field"
)

// User schema.
type User struct {
    ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age"),
        field.String("name"),
        field.String("username").
            Unique(),
        field.Time("created_at").
            Default(time.Now),
    }
}
```

默认所有字段都是必需的，可使用 `Optional` 方法设置为可选字段。

## 类型

目前框架支持以下数据类型：

- Go中所有数值类型。 如 `int`，`uint8`，`float64` 等
- `bool 布尔型`
- `string 字符串`
- `time.Time 时间类型`
- `UUID`
- `[]byte` （仅限SQL）。
- `JSON`（仅限SQL）。
- `Enum`（仅限SQL）。
- `其它类型` （仅限SQL）

```go
package schema

import (
    "time"
    "net/url"

    "github.com/google/uuid"
    "entgo.io/ent"
    "entgo.io/ent/schema/field"
)

// User schema.
type User struct {
    ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age").
            Positive(),
        field.Float("rank").
            Optional(),
        field.Bool("active").
            Default(false),
        field.String("name").
            Unique(),
        field.Time("created_at").
            Default(time.Now),
        field.JSON("url", &url.URL{}).
            Optional(),
        field.JSON("strings", []string{}).
            Optional(),
        field.Enum("state").
            Values("on", "off").
            Optional(),
        field.UUID("uuid", uuid.UUID{}).
            Default(uuid.New),
    }
}
```

想要了解更多关于每种类型是如何映射到数据库类型的，请移步 [数据迁移](migrate.md) 章节。

## ID 字段

`id` 字段内置于架构中，无需声明。 在基于 SQL 的数据库中，它的类型默认为 `int` （可以使用 [代码生成设置](code-gen.md#code-generation-options) 更改）并自动递增。

如果想要配置 `id` 字段在所有表中唯一，可以在运行 schema 迁移时使用 [WithGlobalUniqueID](migrate.md#universal-ids) 选项实现。

如果需要对 `id` 字段进行其他配置，或者要使用由应用程序在实体创建时提供的 `id` （例如UUID），可以覆盖内置 `id` 配置。 如：

```go
// Fields of the Group.
func (Group) Fields() []ent.Field {
    return []ent.Field{
        field.Int("id").
            StructTag(`json:"oid,omitempty"`),
    }
}

// Fields of the Blob.
func (Blob) Fields() []ent.Field {
    return []ent.Field{
        field.UUID("id", uuid.UUID{}).
            Default(uuid.New).
            StorageKey("oid"),
    }
}

// Fields of the Pet.
func (Pet) Fields() []ent.Field {
    return []ent.Field{
        field.String("id").
            MaxLen(25).
            NotEmpty().
            Unique().
            Immutable(),
    }
}
```

如果你需要设置一个自定义函数来生成 ID， 使用 `DefaultFunc` 方法来指定一个函数，每次ID将由此函数生成。 更多信息请参阅 [相关常见问题](faq.md#how-do-i-use-a-custom-generator-of-ids)。

```go
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int64("id").
            DefaultFunc(func() int64 {
                // An example of a dumb ID generator - use a production-ready alternative instead.
                return time.Now().Unix() << 8 | atomic.AddInt64(&counter, 1) % 256
            }),
    }
}
```

## 数据库字段类型

每个数据库方言都有自己Go类型与数据库类型的映射。 例如，MySQL 方言将Go类型为 `float64` 的字段创建为 `double` 的数据库字段。 当然，我们也可以通过 `SchemaType` 方法来重写默认的类型映射。

```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/dialect"
    "entgo.io/ent/schema/field"
)

// Card schema.
type Card struct {
    ent.Schema
}

// Fields of the Card.
func (Card) Fields() []ent.Field {
    return []ent.Field{
        field.Float("amount").
            SchemaType(map[string]string{
                dialect.MySQL:    "decimal(6,2)",   // Override MySQL.
                dialect.Postgres: "numeric",        // Override Postgres.
            }),
    }
}
```

## Go 类型

字段的默认类型是基本的 Go 类型。 例如，对于字符串字段，类型是 `string`, 对于时间字段，类型是 `time.Time`。 `GoType` 方法提供了以自定义类型覆盖 默认类型的选项。

自定义类型必须是可转换为Go基本类型，或者实现 [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field?tab=doc#ValueScanner) 接口的类型。


```go
package schema

import (
    "database/sql"

    "entgo.io/ent"
    "entgo.io/ent/dialect"
    "entgo.io/ent/schema/field"
    "github.com/shopspring/decimal"
)

// Amount is a custom Go type that's convertible to the basic float64 type.
type Amount float64

// Card schema.
type Card struct {
    ent.Schema
}

// Fields of the Card.
func (Card) Fields() []ent.Field {
    return []ent.Field{
        field.Float("amount").
            GoType(Amount(0)),
        field.String("name").
            Optional().
            // A ValueScanner type.
            GoType(&sql.NullString{}),
        field.Enum("role").
            // A convertible type to string.
            GoType(role.Role("")),
        field.Float("decimal").
            // A ValueScanner type mixed with SchemaType.
            GoType(decimal.Decimal{}).
            SchemaType(map[string]string{
                dialect.MySQL:    "decimal(6,2)",
                dialect.Postgres: "numeric",
            }),
    }
}
```

## 其它字段

Other 代表一个不适合任何标准字段类型的字段。 示例为 Postgres 中的 Rage 类型或 Geospatial 类型

```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/dialect"
    "entgo.io/ent/schema/field"

    "github.com/jackc/pgtype"
)

// User schema.
type User struct {
    ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Other("duration", &pgtype.Tstzrange{}).
            SchemaType(map[string]string{
                dialect.Postgres: "tstzrange",
            }),
    }
}
```

## 默认值

**非唯一** 字段可使用 `Default` 和 `UpdateDefault` 方法为其设置默认值。 你也可以指定 `DefaultFunc` 方法来自定义默认值生成。

```go
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Time("created_at").
            Default(time.Now),
        field.Time("updated_at").
            Default(time.Now).
            UpdateDefault(time.Now),
        field.String("name").
            Default("unknown"),
        field.String("cuid").
            DefaultFunc(cuid.New),
        field.JSON("dirs", []http.Dir{}).
            Default([]http.Dir{"/tmp"}),
    }
}
```

可以通过 [`entsql.Annotation`](https://pkg.go.dev/entgo.io/ent@master/dialect/entsql#Annotation) 将像函数调用的SQL特定表达式添加到默认值配置中：

```go
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        // Add a new field with CURRENT_TIMESTAMP
        // as a default value to all previous rows.
        field.Time("created_at").
            Default(time.Now).
            Annotations(&entsql.Annotation{
                Default: "CURRENT_TIMESTAMP",
            }),
    }
}
```

为避免你指定的 `DefaultFunc` 方法也返回了一个错误，最好使用 [schema-hooks](hooks.md#schema-hooks) 处理它。 更多信息请参阅 [相关常见问题](faq.md#how-to-use-a-custom-generator-of-ids)。

## 校验器

字段校验器是一个 `func(T) error` 类型的函数，定义在 schema 的 `Validate` 方法中，字段在创建或更新前会执行此方法。

支持 `string` 类型和所有数值类型。

```go
package schema

import (
    "errors"
    "regexp"
    "strings"
    "time"

    "entgo.io/ent"
    "entgo.io/ent/schema/field"
)


// Group schema.
type Group struct {
    ent.Schema
}

// Fields of the group.
func (Group) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            Match(regexp.MustCompile("[a-zA-Z_]+$")).
            Validate(func(s string) error {
                if strings.ToLower(s) == s {
                    return errors.New("group name must begin with uppercase")
                }
                return nil
            }),
    }
}
```

又如：编写一个可复用的校验器

```go
import (
    "entgo.io/ent/dialect/entsql"
    "entgo.io/ent/schema/field"
)

// MaxRuneCount validates the rune length of a string by using the unicode/utf8 package.
func MaxRuneCount(maxLen int) func(s string) error {
    return func(s string) error {
        if utf8.RuneCountInString(s) > maxLen {
            return errors.New("value is more than the max length")
        }
        return nil
    }
}

field.String("name").
    // If using a SQL-database: change the underlying data type to varchar(10).
    Annotations(entsql.Annotation{
        Size: 10,
    }).
    Validate(MaxRuneCount(10))
field.String("nickname").
    //  If using a SQL-database: change the underlying data type to varchar(20).
    Annotations(entsql.Annotation{
        Size: 20,
    }).
    Validate(MaxRuneCount(20))
```

## 内置校验器

框架为每个类型提供了几个内置的验证器：

- 数值类型：
  - `Positive()`
  - `Negative()`
  - `NonNegative()`
  - `Min(i)` - 验证给定的值 > i。
  - `Max(i)` - 验证给定的值 < i。
  - `Range(i, j)` - 验证给定值在 [i, j] 之间。

- `string 字符串`
  - `MinLen(i)`
  - `MaxLen(i)`
  - `Match(regexp.Regexp)`
  - `NotEmpty`

- `[]byte`
  - `MaxLen(i)`
  - `MinLen(i)`
  - `NotEmpty`

## Optional 可选项

可选字段为创建时非必须的字段，在数据库中被设置为 null。 和 edges 不同，**字段默认都为必需字段**，可通过 `Optional` 方法显示的设为可选字段。


```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("required_name"),
        field.String("optional_name").
            Optional(),
    }
}
```

## Nillable 空值
Sometimes you want to be able to distinguish between the zero value of fields and `nil`. For example, if the database column contains `0` or `NULL`.  The `Nillable` option exists exactly for this.

如果你有一个类型为 `T` 的 `可选字段`，设置为 `Nillable` 后，将生成一个类型为 `*T` 的结构体字段。 因此，如果数据库返回 `NULL` 字段， 结构体字段将为 `nil` 值。 Otherwise, it will contain a pointer to the actual value.

例如，在这个 schema 中：
```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("required_name"),
        field.String("optional_name").
            Optional(),
        field.String("nillable_name").
            Optional().
            Nillable(),
    }
}
```

`User` 生成的结构体如下：

```go title="ent/user.go"
package ent

// User entity.
type User struct {
    RequiredName string `json:"required_name,omitempty"`
    OptionalName string `json:"optional_name,omitempty"`
    NillableName *string `json:"nillable_name,omitempty"`
}
```

#### `Nillable` 必要字段

`Nillable` 字段同样有助与避免JSON反序列化遇到未被选择的空值， 如 `time.Time` 字段

```go
// Fields of the task.
func (Task) Fields() []ent.Field {
    return []ent.Field{
        field.Time("created_at").
            Default(time.Now),
        field.Time("nillable_created_at").
            Default(time.Now).
            Nillable(),
    }
}
```

生成的 `Task` 实例如下:

```go title="ent/task.go"
package ent

// Task entity.
type Task struct {
    // CreatedAt 存储 "created_at" 字段.
    CreatedAt time.Time `json:"created_at,omitempty"`
    // NillableCreatedAt 存储 "nillable_created_at" 字段.
    NillableCreatedAt *time.Time `json:"nillable_created_at,omitempty"`
}
```

生成的序列 `json.Marshal` 如下:

```go
b, _ := json.Marshal(Task{})
fmt.Printf("%s\n", b)
//highlight-next-line-info
// {"created_at":"0001-01-01T00:00:00Z"}

now := time.Now()
b, _ = json.Marshal(Task{CreatedAt: now, NillableCreatedAt: &now})
fmt.Printf("%s\n", b)
//highlight-next-line-info
// {"created_at":"2009-11-10T23:00:00Z","nillable_created_at":"2009-11-10T23:00:00Z"}
```

## Immutable 不可变的

Immutable 字段只能在创建时设定，之后不能修改. 例如 在生成的update代码中将不会有 set 方法

```go {6}
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Time("created_at").
            Default(time.Now).
            Immutable(),
    }
}
```

## 唯一键
`Unique` 字段用于设定只能存储唯一值的字段. 注意使用 `Unique`后不能使用 `Default`.

```go {5}
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("nickname").
            Unique(),
    }
}
```

## 注释

 `.Comment()` 用于添加注释. 这个注释将会先于实例结构生成. 使用 `\n` 换行.

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            Default("John Doe").
            Comment("Name of the user.\n If not specified, defaults to \"John Doe\"."),
    }
}
```

## 自定义字段

使用 `StorageKey` 来设定自定义字段名. 它将被映射到数据库和Gremlin中.

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            StorageKey("old_name"),
    }
}
```

## 索引
索引可以添加在多个字段和边中. 注意这只能用在 SQL 类型的数据库中.

查看更多 [索引](schema-indexes.md) .

## 结构标签

自定义 tag 可以用 `StructTag` 添加. 注意如果没有添加 `json` 标签或者为添加任何标签, 默认会自动创建 `json` 标签

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            StructTag(`gqlgen:"gql_name"`),
    }
}
```

## 额外的结构字段

默认情况下 `ent` 只生成在 `schema.Fields` 设定的字段. 例如下面:

```go
// User schema.
type User struct {
    ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age").
            Optional().
            Nillable(),
        field.String("name").
            StructTag(`gqlgen:"gql_name"`),
    }
}
```

生成的代码为:

```go
// User is the model entity for the User schema.
type User struct {
    // Age holds the value of the "age" field.
    Age  *int   `json:"age,omitempty"`
    // Name holds the value of the "name" field.
    Name string `json:"name,omitempty" gqlgen:"gql_name"`
}
```

为了添加额外的字段 **不存在数据库中**, 使用 [额外模板](code-gen.md/#external-templates). 例如:

```gotemplate
{{ define "model/fields/additional" }}
    {{- if eq $.Name "User" }}
        // StaticField defined by template.
        StaticField string `json:"static,omitempty"`
    {{- end }}
{{ end }}
```

生成的代码为:

```go
// User is the model entity for the User schema.
type User struct {
    // Age holds the value of the "age" field.
    Age  *int   `json:"age,omitempty"`
    // Name holds the value of the "name" field.
    Name string `json:"name,omitempty" gqlgen:"gql_name"`
    // StaticField defined by template.
    StaticField string `json:"static,omitempty"`
}
```

## 敏感字段

String 字段可以使用 `Sensitive` 定义为敏感字段. 敏感字段将会不会被打印或者序列化到json中.

注意使用 `Sensitive`  后不能再自定义字段名 (StructTag).

```go
// User schema.
type User struct {
    ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("password").
            Sensitive(),
    }
}
```

## 枚举字段

 `Enum` 构造器允许使用一系列数据创建枚举字段.

```go
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("first_name"),
        field.String("last_name"),
        field.Enum("size").
            Values("big", "small"),
    }
}
```

当一个自定义 [`GoType`](#go-type) 被使用, 他必须被转化成基本类型 `string` 或者实现 [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field#ValueScanner) 接口.

[EnumValues](https://pkg.go.dev/entgo.io/ent/schema/field#EnumValues) 接口还需要 GoType 告诉 Ent 哪些字段是被允许的.

下面例子展示了 `Enum` 字段使用 GoType 转化成 `string`:

```go
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("first_name"),
        field.String("last_name"),
        // A convertible type to string.
        field.Enum("shape").
            GoType(property.Shape("")),
    }
}
```

实现 [EnumValues](https://pkg.go.dev/entgo.io/ent/schema/field#EnumValues) 接口.
```go
package property

type Shape string

const (
    Triangle Shape = "TRIANGLE"
    Circle   Shape = "CIRCLE"
)

// Values provides list valid values for Enum.
func (Shape) Values() (kinds []string) {
    for _, s := range []Shape{Triangle, Circle} {
        kinds = append(kinds, string(s))
    }
    return
}

```
下面的 `Enum` 字段使用自定义 GoType 转化成 `string`, 但是实现的是 [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field#ValueScanner) 接口:

```go
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("first_name"),
        field.String("last_name"),
        // Add conversion to and from string
        field.Enum("level").
            GoType(property.Level(0)),
    }
}
```
实现 [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field?tab=doc#ValueScanner) 接口.

```go
package property

import "database/sql/driver"

type Level int

const (
    Unknown Level = iota
    Low
    High
)

func (p Level) String() string {
    switch p {
    case Low:
        return "LOW"
    case High:
        return "HIGH"
    default:
        return "UNKNOWN"
    }
}

// Values provides list valid values for Enum.
func (Level) Values() []string {
    return []string{Unknown.String(), Low.String(), High.String()}
}

// Value provides the DB a string from int.
func (p Level) Value() (driver.Value, error) {
    return p.String(), nil
}

// Scan tells our code how to read the enum into our type.
func (p *Level) Scan(val any) error {
    var s string
    switch v := val.(type) {
    case nil:
        return nil
    case string:
        s = v
    case []uint8:
        s = string(v)
    }
    switch s {
    case "LOW":
        *p = Low
    case "HIGH":
        *p = High
    default:
        *p = Unknown
    }
    return nil
}
```

合在一起的效果:

```go
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("first_name"),
        field.String("last_name"),
        field.Enum("size").
            Values("big", "small"),
        // 一个自动转换的 Enum 字段.
        field.Enum("shape").
            GoType(property.Shape("")),
        // 添加转换到一个 string 字段.
        field.Enum("level").
            GoType(property.Level(0)),
    }
}
```

代码生成后使用非常简单:

```go 
client.User.Create().
    SetFirstName("John").
    SetLastName("Dow").
    SetSize(user.SizeSmall).
    SetShape(property.Triangle).
    SetLevel(property.Low).
    SaveX(context.Background())

john := client.User.Query().FirstX(context.Background())
fmt.Println(john)
// User(id=1, first_name=John, last_name=Dow, size=small, shape=TRIANGLE, level=LOW)
```

## 注解

`Annotations` 用于在代码生成中将任意元数据附加到字段对象。 模板扩展可以检索此元数据并在其模板中使用它

注意 metadata 必须序列化成 JSON 序列 (如. struct, map 或者 slice).

```go
// User schema.
type User struct {
    ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Time("creation_date").
            Annotations(entgql.Annotation{
                OrderField: "CREATED_AT",
            }),
    }
}
```

了解更多 [模板文档](templates.md#annotations).

## 命名转换

默认命名方式为 `snake_case`.  `ent` 允许生成的代码为 `PascalCase` 格式. 如果想用 `PascalCase` 格式, 你可以使用 `StorageKey` 或 `StructTag` 方法自己设置.
