# CHash
## 使用
- 构造 redis 客户端实例<br/><br/>
```go
import "github.com/xiaoxuxiansheng/redmq/redis"
func main(){
    redisClient := redis.NewClient("tcp","my_address","my_password")
    // ...
}
```
