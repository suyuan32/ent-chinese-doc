**ent** 是一个简单但功能强大的 Go 实体框架，它可以轻松构建和维护具有大型数据模型的应用程序，并遵循以下原则：

- 简单地使用数据库结构作为图结构。
- 使用Go代码定义结构。
- 基于代码生成的静态类型。
- 容易地进行数据库查询和图遍历。
- 容易地使用Go模板扩展和自定义。

![gopher-schema-as-code](https://entgo.io/images/assets/gopher-schema-as-code.png)

## 初始化Go环境

如果你的项目目录在 [GOPATH](https://github.com/golang/go/wiki/GOPATH) 之外 或 不熟悉 GOPATH， 可以为项目设置 [Go module](https://github.com/golang/go/wiki/Modules#quick-start)：

```console
go mod init <project>
```

## 创建你的第一个模式

进入你项目的根目录，然后运行：

```console
go run -mod=mod entgo.io/ent/cmd/ent init User
```

以上的命令会在`<project>/ent/schema/`目录下产生`User`的数据模式（数据模式是数据库系统设计中的专业术语，若对该部分有任何理解问题，请查阅数据库系统的相关书籍）：

```go title="<project>/ent/schema/user.go"

package schema

import "entgo.io/ent"

// User在User实体中组合了ent默认的数据库模式定义
type User struct {
    ent.Schema
}

// User的字段
func (User) Fields() []ent.Field {
    return nil
}

// User的边
func (User) Edges() []ent.Edge {
    return nil
}

```

为`User` 模式添加两个字段：

```go title="<project>/ent/schema/user.go"

package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/field"
)

// User的字段
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age").
            Positive(),
        field.String("name").
            Default("unknown"),
    }
}
```

从项目的根目录下像如下命令那样，运行`go generate`：

```go
go generate ./ent
```

上述命令，将产生如下的文件：
```console {12-20}
ent
├── client.go
├── config.go
├── context.go
├── ent.go
├── generate.go
├── mutation.go
... truncated
├── schema
│   └── user.go
├── tx.go
├── user
│   ├── user.go
│   └── where.go
├── user.go
├── user_create.go
├── user_delete.go
├── user_query.go
└── user_update.go
```


## 创建你的第一个实体

首先，创建一个`ent.Client`。

<Tabs
defaultValue="sqlite"
values={[
{label: 'SQLite', value: 'sqlite'},
{label: 'PostgreSQL', value: 'postgres'},
{label: 'MySQL', value: 'mysql'},
]}>
<TabItem value="sqlite">

```go title="<project>/start/start.go"
package main

import (
    "context"
    "log"

    "<project>/ent"

    _ "github.com/mattn/go-sqlite3"
)

func main() {
    client, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
    if err != nil {
        log.Fatalf("failed opening connection to sqlite: %v", err)
    }
    defer client.Close()
    // 运行自动迁移工具。
    if err := client.Schema.Create(context.Background()); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

</TabItem>
<TabItem value="postgres">

```go title="<project>/start/start.go"
package main

import (
    "context"
    "log"

    "<project>/ent"

    _ "github.com/lib/pq"
)

func main() {
    client, err := ent.Open("postgres","host=<host> port=<port> user=<user> dbname=<database> password=<pass>")
    if err != nil {
        log.Fatalf("failed opening connection to postgres: %v", err)
    }
    defer client.Close()
    // 运行自动迁移工具。
    if err := client.Schema.Create(context.Background()); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

</TabItem>
<TabItem value="mysql">

```go title="<project>/start/start.go"
package main

import (
    "context"
    "log"

    "<project>/ent"

    _ "github.com/go-sql-driver/mysql"
)

func main() {
    client, err := ent.Open("mysql", "<user>:<pass>@tcp(<host>:<port>)/<database>?parseTime=True")
    if err != nil {
        log.Fatalf("failed opening connection to mysql: %v", err)
    }
    defer client.Close()
    // 运行自动迁移工具。
    if err := client.Schema.Create(context.Background()); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

</TabItem>
</Tabs>

现在，我们准备创建我们的用户。 让我们写一个 `CreateUser` 函数，比如：

```go title="<project>/start/start.go"
func CreateUser(ctx context.Context, client *ent.Client) (*ent.User, error) {
    u, err := client.User.
        Create().
        SetAge(30).
        SetName("a8m").
        Save(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed creating user: %w", err)
    }
    log.Println("user was created: ", u)
    return u, nil
}
```

## 查询你的实体

`ent` 为每个实体结构生成一个package，包含其条件、默认值、验证器、有关存储元素的附加信息 (字段名、主键等) 。

```go title="<project>/start/start.go"
package main

import (
    "log"

    "<project>/ent"
    "<project>/ent/user"
)

func QueryUser(ctx context.Context, client *ent.Client) (*ent.User, error) {
    u, err := client.User.
        Query().
        Where(user.Name("a8m")).
        // `Only` 如果没有发现用户则报错,
        // 否则正常返回。
        Only(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed querying user: %w", err)
    }
    log.Println("user returned: ", u)
    return u, nil
}
```


## 添加你的第一条边 (关系)
在这部分教程中，我们想声明一条到另一个 实体 的 边(关系) 。 让我们另外创建2个实体，分别为 `Car` 和 `Group`，并添加一些字段。 我们使用 `ent` CLI 生成初始的结构：

```console
go run -mod=mod entgo.io/ent/cmd/ent init Car Group
```

然后我们手动添加其他字段：

```go title="<project>/ent/schema/car.go"
// Fields of the Car.
func (Car) Fields() []ent.Field {
    return []ent.Field{
        field.String("model"),
        field.Time("registered_at"),
    }
}
```

```go title="<project>/ent/schema/group.go"
// Fields of the Group.
func (Group) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            // 使用正则验证 组 名称
            Match(regexp.MustCompile("[a-zA-Z_]+$")),
    }
}
```

让我们来定义第一个关系。 从 `User` 到 `Car` 的关系，定义了一个用户可以**拥有1辆或多辆**汽车，但每辆汽车**只有一个**车主（一对多关系）。

![er-user-cars](https://entgo.io/images/assets/re_user_cars.png)

让我们将 `"cars"` 关系添加到 `User` 结构中，并运行 `go generate ./ent` 。

```go title="<project>/ent/schema/user.go"

// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("cars", Car.Type),
    }
}
```

作为示例，我们创建2辆汽车并将它们添加到某个用户。

```go title="<project>/start/start.go"

