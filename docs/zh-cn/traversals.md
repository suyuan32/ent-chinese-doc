通过例子的展示, 我们将生成下面的图结构:


![er-traversal-graph](https://entgo.io/images/assets/er_traversal_graph.png)

第一步是生成三个表: `Pet`, `User`, `Group`.

```console
go run -mod=mod entgo.io/ent/cmd/ent init Pet User Group
```

添加必要的字段和边:

`ent/schema/pet.go`

```go
// Pet holds the schema definition for the Pet entity.
type Pet struct {
    ent.Schema
}

// Fields of the Pet.
func (Pet) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
    }
}

// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("friends", Pet.Type),
        edge.From("owner", User.Type).
            Ref("pets").
            Unique(),
    }
}
```

`ent/schema/user.go`

```go
// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age"),
        field.String("name"),
    }
}

// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("pets", Pet.Type),
        edge.To("friends", User.Type),
        edge.From("groups", Group.Type).
            Ref("users"),
        edge.From("manage", Group.Type).
            Ref("admin"),
    }
}
```

`ent/schema/group.go`

```go
// Group holds the schema definition for the Group entity.
type Group struct {
    ent.Schema
}

// Fields of the Group.
func (Group) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
    }
}

// Edges of the Group.
func (Group) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("users", User.Type),
        edge.To("admin", User.Type).
            Unique(),
    }
}
```

让我们编写用于将顶点和边填充到图中的代码：

```go
func Gen(ctx context.Context, client *ent.Client) error {
    hub, err := client.Group.
        Create().
        SetName("Github").
        Save(ctx)
    if err != nil {
        return fmt.Errorf("failed creating the group: %w", err)
    }
    // Create the admin of the group.
    // Unlike `Save`, `SaveX` panics if an error occurs.
    dan := client.User.
        Create().
        SetAge(29).
        SetName("Dan").
        AddManage(hub).
        SaveX(ctx)

    // Create "Ariel" and its pets.
    a8m := client.User.
        Create().
        SetAge(30).
        SetName("Ariel").
        AddGroups(hub).
        AddFriends(dan).
        SaveX(ctx)
    pedro := client.Pet.
        Create().
        SetName("Pedro").
        SetOwner(a8m).
        SaveX(ctx)
    xabi := client.Pet.
        Create().
        SetName("Xabi").
        SetOwner(a8m).
        SaveX(ctx)

    // Create "Alex" and its pets.
    alex := client.User.
        Create().
        SetAge(37).
        SetName("Alex").
        SaveX(ctx)
    coco := client.Pet.
        Create().
        SetName("Coco").
        SetOwner(alex).
        AddFriends(pedro).
        SaveX(ctx)

    fmt.Println("Pets created:", pedro, xabi, coco)
    // Output:
    // Pets created: Pet(id=1, name=Pedro) Pet(id=2, name=Xabi) Pet(id=3, name=Coco)
    return nil
}
```

让我们回顾一些遍历，并展示它们的代码:

![er-traversal-graph-gopher](https://entgo.io/images/assets/er_traversal_graph_gopher.png)

上面的遍历从 `Group` 实体开始, 然后到 `admin` (edge), 接着到朋友 `friends` (edge), 获得他们的宠物 `pets` (edge), 然后获得宠物的朋友 pet's `friends` (edge), 然后查询他们的主人.

```go
func Traverse(ctx context.Context, client *ent.Client) error {
    owner, err := client.Group.         // GroupClient.
        Query().                        // Query builder.
        Where(group.Name("Github")).    // Filter only Github group (only 1).
        QueryAdmin().                   // Getting Dan.
        QueryFriends().                 // Getting Dan's friends: [Ariel].
        QueryPets().                    // Their pets: [Pedro, Xabi].
        QueryFriends().                 // Pedro's friends: [Coco], Xabi's friends: [].
        QueryOwner().                   // Coco's owner: Alex.
        Only(ctx)                       // Expect only one entity to return in the query.
    if err != nil {
        return fmt.Errorf("failed querying the owner: %w", err)
    }
    fmt.Println(owner)
    // Output:
    // User(id=3, age=37, name=Alex)
    return nil
}
```

下面的图表示什么？

![er-traversal-graph-gopher-query](https://entgo.io/images/assets/er_traversal_graph_gopher_query.png)

我们想获得有一个主人(owner)是某个Group的朋友(Friend)的所有宠物(pets).

```go
func Traverse(ctx context.Context, client *ent.Client) error {
    pets, err := client.Pet.
        Query().
        Where(
            pet.HasOwnerWith(
                user.HasFriendsWith(
                    user.HasManage(),
                ),
            ),
        ).
        All(ctx)
    if err != nil {
        return fmt.Errorf("failed querying the pets: %w", err)
    }
    fmt.Println(pets)
    // Output:
    // [Pet(id=1, name=Pedro) Pet(id=2, name=Xabi)]
    return nil
}
```

完整的例子请查看 [GitHub](https://github.com/ent/ent/tree/master/examples/traversal).
