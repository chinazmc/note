---
sr-due: 2022-10-18
sr-interval: 4
sr-ease: 270
---

#限流  #note 


之前文章写了GOLANG和PHP实现商品秒杀功能，最近疫情大家都在家里忙着抢菜。但是很多人反馈，卖菜APP能够加入购物车，但是下单时却出了意外。这种情况基本就是平台做了限流处理。

今天就为大家分享和演示go 、redis和lua脚本实现的一种简单限流方法：滑动窗口，直接上代码
```java
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"github.com/go-redis/redis"
	"net/http"
	"time"
)

var redisDb *redis.Client
var succSort int = 0

var luaScript = `

	local key         = KEYS[1];-- 键 key
	local now_time    = ARGV[1];-- 值 value
	local before_time = ARGV[2];-- 间隔时间之前，纳秒
	local period      = ARGV[3];-- 时间间隔，多少秒内，设置过期时间
	local requests    = ARGV[4];-- 时间间隔内的请求次数

	redis.pcall("zadd", key, now_time, now_time);-- zset结构设置一个key，zadd(key,value,scores)
	redis.pcall("zremrangebyscore", key, 0, before_time); -- 移除scores范围在0到bofore_time之间的值，即移除时间窗口之前的行为记录，剩下的都是时间窗口内的
	local count = redis.pcall("zcard", key); -- 获取窗口内的请求数量
	redis.pcall("expire", key, period); -- 设置 zset 过期时间，避免不活跃用户持续占用内存

	if tonumber(count) > tonumber(requests) then
		return 2;
	end
	return 1;`

var evalSha string

func init() {
	initRedisClient()
}

func initRedisClient() {

	redisDb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "",
		DB:       0,
	})

	var err error
	evalSha, err = redisDb.ScriptLoad(luaScript).Result()
	if err != nil {
		panic(err)
	}
}

func main() {
	// 1.创建路由
	r := gin.Default()
	// 2.绑定路由规则，执行的函数
	// gin.Context，封装了request和response
	r.GET("/count", func(c *gin.Context) {
		doRequest()
		c.String(http.StatusOK, "ok")
	})

	// 3.监听端口，默认在8080
	// Run("里面不指定端口号默认为8080")
	r.Run(":8000")

}

func doRequest() {
	success := isPermited("oossnaa", "add/Cart", 3, 10)
	if success {
		succSort++
		fmt.Println("成功", succSort) //处理下单逻辑
	} else {
		fmt.Println("失败") // 返回请求或者抛出异常、panic
	}
}

func isPermited(uid string, action string, period, maxCount int) bool {
	key := fmt.Sprintf("%v_%v", uid, action)
	now := time.Now().UnixNano() //纳秒
	beforeTime := now - int64(period*1000000000)
	res, err := redisDb.EvalSha(evalSha, []string{key}, now, beforeTime, period, maxCount).Result()
	if err != nil {
		panic(err)
	}
	if res.(int64) == int64(2) {
		return false
	}
	return true
}
```
使用了gin框架，现在AB工具来压测下，实际也可以在代码中for循环大量请求来测试


ab -n 20 -t 3 "http://localhost:8000/count"

本次案例需求是3秒的时间窗口一个用户的请求数不超过10个，所以AB 命令加上 -t 3

-t的注释是：测试所进行的最大秒数。其内部隐含值是-n 50000。它可以使对服务器的测试限制在一个固定的总时间以内。默认时，没有时间限制。

我对-n 20 -t 3 的理解是，3秒内执行20个请求，但是实际的结果是3秒内有1000多个请求，这个有点不太明白，还需要各位大佬指点。

但是并不影响本次案例的需要背景，便于查看我将成功的请求加上序号，最后看结果：


只有十个请求成功了，其余的失败了。

回顾前期文章

go+redis+lua实现秒杀

php+redis+lua实现秒杀

后续分享和演示限流的另外两个方法：令牌桶和漏斗算法 作者：沪猿小韩 https://www.bilibili.com/read/cv15928462/ 出处：bilibili