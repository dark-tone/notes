# 中间件
## 定义中间件
```
package middlewares

func CustomMiddleWare(ctx *gin.Context) {
	ctx.Set("customKey", "gt")
	start := time.Now()
	fmt.Println("start time:" + start.String())

    // 执行往后的方法，类似于nodejs的洋葱圈模型
	ctx.Next()

	end := time.Now()
	total := time.Since(start)
	fmt.Println("end time:" + end.String())
	fmt.Println("total time:" + total.String())
}
```

## 使用中间件
**路由级中间件**直接加在参数后面即可：
```
// 可以加在路由组内，也可加在指定的一条路由内
articleGroup := e.Group("/article")
articleGroup.GET("/:id", middlewares.CustomMiddleWare, article.Get)
```
**全局中间件**使用Use方法：
```
r := gin.New()
r.Use(CustomMiddleWare)
```

## 执行顺序
全局中间件 -> 路由组中间件 -> 路由中间件

## 示例
以上述定义的中间件为例，获取文章的路由加上middlewares.CustomMiddleWare，以及控制器代码打印customKey
```
func Get(ctx *gin.Context) {
	……
	var ck, _ = ctx.Get("customKey")
	fmt.Println("customKey is " + ck.(string))
	……
}
```
执行情况如下：
```
start time:2021-11-02 07:59:12.9443586 +0800 CST m=+4.166997401
customKey is gt
end time:2021-11-02 07:59:12.9922363 +0800 CST m=+4.214875101
total time:47.8777ms
```