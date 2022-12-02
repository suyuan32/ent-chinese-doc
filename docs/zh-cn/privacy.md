schema中的`Policy`选项允许为数据库中的查询和可变实体进行隐私策略的配置。

![gopher-privacy](https://entgo.io/images/assets/gopher-privacy-opacity.png)

隐私层的主要优点是，您只需在schema中编写编写隐私政策**一次**，它就会**总是**被评估是否为隐私数据。无论你的代码在何处执行查询和更改，它都要经过隐私层。

在本教程中，我们将首先介绍我们在框架中使用的基本术语，然后是为您的项目配置策略功能的部分，最后是一些示例。

## 基础术语

### 策略（Policy）

`ent.Policy`接口包含两个方法：`EvalQuery`和`EvalMutation`。 第一个方法用于定义读策略，第二个方法用于定义写策略。 每个策略可以包含0条或多条隐私规则(见后文)。这些规则的评估顺序与它们在模式中声明的顺序相同。

如果对所有规则都进行了评估而没有返回错误，则评估成功完成，并且执行的操作可以访问目标节点。

![privacy-rules](https://entgo.io/images/assets/permission_1.png)

但是，如果某个评估的规则返回错误或决定执行“privacy.Deny”（见下文），则执行的操作返回错误，并被取消。

![privacy-deny](https://entgo.io/images/assets/permission_2.png)

### 隐私规则（Privacy Rules）

每个策略（变更或查询）都包含一个或多个隐私规则。 这些规则的函数签名如下：

```go
// EvalQuery defines the a read-policy rule.
func(Policy) EvalQuery(context.Context, Query) error

// EvalMutation defines the a write-policy rule.
func(Policy) EvalMutation(context.Context, Mutation) error
```

### 隐私决策操作（Privacy Decisions）

可以帮助您控制隐私规则评估的三种决策类型。

- `privacy.Allow` - 如果从隐私规则返回，则评估停止（将跳过下一条规则），并且执行的操作（查询或变异）可以访问目标节点。

- `privacy.Deny` - 如果从隐私规则返回，则评估停止（将跳过下一条规则），并取消执行的操作。 这相当于返回任何错误。

- `privacy.Skip` - 跳过当前规则，跳转到下一条隐私规则。 这相当于返回一个 `nil` 错误。

![privacy-allow](https://entgo.io/images/assets/permission_3.png)

现在我们已经介绍了基本术语，让我们开始编写一些代码。

## 配置（Configuration）

为了在代码生成中启用隐私选项，请使用以下两个选项之一启用“隐私”功能：

1\。 如果您使用默认的 go generate 配置，请将 `--feature privacy` 选项添加到 `ent/generate.go` 文件，如下所示：

```go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate --feature privacy ./schema
```

建议添加 [`schema/snapshot`](./zh-cn/features.md#自动解决合并冲突) feature-flag 以及 `privacy` 以增强开发体验（例如 `--feature privacy,schema/ 快照`)

2\. 如果您使用的是 GraphQL 文档中的配置，请按如下方式添加功能标志：

```go {20}
// Copyright 2019-present Facebook Inc. All rights reserved.
// This source code is licensed under the Apache 2.0 license found
// in the LICENSE file in the root directory of this source tree.

// +build ignore

package main


import (
    "log"

    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
    "entgo.io/contrib/entgql"
)

func main() {
    opts := []entc.Option{
        entc.FeatureNames("privacy"),
    }
    err := entc.Generate("./schema", &gen.Config{
        Templates: entgql.AllTemplates,
    }, opts...)
    if err != nil {
        log.Fatalf("running ent codegen: %v", err)
    }
}
```

> 重要： 你应该注意到，类似于 [schema hooks](hooks.md#钩子注册)，如果你在你的模式中使用 **`Policy`** 选项，你**必须**在 main 中添加以下导入 包，因为在模式包和生成的 ent 包之间可以循环导入：
```go
import _ "<project>/ent/runtime"
```


## 例子

### 仅限管理员（Admin Only）

我们从一个简单的应用程序示例开始，该应用程序允许任何用户读取任何数据，并且只接受来自具有管理员角色的用户的更改。 为了示例的目的，我们将创建 2 个额外的包：

- `rule` - 用于在我们的模式中保存不同的隐私规则。
- `viewer` - 用于获取和设置执行操作的用户/查看者。 在这个简单的示例中，它可以是普通用户或管理员。

运行代码生成（带有隐私功能标志）后，我们添加带有 2 个生成的策略规则的 `Policy` 方法。

```go title="examples/privacyadmin/ent/schema/user.go"
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/examples/privacyadmin/ent/privacy"
)

// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

// Policy defines the privacy policy of the User.
func (User) Policy() ent.Policy {
    return privacy.Policy{
        Mutation: privacy.MutationPolicy{
            // Deny if not set otherwise. 
            privacy.AlwaysDenyRule(),
        },
        Query: privacy.QueryPolicy{
            // Allow any viewer to read anything.
            privacy.AlwaysAllowRule(),
        },
    }
}
```

我们定义了一个拒绝任何变更（mutation）并接受任何查询的策略。 然而，如上所述，在这个例子中，我们只接受来自具有管理员角色的查看者的变更（mutation）。 让我们创建 2 条隐私规则来执行此操作：

```go title="examples/privacyadmin/rule/rule.go"
package rule

import (
    "context"

    "entgo.io/ent/examples/privacyadmin/ent/privacy"
    "entgo.io/ent/examples/privacyadmin/viewer"
)

// DenyIfNoViewer is a rule that returns Deny decision if the viewer is
// missing in the context.
func DenyIfNoViewer() privacy.QueryMutationRule {
    return privacy.ContextQueryMutationRule(func(ctx context.Context) error {
        view := viewer.FromContext(ctx)
        if view == nil {
            return privacy.Denyf("viewer-context is missing")
        }
        // Skip to the next privacy rule (equivalent to returning nil).
        return privacy.Skip
    })
}

// AllowIfAdmin is a rule that returns Allow decision if the viewer is admin.
func AllowIfAdmin() privacy.QueryMutationRule {
    return privacy.ContextQueryMutationRule(func(ctx context.Context) error {
        view := viewer.FromContext(ctx)
        if view.Admin() {
            return privacy.Allow
        }
        // Skip to the next privacy rule (equivalent to returning nil).
        return privacy.Skip
    })
}
```

如您所见，第一条规则“DenyIfNoViewer”确保每个操作在其上下文中都有一个查看器，否则操作将被拒绝。 第二条规则“AllowIfAdmin”，接受来自具有管理员角色的查看者的任何操作。 让我们将它们添加到Schema中，并运行代码生成：

```go title="examples/privacyadmin/ent/schema/user.go"
// Policy defines the privacy policy of the User.
func (User) Policy() ent.Policy {
    return privacy.Policy{
        Mutation: privacy.MutationPolicy{
            rule.DenyIfNoViewer(),
            rule.AllowIfAdmin(),
            privacy.AlwaysDenyRule(),
        },
        Query: privacy.QueryPolicy{
            privacy.AlwaysAllowRule(),
        },
    }
}
```

由于我们首先定义了 `DenyIfNoViewer`，它将在所有其他规则之前执行，并且访问 `viewer.Viewer` 对象在 `AllowIfAdmin` 规则中是安全的。

添加上述规则并运行代码生成后，我们希望将隐私层逻辑应用于“ent.Client”操作。

```go title="examples/privacyadmin/example_test.go"
func Do(ctx context.Context, client *ent.Client) error {
    // Expect operation to fail, because viewer-context
    // is missing (first mutation rule check).
    if err := client.User.Create().Exec(ctx); !errors.Is(err, privacy.Deny) {
        return fmt.Errorf("expect operation to fail, but got %w", err)
    }
    // Apply the same operation with "Admin" role.
    admin := viewer.NewContext(ctx, viewer.UserViewer{Role: viewer.Admin})
    if err := client.User.Create().Exec(admin); err != nil {
        return fmt.Errorf("expect operation to pass, but got %w", err)
    }
    // Apply the same operation with "ViewOnly" role.
    viewOnly := viewer.NewContext(ctx, viewer.UserViewer{Role: viewer.View})
    if err := client.User.Create().Exec(viewOnly); !errors.Is(err, privacy.Deny) {
        return fmt.Errorf("expect operation to fail, but got %w", err)
    }
    // Allow all viewers to query users.
    for _, ctx := range []context.Context{ctx, viewOnly, admin} {
        // Operation should pass for all viewers.
        count := client.User.Query().CountX(ctx)
        fmt.Println(count)
    }
    return nil
}
```

### 决策上下文（Decision Context）

有时，我们想将特定的隐私决定绑定到 `context.Context`。 在这种情况下，我们可以使用 `privacy.DecisionContext` 函数创建一个新的上下文，并附加一个隐私决策。

```go title="examples/privacyadmin/example_test.go"
func Do(ctx context.Context, client *ent.Client) error {
    // Bind a privacy decision to the context (bypass all other rules).
    allow := privacy.DecisionContext(ctx, privacy.Allow)
    if err := client.User.Create().Exec(allow); err != nil {
        return fmt.Errorf("expect operation to pass, but got %w", err)
    }
    return nil
}
```

完整的例子存在于 [GitHub](https://github.com/ent/ent/tree/master/examples/privacyadmin).

### 多租户（Multi Tenancy）

在此示例中，我们将创建一个包含 3 种实体类型的模式 - `Tenant`、`User` 和 `Group`。 帮助程序包 `viewer` 和 `rule`（如上所述）也存在于此示例中，以帮助我们构建应用程序。

![tenant-example](https://entgo.io/images/assets/tenant_medium.png)

让我们开始一点一点地构建这个应用程序。 我们首先创建 3 个不同的模式 (完整代码 [here](https://github.com/ent/ent/tree/master/examples/privacytenant/ent/schema)), 因为我们想在它们之间共享一些逻辑，所以我们创建了另一个 [mixed-in schema](schema-mixin.md) 并将其添加到所有其他模式(schema)，如下所示：

```go title="examples/privacytenant/ent/schema/mixin.go"
// BaseMixin for all schemas in the graph.
type BaseMixin struct {
    mixin.Schema
}

// Policy defines the privacy policy of the BaseMixin.
func (BaseMixin) Policy() ent.Policy {
    return privacy.Policy{
        Query: privacy.QueryPolicy{
            // Deny any query operation in case
            // there is no "viewer context".
            rule.DenyIfNoViewer(),
            // Allow admins to query any information.
            rule.AllowIfAdmin(),
        },
        Mutation: privacy.MutationPolicy{
            // Deny any mutation operation in case
            // there is no "viewer context".
            rule.DenyIfNoViewer(),
        },
    }
}
```

```go title="examples/privacytenant/ent/schema/tenant.go"
// Mixin of the Tenant schema.
func (Tenant) Mixin() []ent.Mixin {
    return []ent.Mixin{
        BaseMixin{},
    }
}
```

如第一个示例中所述，如果 context.Context 不包含 viewer.Viewer 信息，则 `DenyIfNoViewer` 隐私规则会拒绝操作。

与前面的示例类似，我们要添加一个约束，即只有管理员用户才能创建租户（否则拒绝）。 我们通过复制上面的 `AllowIfAdmin` 规则，并将其添加到 `Tenant` 模式的 `Policy` 来实现：

```go title="examples/privacytenant/ent/schema/tenant.go"
// Policy defines the privacy policy of the User.
func (Tenant) Policy() ent.Policy {
    return privacy.Policy{
        Mutation: privacy.MutationPolicy{
            // For Tenant type, we only allow admin users to mutate
            // the tenant information and deny otherwise.
            rule.AllowIfAdmin(),
            privacy.AlwaysDenyRule(),
        },
    }
}
```

然后，我们期望下面的代码能够成功运行：

```go title="examples/privacytenant/example_test.go"

func Example_CreateTenants(ctx context.Context, client *ent.Client) {
    // Expect operation to fail in case viewer-context is missing.
    // First mutation privacy policy rule defined in BaseMixin.
    if err := client.Tenant.Create().Exec(ctx); !errors.Is(err, privacy.Deny) {
        log.Fatal("expect tenant creation to fail, but got:", err)
    }

    // Expect operation to fail in case the ent.User in the viewer-context
    // is not an admin user. Privacy policy defined in the Tenant schema.
    viewCtx := viewer.NewContext(ctx, viewer.UserViewer{Role: viewer.View})
    if err := client.Tenant.Create().Exec(viewCtx); !errors.Is(err, privacy.Deny) {
        log.Fatal("expect tenant creation to fail, but got:", err)
    }

    // Operations should pass successfully as the user in the viewer-context
    // is an admin user. First mutation privacy policy in Tenant schema.
    adminCtx := viewer.NewContext(ctx, viewer.UserViewer{Role: viewer.Admin})
    hub, err := client.Tenant.Create().SetName("GitHub").Save(adminCtx)
    if err != nil {
        log.Fatal("expect tenant creation to pass, but got:", err)
    }
    fmt.Println(hub)

    lab, err := client.Tenant.Create().SetName("GitLab").Save(adminCtx)
    if err != nil {
        log.Fatal("expect tenant creation to pass, but got:", err)
    }
    fmt.Println(lab)

    // Output:
    // Tenant(id=1, name=GitHub)
    // Tenant(id=2, name=GitLab)
}
```

我们继续在我们的数据模型中添加其余的边缘(edge)（见上图），并且由于“用户”和“组”都具有“租户”模式的边缘(edge)，我们创建了一个共享的[混合模式] (schema-mixin.md) 为此命名为“TenantMixin”：

```go title="examples/privacytenant/ent/schema/mixin.go"
// TenantMixin for embedding the tenant info in different schemas.
type TenantMixin struct {
    mixin.Schema
}

// Fields for all schemas that embed TenantMixin.
func (TenantMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Int("tenant_id").
            Immutable(),
    }
}

// Edges for all schemas that embed TenantMixin.
func (TenantMixin) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("tenant", Tenant.Type).
            Field("tenant_id").
            Unique().
            Required().
            Immutable(),
    }
}
```

#### 过滤规则 (Filter Rules)

接下来，我们可能想要执行一个规则，将查看者限制为仅查询与其所属租户相关联的组和用户。 对于这样的用例，Ent 有一种名为“Filter”的额外隐私规则。 我们可以使用“过滤器”规则根据查看者的身份过滤掉实体。 与我们之前讨论的规则不同，“过滤器”规则除了返回隐私决策外，还可以限制查看者可以进行的查询范围。

> 注意需要使用 [`entql`](./zh-cn/features.md#entql过滤) 功能标志启用隐私过滤选项（参见说明 [above](#configuration)）。

```go title="examples/privacytenant/rule/rule.go"
// FilterTenantRule is a query/mutation rule that filters out entities that are not in the tenant.
func FilterTenantRule() privacy.QueryMutationRule {
    // TenantsFilter is an interface to wrap WhereHasTenantWith()
    // predicate that is used by both `Group` and `User` schemas.
    type TenantsFilter interface {
        WhereTenantID(entql.IntP)
    }
    return privacy.FilterFunc(func(ctx context.Context, f privacy.Filter) error {
        view := viewer.FromContext(ctx)
        tid, ok := view.Tenant()
        if !ok {
            return privacy.Denyf("missing tenant information in viewer")
        }
        tf, ok := f.(TenantsFilter)
        if !ok {
            return privacy.Denyf("unexpected filter type %T", f)
        }
        // Make sure that a tenant reads only entities that have an edge to it.
        tf.WhereTenantID(entql.IntEQ(tid))
        // Skip to the next privacy rule (equivalent to return nil).
        return privacy.Skip
    })
}
```

创建“FilterTenantRule”隐私规则后，我们将其添加到“TenantMixin”中，以确保**所有使用此 mixin 的模式**也将具有此隐私规则。

```go title="examples/privacytenant/ent/schema/mixin.go"
// Policy for all schemas that embed TenantMixin.
func (TenantMixin) Policy() ent.Policy {
    return rule.FilterTenantRule()
}
```

然后，在运行代码生成之后，我们希望隐私规则对客户端操作生效。

```go title="examples/privacytenant/example_test.go"

func Example_TenantView(ctx context.Context, client *ent.Client) {
    // Operations should pass successfully as the user in the viewer-context
    // is an admin user. First mutation privacy policy in Tenant schema.
    adminCtx := viewer.NewContext(ctx, viewer.UserViewer{Role: viewer.Admin})
    hub := client.Tenant.Create().SetName("GitHub").SaveX(adminCtx)
    lab := client.Tenant.Create().SetName("GitLab").SaveX(adminCtx)

    // Create 2 tenant-specific viewer contexts.
    hubView := viewer.NewContext(ctx, viewer.UserViewer{T: hub})
    labView := viewer.NewContext(ctx, viewer.UserViewer{T: lab})

    // Create 2 users in each tenant.
    hubUsers := client.User.CreateBulk(
        client.User.Create().SetName("a8m").SetTenant(hub),
        client.User.Create().SetName("nati").SetTenant(hub),
    ).SaveX(hubView)
    fmt.Println(hubUsers)

    labUsers := client.User.CreateBulk(
        client.User.Create().SetName("foo").SetTenant(lab),
        client.User.Create().SetName("bar").SetTenant(lab),
    ).SaveX(labView)
    fmt.Println(labUsers)

    // Query users should fail in case viewer-context is missing.
    if _, err := client.User.Query().Count(ctx); !errors.Is(err, privacy.Deny) {
        log.Fatal("expect user query to fail, but got:", err)
    }

    // Ensure each tenant can see only its users.
    // First and only rule in TenantMixin.
    fmt.Println(client.User.Query().Select(user.FieldName).StringsX(hubView))
    fmt.Println(client.User.Query().CountX(hubView))
    fmt.Println(client.User.Query().Select(user.FieldName).StringsX(labView))
    fmt.Println(client.User.Query().CountX(labView))

    // Expect admin users to see everything. First
    // query privacy policy defined in BaseMixin.
    fmt.Println(client.User.Query().CountX(adminCtx)) // 4

    // Update operation with specific tenant-view should update
    // only the tenant in the viewer-context.
    client.User.Update().SetFoods([]string{"pizza"}).SaveX(hubView)
    fmt.Println(client.User.Query().AllX(hubView))
    fmt.Println(client.User.Query().AllX(labView))

    // Delete operation with specific tenant-view should delete
    // only the tenant in the viewer-context.
    client.User.Delete().ExecX(labView)
    fmt.Println(
        client.User.Query().CountX(hubView), // 2
        client.User.Query().CountX(labView), // 0
    )

    // DeleteOne with wrong viewer-context is nop.
    client.User.DeleteOne(hubUsers[0]).ExecX(labView)
    fmt.Println(client.User.Query().CountX(hubView)) // 2

    // Unlike queries, admin users are not allowed to mutate tenant specific data.
    if err := client.User.DeleteOne(hubUsers[0]).Exec(adminCtx); !errors.Is(err, privacy.Deny) {
        log.Fatal("expect user deletion to fail, but got:", err)
    }

    // Output:
    // [User(id=1, tenant_id=1, name=a8m, foods=[]) User(id=2, tenant_id=1, name=nati, foods=[])]
    // [User(id=3, tenant_id=2, name=foo, foods=[]) User(id=4, tenant_id=2, name=bar, foods=[])]
    // [a8m nati]
    // 2
    // [foo bar]
    // 2
    // 4
    // [User(id=1, tenant_id=1, name=a8m, foods=[pizza]) User(id=2, tenant_id=1, name=nati, foods=[pizza])]
    // [User(id=3, tenant_id=2, name=foo, foods=[]) User(id=4, tenant_id=2, name=bar, foods=[])]
    // 2 0
    // 2
}
```

我们在 `Group` 模式(schema)上使用另一个名为 `DenyMismatchedTenants` 的隐私规则来结束我们的示例。 如果关联用户不属于与组相同的租户，则“DenyMismatchedTenants”规则会拒绝创建组。

```go title="examples/privacytenant/rule/rule.go"
// DenyMismatchedTenants is a rule that runs only on create operations and returns a deny
// decision if the operation tries to add users to groups that are not in the same tenant.
func DenyMismatchedTenants() privacy.MutationRule {
    return privacy.GroupMutationRuleFunc(func(ctx context.Context, m *ent.GroupMutation) error {
        tid, exists := m.TenantID()
        if !exists {
            return privacy.Denyf("missing tenant information in mutation")
        }
        users := m.UsersIDs()
        // If there are no users in the mutation, skip this rule-check.
        if len(users) == 0 {
            return privacy.Skip
        }
        // Query the tenant-ids of all attached users. Expect all users to be connected to the same tenant
        // as the group. Note, we use privacy.DecisionContext to skip the FilterTenantRule defined above.
        ids, err := m.Client().User.Query().Where(user.IDIn(users...)).Select(user.FieldTenantID).Ints(privacy.DecisionContext(ctx, privacy.Allow))
        if err != nil {
            return privacy.Denyf("querying the tenant-ids %v", err)
        }
        if len(ids) != len(users) {
            return privacy.Denyf("one the attached users is not connected to a tenant %v", err)
        }
        for _, id := range ids {
            if id != tid {
                return privacy.Denyf("mismatch tenant-ids for group/users %d != %d", tid, id)
            }
        }
        // Skip to the next privacy rule (equivalent to return nil).
        return privacy.Skip
    })
}
```

我们将此规则添加到“Group”模式(schema)并运行代码生成。

```go title="examples/privacytenant/ent/schema/group.go"
// Policy defines the privacy policy of the Group.
func (Group) Policy() ent.Policy {
    return privacy.Policy{
        Mutation: privacy.MutationPolicy{
            // Limit DenyMismatchedTenants only for
            // Create operation
            privacy.OnMutationOperation(
                rule.DenyMismatchedTenants(),
                ent.OpCreate,
            ),
        },
    }
}
```

同样，我们希望隐私规则对客户端操作生效。

```go title="examples/privacytenant/example_test.go"
func Example_DenyMismatchedTenants(ctx context.Context, client *ent.Client) {
    // Operation should pass successfully as the user in the viewer-context
    // is an admin user. First mutation privacy policy in Tenant schema.
    adminCtx := viewer.NewContext(ctx, viewer.UserViewer{Role: viewer.Admin})
    hub := client.Tenant.Create().SetName("GitHub").SaveX(adminCtx)
    lab := client.Tenant.Create().SetName("GitLab").SaveX(adminCtx)

    // Create 2 tenant-specific viewer contexts.
    hubView := viewer.NewContext(ctx, viewer.UserViewer{T: hub})
    labView := viewer.NewContext(ctx, viewer.UserViewer{T: lab})

    // Create 2 users in each tenant.
    hubUsers := client.User.CreateBulk(
        client.User.Create().SetName("a8m").SetTenant(hub),
        client.User.Create().SetName("nati").SetTenant(hub),
    ).SaveX(hubView)
    fmt.Println(hubUsers)

    labUsers := client.User.CreateBulk(
        client.User.Create().SetName("foo").SetTenant(lab),
        client.User.Create().SetName("bar").SetTenant(lab),
    ).SaveX(labView)
    fmt.Println(labUsers)

    // Expect operation to fail as the DenyMismatchedTenants rule makes
    // sure the group and the users are connected to the same tenant.
    if err := client.Group.Create().SetName("entgo.io").SetTenant(hub).AddUsers(labUsers...).Exec(hubView); !errors.Is(err, privacy.Deny) {
        log.Fatal("expect operation to fail, since labUsers are not connected to the same tenant")
    }
    if err := client.Group.Create().SetName("entgo.io").SetTenant(hub).AddUsers(hubUsers[0], labUsers[0]).Exec(hubView); !errors.Is(err, privacy.Deny) {
        log.Fatal("expect operation to fail, since labUsers[0] is not connected to the same tenant")
    }
    // Expect mutation to pass as all users belong to the same tenant as the group.
    entgo := client.Group.Create().SetName("entgo.io").SetTenant(hub).AddUsers(hubUsers...).SaveX(hubView)
    fmt.Println(entgo)

    // Output:
    // [User(id=1, tenant_id=1, name=a8m, foods=[]) User(id=2, tenant_id=1, name=nati, foods=[])]
    // [User(id=3, tenant_id=2, name=foo, foods=[]) User(id=4, tenant_id=2, name=bar, foods=[])]
    // Group(id=1, tenant_id=1, name=entgo.io)
}
```

在某些情况下，我们希望拒绝用户对不属于其租户的实体的操作**而不从数据库加载这些实体**（与上面的“DenyMismatchedTenants”示例不同）。 为实现这一目标，我们还依赖 `FilterTenantRule` 规则来添加其对变更（mutation）的过滤，并期望操作失败并返回 `NotFoundError`，以防 `tenant_id` 列与存储在查看器上下文中的列不匹配。

```go title="examples/privacytenant/example_test.go"
func Example_DenyMismatchedView(ctx context.Context, client *ent.Client) {
    // Continuation of the code above.

    // Expect operation to fail, because the FilterTenantRule rule makes sure
    // that tenants can update and delete only their groups.
    if err := entgo.Update().SetName("fail.go").Exec(labView); !ent.IsNotFound(err) {
        log.Fatal("expect operation to fail, since the group (entgo) is managed by a different tenant (hub), but got:", err)
    }

    // Operation should pass in case it was applied with the right viewer-context.
    entgo = entgo.Update().SetName("entgo").SaveX(hubView)
    fmt.Println(entgo)

    // Output:
    // Group(id=1, tenant_id=1, name=entgo)
}
```

完整示例请详见[Github](https://github.com/ent/ent/tree/master/examples/privacytenant)。

请注意，此文档仍处于更新中。
