# go语言的三件套|青训营笔记

**这是我参与「第五届青训营 」伴学笔记创作活动的第5天**

## Gorm

首先介绍一下Gorm框架是一个已经迭代了10年+的功能强大的ORM[对象关系映射](https://baike.baidu.com/item/ORM%E6%A1%86%E6%9E%B6/15541111?fr=aladdin)在字节跳动内部已经被广泛的使用且拥有非常丰富的开源扩展

### 基本约定

Gorm的约定：

GORM 倾向于约定，而不是配置。默认情况下，GORM 使用 `ID` 作为主键，使用结构体名的 `蛇形复数` 作为表名，字段名的 `蛇形` 作为列名，并使用 `CreatedAt`、`UpdatedAt` 字段追踪创建、更新时间

### 基本操作crud

![显示](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c838fa170244429ebd606496db92e54b~tplv-k3u1fbpfcp-watermark.image?)

这张图片就展示了gorm的一些基本操作类似于java的jdbc的一些操作

### 支持的数据库

Gorm目前支持的数据库有mysql、sqlserver、postgresql、sqlite。Gorm通过驱动来连接数据库，如果需要连接其他类型的数据库可以自己复用或者自行的开发驱动来使用。

举个例子以sqlserver举例子：

```go
import(
    "gorm.io/driver/sqlserver"
    "gorm.io/gorm"
)
dsn := "sqlserver://gorm:LoremIpsum86@localhost:9930?database=gorm"
db, err :=gorm.Open(sqlserver.Open(dsn),&gorm.Config{})
```

[dsn详解](http://github.com/go-sql-driver/mysql#dsn-data-source-name)

### 创建数据

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38c657ba5f70488b8835ed0bc225a7a3~tplv-k3u1fbpfcp-watermark.image?)

一般我个人用的是：GORM 允许使用 SQL 表达式插入数据，有两种方法实现这个目标。根据 `map[string]interface{}` 或 [自定义数据类型](https://gorm.cn/zh_CN/docs/data_types.html#gorm_valuer_interface) 创建，例如：

```go
// 根据 map Create
DB.Model(User{}).Create(map[string]interface{}{
  "Name": "tq",
  "Location": clause.Expr{SQL: "ST_PointFromText(?)", Vars: []interface{}{"POINT(100 100)"}},
})
// INSERT INTO `users` (`name`,`point`) VALUES ("jinzhu",ST_PointFromText("POINT(100 100)"));

// 根据自定义数据类型 Create
type Location struct {
    X, Y int
}

// Scan 实现了 sql.Scanner 接口
func (loc *Location) Scan(v interface{}) error {
  // Scan a value into struct from database driver
}

func (loc Location) GormDataType() string {
  return "geometry"
}

func (loc Location) GormValue(ctx context.Context, db *gorm.DB) clause.Expr {
  return clause.Expr{
    SQL:  "ST_PointFromText(?)",
    Vars: []interface{}{fmt.Sprintf("POINT(%d %d)", loc.X, loc.Y)},
  }
}

type User struct {
  Name     string
  Location Location
}

DB.Create(&User{
  Name:     "jinzhu",
  Location: Location{X: 100, Y: 100},
})
// INSERT INTO `users` (`name`,`point`) VALUES ("jinzhu",ST_PointFromText("POINT(100 100)"))

```

提示以下：如何使Upsert，可以使用clause.OnConflict处理数据冲突

```go
//以不处理冲突为例，
p :=&Product{Code:"D42",ID:1}
```

### 查询数据

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dfcbaeb2d3b84b518942063827a99c79~tplv-k3u1fbpfcp-watermark.image?)

以上代码就是查询的操作，注意：First的使用踩坑，使用First时，需要注意查询不到数据会返回ErrRecordNotFound。使用find查询多条数据，查询不到数据的时候不会报错。

使用结构图作为查询条件：当使用结构体作为查询条件的时候，GORM只会查询非零字段，这就意味着如果要查询"0" “” false等其他零值，该字段就需要用map来进行构建了

### 更新数据

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8a02cef5a1946289435e8f238e32f83~tplv-k3u1fbpfcp-watermark.image?)

使用Struct更新时候，只会更新非零值，如果需要更新零值，可以使用map更新或者使用select来选择字段

### 删除数据

#### 物理删除

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f86fa797bb543e980da675252dba7c8~tplv-k3u1fbpfcp-watermark.image?)

gorm提供了gorm.DeletedAt用于帮助用户实现软删除，拥有软删除能力的Model调用Delete时、记录不会被从数据库真正的删除

#### 软删除

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed70799cebf14153b048ab89549eeeb8~tplv-k3u1fbpfcp-watermark.image?)

### GORM总结

以上就是基础的crud的操作，还有其余的方式例如事务和hook（在创建、查询、更新、删除等操作之前、之后自动调用的函数）

## Kitex

kitex是字节内部自行开发的go微服务的[RPC](https://baike.baidu.com/item/%E8%BF%9C%E7%A8%8B%E8%BF%87%E7%A8%8B%E8%B0%83%E7%94%A8/7854346?fromtitle=RPC&fromid=609861&fr=aladdin)框架，具有高性能，强扩展的主要特点，支持多协议并且拥有丰富开源扩展。

在作者的个人理解其实可以类比到java的远程调用协议中的rmi。目前Kitex支持的环境是linux对windows还不够完善，所以用windows的同学可以进行安装一个虚拟机linux系统然后来和主机进行通信，只需要在同一个网段下面 即可，kitex的端口号默认为8888。在这里作者不过多赘述详细请去[这里](https://www.cloudwego.io/zh/docs/kitex)

## Hertz

hertz字节内部的http框架，参考了其他开源框架的优势，结合字节内部需求，具有高易用性，高性能，高扩展性的特点。

### 基本使用

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74fdf4352c6d451aa623e3b9d002cd87~tplv-k3u1fbpfcp-watermark.image?)

使用hertz实现、服务器监听8080端口并注册了一个get方法的路由函数，其实很类似于java springboot中@RequestMapping 来注册一个路由

### hertz路由

hertz提供了get、post、put、delete、any等方法路由。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8c6ccac6d054edeb43b33e90e0b5725~tplv-k3u1fbpfcp-watermark.image?)

也提供了路由组的能力，用于支持路由分组的功能

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1c4c81f076f43ce8754ea8ab8706e48~tplv-k3u1fbpfcp-watermark.image?)

提供了参数路由和通配路由等，路由优先级 静态路由>命名路由>通配路由

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a481ebd677754037abfa7dc64f98398b~tplv-k3u1fbpfcp-watermark.image?)

### hertz中间件

hertz中间件主要分为客户端中间件和服务端中间件如下

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fdbc6a89e05c4e6d8b61558ecf432a54~tplv-k3u1fbpfcp-watermark.image?)

### hertz总结

在我看来其实和springboot里面的controller层是异曲同工之妙的。

## 引用

-   [go官网之go语言介绍](https://golang.google.cn/doc/effective_go#introduction)
-   [go语言的菜鸟教程](https://www.runoob.com/go/go-tutorial.html)

