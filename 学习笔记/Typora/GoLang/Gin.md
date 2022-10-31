

GoLang1.3之后采用





创建文件夹



使用`go env -w GO111MODULE=on`管理`go`项目依赖

`go mod init 项目名称`

下载`gin`，

设置代理`go env -w GOPROXY=https://goproxy.io,direct`，下载`gin`,`go get github.com/gin-gonic/gin`

下载





初始化





创建请求

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	// 创建默认路由请求
	engine := gin.Default()

	// 创建一个GET请求
	engine.GET("/get", func(c *gin.Context) {
		c.String(200, "Hello World")
	})

	// 创建一个POST请求
	engine.POST("/post", func(c *gin.Context) {
		c.String(200, "这是一个post方法")
	})

	// 创建一个PUT请求
	engine.PUT("/put", func(c *gin.Context) {
		c.String(200, "这是一个put方法")
	})

	// 创建一个DELETE请求
	engine.DELETE("/delete", func(c *gin.Context) {
		c.String(200, "这是一个delete方法")
	})

	err := engine.Run(":8000")
	if err != nil {
		return
	}

}

```



安装热部署`fresh`插件（以下操作请在自己的`go`项目中执行）

```shell
# 下载插件
go get github.com/pilu/fresh
# 编译插件
go install github.com/pilu/fresh
# 使用插件（不进行编译的话，会报错 fresh 不是命令）
fresh
```



创建一个返回值为`json`的请求

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

type Article struct {
	Title   string `json:"title"`
	Desc    string `json:"desc"`
	Content string `json:"content"`
}

func main() {
	// 创建默认路由请求
	engine := gin.Default()

	// 返回json
	engine.GET("/json1", func(c *gin.Context) {
		c.JSON(http.StatusOK, map[string]interface{}{
			"success": true,
			"message": "你好我是函数json2",
		})
	})

	// 返回json
	engine.GET("/json2", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"success": true,
			"message": "你好我是函数json1",
		})
	})

	// 返回json对象
	engine.GET("/json3", func(c *gin.Context) {
		result := Article{
			Title:   "我是一个标题",
			Desc:    "描述",
			Content: "不忘初心方得始终",
		}
		c.JSON(http.StatusOK, result)
	})

	// 用来解决跨域问题，回调前端函数
	engine.GET("/jsonp", func(c *gin.Context) {
		result := Article{
			Title:   "我是一个标题-jsonp",
			Desc:    "描述",
			Content: "不忘初心方得始终",
		}
		c.JSON(http.StatusOK, result)
	})

	// 返回一个xml
	engine.GET("/xml", func(c *gin.Context) {
		c.XML(http.StatusOK, gin.H{
			"success": true,
			"msg":     "这是一个xml",
		})
	})

	// 配置模板路径
	engine.LoadHTMLGlob("templates/*")
	// 返回一个渲染页面
	engine.GET("/html", func(c *gin.Context) {
		// goods.html 前端模板名称
		c.HTML(http.StatusOK, "goods.html", gin.H{
			"title": "我是自定义模板",
		})
	})
    
    // GET 接收参数
	engine.GET("/query", func(c *gin.Context) {
		result := Article{
			// 获取请求参数
			Title: c.Query("title"),
			// 获取请求参数，不存在时返回默认值
			Desc: c.DefaultQuery("desc", "22"),
		}
		c.JSON(http.StatusOK, result)
	})

	err := engine.Run(":8000")
	if err != nil {
		return
	}

}

```





请求参数接收

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

type Article struct {
    // json:"title" 数据返回前端时的属性，类似@RestController
    // form:"title"	接受来自前端的属性，类似@RequestBody
	Title   string `json:"title" form:"title"`
	Desc    string `json:"desc" form:"desc"`
	Content string `json:"content" form:"content"`
}

func main() {

	// 创建默认路由请求
	engine := gin.Default()

	// GET 接收参数
	engine.GET("/getQuery", func(c *gin.Context) {
		result := Article{
			// 获取请求参数
			Title: c.Query("title"),
			// 获取请求参数，不存在时返回默认值
			Desc: c.DefaultQuery("desc", "22"),
		}
		c.JSON(http.StatusOK, result)
	})

	// GET 接收参数，并转换为对象
	engine.GET("/getQueryObject", func(c *gin.Context) {
		// 声明结构体
		article := &Article{}
		// c.ShouldBind(&article) 把请求参数绑定到结构体中，请求参数必须和结构体中的form中的属性名对应
		if err := c.ShouldBind(&article); err == nil {
			c.JSON(http.StatusOK, article)
		} else {
			c.JSON(http.StatusOK, gin.H{
				"err": err.Error(),
			})
		}
	})

	// GET 页面跳转
	// 配置模板路径
	engine.LoadHTMLGlob("templates/*")
	engine.GET("/jump", func(c *gin.Context) {
		c.HTML(http.StatusOK, "goods.html", gin.H{
			"title": "不忘初心",
			"desc":  "方得始终",
		})
	})

	// POST 接收参数
	engine.POST("/postQuery", func(c *gin.Context) {
		result := Article{
			// 获取请求参数
			Title: c.PostForm("title"),
			// 获取请求参数，不存在时返回默认值
			Desc: c.DefaultPostForm("desc", "22"),
		}
		c.JSON(http.StatusOK, result)
	})

	err := engine.Run(":8000")
	if err != nil {
		return
	}

}

```





## 自定义控制器（类比`java`中的`controller`）

1.   项目根目录创建`admin`目录，创建自定义调度器`userController`类（类比`java`中的`controller`）
2.   项目根目录创建`routers`目录，创建自定义路由引擎`userRouters`类，注意导入`admin`包
3.   项目根目录创建`main`类，创建`main`方法，注意导入`routers`包

### 自定义控制器类

```go
// Package admin 包名称
package admin

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

// QueryUserList 函数，处理业务（类别controller中的函数）
func QueryUserList(c *gin.Context) {
	c.String(http.StatusOK, "用户列表")
}
```

### 自定义路由引擎类

```go
// Package routers 包名称
package routers

import (
	// 引入自定义控制器
	// demo：项目名
	// controller：包名称
	// admin：包名称
	"demo/controller/admin"
	"github.com/gin-gonic/gin"
)

// AdminRoutersInit 自定义路由引擎函数
func AdminRoutersInit(r *gin.Engine) {
	// 自定义路由引擎分组
	adminRouters := r.Group("/admin")
	{
		// 注册userController类的QueryUserList函数
		// admin.QueryUserList 注意不要带括号，不带括号是注册，带括号为执行
		adminRouters.GET("/queryUserList", admin.QueryUserList)
	}
}
```

### 启动类

```go
// 包名称
package main

import (
	// 引入自定义路由引擎
	// demo：项目名
	// routers：包名称
	"demo/routers"
	"github.com/gin-gonic/gin"
)

func main() {
	// 创建默认路由请求
	engine := gin.Default()
	// 执行允许自定义路由类
	routers.AdminRoutersInit(engine)
	// 运行项目，端口号指定为8000
	err := engine.Run(":8000")
	// 存在异常终止运行
	if err != nil {
		return
	}
}
```







```go

```







```go

```







```go

```







```go

```







```go

```







```go

```







```go

```







```go

```







```go

```







