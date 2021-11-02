# 参数校验
gin框架使用<code>github.com/go-playground/validator</code>进行参数校验，只需要在定义结构体时使用<code>binding</code>的tag标识相关校验规则，就可以进行参数校验了。具体的用法以及常用的tag可查看官方文档<code>https://pkg.go.dev/github.com/go-playground/validator</code>

## 示例
```
// 定义结构体
type articleCreateInfo struct {
	Title string `form:"title" binding:"required,max=15"`
	Content string `form:"content" binding:"required"`
}
func Create(ctx *gin.Context) {
	var articleInfo articleCreateInfo
    // 使用ShouldBind获取参数并进行校验
	if err := ctx.ShouldBind(&articleInfo); err != nil {
		ctx.JSON(http.StatusForbidden, models.ResponseResult{Code: -1, Msg: err.Error()})
		return
	}
	article := db.Article{
		Title: articleInfo.Title,
		Content: articleInfo.Content,
	}
	db.GetDb().Create(&article)
	ctx.JSON(http.StatusOK, models.ResponseResult{})
}
```

## 参数获取
gin框架会根据请求头部的Content-Type以及对应的结构体的tag来获取参数，比如头部为<code>application/x-www-form-urlencoded</code>时，会根据tag里面的form字段获取对应的键名。常见的参数获取tag对应关系如下：
```
application/json - json
application/x-www-form-urlencoded - form
(restful规范的参数，比如/:id) - uri
```
绑定参数方法：
```
ctx.ShouldBind
ctx.ShouldBindUri
ctx.Bind（会抛错，并设置statusCode）
……
```

## 参数校验
在结构体中使用binding加对应的参数校验规则即可，常用的校验如下：
```
required    必填
max     最大值
min     最小值
len     长度
eq      等于
ne      不等于
oneof   其中一个
……
```
详细可查看官方文档<code>https://pkg.go.dev/github.com/go-playground/validator#section-documentation</code>

## 自定义参数校验
当默认的校验规则不符合业务时，可自定义参数校验方法，示例如下：
```
func main() {
	r := gin.Default()

	// 设置路由
	routers.SetRouters(r)

	// 注册自定义参数校验方法
	v, ok := binding.Validator.Engine().(*validator.Validate)
	if ok {
		// 自定义验证方法
		v.RegisterValidation("checkMobile", checkMobile)
	}

    // 示例
    type articleCreateInfo struct {
	Title string `form:"title" binding:"required,checkMobile"`
}
}

// 自定义参数校验方法，个人建议将该部分单独放在一个package中进行集中维护
func checkMobile(fl validator.FieldLevel) bool {
	mobileStr := fl.Field().String()
	mobileNum, _ := strconv.Atoi(mobileStr)
	if mobileNum > 10 {
		return true
	} else {
		return false
	}
}
```

## 自定义错误提示信息
（略）

# 参考文章
[学会gin参数校验之validator库，看这一篇就足够了](https://blog.csdn.net/qq_39397165/article/details/108173108)

[Gin框架系列04：趣谈参数绑定与校验 ](https://www.cnblogs.com/pingyeaa/p/12674589.html)