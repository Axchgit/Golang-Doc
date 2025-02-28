# swag

swaggo/swag是Swagger API 2.0在go语言中的一个实现，通过在书写指定格式的注释就可以生成`swagger.json`和`swagger.yaml`类型的接口文档，方便导出和导入。

仓库：[swaggo/swag: Automatically generate RESTful API documentation with Swagger 2.0 for Go. (github.com)](https://github.com/swaggo/swag)

文档：[swaggo/swag: Automatically generate RESTful API documentation with Swagger 2.0 for Go. (github.com)](https://github.com/swaggo/swag#readme)

swag默认支持的web框架如下所示，本文以gin为例子，来演示gin结合swagger快速生成接口文档的例子。

- [gin](http://github.com/swaggo/gin-swagger)
- [echo](http://github.com/swaggo/echo-swagger)
- [buffalo](https://github.com/swaggo/buffalo-swagger)
- [net/http](https://github.com/swaggo/http-swagger)
- [gorilla/mux](https://github.com/swaggo/http-swagger)
- [go-chi/chi](https://github.com/swaggo/http-swagger)
- [flamingo](https://github.com/i-love-flamingo/swagger)
- [fiber](https://github.com/gofiber/swagger)
- [atreugo](https://github.com/Nerzal/atreugo-swagger)
- [hertz](https://github.com/hertz-contrib/swagger)



::: tip

如果不熟悉swagger语法，可以前往[About Swagger Specification | Documentation | Swagger](https://swagger.io/docs/specification/about/)

:::

## 安装

首先下载swagger命令行工具

```
go install github.com/swaggo/swag/cmd/swag@latest
```

然后下载swagger源码依赖

```
go get github.com/swaggo/swag
```

::: tip

为避免出现问题，两者版本必须保持一致。

:::

然后下载swagger的静态文件库，html，css，js之类的，都被嵌到了go代码中。

```
go get github.com/swaggo/files@latest
```

最后下载swagger的gin适配库

```
go get github.com/swaggo/gin-swagger@latest
```

因为本文是只用gin做示例，其他web框架的适配器请自行了解，基本都是大同小异。



## 简单使用

使用go mod创建一个最基本的go项目，新建`main.go`，写入如下内容。

```go
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
)

// @title           Swagger Example API
// @version         1.0
// @description     This is a sample server celler server.

// @contact.name   API Support
// @contact.url    http://www.swagger.io/support
// @contact.email  support@swagger.io

// @BasePath  /api/v1
func main() {
	engine := gin.Default()
	engine.GET("/api/v1/ping", Ping)
	engine.Run(":80")
}

// Ping godoc
// @Summary      say hello world
// @Description  return hello world json format content
// @param       name query    string  true  "name"
// @Tags         system
// @Produce      json
// @Router       /ping [get]
func Ping(ctx *gin.Context) {
	ctx.JSON(200, gin.H{
		"message": fmt.Sprintf("Hello World!%s", ctx.Query("name")),
	})
}
```

这是一个很简单的gin web例子，main函数上的注释是文档的基本信息，Ping函数则是一个普通的接口。接下来执行命令生成文档，默认是在`main.go`同级的docs目录下

```
swag init
```

修改`main.go`代码

```go
package main

import (
    "fmt"
    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger "github.com/swaggo/gin-swagger"
    // 匿名导入生成的接口文档包
    _ "golearn/docs"
)

// @title           Swagger Example API
// @version         1.0
// @description     This is a sample server celler server.

// @contact.name   API Support
// @contact.url    http://www.swagger.io/support
// @contact.email  support@swagger.io

// @BasePath  /api/v1
func main() {
    engine := gin.Default()
    // 注册swagger静态文件路由
    engine.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
    engine.GET("/api/v1/ping", Ping)
    engine.Run(":80")
}

// Ping godoc
// @Summary      say hello world
// @Description  return hello world json format content
// @param       name query    string  true  "name"
// @Tags         system
// @Produce      json
// @Router       /ping [get]
func Ping(ctx *gin.Context) {
    ctx.JSON(200, gin.H{
       "message": fmt.Sprintf("Hello World!%s", ctx.Query("name")),
    })
}
```

运行程序，访问`127.0.0.1/swagger/index.html`，界面如下

![](https://public-1308755698.cos.ap-chongqing.myqcloud.com//img/202308132014682.png)

如此便运行起了一个基本的接口文档。接下来除了一些特别要注意的点，基本上和其他语言使用起来没有什么太大的差别。



## 配置

swag实际上是将多个不同的swagger实例存放在一个map中，ginSwagger作为适配器从实例中读取`doc.json`也就是API接口的定义文件，swaggerFiles提供静态的HTML文件用于展示网页，解析API定义并生成界面，整个流程明白以后，就可以进行自定义的操作了。

```go
// Name is a unique name be used to register swag instance.
// 默认实例名称
const Name = "swagger"

var (
	swaggerMu sync.RWMutex
    // 实例表
	swags     map[string]Swagger
)
```

```go
func CustomWrapHandler(config *Config, handler *webdav.Handler) gin.HandlerFunc {
    var once sync.Once

    if config.InstanceName == "" {
       config.InstanceName = swag.Name
    }

    if config.Title == "" {
       config.Title = "Swagger UI"
    }

    // create a template with name
    index, _ := template.New("swagger_index.html").Parse(swaggerIndexTpl)

    var matcher = regexp.MustCompile(`(.*)(index\.html|doc\.json|favicon-16x16\.png|favicon-32x32\.png|/oauth2-redirect\.html|swagger-ui\.css|swagger-ui\.css\.map|swagger-ui\.js|swagger-ui\.js\.map|swagger-ui-bundle\.js|swagger-ui-bundle\.js\.map|swagger-ui-standalone-preset\.js|swagger-ui-standalone-preset\.js\.map)[?|.]*`)

    return func(ctx *gin.Context) {
       if ctx.Request.Method != http.MethodGet {
          ctx.AbortWithStatus(http.StatusMethodNotAllowed)

          return
       }
	
       // 路由匹配
       matches := matcher.FindStringSubmatch(ctx.Request.RequestURI)

       if len(matches) != 3 {
          ctx.String(http.StatusNotFound, http.StatusText(http.StatusNotFound))

          return
       }

       path := matches[2]
       once.Do(func() {
          handler.Prefix = matches[1]
       })
		
       switch filepath.Ext(path) {
       case ".html":
          ctx.Header("Content-Type", "text/html; charset=utf-8")
       case ".css":
          ctx.Header("Content-Type", "text/css; charset=utf-8")
       case ".js":
          ctx.Header("Content-Type", "application/javascript")
       case ".png":
          ctx.Header("Content-Type", "image/png")
       case ".json":
          ctx.Header("Content-Type", "application/json; charset=utf-8")
       }

       switch path {
       // 主页
       case "index.html":
          _ = index.Execute(ctx.Writer, config.toSwaggerConfig())
       // API描述文件
       case "doc.json":
          doc, err := swag.ReadDoc(config.InstanceName)
          if err != nil {
             ctx.AbortWithStatus(http.StatusInternalServerError)

             return
          }

          ctx.String(http.StatusOK, doc)
       default:
          handler.ServeHTTP(ctx.Writer, ctx.Request)
       }
    }
}
```

通过生成的go代码来自动完成实例注册，下方是自动生成的`docs.go`的部分代码

```go
// SwaggerInfo holds exported Swagger Info so clients can modify it
var SwaggerInfo = &swag.Spec{
	Version:          "",
	Host:             "",
	BasePath:         "",
	Schemes:          []string{},
	Title:            "",
	Description:      "",
	InfoInstanceName: "swagger",
	SwaggerTemplate:  docTemplate,
	LeftDelim:        "{{",
	RightDelim:       "}}",
}

func init() {
    // 注册实例
	swag.Register(SwaggerInfo.InstanceName(), SwaggerInfo)
}
```

可以看到在`init`函数中有一个Register函数用来注册当前实例，如果想要修改实例名称不建议在该文件进行编辑，因为`docs.go`文件是自动生成的，只需要在生成代码时使用`--instanceName appapi`参数。为了方便，可以使用go generate命令嵌入的到go文件中，方便自动生成代码，如下。

```go
// swagger declarative api comment

// @title App Internal API Documentation
// @version v1.0.0
// @description Wilson api documentation
// @BasePath /api/v1
//go:generate swag init --generatedTime --instanceName appapi -g api.go -d ./ --output ./swagger
```

个人并不喜欢将swagger的通用信息注释写在`main.go`或`main`函数上，将这些注释写在`go generate`上方最合适不过。

::: tip

如果需要多个实例，务必保持实例名称唯一，否则会`panic`

:::

为了定制化一些配置，需要用`ginSwagger.CustomWrapHandler`，它相比前者多了一个Config参数，释义如下

```go
// Config stores ginSwagger configuration variables.
type Config struct {
	// The url pointing to API definition (normally swagger.json or swagger.yaml). Default is `doc.json`.
	URL                      string
    // 接口列表展开状态
	DocExpansion             string
    // 实例名称
	InstanceName             string
    // 标题
	Title                    string
    // 展开深度
	DefaultModelsExpandDepth int
    // 顾名思义
	DeepLinking              bool
	PersistAuthorization     bool
	Oauth2DefaultClientID    string
}
```

使用`swaggerFiles.NewHandler()`来替代默认的Handler，在多个实例时尤其要如此。

```go
engine.GET(openapi.ApiDoc, ginSwagger.CustomWrapHandler(openapi.Config, swaggerFiles.NewHandler()))
```

除此之外还可以进行类型重写等一系列操作，都是比较简单的，更多内容可以阅读官方文档。



## 注意事项

1. swag是根据注释来生成openapi的接口描述文件的，在生成时，指定的目录必须要包含接口文档的基本信息，默认是在`main.go`里面查找

2. `swag init`默认指定的是当前目录，值为`./`，可以使用`swag init -d`指定多个目录，使用逗号分隔，第一个指定的目录必须包含接口文档的基本信息。例如

    ```
    swag init -d ./,./api
    ```

3. `-g`，接口文档的基本信息的存放文件可以自定义文件名，默认是`main.go`，在生成文档时，使用`-g`参数指定文件名

    ```
    swag init -g api.go -d ./,./api
    ```

    该命令的意思是在`./api.go`解析接口文档的基本信息，同时在`./`和`./api`两个目录下查找和解析其他接口的注释信息并生成对应的文档。

4. `-o`参数可以指定文档描述文件的输出路径，默认是`./docs`，例:

    ```
    swag init -o ./api/docs
    ```

5. `--ot`可以指定输出文件类型，默认是（docs.go,swagger.json,swagger.yaml），如果想要使用go程序加载swagger ui，go文件是必不可少的。

    ```
    swag init --ot go,yaml
    ```

    其余生成的json和yaml文件可以方便在其他接口管理软件上导入数据。

6. 注释写在哪里都一样，就算不写在函数上也一样能解析，只是写在函数上可读性好一些，本质上还是一个以注释形式和go源代码嵌在一起的DSL。

7. swag还支持很多其他的参数，可以使用`swag init -h`查看。



