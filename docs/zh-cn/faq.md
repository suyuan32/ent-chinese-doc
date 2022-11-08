## 问题

[如何从struct `T`创建一个实体?](#how-to-create-an-entity-from-a-struct-t)  
[如何创建一个struct (或者mutation) 级别的校验器?](#how-to-create-a-mutation-level-validator)  
[如何编写审核日志扩展?](#how-to-write-an-audit-log-extension)  
[如何编写自定义的sql查询谓语?](#how-to-write-custom-predicates)  
[如何将自定义的sql查询谓语添加到ent代码生成器的资源中?](#how-to-add-custom-predicates-to-the-codegen-assets)  
[如何在PostgreSQL中定义一个网络地址字段?](#how-to-define-a-network-address-field-in-postgresql)  
[如何在MySQL中自定义时间字段类型为`DATETIME`?](#how-to-customize-time-fields-to-type-datetime-in-mysql)  
[如何自定义IDs的产生?](#how-to-use-a-custom-generator-of-ids)  
[如何使用全局自定义XID作为唯一ID?](#how-to-use-a-custom-xid-globally-unique-id)  
[如何在MySQL中定义空间数据类型字段?](#how-to-define-a-spatial-data-type-field-in-mysql)  
[如何扩展生成的模型?](#how-to-extend-the-generated-models)  
[如何扩展生成的构造器?](#how-to-extend-the-generated-builders)   
[如何将原型对象存储在BLOB列中?](#how-to-store-protobuf-objects-in-a-blob-column)  
[如何在表中添加一个 `CHECK` 约束?](#how-to-add-check-constraints-to-table)  
[如何定义一个自定义精度数值字段?](#how-to-define-a-custom-precision-numeric-field)  
[如何配置两个及以上的 `数据库` 并且实现各自的读写?](#how-to-configure-two-or-more-db-to-separate-read-and-write)  
[如何更改MySQL表的charachter set或者collation?](#how-to-change-the-character-set-andor-collation-of-a-mysql-table)

## 回答

#### 如何通过结构体 `T` 创建一个实体？

构建器不支持从给定的结构体 `T` 中设置实体字段 (或关系)。 原因是，在更新数据库时无法区分空值和实际的0值 (例如， `&ent.T{Age: 0, Name: ""}`). 设置这些值的时候，可能在数据中写入错误的值或者更新不必要的列。

然而， [外部模板](templates.md) 选项能让你通过添加自定义逻辑，来扩展默认代码生成资源。 例如使用以下模板，可以为每个创建构造器生成一个方法，接受一个结构体作为输入并配置构造器：

```gotemplate
{{ range $n := $.Nodes }}
    {{ $builder := $n.CreateName }}
    {{ $receiver := receiver $builder }}

    func ({{ $receiver }} *{{ $builder }}) Set{{ $n.Name }}(input *{{ $n.Name }}) *{{ $builder }} {
        {{- range $f := $n.Fields }}
            {{- $setter := print "Set" $f.StructField }}
            {{ $receiver }}.{{ $setter }}(input.{{ $f.StructField }})
        {{- end }}
        return {{ $receiver }}
    }
{{ end }}
```

#### 如何创建一个更变时校验器？

要实现一个更变时校验器， 你既可以使用 [schema hooks](hooks.md#schema-hooks) 来验证应用于一个实体类型的更变， 也可以使用 [transaction hooks](transactions.md#hooks) 来验证应用于多个实体类型的更变 (好比一次GraphQL更变)。 举个例子：

```go
// VersionHook是一个假的例子，它可以验证 "version" 字段
// 在每次更新时增加1。 注意，这只是一个假的例子，
// 且并不保证数据库的一致性。
func VersionHook() ent.Hook {
    type OldSetVersion interface {
        SetVersion(int)
        Version() (int, bool)
        OldVersion(context.Context) (int, error)
    }
    return func(next ent.Mutator) ent.Mutator {
        return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
            ver, ok := m.(OldSetVersion)
            if !ok {
                return next.Mutate(ctx, m)
            }
            oldV, err := ver.OldVersion(ctx)
            if err != nil {
                return nil, err
            }
            curV, exists := ver.Version()
            if !exists {
                return nil, fmt.Errorf("version field is required in update mutation")
            }
            if curV != oldV+1 {
                return nil, fmt.Errorf("version field must be incremented by 1")
            }
            // 应添加一个SQL断言，验证 "version" 列是否等于 "oldV"
            // (确保它在更变过程中没有被其他人改变)
            return next.Mutate(ctx, m)
        })
    }
}
```

#### 如何编写一个审计日志的扩展？

编写此扩展的推荐方式是使用 [ent.Mixin](schema-mixin.md)。 使用 `Fields` 选项来为所有引入该 mixed-schema 的 Schema 设置相同的字段，并使用 `Hooks` 选项来为所有正在使用这些 Schema 的变更操作附加一个钩子。 下面是一个例子，基于 [代码仓库 Issue 追踪器](https://github.com/ent/ent/issues/830) 中的讨论。

```go
// AuditMixin 实现了 ent.Mixin，
// 用于 schema 包内通用的审计日志功能。
type AuditMixin struct{
    mixin.Schema
}

// Fields of the AuditMixin.
func (AuditMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Time("created_at").
            Immutable().
            Default(time.Now),
        field.Int("created_by").
            Optional(),
        field.Time("updated_at").
            Default(time.Now).
            UpdateDefault(time.Now),
        field.Int("updated_by").
            Optional(),
    }
}

// AuditMixin 的钩子
func (AuditMixin) Hooks() []ent.Hook {
    return []ent.Hook{
        hooks.AuditHook,
    }
}

// AuditHook 是审计日志钩子的示例
func AuditHook(next ent.Mutator) ent.Mutator {
    // AuditLogger 声明了所有嵌入 AuditLog mixin 的
    // Schema 更变所共有的方法。 若变量 "existence "为真，
    // 则该字段存在于此更变中 (例如被其它的钩子设置过)。
    type AuditLogger interface {
        SetCreatedAt(time.Time)
        CreatedAt() (value time.Time, exists bool)
        SetCreatedBy(int)
        CreatedBy() (id int, exists bool)
        SetUpdatedAt(time.Time)
        UpdatedAt() (value time.Time, exists bool)
        SetUpdatedBy(int)
        UpdatedBy() (id int, exists bool)
    }
    return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
        ml, ok := m.(AuditLogger)
        if !ok {
            return nil, fmt.Errorf("unexpected audit-log call from mutation type %T", m)
        }
        usr, err := viewer.UserFromContext(ctx)
        if err != nil {
            return nil, err
        }
        switch op := m.Op(); {
        case op.Is(ent.OpCreate):
            ml.SetCreatedAt(time.Now())
            if _, exists := ml.CreatedBy(); !exists {
                ml.SetCreatedBy(usr.ID)
            }
        case op.Is(ent.OpUpdateOne | ent.OpUpdate):
            ml.SetUpdatedAt(time.Now())
            if _, exists := ml.UpdatedBy(); !exists {
                ml.SetUpdatedBy(usr.ID)
            }
        }
        return next.Mutate(ctx, m)
    })
}
```

#### 如何编写自定义的断言？

用户可以在执行查询之前可以提供自定义的断言。 例如：

```go
pets := client.Pet.
    Query().
    Where(predicate.Pet(func(s *sql.Selector) {
        s.Where(sql.InInts(pet.OwnerColumn, 1, 2, 3))
    })).
    AllX(ctx)

users := client.User.
    Query().
    Where(predicate.User(func(s *sql.Selector) {
        s.Where(sqljson.ValueContains(user.FieldTags, "tag"))
    })).
    AllX(ctx)
```

更多例子，请到 [断言](predicates.md#custom-predicates) 页面，或者在代码仓库Issue跟踪器中搜索更多的高级例子，如 [issue-842](https://github.com/ent/ent/issues/842#issuecomment-707896368)。

#### 如何将自定义的sql查询谓语添加到ent代码生成器的资源中?

[template](templates.md)选项提供了扩展或者复写默认代码生成资源的能力 为了产生类型安全的查询谓语，[对于上述例子](#how-to-write-custom-predicates)使用如下模板选项实现相同效果：

```gotemplate
{{/* 添加了"<F>Glob"谓语的模板作用于所有的string字段 */}}
{{ define "where/additional/strings" }}
    {{ range $f := $.Fields }}
        {{ if $f.IsString }}
            {{ $func := print $f.StructField "Glob" }}
            // {{ $func }} applies the Glob predicate on the {{ quote $f.Name }} field.
            func {{ $func }}(pattern string) predicate.{{ $.Name }} {
                return predicate.{{ $.Name }}(func(s *sql.Selector) {
                    s.Where(sql.P(func(b *sql.Builder) {
                        b.Ident(s.C({{ $f.Constant }})).WriteString(" glob" ).Arg(pattern)
                    }))
                })
            }
        {{ end }}
    {{ end }}
{{ end }}
```

#### 如何在 PostgreSQL 中定义一个网络地址字段？

[GoType](schema-fields.md#go-type) 和 [SchemaType](schema-fields.md#database-type) 选项允许用户定义数据库特定字段。 例如，要定义 [`macaddr`](https://www.postgresql.org/docs/13/datatype-net-types.html#DATATYPE-MACADDR) 字段，请使用以下配置：

```go
func (T) Fields() []ent.Field {
    return []ent.Field{
        field.String("mac").
            GoType(&MAC{}).
            SchemaType(map[string]string{
                dialect.Postgres: "macaddr",
            }).
            Validate(func(s string) error {
                _, err := net.ParseMAC(s)
                return err
            }),
    }
}

// MAC表示物理硬件地址
type MAC struct {
    net.HardwareAddr
}

// Scan继承了Scanner接口
func (m *MAC) Scan(value any) (err error) {
    switch v := value.(type) {
    case nil:
    case []byte:
        m.HardwareAddr, err = net.ParseMAC(string(v))
    case string:
        m.HardwareAddr, err = net.ParseMAC(v)
    default:
        err = fmt.Errorf("unexpected type %T", v)
    }
    return
}

// Value继承了Valuer驱动接口
func (m MAC) Value() (driver.Value, error) {
    return m.HardwareAddr.String(), nil
}
```
注意，如果数据库不支持`macaddr`类型(例如：SQLite还在测试中)，字段将会回退为它的原生类型(即`string`)

`inet` 示例：

```go
func (T) Fields() []ent.Field {
    return []ent.Field{
        field.String("ip").
            GoType(&Inet{}).
            SchemaType(map[string]string{
                dialect.Postgres: "inet",
            }).
            Validate(func(s string) error {
                if net.ParseIP(s) == nil {
                    return fmt.Errorf("invalid value for ip %q", s)
                }
                return nil
            }),
    }
}

// Inet表示单个IP地址
type Inet struct {
    net.IP
}

// Scan继承了Scanner接口
func (i *Inet) Scan(value any) (err error) {
    switch v := value.(type) {
    case nil:
    case []byte:
        if i.IP = net.ParseIP(string(v)); i.IP == nil {
            err = fmt.Errorf("invalid value for ip %q", v)
        }
    case string:
        if i.IP = net.ParseIP(v); i.IP == nil {
            err = fmt.Errorf("invalid value for ip %q", v)
        }
    default:
        err = fmt.Errorf("unexpected type %T", v)
    }
    return
}

// Value继承了Valuer驱动接口
func (i Inet) Value() (driver.Value, error) {
    return i.IP.String(), nil
}
```

#### 如何在MySQL中自定义时间字段类型为`DATETIME`?

`Time`字段在数据库模式中默认使用MySQL `TIMESTAMP`类型，但是这个类型只支持 '1970-01-01 00:00:01' UTC到'2038-01-19 03:14:07' UTC之间的范围(参见[MySQL文档](https://dev.mysql.com/doc/refman/5.6/en/datetime.html))

为了自定义范围更广的时间域，请使用MySQL的`DATETIME`属性，如下所示:
```go
field.Time("birth_date").
    Optional().
    SchemaType(map[string]string{
        dialect.MySQL: "datetime",
    }),
```

#### 如何使用自定义的 ID 生成器？

如果你在你的数据库使用自定义的ID生成器替换自增IDs(例如：推特的[Snowflake](https://github.com/twitter-archive/snowflake/tree/snowflake-2010))，你需要在资源创建的时候自动调用生成器写入自定义的ID字段。

取决于你自己，你可以使用`DefaultFunc`或者模式钩子(schema hooks) 实现。 如果生成器并不需要返回错误，那么使用`DefaultFunc`就是最佳选择。否则的话，在资源创建的时候配置一个钩子会帮助你捕获到完整的错误信息。 可以在[ID字段](schema-fields.md#id-field)章节查看如何使用`DefaultFunc`的例子。

以[sonyflake](https://github.com/sony/sonyflake)为例，下面是一个如何使用自定义钩子生成器的例子。

```go
// BaseMixin将在不同的数据库模式之间共享
type BaseMixin struct {
    mixin.Schema
}

// Fields of the Mixin.
func (BaseMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Uint64("id"),
    }
}

// Mixin的钩子
func (BaseMixin) Hooks() []ent.Hook {
    return []ent.Hook{
        hook.On(IDHook(), ent.OpCreate),
    }
}

func IDHook() ent.Hook {
    sf := sonyflake.NewSonyflake(sonyflake.Settings{})
    type IDSetter interface {
        SetID(uint64)
    }
    return func(next ent.Mutator) ent.Mutator {
        return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
            is, ok := m.(IDSetter)
            if !ok {
                return nil, fmt.Errorf("unexpected mutation %T", m)
            }
            id, err := sf.NextID()
            if err != nil {
                return nil, err
            }
            is.SetID(id)
            return next.Mutate(ctx, m)
        })
    }
}

// User约束了User实体的模式定义
type User struct {
    ent.Schema
}

// User的Mixin.
func (User) Mixin() []ent.Mixin {
    return []ent.Mixin{
        // 在用户模式中嵌入BaseMixin
        BaseMixin{},
    }
}
```

#### 如何使用全局自定义XID作为唯一ID?

Package [xid](https://github.com/rs/xid) is a globally unique ID generator library that uses the [Mongo Object ID](https://docs.mongodb.org/manual/reference/object-id/) algorithm to generate a 12 byte, 20 character ID with no configuration. The xid package comes with [database/sql](https://pkg.go.dev/database/sql) `sql.Scanner` and `driver.Valuer` interfaces required by Ent for serialization.

To store an XID in any string field use the [GoType](schema-fields.md#go-type) schema configuration:

```go
// Fields of type T.
func (T) Fields() []ent.Field {
    return []ent.Field{
        field.String("id").
            GoType(xid.ID{}).
            DefaultFunc(xid.New),
    }
}
```

Or as a reusable [Mixin](schema-mixin.md) across multiple schemas:

```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/mixin"
    "github.com/rs/xid"
)

// BaseMixin to be shared will all different schemas.
type BaseMixin struct {
    mixin.Schema
}

// Fields of the User.
func (BaseMixin) Fields() []ent.Field {
    return []ent.Field{
        field.String("id").
            GoType(xid.ID{}).
            DefaultFunc(xid.New),
    }
}

// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

// Mixin of the User.
func (User) Mixin() []ent.Mixin {
    return []ent.Mixin{
        // Embed the BaseMixin in the user schema.
        BaseMixin{},
    }
}
```

In order to use extended identifiers (XIDs) with gqlgen, follow the configuration mentioned in the [issue tracker](https://github.com/ent/ent/issues/1526#issuecomment-831034884).

#### How to define a spatial data type field in MySQL?

The [GoType](schema-fields.md#go-type) and the [SchemaType](schema-fields.md#database-type) options allow users to define database-specific fields. For example, in order to define a [`POINT`](https://dev.mysql.com/doc/refman/8.0/en/spatial-type-overview.html) field, use the following configuration:

```go
// Fields of the Location.
func (Location) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
        field.Other("coords", &Point{}).
            SchemaType(Point{}.SchemaType()),
    }
}
```

```go
package schema

import (
    "database/sql/driver"
    "fmt"

    "entgo.io/ent/dialect"
    "entgo.io/ent/dialect/sql"
    "github.com/paulmach/orb"
    "github.com/paulmach/orb/encoding/wkb"
)

// A Point consists of (X,Y) or (Lat, Lon) coordinates
// and it is stored in MySQL the POINT spatial data type.
type Point [2]float64

// Scan implements the Scanner interface.
func (p *Point) Scan(value any) error {
    bin, ok := value.([]byte)
    if !ok {
        return fmt.Errorf("invalid binary value for point")
    }
    var op orb.Point
    if err := wkb.Scanner(&op).Scan(bin[4:]); err != nil {
        return err
    }
    p[0], p[1] = op.X(), op.Y()
    return nil
}

// Value implements the driver Valuer interface.
func (p Point) Value() (driver.Value, error) {
    op := orb.Point{p[0], p[1]}
    return wkb.Value(op).Value()
}

// FormatParam implements the sql.ParamFormatter interface to tell the SQL
// builder that the placeholder for a Point parameter needs to be formatted.
func (p Point) FormatParam(placeholder string, info *sql.StmtInfo) string {
    if info.Dialect == dialect.MySQL {
        return "ST_GeomFromWKB(" + placeholder + ")"
    }
    return placeholder
}

// SchemaType defines the schema-type of the Point object.
func (Point) SchemaType() map[string]string {
    return map[string]string{
        dialect.MySQL: "POINT",
    }
}
```

A full example exists in the [example repository](https://github.com/a8m/entspatial).


#### How to extend the generated models?

Ent supports extending generated types (both global types and models) using custom templates. For example, in order to add additional struct fields or methods to the generated model, we can override the `model/fields/additional` template like in this [example](https://github.com/ent/ent/blob/dd4792f5b30bdd2db0d9a593a977a54cb3f0c1ce/examples/entcpkg/ent/template/static.tmpl).


If your custom fields/methods require additional imports, you can add those imports using custom templates as well:

```gotemplate
{{- define "import/additional/field_types" -}}
    "github.com/path/to/your/custom/type"
{{- end -}}

{{- define "import/additional/client_dependencies" -}}
    "github.com/path/to/your/custom/type"
{{- end -}}
```

#### How to extend the generated builders?

See the *[Injecting External Dependencies](code-gen.md#external-dependencies)* section, or follow the example on [GitHub](https://github.com/ent/ent/tree/master/examples/entcpkg).

#### How to store Protobuf objects in a BLOB column?

Assuming we have a Protobuf message defined:
```protobuf
syntax = "proto3";

package pb;

option go_package = "project/pb";

message Hi {
  string Greeting = 1;
}
```

We add receiver methods to the generated protobuf struct such that it implements [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field#ValueScanner)

```go
func (x *Hi) Value() (driver.Value, error) {
    return proto.Marshal(x)
}

func (x *Hi) Scan(src any) error {
    if src == nil {
        return nil
    }
    if b, ok := src.([]byte); ok {
        if err := proto.Unmarshal(b, x); err != nil {
            return err
        }
        return nil
    }
    return fmt.Errorf("unexpected type %T", src)
}
```

We add a new `field.Bytes` to our schema, setting the generated protobuf struct as its underlying `GoType`:

```go
// Fields of the Message.
func (Message) Fields() []ent.Field {
    return []ent.Field{
        field.Bytes("hi").
            GoType(&pb.Hi{}),
    }
}
```

Test that it works:

```go
package main

import (
    "context"
    "testing"

    "project/ent/enttest"
    "project/pb"

    _ "github.com/mattn/go-sqlite3"
    "github.com/stretchr/testify/require"
)

func TestMain(t *testing.T) {
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
    defer client.Close()

    msg := client.Message.Create().
        SetHi(&pb.Hi{
            Greeting: "hello",
        }).
        SaveX(context.TODO())

    ret := client.Message.GetX(context.TODO(), msg.ID)
    require.Equal(t, "hello", ret.Hi.Greeting)
}
```

#### How to add `CHECK` constraints to table?

The [`entsql.Annotation`](schema-annotations.md) option allows adding custom `CHECK` constraints to the `CREATE TABLE` statement. In order to add `CHECK` constraints to your schema, use the following example:

```go
func (User) Annotations() []schema.Annotation {
    return []schema.Annotation{
        &entsql.Annotation{
            // The `Check` option allows adding an
            // unnamed CHECK constraint to table DDL.
            Check: "website <> 'entgo.io'",

            // The `Checks` option allows adding multiple CHECK constraints
            // to table creation. The keys are used as the constraint names.
            Checks: map[string]string{
                "valid_nickname":  "nickname <> firstname",
                "valid_firstname": "length(first_name) > 1",
            },
        },
    }
}
```

#### How to define a custom precision numeric field?

Using [GoType](schema-fields.md#go-type) and [SchemaType](schema-fields.md#database-type) it is possible to define custom precision numeric fields. For example, defining a field that uses [big.Int](https://pkg.go.dev/math/big).

```go
func (T) Fields() []ent.Field {
    return []ent.Field{
        field.Int("precise").
            GoType(new(BigInt)).
            SchemaType(map[string]string{
                dialect.SQLite:   "numeric(78, 0)",
                dialect.Postgres: "numeric(78, 0)",
            }),
    }
}

type BigInt struct {
    big.Int
}

func (b *BigInt) Scan(src any) error {
    var i sql.NullString
    if err := i.Scan(src); err != nil {
        return err
    }
    if !i.Valid {
        return nil
    }
    if _, ok := b.Int.SetString(i.String, 10); ok {
        return nil
    }
    return fmt.Errorf("could not scan type %T with value %v into BigInt", src, src)
}

func (b *BigInt) Value() (driver.Value, error) {
    return b.String(), nil
}
```

#### How to configure two or more `DB` to separate read and write?

You can wrap the `dialect.Driver` with your own driver and implement this logic. For example.

You can extend it, add support for multiple read replicas and add some load-balancing magic.

```go
func main() {
    // ...
    wd, err := sql.Open(dialect.MySQL, "root:pass@tcp(<addr>)/<database>?parseTime=True")
    if err != nil {
        log.Fatal(err)
    }
    rd, err := sql.Open(dialect.MySQL, "readonly:pass@tcp(<addr>)/<database>?parseTime=True")
    if err != nil {
        log.Fatal(err)
    }
    client := ent.NewClient(ent.Driver(&multiDriver{w: wd, r: rd}))
    defer client.Close()
    // Use the client here.
}


type multiDriver struct {
    r, w dialect.Driver
}

var _ dialect.Driver = (*multiDriver)(nil)

func (d *multiDriver) Query(ctx context.Context, query string, args, v any) error {
    return d.r.Query(ctx, query, args, v)
}

func (d *multiDriver) Exec(ctx context.Context, query string, args, v any) error {
    return d.w.Exec(ctx, query, args, v)
}

func (d *multiDriver) Tx(ctx context.Context) (dialect.Tx, error) {
    return d.w.Tx(ctx)
}

func (d *multiDriver) BeginTx(ctx context.Context, opts *sql.TxOptions) (dialect.Tx, error) {
    return d.w.(interface {
        BeginTx(context.Context, *sql.TxOptions) (dialect.Tx, error)
    }).BeginTx(ctx, opts)
}

func (d *multiDriver) Close() error {
    rerr := d.r.Close()
    werr := d.w.Close()
    if rerr != nil {
        return rerr
    }
    if werr != nil {
        return werr
    }
    return nil
}
```

#### How to change the character set and/or collation of a MySQL table?

By default for MySQL the character set `utf8mb4` is used and the collation of `utf8mb4_bin`. However if you'd like to change the schema's character set and/or collation you need to use an annotation.

Here's an example where we set the character set to `ascii` and the collation to `ascii_general_ci`.

```go
// Annotations of the Entity.
func (Entity) Annotations() []schema.Annotation {
    return []schema.Annotation{
        entsql.Annotation{
            Charset:   "ascii",
            Collation: "ascii_general_ci",
        },
    }
}
```