import (
    "<project>/ent"
    "<project>/ent/car"
    "<project>/ent/user"
)

func CreateCars(ctx context.Context, client *ent.Client) (*ent.User, error) {
    // 创建一辆 "Tesla" 汽车.
    tesla, err := client.Car.
        Create().
        SetModel("Tesla").
        SetRegisteredAt(time.Now()).
        Save(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed creating car: %w", err)
    }
    log.Println("car was created: ", tesla)

    // 创建一辆 "Ford" 汽车.
    ford, err := client.Car.
        Create().
        SetModel("Ford").
        SetRegisteredAt(time.Now()).
        Save(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed creating car: %w", err)
    }
    log.Println("car was created: ", ford)

    // 创建一个用户，并给他添加两辆汽车.
    a8m, err := client.User.
        Create().
        SetAge(30).
        SetName("a8m").
        AddCars(tesla, ford).
        Save(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed creating user: %w", err)
    }
    log.Println("user was created: ", a8m)
    return a8m, nil
}

```
想要查询 `cars` 边 (关系) 怎么办？ 请参考下面这样：

```go title="<project>/start/start.go"
import (
    "log"

    "<project>/ent"
    "<project>/ent/car"
)

func QueryCars(ctx context.Context, a8m *ent.User) error {
    cars, err := a8m.QueryCars().All(ctx)
    if err != nil {
        return fmt.Errorf("failed querying user cars: %w", err)
    }
    log.Println("returned cars:", cars)

    // 查询某辆汽车。
    ford, err := a8m.QueryCars().
        Where(car.Model("Ford")).
        Only(ctx)
    if err != nil {
        return fmt.Errorf("failed querying user cars: %w", err)
    }
    log.Println(ford)
    return nil
}
```

## 添加您的第一条反向边(反向引用)
假定我们有一个 `Car` 对象，我们想要得到它的所有者；即这辆汽车所属的用户。 为此，我们有另一种“反向”的边，通过 `edge.From` 函数定义。

![er-cars-owner](https://entgo.io/images/assets/re_cars_owner.png)

在上图中新建的边是半透明的，以强调我们没有在数据库中创建另一个关联。 它是对真实边（关系）的反向引用。

让我们把一个名为 `owner` 的反向边添加到 `Car` 的结构中, 在 `User` 结构中引用它到 `cars` 关系 然后运行 `go generate ./ent`

```go title="<project>/ent/schema/car.go"
// Edges of the Car.
func (Car) Edges() []ent.Edge {
    return []ent.Edge{
        // 创建一个名为 "owner" 的 `User` 反向边
        // 并关联到 "cars" 边 (在 用户 结构中)
        // 并显式的使用 `Ref` 方法。
        edge.From("owner", User.Type).
            Ref("cars").
            // 设置边为唯一确保每辆车都只有一个拥有者
            Unique(),
    }
}
```
我们继续使用用户和汽车，作为查询反向边的例子。

```go title="<project>/start/start.go"
import (
    "fmt"
    "log"

    "<project>/ent"
    "<project>/ent/user"
)

