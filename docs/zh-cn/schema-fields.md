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

#### `Nillable` required fields

`Nillable` fields are also helpful for avoiding zero values in JSON marshaling for fields that have not been `Select`ed in the query. For example, a `time.Time` field.

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

The generated struct for the `Task` entity will be as follows:

```go title="ent/task.go"
package ent

// Task entity.
type Task struct {
    // CreatedAt holds the value of the "created_at" field.
    CreatedAt time.Time `json:"created_at,omitempty"`
    // NillableCreatedAt holds the value of the "nillable_created_at" field.
    NillableCreatedAt *time.Time `json:"nillable_created_at,omitempty"`
}
```

And the result of `json.Marshal` is:

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

Immutable fields are fields that can be set only in the creation of the entity. i.e., no setters will be generated for the update builders of the entity.

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
Fields can be defined as unique using the `Unique` method. Note that unique fields cannot have default values.

```go {5}
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("nickname").
            Unique(),
    }
}
```

## Comments

A comment can be added to a field using the `.Comment()` method. This comment appears before the field in the generated entity code. Newlines are supported using the `\n` escape sequence.

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

## Storage Key

Custom storage name can be configured using the `StorageKey` method. It's mapped to a column name in SQL dialects and to property name in Gremlin.

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            StorageKey("old_name"),
    }
}
```

## Indexes
Indexes can be defined on multi fields and some types of edges as well. However, you should note, that this is currently an SQL-only feature.

Read more about this in the [Indexes](schema-indexes.md) section.

## Struct Tags

Custom struct tags can be added to the generated entities using the `StructTag` method. Note that if this option was not provided, or provided and did not contain the `json` tag, the default `json` tag will be added with the field name.

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            StructTag(`gqlgen:"gql_name"`),
    }
}
```

## Additional Struct Fields

By default, `ent` generates the entity model with fields that are configured in the `schema.Fields` method. For example, given this schema configuration:

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

The generated model will be as follows:

```go
// User is the model entity for the User schema.
type User struct {
    // Age holds the value of the "age" field.
    Age  *int   `json:"age,omitempty"`
    // Name holds the value of the "name" field.
    Name string `json:"name,omitempty" gqlgen:"gql_name"`
}
```

In order to add additional fields to the generated struct **that are not stored in the database**, use [external templates](code-gen.md/#external-templates). For example:

```gotemplate
{{ define "model/fields/additional" }}
    {{- if eq $.Name "User" }}
        // StaticField defined by template.
        StaticField string `json:"static,omitempty"`
    {{- end }}
{{ end }}
```

The generated model will be as follows:

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

## Sensitive Fields

String fields can be defined as sensitive using the `Sensitive` method. Sensitive fields won't be printed and they will be omitted when encoding.

Note that sensitive fields cannot have struct tags.

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

## Enum Fields

The `Enum` builder allows creating enum fields with a list of permitted values.

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

When a custom [`GoType`](#go-type) is being used, it is must be convertible to the basic `string` type or it needs to implement the [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field#ValueScanner) interface.

The [EnumValues](https://pkg.go.dev/entgo.io/ent/schema/field#EnumValues) interface is also required by the custom Go type to tell Ent what are the permitted values of the enum.

The following example shows how to define an `Enum` field with a custom Go type that is convertible to `string`:

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

Implement the [EnumValues](https://pkg.go.dev/entgo.io/ent/schema/field#EnumValues) interface.
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
The following example shows how to define an `Enum` field with a custom Go type that is not convertible to `string`, but it implements the [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field#ValueScanner) interface:

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
Implement also the [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field?tab=doc#ValueScanner) interface.

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

Combining it all together:
```go
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("first_name"),
        field.String("last_name"),
        field.Enum("size").
            Values("big", "small"),
        // A convertible type to string.
        field.Enum("shape").
            GoType(property.Shape("")),
        // Add conversion to and from string.
        field.Enum("level").
            GoType(property.Level(0)),
    }
}
```

After code generation usage is trivial:
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

## Annotations

`Annotations` is used to attach arbitrary metadata to the field object in code generation. Template extensions can retrieve this metadata and use it inside their templates.

Note that the metadata object must be serializable to a JSON raw value (e.g. struct, map or slice).

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

Read more about annotations and their usage in templates in the [template doc](templates.md#annotations).

## Naming Convention

By convention field names should use `snake_case`. The corresponding struct fields generated by `ent` will follow the Go convention of using `PascalCase`. In cases where `PascalCase` is desired, you can do so with the `StorageKey` or `StructTag` methods.
