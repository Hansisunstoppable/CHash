## CHash
实现了一致性哈希负载均衡策略，优化集群节点的数据分配，并在节点增减时最小化迁移成本。
### 使用方法
```
go get github.com/Hansisunstoppable/CHash
```

### 功能特性
- 负载均衡：实现一致性哈希环，优化数据分配，确保集群节点间的负载均衡，适用于有状态服务。
- 最小化迁移：在节点增减场景下，仅需局部数据迁移，显著降低扩缩容过程中的数据迁移成本。
- 虚拟节点：设计虚拟节点和带权路由机制，确保数据分布均匀，支持节点计算/存储能力的差异化需求。
- 并发迁移：实现并发数据迁移方法，利用 Go 协程和 WaitGroup 统一调度虚拟节点迁移任务，结合 panic 恢复机制提升迁移可靠性和性能。
- 可扩展性：将哈希环和哈希编码器抽象为通用接口，允许用户自定义哈希算法和路由策略，增强模块灵活性。
- 多场景支持：提供单机哈希环（基于跳表实现）用于本地缓存分片，以及分布式哈希环（基于 Redis ZSet 实现）用于高可用分布式系统。
- 高性能：集成 Murmur3 哈希算法，确保高效且均匀的哈希分布，支持用户自定义哈希实现。

### 快速开始
#### 单机跳表实现
```go
package main

import (
	"context"
	"fmt"
	"github.com/Hansisunstoppable/CHash"
	"github.com/Hansisunstoppable/CHash/local"
)

func main() {
	// 初始化本地跳表哈希环
	hashRing := local.NewSkiplistHashRing()
	// 初始化一致性哈希模块
	consistentHash := consistent_hash.NewConsistentHash(
		hashRing,
		consistent_hash.NewMurmurHasher(),
		nil, // 可选：自定义迁移逻辑
		consistent_hash.WithReplicas(5), // 每个节点对应 5 个虚拟节点
		consistent_hash.WithLockExpireSeconds(5), // 锁超时时间 5 秒
	)

	ctx := context.Background()

	// 添加节点
	if err := consistentHash.AddNode(ctx, "node_a", 2); err != nil {
		fmt.Println("添加节点 node_a 失败:", err)
		return
	}
	if err := consistentHash.AddNode(ctx, "node_b", 1); err != nil {
		fmt.Println("添加节点 node_b 失败:", err)
		return
	}

	// 查询数据归属
	dataKeys := []string{"data_a", "data_b", "data_c"}
	for _, key := range dataKeys {
		node, err := consistentHash.GetNode(ctx, key)
		if err != nil {
			fmt.Println("查询节点失败:", err)
			return
		}
		fmt.Printf("数据 %s 归属于节点: %s\n", key, node)
	}

	// 添加新节点，触发数据迁移
	if err := consistentHash.AddNode(ctx, "node_c", 1); err != nil {
		fmt.Println("添加节点 node_c 失败:", err)
		return
	}

	// 再次查询数据归属，验证迁移效果
	for _, key := range dataKeys {
		node, err := consistentHash.GetNode(ctx, key)
		if err != nil {
			fmt.Println("查询节点失败:", err)
			return
		}
		fmt.Printf("添加节点 node_c 后，数据 %s 归属于节点: %s\n", key, node)
	}
}
```
#### Redis 实现
```go
package main

import (
	"context"
	"fmt"
	"github.com/Hansisunstoppable/CHash"
	"github.com/Hansisunstoppable/CHash/redis"
)

func main() {
	// 初始化 Redis 客户端
	redisClient := redis.NewClient("tcp", "你的_redis_地址", "你的_redis_密码")
	// 初始化 Redis 哈希环
	hashRing := redis.NewRedisHashRing("唯一哈希环_id", redisClient)
	// 初始化一致性哈希模块
	consistentHash := consistent_hash.NewConsistentHash(
		hashRing,
		consistent_hash.NewMurmurHasher(),
		nil, // 可选：自定义迁移逻辑
	)

	ctx := context.Background()

	// 添加节点
	if err := consistentHash.AddNode(ctx, "node_a", 2); err != nil {
		fmt.Println("添加节点 node_a 失败:", err)
		return
	}
	if err := consistentHash.AddNode(ctx, "node_b", 1); err != nil {
		fmt.Println("添加节点 node_b 失败:", err)
		return
	}

	// 查询数据归属
	dataKeys := []string{"data_a", "data_b", "data_c"}
	for _, key := range dataKeys {
		node, err := consistentHash.GetNode(ctx, key)
		if err != nil {
			fmt.Println("查询节点失败:", err)
			return
		}
		fmt.Printf("数据 %s 归属于节点: %s\n", key, node)
	}
}
```