func QueryCarUsers(ctx context.Context, a8m *ent.User) error {
    cars, err := a8m.QueryCars().All(ctx)
    if err != nil {
        return fmt.Errorf("failed querying user cars: %w", err)
    }
    // 通过反边查询。
    for _, c := range cars {
        owner, err := c.QueryOwner().Only(ctx)
        if err != nil {
            return fmt.Errorf("failed querying car %q owner: %w", c.Model, err)
        }
        log.Printf("car %q owner: %q\n", c.Model, owner.Name)
    }
    return nil
}
```

## 创建你的第二条边

继续我们的例子，在用户和组之间创建一个M2M（多对多）关系。

![er-group-users](https://entgo.io/images/assets/re_group_users.png)

如图所示，每个组实体可以**拥有许多**用户，而一个用户可以**关联到多个**组。 一个简单的 "多对多 "关系。 上图中，`Group` 结构是 `users` 关系的所有者， 而 `User` 实体对这个关系有一个名为 `groups` 的反向引用。 让我们在结构中定义这种关系。

```go title="<project>/ent/schema/group.go"
// Edges of the Group.
func (Group) Edges() []ent.Edge {
   return []ent.Edge{
       edge.To("users", User.Type),
   }
}
```

```go title="<project>/ent/schema/user.go"
// Edges of the User.
func (User) Edges() []ent.Edge {
   return []ent.Edge{
       edge.To("cars", Car.Type),
       // 创建一个名为 "groups" 的 `Group` 的反向边
       // 并关联至 "users" 边 (在 组 结构中)
       // 并显式的使用 `Ref` 方法.
       edge.From("groups", Group.Type).
           Ref("users"),
   }
}
```

在schema目录的上级目录中运行`ent`来重新生成资源文件。
```console
go generate ./ent
```

## 进行第一次图遍历

为了进行我们的第一次图遍历，我们需要生成一些数据 (节点和边，或者说，实体和关系)。 创建如下图所示的框架：

![re-graph](https://entgo.io/images/assets/re_graph_getting_started.png)


```go title="<project>/start/start.go"
func CreateGraph(ctx context.Context, client *ent.Client) error {
    // 首先创建用户
    a8m, err := client.User.
        Create().
        SetAge(30).
        SetName("Ariel").
        Save(ctx)
    if err != nil {
        return err
    }
    neta, err := client.User.
        Create().
        SetAge(28).
        SetName("Neta").
        Save(ctx)
    if err != nil {
        return err
    }
    // 然后创建汽车，并关联到创建的用户
    err = client.Car.
        Create().
        SetModel("Tesla").
        SetRegisteredAt(time.Now()).
        // 把汽车关联到 Ariel.
        SetOwner(a8m).
        Exec(ctx)
    if err != nil {
        return err
    }
    err = client.Car.
        Create().
        SetModel("Mazda").
        SetRegisteredAt(time.Now()).
        // 把汽车关联到 Ariel.
        SetOwner(a8m).
        Exec(ctx)
    if err != nil {
        return err
    }
    err = client.Car.
        Create().
        SetModel("Ford").
        SetRegisteredAt(time.Now()).
        // 关联汽车到 Neta。
        SetOwner(neta).
        Exec(ctx)
    if err != nil {
        return err
    }
    // 创建组，并把用户添加到组中
    err = client.Group.
        Create().
        SetName("GitLab").
        AddUsers(neta, a8m).
        Exec(ctx)
    if err != nil {
        return err
    }
    err = client.Group.
        Create().
        SetName("GitHub").
        AddUsers(a8m).
        Exec(ctx)
    if err != nil {
        return err
    }
    log.Println("The graph was created successfully")
    return nil
}
```

现在我们有一个含数据的图，我们可以对它运行一些查询：

1. 获取名为 "GitHub" 的群组内所有用户的汽车。

    ```go title="<project>/start/start.go"
    import (
        "log"

        "<project>/ent"
        "<project>/ent/group"
    )

    func QueryGithub(ctx context.Context, client *ent.Client) error {
        cars, err := client.Group.
            Query().
            Where(group.Name("GitHub")). // (Group(Name=GitHub),)
            QueryUsers().                // (User(Name=Ariel, Age=30),)
            QueryCars().                 // (Car(Model=Tesla, RegisteredAt=<Time>), Car(Model=Mazda, RegisteredAt=<Time>),)
            All(ctx)
        if err != nil {
            return fmt.Errorf("failed getting cars: %w", err)
        }
        log.Println("cars returned:", cars)
        // 输出: (Car(Model=Tesla, RegisteredAt=<Time>), Car(Model=Mazda, RegisteredAt=<Time>),)
        return nil
    }
    ```

2. 修改上面的查询，遍历 *Ariel* 用户。

    ```go title="<project>/start/start.go"
    import (
        "log"

        "<project>/ent"
        "<project>/ent/car"
    )

    func QueryArielCars(ctx context.Context, client *ent.Client) error {
        // 从之前的步骤中获取 "Ariel".
        a8m := client.User.
            Query().
            Where(
                user.HasCars(),
                user.Name("Ariel"),
            ).
            OnlyX(ctx)
        cars, err := a8m.                       // 获取组，a8m 关联到:
                QueryGroups().                  // (Group(Name=GitHub), Group(Name=GitLab),)
                QueryUsers().                   // (User(Name=Ariel, Age=30), User(Name=Neta, Age=28),)
                QueryCars().                    //
                Where(                          //
                    car.Not(                    //  获取 Neta 和 Ariel 的汽车，但是
                        car.Model("Mazda"),     //  过滤名为 "Mazda" 的汽车
                    ),                          //
                ).                              //
                All(ctx)
        if err != nil {
            return fmt.Errorf("failed getting cars: %w", err)
        }
        log.Println("cars returned:", cars)
        // 输出: (Car(Model=Tesla, RegisteredAt=<Time>), Car(Model=Ford, RegisteredAt=<Time>),)
        return nil
    }
    ```

3. 获取所有拥有用户的组 (通过额外 [look-aside] 断言查询)：

    ```go title="<project>/start/start.go"
    import (
        "log"

        "<project>/ent"
        "<project>/ent/group"
    )

    func QueryGroupWithUsers(ctx context.Context, client *ent.Client) error {
        groups, err := client.Group.
            Query().
            Where(group.HasUsers()).
            All(ctx)
        if err != nil {
            return fmt.Errorf("failed getting groups: %w", err)
        }
        log.Println("groups returned:", groups)
        // 输出: (Group(Name=GitHub), Group(Name=GitLab),)
        return nil
    }
    ```

## 完整示例

完整示例请参阅 [GitHub](https://github.com/ent/ent/tree/master/examples/start).
