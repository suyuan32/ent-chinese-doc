### 简介

Ent [扩展API（Extension API）](https://pkg.go.dev/entgo.io/ent/entc#Extension) 用于实现代码生成功能，配合 [代码生成钩子(codegen hooks)](code-gen.md#code-generation-hooks), [模板(templates)](templates.md) 和 [注解(annotations)](templates.md#annotations) 来创建可重用的组件，为 Ent 的核心添加丰富功能. 例如, Ent 的工具 [entgql plugin](https://pkg.go.dev/entgo.io/contrib/entgql#Extension) 暴露了 `Extension` 接口用于从 Ent Schema自动生成 GraphQL 服务器.

### 定义一个新的扩展

所有扩展都必须实现 [Extension](https://pkg.go.dev/entgo.io/ent/entc#Extension) 接口中的方法:

```go
type Extension interface {
    // Hooks holds an optional list of Hooks to apply
    // on the graph before/after the code-generation.
    Hooks() []gen.Hook

    // Annotations injects global annotations to the gen.Config object that
    // can be accessed globally in all templates. Unlike schema annotations,
    // being serializable to JSON raw value is not mandatory.
    //
    //  {{- with $.Config.Annotations.GQL }}
    //      {{/* Annotation usage goes here. */}}
    //  {{- end }}
    //
    Annotations() []Annotation

    // Templates specifies a list of alternative templates
    // to execute or to override the default.
    Templates() []*gen.Template

    // Options specifies a list of entc.Options to evaluate on
    // the gen.Config before executing the code generation.
    Options() []Option
}
```
为了简化新扩展的开发，开发人员可以嵌入 [entc.DefaultExtension](https://pkg.go.dev/entgo.io/ent/entc#DefaultExtension) 在不实现所有方法的情况下创建扩展:

```go
package hello

// GreetExtension implements entc.Extension.
type GreetExtension struct {
    entc.DefaultExtension
}
```

### 添加模板

Ent 支持添加模板 [external templates](templates.md) 用于在代码生成期间添加新方法. 要将此类外部模板捆绑在扩展上，请实现 `Templates` 方法:
```gotemplate title="templates/greet.tmpl"
{{/* Tell Intellij/GoLand to enable the autocompletion based on the *gen.Graph type. */}}
{{/* gotype: entgo.io/ent/entc/gen.Graph */}}

{{ define "greet" }}

{{/* Add the base header for the generated file */}}
{{ $pkg := base $.Config.Package }}
{{ template "header" $ }}

{{/* Loop over all nodes and add the Greet method */}}
{{ range $n := $.Nodes }}
    {{ $receiver := $n.Receiver }}
    func ({{ $receiver }} *{{ $n.Name }}) Greet() string {
        return "Hello, {{ $n.Name }}"
    }
{{ end }}

{{ end }}
```
```go
func (*GreetExtension) Templates() []*gen.Template {
    return []*gen.Template{
        gen.MustParse(gen.NewTemplate("greet").ParseFiles("templates/greet.tmpl")),
    }
}
```

### 添加全局注释

Annotations 是我们为用户提供扩展 API 来修改代码生成行为的便捷方式. 要向我们的扩展添加注释，请实现 `Annotations` 方法. 查看下面的 `GreetExtension`， 我们希望为用户提供生成的代码中配置问候语的功能:

```go
// GreetingWord implements entc.Annotation.
type GreetingWord string

// Name of the annotation. Used by the codegen templates.
func (GreetingWord) Name() string {
    return "GreetingWord"
}
```
然后添加 `GreetExtension` 结构:
```go
type GreetExtension struct {
    entc.DefaultExtension
    word GreetingWord
}
```
然后实现 `Annotations` 方法:
```go
func (s *GreetExtension) Annotations() []entc.Annotation {
    return []entc.Annotation{
        s.word,
    }
}
```
现在你可以在你的模板中使用 `GreetingWord` 注解:
```gotemplate
func ({{ $receiver }} *{{ $n.Name }}) Greet() string {
    return "{{ $.Annotations.GreetingWord }}, {{ $n.Name }}"
}
```

### 添加钩子

entc 包提供了一个选项在代码生成阶段添加一个 [钩子（hooks）](code-gen.md#code-generation-hooks) (middlewares) 列表. 此选项非常适合为Schema添加自定义验证器，或使用图形模式(Graph schema)生成额外的资源。 要将代码生成钩子与您的扩展捆绑在一起，请实现“Hooks”方法：

```go
func (s *GreetExtension) Hooks() []gen.Hook {
    return []gen.Hook{
        DisallowTypeName("Shalom"),
    }
}

// DisallowTypeName ensures there is no ent.Schema with the given name in the graph.
func DisallowTypeName(name string) gen.Hook {
    return func(next gen.Generator) gen.Generator {
        return gen.GenerateFunc(func(g *gen.Graph) error {
            for _, node := range g.Nodes {
                if node.Name == name {
                    return fmt.Errorf("entc: validation failed, type named %q not allowed", name)
                }
            }
            return next.Generate(g)
        })
    }
}
```

### 在代码生成中使用扩展

要在我们的代码生成配置中使用扩展，请使用 entc.Extensions，这是一个返回应用我们选择的扩展的 entc.Option 的辅助方法：

```go title="ent/entc.go"
//+build ignore

package main

import (
    "fmt"
    "log"

    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
)

func main() {
    err := entc.Generate("./schema",
        &gen.Config{},
        entc.Extensions(&GreetExtension{
            word: GreetingWord("Shalom"),
        }),
    )
    if err != nil {
        log.Fatal("running ent codegen:", err)
    }
}
```

### 受欢迎的扩展

- **[elk (discontinued)](https://github.com/masseelch/elk)**  
  `elk` 是从 Ent 模式生成 RESTful API 端点的扩展。 该扩展从 Ent 架构生成 HTTP CRUD 处理程序，以及一个 OpenAPI JSON 文件。 通过使用它，您可以轻松地为您的应用程序构建一个 RESTful HTTP 服务器。 请注意，`elk` 已停用，取而代之的是 `entoas`。 一个实现生成器正在开发中。
  查看 [帖子](https://entgo.io/blog/2021/07/29/generate-a-fully-working-go-crud-http-api-with-ent) 了解如何使用 `elk`, 和 [帖子](https://entgo.io/blog/2021/09/10/openapi-generator) 了解如何生成 [OpenAPI Specification](https://swagger.io/resources/open-api/).

- **[entoas](https://github.com/ent/contrib/tree/master/entoas)** `entoas` 是一个源于 `elk` 的扩展，并被移植到它自己的扩展中，现在是 OpenAPI 规范文档的官方生成器。 您可以使用它来快速开发 RESTful HTTP 服务器。 我们很快将发布一个新的扩展，提供一个生成的实现，集成了使用 ent 的 entoas 提供的文档

- **[entgql](https://github.com/ent/contrib/tree/master/entgql)**  
  这个扩展帮助用户从Ent schema生成 [GraphQL](https://graphql.org/) 服务. `entgql` 整合了 [gqlgen](https://github.com/99designs/gqlgen), 一个流行的、模式优先的 Go 库，用于构建 GraphQL 服务器。 该扩展包括生成类型安全的 GraphQL 过滤器，使用户能够毫不费力地将 GraphQL 查询映射到 Ent 查询。
  查看 [教程](https://entgo.io/docs/tutorial-todo-gql) 学习使用.

- **[entproto](https://github.com/ent/contrib/tree/master/entproto)**  
  `entproto` 从 Ent schema生成 Protobuf 消息定义和 gRPC 服务定义. 这个项目还包含 `protoc-gen-entgrpc`, 一个 `protoc` (Protobuf 编译器) 工具用于生成由 Entproto 生成的 gRPC 服务定义的工作实现。 通过这种方式，我们可以轻松创建一个 gRPC 服务器，无需编写任何代码（除了定义 Ent 模式）即可为我们的服务提供服务请求！
  要了解如何使用和设置 entproto，请阅读 [本教程](https://entgo.io/docs/grpc-intro). 有关更多背景信息，您可以阅读 [帖子](https://entgo.io/blog/2021/03/18/generating-a-grpc-server-with-ent), or [this blog post](https://entgo.io/blog/2021/06/28/gprc-ready-for-use/) 讨论更多 `entproto` 特性.

- **[entviz](https://github.com/hedwigz/entviz)**  
  `entviz` 是从 Ent 模式生成可视化图表的扩展。 这些图表在 Web 浏览器中可视化架构，并在我们继续编码时保持更新。 `entviz` 可以这样配置，每次我们重新生成模式时，图表都会自动更新，以便于查看所做的更改。
  了解如何将 `entviz` 集成到您的项目中  [帖子](https://entgo.io/blog/2021/08/26/visualizing-your-data-graph-using-entviz).
