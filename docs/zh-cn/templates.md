`ent` 接受外部 [Go 模板](https://golang.org/pkg/text/template) 以使用 `--template` 标志执行。 如果外部模板名在 `ent` 中已经定义过，它会覆盖已经定义的外部模板。 如果没有定义过，在命令执行完成后，会生成一个与外部模板同名的文件。 例如：

`stringer.tmpl` - 这个外部模板示例文件会生成一个叫做`ent/stringer.go`的文件。

```gotemplate
{{/* 下一行的作用是，告诉 Intellij/GoLand 等IDE，基于 gen.Graph 类型来启用自动补全功能。 */}}
{{/* gotype: entgo.io/ent/entc/gen.Graph */}}

{{ define "stringer" }}

{{/* Add the base header for the generated file */}}
{{ $pkg := base $.Config.Package }}
{{ template "header" $ }}

{{/* Loop over all nodes and implement the "GoStringer" interface */}}
{{ range $n := $.Nodes }}
    {{ $receiver := $n.Receiver }}
    func ({{ $receiver }} *{{ $n.Name }}) GoString() string {
        if {{ $receiver }} == nil {
            return fmt.Sprintf("{{ $n.Name }}(nil)")
        }
        return {{ $receiver }}.String()
    }
{{ end }}

{{ end }}
```

`debug.tmpl` - 这个模板文件会生成一个叫做`ent/debug.go`的文件

```gotemplate
{{ define "debug" }}

{{/* 此模板用于给运行在debug模式下的每个 client<T> 添加一些功能。 */}}

{{/* 给生成的文件添加头部信息 */}}
{{ $pkg := base $.Config.Package }}
{{ template "header" $ }}

{{/* 循环遍历所有节点，并且给配置项添加 "Debug" 方法 */}}
{{ range $n := $.Nodes }}
    {{ $client := print $n.Name "Client" }}
    func (c *{{ $client }}) Debug() *{{ $client }} {
        if c.debug {
            return c
        }
        cfg := config{driver: dialect.Debug(c.driver, c.log), log: c.log, debug: true, hooks: c.hooks}
        return &{{ $client }}{config: cfg}
    }
{{ end }}

{{ end }}
```

通过模板名对现有模板进行覆盖。 示例如下：
```gotemplate
{{/* 此模板用于给实体添加一些字段。 */}}
{{ define "model/fields/additional" }}
    {{- /* 给 "Card" 实体添加静态字段。 */}}
    {{- if eq $.Name "Card" }}
        // 通过模板定义静态字段。
        StaticField string `json:"static_field,omitempty"`
    {{- end }}
{{ end }}
```

## Helper Templates

As mentioned above, `ent` writes each template's execution output to a file named the same as the template. For example, the output from a template defined as `{{ define "stringer" }}` will be written to a file named `ent/stringer.go`.

By default, `ent` writes each template declared with `{{ define "<name>" }}` to a file. However, it is sometimes desirable to define helper templates - templates that will not be invoked directly but rather be executed by other templates. To facilitate this use case, `ent` supports two naming formats that designate a template as a helper. The formats are:

1\. `{{ define "helper/.+" }}` for global helper templates. For example:

```gotemplate
{{ define "helper/foo" }}
    {{/* Logic goes here. */}}
{{ end }}

{{ define "helper/bar/baz" }}
    {{/* Logic goes here. */}}
{{ end }}
```

2\. `{{ define "<root-template>/helper/.+" }}` for local helper templates. A template is considered as "root" if its execution output is written to a file. For example:

```gotemplate
{{/* A root template that is executed on the `gen.Graph` and will be written to a file named: `ent/http.go`.*/}}
{{ define "http" }}
    {{ range $n := $.Nodes }}
        {{ template "http/helper/get" $n }}
        {{ template "http/helper/post" $n }}
    {{ end }}
{{ end }}

{{/* A helper template that is executed on `gen.Type` */}}
{{ define "http/helper/get" }}
    {{/* Logic goes here. */}}
{{ end }}

{{/* A helper template that is executed on `gen.Type` */}}
{{ define "http/helper/post" }}
    {{/* Logic goes here. */}}
{{ end }}
```

## Annotations
Schema annotations allow attaching metadata to fields and edges and inject them to external templates.  
An annotation must be a Go type that is serializable to JSON raw value (e.g. struct, map or slice) and implement the [Annotation](https://pkg.go.dev/entgo.io/ent/schema?tab=doc#Annotation) interface.

Here's an example of an annotation and its usage in schema and template:

1\. 定义一个注解

```go
package entgql

// Annotation annotates fields with metadata for templates.
type Annotation struct {
    // OrderField is the ordering field as defined in graphql schema.
    OrderField string
}

// Name implements ent.Annotation interface.
func (Annotation) Name() string {
    return "EntGQL"
}
```

2\. Annotation usage in ent/schema:

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

3\. 注解在外部模板中的用法：

```gotemplate
{{ range $node := $.Nodes }}
    {{ range $f := $node.Fields }}
        {{/* Get the annotation by its name. See: Annotation.Name */}}
        {{ if $annotation := $f.Annotations.EntGQL }}
            {{/* Get the field from the annotation. */}}
            {{ $orderField := $annotation.OrderField }}
        {{ end }}
    {{ end }}
{{ end }}
```

## Global Annotations

Global annotation is a type of annotation that is injected into the `gen.Config` object and can be accessed globally in all templates. For example, an annotation that holds a config file information (e.g. `gqlgen.yml` or `swagger.yml`) add can accessed in all templates:

1\. An annotation definition:

```go
package gqlconfig

import (
    "entgo.io/ent/schema"
    "github.com/99designs/gqlgen/codegen/config"
)

// Annotation defines a custom annotation
// to be inject globally to all templates.
type Annotation struct {
    Config *config.Config
}

func (Annotation) Name() string {
    return "GQL"
}

var _ schema.Annotation = (*Annotation)(nil)
```

2\. Annotation usage in `ent/entc.go`:

```go
func main() {
    cfg, err := config.LoadConfig("<path to gqlgen.yml>")
    if err != nil {
        log.Fatalf("loading gqlgen config: %v", err)
    }
    opts := []entc.Option{
        entc.TemplateDir("./template"),
        entc.Annotations(gqlconfig.Annotation{Config: cfg}),
    }
    err = entc.Generate("./schema", &gen.Config{
        Templates: entgql.AllTemplates,
    }, opts...)
    if err != nil {
        log.Fatalf("running ent codegen: %v", err)
    }
}
```

3\. Annotation usage in external templates:

```gotemplate
{{- with $.Annotations.GQL.Config.StructTag }}
    {{/* Access the GQL configuration on *gen.Graph */}}
{{- end }}

{{ range $node := $.Nodes }}
    {{- with $node.Config.Annotations.GQL.Config.StructTag }}
        {{/* Access the GQL configuration on *gen.Type */}}
    {{- end }}
{{ end }}
```

## Examples
- 一个实现GraphQL `Node` API的自定义模板 - [Github](https://github.com/ent/ent/blob/master/entc/integration/template/ent/template/node.tmpl)。

- 一个用自定义函数执行外部模板的例子。 参阅[配置](https://github.com/ent/ent/blob/master/examples/entcpkg/ent/entc.go)和它的[README](https://github.com/ent/ent/blob/master/examples/entcpkg)文件。

## Documentation

Templates are executed on either a specific node type, or the entire schema graph. For API documentation, see the <a target="_blank" href="https://pkg.go.dev/entgo.io/ent/entc/gen?tab=doc">GoDoc</a>.

## AutoCompletion

JetBrains users can add the following template annotation to enable the autocompletion in their templates:

```gotemplate
{{/* The line below tells Intellij/GoLand to enable the autocompletion based on the *gen.Graph type. */}}
{{/* gotype: entgo.io/ent/entc/gen.Graph */}}

{{ define "template" }}
    {{/* ... */}}
{{ end }}
```

See it in action:

![template-autocomplete](https://entgo.io/images/assets/template-autocomplete.gif)
