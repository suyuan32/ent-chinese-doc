## 概述

边是实体之间的关系（或者关联）。 例如，用户所拥有的（多只）宠物，或者用户组所关联的（多个）用户。


![er-group-users](https://entgo.io/images/assets/er_user_pets_groups.png)

在上面的例子中，你可以看到使用边声明的2种关系。 让我们接着往下看。

1\. `pets` / `owner` 边；用户的宠物和宠物的主人 -

`ent/schema/user.go`
```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/edge"
)

// User schema.
type User struct {
    ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        // ...
    }
}

// Edges of the user.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("pets", Pet.Type),
    }
}
```


`ent/schema/pet.go`
```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/edge"
)

// Pet holds the schema definition for the Pet entity.
type Pet struct {
    ent.Schema
}

// Fields of the Pet.
func (Pet) Fields() []ent.Field {
    return []ent.Field{
        // ...
    }
}

// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("owner", User.Type).
            Ref("pets").
            Unique(),
    }
}
```

你可以看到，一个 `User` 实体可以拥有 **多个** 宠物，但是一个 `Pet` 实体只能拥有 **一名** 主人。  
在关系定义中，`pets` 边是一个 *O2M* （一对多）的关系，`owner` 边是一个 *M2O* （多对一）的关系。

`User` 实体**拥有** `pets/owner` 的关系因为使用了 `edge.To`， `Pet` 实体则仅有一个反向引用，是靠 `edge.From` 以及 `Ref` 方法实现。

`Ref` 方法明确了在 `User` 实体中我们所引用的那条边，因为存在从一个实体到其他实体多条引用的场景。

边/关系的数量可以使用 `Unique` 方法加以约束， 后续将作更多介绍。

2\. `users` / `groups` 边； 用户组的用户和用户所属的用户组 -

`ent/schema/group.go`
```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/edge"
)

// Group schema.
type Group struct {
    ent.Schema
}

// Fields of the group.
func (Group) Fields() []ent.Field {
    return []ent.Field{
        // ...
    }
}

// Edges of the group.
func (Group) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("users", User.Type),
    }
}
```

`ent/schema/user.go`
```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/edge"
)

// User schema.
type User struct {
    ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        // ...
    }
}

// Edges of the user.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("groups", Group.Type).
            Ref("users"),
        // "pets" declared in the example above.
        edge.To("pets", Pet.Type),
    }
}
```

如你所见，一个用户组能**拥有多名**用户，同时一名用户能**属于多个**用户组。  
在定义关系时，`users`的边是一个*M2M* （多对多）关系，同时`groups` 边同样也是一个*M2M*（多对多）关系。

## To 和 From

`edge.To` 和 `edge.From` 是两个用于创建边/关系的生成器。

一个实体使用 `edge.To` 生成器定义一条边以构建关系模式，和使用 `edge.From` 生成器仅定义一个反向引用关系完全是两回事（命名不同）。

让我们通过一些例子来看看如何使用边定义不同的关系模式。

## 关系

- [一对一（ O2O）两者之间](#o2o-two-types)
- [一对一（O2O）自引用](#o2o-same-type)
- [一对一（O2O）双向自引用](#o2o-bidirectional)
- [一对多（O2M ）两者之间](#o2m-two-types)
- [一对多（O2M ）自引用](#o2m-same-type)
- [多对多（M2M ）两者之间](#m2m-two-types)
- [多对多（M2M ）自引用](#m2m-same-type)
- [多对多（M2M）双向自引用](#m2m-bidirectional)

## 一对一（ O2O）两者之间

![er-user-card](https://entgo.io/images/assets/er_user_card.png)

在本例中，一名用户 **仅有一张** 信用卡，同时一张卡 **只属一名** 用户。

`User` 实体基于 `edge.To` 定义了名下的卡并将该引用关系命名 `card`，同时 `Card` 实体基于 `edge.From` 定义了一个相对的反向引用并命名 `owner`。


`ent/schema/user.go`
```go
// Edges of the user.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("card", Card.Type).
            Unique(),
    }
}
```

`ent/schema/card.go`
```go
// Edges of the Card.
func (Card) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("owner", User.Type).
            Ref("card").
            Unique().
            // We add the "Required" method to the builder
            // to make this edge required on entity creation.
            // i.e. Card cannot be created without its owner.
            Required(),
    }
}
```

与这些边的交互API如下：
```go
func Do(ctx context.Context, client *ent.Client) error {
    a8m, err := client.User.
        Create().
        SetAge(30).
        SetName("Mashraki").
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating user: %w", err)
    }
    log.Println("user:", a8m)
    card1, err := client.Card.
        Create().
        SetOwner(a8m).
        SetNumber("1020").
        SetExpired(time.Now().Add(time.Minute)).
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating card: %w", err)
    }
    log.Println("card:", card1)
    // Only returns the card of the user,
    // and expects that there's only one.
    card2, err := a8m.QueryCard().Only(ctx)
    if err != nil {
        return fmt.Errorf("querying card: %w", err)
    }
    log.Println("card:", card2)
    // The Card entity is able to query its owner using
    // its back-reference.
    owner, err := card2.QueryOwner().Only(ctx)
    if err != nil {
        return fmt.Errorf("querying owner: %w", err)
    }
    log.Println("owner:", owner)
    return nil
}
```

完整示例请参考 [GitHub](https://github.com/ent/ent/tree/master/examples/o2o2types)。

## 一对一（O2O）自引用

![er-linked-list](https://entgo.io/images/assets/er_linked_list.png)

在这个链表的例子中（Linked-List），我们有一个 **递归关系** 名为 `next`/`prev`。 表中的每个结点 **仅能拥有一个** `next` 结点。 如果结点A链到（使用 `next`）结点B，B即可使用 `prev` （反向关系引用）。

`ent/schema/node.go`
```go
// Edges of the Node.
func (Node) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("next", Node.Type).
            Unique().
            From("prev").
            Unique(),
    }
}
```

如你所见，在自引用的关系中，你可以把边和它的引用在同一个构造器中声明。

```diff
func (Node) Edges() []ent.Edge {
    return []ent.Edge{
+       edge.To("next", Node.Type).
+           Unique().
+           From("prev").
+           Unique(),

-       edge.To("next", Node.Type).
-           Unique(),
-       edge.From("prev", Node.Type).
-           Ref("next).
-           Unique(),
    }
}
```

与这些边交互的 API 如下：

```go
func Do(ctx context.Context, client *ent.Client) error {
    head, err := client.Node.
        Create().
        SetValue(1).
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating the head: %w", err)
    }
    curr := head
    // Generate the following linked-list: 1<->2<->3<->4<->5.
    for i := 0; i < 4; i++ {
        curr, err = client.Node.
            Create().
            SetValue(curr.Value + 1).
            SetPrev(curr).
            Save(ctx)
        if err != nil {
            return err
        }
    }

    // Loop over the list and print it. `FirstX` panics if an error occur.
    for curr = head; curr != nil; curr = curr.QueryNext().FirstX(ctx) {
        fmt.Printf("%d ", curr.Value)
    }
    // Output: 1 2 3 4 5

    // Make the linked-list circular:
    // The tail of the list, has no "next".
    tail, err := client.Node.
        Query().
        Where(node.Not(node.HasNext())).
        Only(ctx)
    if err != nil {
        return fmt.Errorf("getting the tail of the list: %v", tail)
    }
    tail, err = tail.Update().SetNext(head).Save(ctx)
    if err != nil {
        return err
    }
    // Check that the change actually applied:
    prev, err := head.QueryPrev().Only(ctx)
    if err != nil {
        return fmt.Errorf("getting head's prev: %w", err)
    }
    fmt.Printf("\n%v", prev.Value == tail.Value)
    // Output: true
    return nil
}
```

完整示例请参考 [GitHub](https://github.com/ent/ent/tree/master/examples/o2orecur)。

## 一对一（O2O）双向自引用

![er-user-spouse](https://entgo.io/images/assets/er_user_spouse.png)

在这个用户配偶的例子中，我们定义了一个 **对称的一对一（O2O）关系** 名为 `spouse`。 每名用户**仅能有一名**配偶。 如果A将其配偶设置成（使用 `spouse`）B，B则可使用`spouse` 边查到其相应的配偶。

注意在双向自引用场景下没有所有者/反向引用关系。

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("spouse", User.Type).
            Unique(),
    }
}
```

与这些边的交互API如下：

```go
func Do(ctx context.Context, client *ent.Client) error {
    a8m, err := client.User.
        Create().
        SetAge(30).
        SetName("a8m").
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating user: %w", err)
    }
    nati, err := client.User.
        Create().
        SetAge(28).
        SetName("nati").
        SetSpouse(a8m).
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating user: %w", err)
    }

    // Query the spouse edge.
    // Unlike `Only`, `OnlyX` panics if an error occurs.
    spouse := nati.QuerySpouse().OnlyX(ctx)
    fmt.Println(spouse.Name)
    // Output: a8m

    spouse = a8m.QuerySpouse().OnlyX(ctx)
    fmt.Println(spouse.Name)
    // Output: nati

    // Query how many users have a spouse.
    // Unlike `Count`, `CountX` panics if an error occurs.
    count := client.User.
        Query().
        Where(user.HasSpouse()).
        CountX(ctx)
    fmt.Println(count)
    // Output: 2

    // Get the user, that has a spouse with name="a8m".
    spouse = client.User.
        Query().
        Where(user.HasSpouseWith(user.Name("a8m"))).
        OnlyX(ctx)
    fmt.Println(spouse.Name)
    // Output: nati
    return nil
}
```

注意，外键列可以配置并暴露为实体字段，如下使用[边字段](#edge-field)选项：

```go {4,14}
// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int("spouse_id").
            Optional(),
    }
}

// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("spouse", User.Type).
            Unique().
            Field("spouse_id"),
    }
}
```

完整示例请参考 [GitHub](https://github.com/ent/ent/tree/master/examples/o2obidi)。

## 一对多（O2M ）两者之间

![er-user-pets](https://entgo.io/images/assets/er_user_pets.png)

在这个用户和宠物例子中，我们在用户与其宠物之间定义了一个一对多（O2M）的关系。 每个用户能够**拥有多个**宠物，一个宠物只能**拥有一个**主人。 如果用户 A 使用 `pets` 边添加了一个宠物 B，B 可以使用 `owner` 边（反向引用的边）获得它的主人。

注意：在 `Pet` 结构 (Schema) 的视角看，也是一个多对一（M2O）的关系。

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("pets", Pet.Type),
    }
}
```

`ent/schema/pet.go`
```go
// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("owner", User.Type).
            Ref("pets").
            Unique(),
    }
}
```

与这些边的交互API如下：

```go
func Do(ctx context.Context, client *ent.Client) error {
    // Create the 2 pets.
    pedro, err := client.Pet.
        Create().
        SetName("pedro").
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating pet: %w", err)
    }
    lola, err := client.Pet.
        Create().
        SetName("lola").
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating pet: %w", err)
    }
    // Create the user, and add its pets on the creation.
    a8m, err := client.User.
        Create().
        SetAge(30).
        SetName("a8m").
        AddPets(pedro, lola).
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating user: %w", err)
    }
    fmt.Println("User created:", a8m)
    // Output: User(id=1, age=30, name=a8m)

    // Query the owner. Unlike `Only`, `OnlyX` panics if an error occurs.
    owner := pedro.QueryOwner().OnlyX(ctx)
    fmt.Println(owner.Name)
    // Output: a8m

    // Traverse the sub-graph. Unlike `Count`, `CountX` panics if an error occurs.
    count := pedro.
        QueryOwner(). // a8m
        QueryPets().  // pedro, lola
        CountX(ctx)   // count
    fmt.Println(count)
    // Output: 2
    return nil
}
```

注意，外键列可以配置并暴露为实体字段，如下使用 [Edge Field](#edge-field) 选项：

```go {4,15}
// Fields of the Pet.
func (Pet) Fields() []ent.Field {
    return []ent.Field{
        field.Int("owner_id").
            Optional(),
    }
}

// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("owner", User.Type).
            Ref("pets").
            Unique().
            Field("owner_id"),
    }
}
```

完整示例请参考 [GitHub](https://github.com/ent/ent/tree/master/examples/o2m2types)。

## 一对多（O2M ）自引用

![er-tree](https://entgo.io/images/assets/er_tree.png)

在这个例子中，我们在树结点及其子结点（或它们的父结点）间定义了一个一对多（O2M）递归关系。  
树中的每个结点 **有多个** 子结点，以及 **一个** 父结点。 如果结点A将B添加到其子结点， B即可使用 `owner` 边查到其所有者。


`ent/schema/node.go`
```go
// Edges of the Node.
func (Node) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("children", Node.Type).
            From("parent").
            Unique(),
    }
}
```

如你所见，在自引用的关系中，你可以把边和它的引用在同一个构造器中声明。

```diff
func (Node) Edges() []ent.Edge {
    return []ent.Edge{
+       edge.To("children", Node.Type).
+           From("parent").
+           Unique(),

-       edge.To("children", Node.Type),
-       edge.From("parent", Node.Type).
-           Ref("children").
-           Unique(),
    }
}
```

与这些边的交互API如下：

```go
func Do(ctx context.Context, client *ent.Client) error {
    root, err := client.Node.
        Create().
        SetValue(2).
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating the root: %w", err)
    }
    // Add additional nodes to the tree:
    //
    //       2
    //     /   \
    //    1     4
    //        /   \
    //       3     5
    //
    // Unlike `Save`, `SaveX` panics if an error occurs.
    n1 := client.Node.
        Create().
        SetValue(1).
        SetParent(root).
        SaveX(ctx)
    n4 := client.Node.
        Create().
        SetValue(4).
        SetParent(root).
        SaveX(ctx)
    n3 := client.Node.
        Create().
        SetValue(3).
        SetParent(n4).
        SaveX(ctx)
    n5 := client.Node.
        Create().
        SetValue(5).
        SetParent(n4).
        SaveX(ctx)

    fmt.Println("Tree leafs", []int{n1.Value, n3.Value, n5.Value})
    // Output: Tree leafs [1 3 5]

    // Get all leafs (nodes without children).
    // Unlike `Int`, `IntX` panics if an error occurs.
    ints := client.Node.
        Query().                             // All nodes.
        Where(node.Not(node.HasChildren())). // Only leafs.
        Order(ent.Asc(node.FieldValue)).     // Order by their `value` field.
        GroupBy(node.FieldValue).            // Extract only the `value` field.
        IntsX(ctx)
    fmt.Println(ints)
    // Output: [1 3 5]

    // Get orphan nodes (nodes without parent).
    // Unlike `Only`, `OnlyX` panics if an error occurs.
    orphan := client.Node.
        Query().
        Where(node.Not(node.HasParent())).
        OnlyX(ctx)
    fmt.Println(orphan)
    // Output: Node(id=1, value=2)

    return nil
}
```

注意，外键列可以配置并暴露为实体字段，如下使用 [Edge Field](#edge-field) 选项：

```go {4,15}
// Fields of the Node.
func (Node) Fields() []ent.Field {
    return []ent.Field{
        field.Int("parent_id").
            Optional(),
    }
}

// Edges of the Node.
func (Node) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("children", Node.Type).
            From("parent").
            Unique().
            Field("parent_id"),
    }
}
```

完整示例请参考 [GitHub](https://github.com/ent/ent/tree/master/examples/o2mrecur)。

## 多对多（M2M ）两者之间

![er-user-groups](https://entgo.io/images/assets/er_user_groups.png)

在这个用户组和用户例子中，我们在用户组与其用户之间定义了一个多对多（M2M）的关系。 每个用户组能够**拥有多个**用户，并且每个用户可以加入到**多个**用户组中。

`ent/schema/group.go`
```go
// Edges of the Group.
func (Group) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("users", User.Type),
    }
}
```

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("groups", Group.Type).
            Ref("users"),
    }
}
```

与这些边的交互API如下：

```go
func Do(ctx context.Context, client *ent.Client) error {
    // Unlike `Save`, `SaveX` panics if an error occurs.
    hub := client.Group.
        Create().
        SetName("GitHub").
        SaveX(ctx)
    lab := client.Group.
        Create().
        SetName("GitLab").
        SaveX(ctx)
    a8m := client.User.
        Create().
        SetAge(30).
        SetName("a8m").
        AddGroups(hub, lab).
        SaveX(ctx)
    nati := client.User.
        Create().
        SetAge(28).
        SetName("nati").
        AddGroups(hub).
        SaveX(ctx)

    // Query the edges.
    groups, err := a8m.
        QueryGroups().
        All(ctx)
    if err != nil {
        return fmt.Errorf("querying a8m groups: %w", err)
    }
    fmt.Println(groups)
    // Output: [Group(id=1, name=GitHub) Group(id=2, name=GitLab)]

    groups, err = nati.
        QueryGroups().
        All(ctx)
    if err != nil {
        return fmt.Errorf("querying nati groups: %w", err)
    }
    fmt.Println(groups)
    // Output: [Group(id=1, name=GitHub)]

    // Traverse the graph.
    users, err := a8m.
        QueryGroups().                                           // [hub, lab]
        Where(group.Not(group.HasUsersWith(user.Name("nati")))). // [lab]
        QueryUsers().                                            // [a8m]
        QueryGroups().                                           // [hub, lab]
        QueryUsers().                                            // [a8m, nati]
        All(ctx)
    if err != nil {
        return fmt.Errorf("traversing the graph: %w", err)
    }
    fmt.Println(users)
    // Output: [User(id=1, age=30, name=a8m) User(id=2, age=28, name=nati)]
    return nil
}
```

完整示例请参考 [GitHub](https://github.com/ent/ent/tree/master/examples/m2m2types)。

## 多对多（M2M ）自引用

![er-following-followers](https://entgo.io/images/assets/er_following_followers.png)

在以下的 粉丝-被关注者 例子中，用户与他们的粉丝是 多对多(M2M) 关系。 每个用户可以关注 **多名** 用户，也可以有 **多名** 粉丝。

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("following", User.Type).
            From("followers"),
    }
}
```


如你所见，在自引用的关系中，你可以把边和它的引用在同一个构造器中声明。

```diff
func (User) Edges() []ent.Edge {
    return []ent.Edge{
+       edge.To("following", User.Type).
+           From("followers"),

-       edge.To("following", User.Type),
-       edge.From("followers", User.Type).
-           Ref("following"),
    }
}
```

与这些边的交互API如下：

```go
func Do(ctx context.Context, client *ent.Client) error {
    // Unlike `Save`, `SaveX` panics if an error occurs.
    a8m := client.User.
        Create().
        SetAge(30).
        SetName("a8m").
        SaveX(ctx)
    nati := client.User.
        Create().
        SetAge(28).
        SetName("nati").
        AddFollowers(a8m).
        SaveX(ctx)

    // Query following/followers:

    flw := a8m.QueryFollowing().AllX(ctx)
    fmt.Println(flw)
    // Output: [User(id=2, age=28, name=nati)]

    flr := a8m.QueryFollowers().AllX(ctx)
    fmt.Println(flr)
    // Output: []

    flw = nati.QueryFollowing().AllX(ctx)
    fmt.Println(flw)
    // Output: []

    flr = nati.QueryFollowers().AllX(ctx)
    fmt.Println(flr)
    // Output: [User(id=1, age=30, name=a8m)]

    // Traverse the graph:

    ages := nati.
        QueryFollowers().       // [a8m]
        QueryFollowing().       // [nati]
        GroupBy(user.FieldAge). // [28]
        IntsX(ctx)
    fmt.Println(ages)
    // Output: [28]

    names := client.User.
        Query().
        Where(user.Not(user.HasFollowers())).
        GroupBy(user.FieldName).
        StringsX(ctx)
    fmt.Println(names)
    // Output: [a8m]
    return nil
}
```

完整示例请参考 [GitHub](https://github.com/ent/ent/tree/master/examples/m2mrecur)。


## 多对多（M2M）双向自引用

![er-user-friends](https://entgo.io/images/assets/er_user_friends.png)

在这个用户与朋友的例子中，我们定义了一个 **对称的多对多（M2M）关系** 名为 `friends`。 每个用户可以 **拥有多个** 朋友。 如果用户 A 成为了 B 的朋友，那么 B 也是 A 的朋友。

注意在双向自引用场景下没有所有者/反向引用关系。

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("friends", User.Type),
    }
}
```

与这些边的交互API如下：

```go
func Do(ctx context.Context, client *ent.Client) error {
    // Unlike `Save`, `SaveX` panics if an error occurs.
    a8m := client.User.
        Create().
        SetAge(30).
        SetName("a8m").
        SaveX(ctx)
    nati := client.User.
        Create().
        SetAge(28).
        SetName("nati").
        AddFriends(a8m).
        SaveX(ctx)

    // Query friends. Unlike `All`, `AllX` panics if an error occurs.
    friends := nati.
        QueryFriends().
        AllX(ctx)
    fmt.Println(friends)
    // Output: [User(id=1, age=30, name=a8m)]

    friends = a8m.
        QueryFriends().
        AllX(ctx)
    fmt.Println(friends)
    // Output: [User(id=2, age=28, name=nati)]

    // Query the graph:
    friends = client.User.
        Query().
        Where(user.HasFriends()).
        AllX(ctx)
    fmt.Println(friends)
    // Output: [User(id=1, age=30, name=a8m) User(id=2, age=28, name=nati)]
    return nil
}
```

完整示例请参考 [GitHub](https://github.com/ent/ent/tree/master/examples/m2mbidi)。

## 边字段

边的 `Field` 选项，允许用户将外键暴露为结构 (Schema) 上的常规字段。 注意，只有持有外键 (边 id) 的关系才能使用这个选项。 Support for non-foreign-key fields in join tables is in progress (as of September 2021) and can be tracked with this [GitHub Issue](https://github.com/ent/ent/issues/1061).

```go
// Fields of the Post.
func (Post) Fields() []ent.Field {
    return []ent.Field{
        field.Int("author_id").
            Optional(),
    }
}

// Edges of the Post.
func (Post) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("author", User.Type).
            // Bind the "author_id" field to this edge.
            Field("author_id").
            Unique(),
    }
}
```

与边字段的交互API如下：

```go
func Do(ctx context.Context, client *ent.Client) error {
    p, err := c.Post.Query().
        Where(post.AuthorID(id)).
        OnlyX(ctx)
    if err != nil {
        log.Fatal(err)  
    }
    fmt.Println(p.AuthorID) // Access the "author" foreign-key.
}
```

更多例子请参考 [GitHub](https://github.com/ent/ent/tree/master/entc/integration/edgefield)。

#### 边(Edge)字段的迁移

正如 [StorageKey](#storagekey) 里提到的, Ent 通过 `edge.To` 配置边的字段 (如. 外键（foreign-keys）) . 因此如果你想添加一个已存在的字段成为边的字段, 你需要使用 `StorageKey` 选项进行配置:

```diff
// Fields of the Post.
func (Post) Fields() []ent.Field {
    return []ent.Field{
+       field.Int("author_id").
+           Optional(),
    }
}

// Edges of the Post.
func (Post) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("author", User.Type).
+           Field("author_id").
+           StorageKey(edge.Column("post_author")).
            Unique(),
    }
}
```

或者配置它在 edge-field :

```diff
// Fields of the Post.
func (Post) Fields() []ent.Field {
    return []ent.Field{
+       field.Int("author_id").
+           StorageKey("post_author").
+           Optional(),
    }
}
```

如果你不确定外键的名称, 请检查你项目里的: `<project>/ent/migrate/schema.go`.

## Required

边可以在实体构建时在构建器里使用 `Required` 方法指定必须条件。

```go
// Edges of the Card.
func (Card) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("owner", User.Type).
            Ref("card").
            Unique().
            Required(),
    }
}
```

就像上面的示例，如果没有所有者就不能创建一个卡的实体。

:::注意 ： 从[v0.10](https://github.com/ent/ent/releases/tag/v0.10.0)版本开始, 外键在必须的边中默认为非空 `NOT NULL`， [自己引用自己](#o2m-same-type). 为了迁移已存在的外键, 使用 [Atlas Migration](migrate.md#atlas-integration) 选项. :::

## 存储字段

默认情况下 Ent 通过边的所有者配置边的 storage-keys (schema 里有 `edge.To` ), 而不是反向引用 (`edge.From`). 因为反向引用是可选的，可以被删除.

为了使用自定义存储字段， 使用 `StorageKey` 方法:

```go
// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("pets", Pet.Type).
            // 设定字段名在 "pets" 表实现一对多的关系.
            StorageKey(edge.Column("owner_id")),
        edge.To("cars", Car.Type).
            // 设定外键实现一对多的关系.
            StorageKey(edge.Symbol("cars_owner_id")),
        edge.To("friends", User.Type).
            // 设定中间表实现多对多的关系.
            StorageKey(edge.Table("friends"), edge.Columns("user_id", "friend_id")),
        edge.To("groups", Group.Type).
            // 设定中间表及表中的字段， 实现多对多关系
            StorageKey(
                edge.Table("groups"),
                edge.Columns("user_id", "group_id"),
                edge.Symbols("groups_id1", "groups_id2")
            ),
    }
}
```

## 结构体标记

使用 `StructTag` 设定自定义结构名称. 注意如果没有添加 `json` 标签或者为添加任何标签, 默认会自动创建 `json` 标签

```go
// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("pets", Pet.Type).
            // 覆盖默认的Json 标签， "pets" 和 "owner" 是一对多的关系.
            StructTag(`json:"owner"`),
    }
}
```

## 索引

索引可以添加在多个字段和边中. 注意这只能用在 SQL 类型的数据库中.

 了解更多 [Indexes](schema-indexes.md) .

## 注解

`Annotations` 用于在代码生成中将任意元数据附加到字段对象。 模板扩展可以检索此元数据并在其模板中使用它

注意 metadata 必须序列化成 JSON 序列 (如. struct, map 或者 slice).

```go
// Pet schema.
type Pet struct {
    ent.Schema
}

// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
    return []ent.Field{
        edge.To("owner", User.Type).
            Ref("pets").
            Unique().
            Annotations(entgql.Annotation{
                OrderField: "OWNER",
            }),
    }
}
```

了解更多 [template doc](templates.md#annotations).

## 命名规范

默认命名方式为 `snake_case`.  `ent` 允许生成的代码为 `PascalCase` 格式. 如果想用 `PascalCase` 格式, 你可以使用 `StorageKey` 或 `StructTag` 方法自己设置.