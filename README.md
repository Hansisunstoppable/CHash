# CHash
Developed a consistent hashing load balancing strategy, optimizing data allocation across cluster nodes and minimizing migration costs during node additions or removals.
- [简体中文](https://github.com/Hansisunstoppable/redis_lock/blob/main/README_ZH.md)
### Usage
```
go get github.com/Hansisunstoppable/CHash
```
### Achieve
- Load Balancing: Implements a consistent hash ring to optimize data allocation, ensuring balanced load across cluster nodes for stateful services.
- Minimal Migration: In node addition or removal scenarios, only partial data migration is required, significantly reducing data transfer costs during scaling.
- Virtual Nodes: Designs virtual node and weighted routing mechanisms to ensure uniform data distribution and support varying computational/storage capacities of nodes.
- Concurrent Migration: Implements a concurrent data migration method using Go goroutines and WaitGroup for unified scheduling of virtual node migrations, with panic recovery to enhance reliability and performance.
- Extensibility: Abstracts the hash ring and hash encoder as generic interfaces, allowing users to customize hash algorithms and routing strategies for greater flexibility.
- Multi-Scenario Support: Provides a single-machine hash ring (skip list implementation) for local cache sharding and a distributed hash ring (Redis ZSet implementation) for high-availability distributed systems.
- High Performance: Integrates the Murmur3 hash algorithm for efficient and uniform hash distribution, with support for user-defined hash implementations.
### Quick Start
#### Single-machine
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
		fmt.Println("Add node_a failed:", err)
		return
	}
	if err := consistentHash.AddNode(ctx, "node_b", 1); err != nil {
		fmt.Println("Add node_b failed:", err)
		return
	}

	// 查询数据归属
	dataKeys := []string{"data_a", "data_b", "data_c"}
	for _, key := range dataKeys {
		node, err := consistentHash.GetNode(ctx, key)
		if err != nil {
			fmt.Println("Get node failed:", err)
			return
		}
		fmt.Printf("Data %s belongs to node: %s\n", key, node)
	}

	// 添加新节点，触发数据迁移
	if err := consistentHash.AddNode(ctx, "node_c", 1); err != nil {
		fmt.Println("Add node_c failed:", err)
		return
	}

	// 再次查询数据归属，验证迁移效果
	for _, key := range dataKeys {
		node, err := consistentHash.GetNode(ctx, key)
		if err != nil {
			fmt.Println("Get node failed:", err)
			return
		}
		fmt.Printf("After adding node_c, data %s belongs to node: %s\n", key, node)
	}
}
```
#### Distributed
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
	redisClient := redis.NewClient("tcp", "your_redis_address", "your_redis_password")
	// 初始化 Redis 哈希环
	hashRing := redis.NewRedisHashRing("unique_hash_ring_id", redisClient)
	// 初始化一致性哈希模块
	consistentHash := consistent_hash.NewConsistentHash(
		hashRing,
		consistent_hash.NewMurmurHasher(),
		nil, // 可选：自定义迁移逻辑
	)

	ctx := context.Background()

	// 添加节点
	if err := consistentHash.AddNode(ctx, "node_a", 2); err != nil {
		fmt.Println("Add node_a failed:", err)
		return
	}
	if err := consistentHash.AddNode(ctx, "node_b", 1); err != nil {
		fmt.Println("Add node_b failed:", err)
		return
	}

	// 查询数据归属
	dataKeys := []string{"data_a", "data_b", "data_c"}
	for _, key := range dataKeys {
		node, err := consistentHash.GetNode(ctx, key)
		if err != nil {
			fmt.Println("Get node failed:", err)
			return
		}
		fmt.Printf("Data %s belongs to node: %s\n", key, node)
	}
}
```
