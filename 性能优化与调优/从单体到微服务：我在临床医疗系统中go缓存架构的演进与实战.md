
## 一、 起点：一个 `map` 和一把锁，解决燃眉之急

项目初期，我们的“机构项目管理系统”还是一个单体应用，基于 Gin 框架开发。很快，我们发现一个性能问题：获取临床试验项目关联的“研究中心列表”接口响应很慢。这个数据有个特点：**读多写少**。一个试验项目一旦启动，其关联的研究中心（医院）在很长一段时间内是固定的。每次都去数据库 `JOIN` 查询，显然是巨大的浪费。

最直接的解决方案，就是在内存里加个缓存。

### 1.1 基础实现：`map` + `sync.RWMutex`

这是最经典、最有效的本地缓存实现。我们选择 `sync.RWMutex`（读写锁）而不是 `sync.Mutex`（互斥锁），原因在于“研究中心列表”的读取频率远高于更新频率。读写锁允许多个 Goroutine 同时读，性能比互斥锁好得多。

```go
package main

import (
	"fmt"
	"log"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
)

// Site 代表一个研究中心
type Site struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

// siteCache 线程安全的本地缓存
var siteCache = struct {
	sync.RWMutex
	data map[string][]Site
}{
	data: make(map[string][]Site),
}

// getSitesFromDB 模拟从数据库查询
func getSitesFromDB(trialID string) ([]Site, error) {
	log.Printf("数据库查询：开始为试验项目 %s 查询研究中心...", trialID)
	time.Sleep(200 * time.Millisecond) // 模拟DB延时
	// 在实际项目中，这里是 gorm 或 sqlx 的查询逻辑
	if trialID == "PRO-001" {
		return []Site{
			{ID: 101, Name: "北京协和医院"},
			{ID: 102, Name: "上海瑞金医院"},
		}, nil
	}
	return nil, fmt.Errorf("未找到试验项目 %s 的研究中心", trialID)
}

func GetTrialSites(c *gin.Context) {
	trialID := c.Param("trialId")

	// 1. 尝试从缓存中读取（使用读锁）
	siteCache.RLock()
	sites, found := siteCache.data[trialID]
	siteCache.RUnlock()

	if found {
		log.Printf("缓存命中：成功获取试验项目 %s 的研究中心列表", trialID)
		c.JSON(200, sites)
		return
	}

	// 2. 缓存未命中，从数据库查询
	log.Printf("缓存未命中：准备为试验项目 %s 查询数据库", trialID)
	sites, err := getSitesFromDB(trialID)
	if err != nil {
		c.JSON(404, gin.H{"error": err.Error()})
		return
	}

	// 3. 将结果写入缓存（使用写锁）
	siteCache.Lock()
	siteCache.data[trialID] = sites
	siteCache.Unlock()
	
	log.Printf("数据回填：已将试验项目 %s 的研究中心列表写入缓存", trialID)
	c.JSON(200, sites)
}

func main() {
	r := gin.Default()
	r.GET("/trials/:trialId/sites", GetTrialSites)
	r.Run(":8080")
}
```

这个简单的改进，让接口的平均响应时间从 250ms 降低到了 10ms 以内，效果立竿见影。

## 二、 规模化后的阵痛：缓存数据一致性与高并发下的“惊群”

随着系统功能越来越复杂，用户量激增，简单的本地缓存开始暴露出新问题。

### 2.1 缓存数据何时失效？—— TTL 过期机制

研究中心信息虽然不常变，但并非一成不变，比如某个中心退出了项目。如果缓存永不过期，就会导致数据不一致。我们需要引入 **TTL（Time-To-Live）**。

我们改造了缓存结构，增加了过期时间和后台清理的 Goroutine。

```go
type CacheItem struct {
	Value      []Site
	Expiration int64 // Unix时间戳（秒）
}

var siteCacheWithTTL = struct {
	sync.RWMutex
	data map[string]CacheItem
}{
	data: make(map[string]CacheItem),
}

// Set 设置缓存，并指定过期时间
func setWithTTL(key string, value []Site, ttl time.Duration) {
	siteCacheWithTTL.Lock()
	defer siteCacheWithTTL.Unlock()
	siteCacheWithTTL.data[key] = CacheItem{
		Value:      value,
		Expiration: time.Now().Add(ttl).Unix(),
	}
}

// Get 读取缓存，并检查是否过期
func getWithTTL(key string) ([]Site, bool) {
	siteCacheWithTTL.RLock()
	defer siteCacheWithTTL.RUnlock()
	item, found := siteCacheWithTTL.data[key]
	if !found {
		return nil, false
	}
	// 惰性删除：访问时发现过期，则认为不存在
	if time.Now().Unix() > item.Expiration {
		return nil, false
	}
	return item.Value, true
}
```

同时，我们启动一个后台 Goroutine 定期清理已过期的键，避免内存无限增长。

### 2.2 热点Key失效的噩梦：缓存击穿与雪崩

在我们的 ePRO 系统中，患者需要填写“生活质量量表”，这个量表的定义数据就是典型的热点 Key。当这个 Key 的缓存在某个瞬间过期时，成百上千个正在填写的患者请求会同时穿透缓存，直接打到数据库，这就是**缓存击穿**。

我们的解决方案是使用 `singleflight` 库来确保对于同一个 Key，在缓存失效的瞬间，只有一个请求会去访问数据库，其他请求则等待这个请求的结果。

```go
import "golang.org/x/sync/singleflight"

var g singleflight.Group

func GetQuestionnaire(id string) (data []byte, err error) {
    // ... 尝试读缓存 ...

    // 缓存未命中，使用singleflight
    v, err, _ := g.Do(id, func() (interface{}, error) {
        // 这个函数体在同一时间只会被一个goroutine执行
        return getQuestionnaireFromDB(id)
    })
    
    if err != nil {
        return nil, err
    }
    
    // ... 结果写入缓存 ...
    
    return v.([]byte), nil
}
```

而**缓存雪崩**则是大量 Key 在同一时间集体失效。我们的预防措施很简单，但非常有效：在设置 TTL 时，增加一个随机的“抖动”时间。比如基础过期时间是 10 分钟，我们实际设置的是 `10分钟 + (0~60秒内的一个随机值)`，这样就把过期时间点分散开了。

## 三、 微服务转型：拥抱 Redis 实现分布式缓存

当公司业务扩展，系统被拆分为多个微服务后，本地缓存的局限性就彻底暴露了：每个服务实例都维护自己的缓存，不仅浪费内存，更导致了数据不一致。比如，“项目管理服务”更新了研究中心信息，但“数据看板服务”的本地缓存还是旧的。

这时，引入像 Redis 这样的分布式缓存就成了必然选择。我们团队全面转向了 `go-zero` 框架，它对 Redis 的集成非常友好。

下面是我们 `patient-service` 中获取患者基本信息的 `logic` 层代码，它完美诠释了**缓存旁路（Cache-Aside）**模式。

**`patient.api` 定义:**
```api
type (
	PatientInfoReq {
		PatientId int64 `path:"patientId"`
	}

	PatientInfoReply {
		Id   int64  `json:"id"`
		Name string `json:"name"`
		Dob  string `json:"dob"` // Date of Birth
	}
)

service patient-api {
	@handler GetPatientInfo
	get /patients/:patientId (PatientInfoReq) returns (PatientInfoReply)
}
```

**`getpatientinfologic.go` 实现:**
```go
package logic

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"patient-service/internal/svc"
	"patient-service/internal/types"
	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/stores/sqlc"
)

// ...

func (l *GetPatientInfoLogic) GetPatientInfo(req *types.PatientInfoReq) (resp *types.PatientInfoReply, err error) {
	// 1. 定义缓存 Key
	cacheKey := fmt.Sprintf("cache:patient:%d", req.PatientId)

	// 2. 尝试从 Redis 获取数据
	val, err := l.svcCtx.RedisClient.Get(cacheKey)
	if err == nil && val != "" {
		logx.Infof("缓存命中: patientId %d", req.PatientId)
		var patient types.PatientInfoReply
		if err = json.Unmarshal([]byte(val), &patient); err == nil {
			return &patient, nil
		}
	}
	// Redis查询出错或值为空，都继续执行数据库查询

	// 3. 缓存未命中，查询数据库
	// l.svcCtx.PatientModel 是 go-zero 自动生成的 model
	dbPatient, err := l.svcCtx.PatientModel.FindOne(l.ctx, req.PatientId)
	if err != nil {
		if err == sqlc.ErrNotFound {
			// 重点：防止缓存穿透，对空结果也进行缓存，但TTL设置得短一些
			l.svcCtx.RedisClient.Setex(cacheKey, "null", 60) // 缓存空值1分钟
			return nil, fmt.Errorf("patient with id %d not found", req.PatientId)
		}
		logx.Errorf("数据库查询失败: patientId %d, error: %v", req.PatientId, err)
		return nil, err
	}

	// 4. 数据回填至 Redis
	resp = &types.PatientInfoReply{
		Id:   dbPatient.Id,
		Name: dbPatient.Name,
		Dob:  dbPatient.Dob.Format("2006-01-02"),
	}
	
	jsonData, jsonErr := json.Marshal(resp)
	if jsonErr == nil {
		// 基础TTL 1小时，加上随机抖动
		ttl := time.Hour + time.Duration(rand.Intn(300)) * time.Second
		l.svcCtx.RedisClient.Setex(cacheKey, string(jsonData), int(ttl.Seconds()))
		logx.Infof("数据回填: patientId %d", req.PatientId)
	}

	return resp, nil
}
```

### 3.1 数据一致性关键决策：先更新数据库，再删除缓存

这是一个面试高频题，也是我们实践中血的教训。当患者信息更新时，我们应该怎么做？

-   **方案A：** 先更新数据库，再更新缓存。**（不推荐）**
-   **方案B：** 先更新缓存，再更新数据库。**（不推荐）**
-   **方案C：** 先更新数据库，再删除缓存。**（我们采用的方案）**
-   **方案D：** 先删除缓存，再更新数据库。**（有风险）**

我们选择 **方案C**。为什么是删除而不是更新？想象一下并发场景：
1.  线程 A 更新了患者地址，然后去更新缓存。
2.  同时，线程 B 也更新了该患者地址，并成功更新了数据库和缓存。
3.  这时，线程 A 的网络延迟导致它的缓存更新操作后到，它会用一个**旧值**覆盖掉线程 B 写入的新值，造成数据不一致。

而删除缓存则不会有这个问题。下次读取时，发现缓存不存在，自然会去数据库加载最新值。这是一种懒加载思想，保证了最终一致性。

## 四、 终极优化：多级缓存应对极端流量

对于像“智能开放平台”这种对外提供 API 的高流量服务，单靠 Redis 有时也会遇到瓶颈（网络 IO、Redis 带宽）。我们借鉴了 CPU 缓存的设计，引入了**多级缓存**架构：**本地内存（L1） + Redis（L2）**。

-   **请求流程**：请求先查 L1 缓存，没有则查 L2 (Redis)，还没有才去查数据库。
-   **数据同步**：从数据库查到数据后，会同时写入 L2 和 L1。
-   **一致性保障**：当数据更新时，我们通过消息队列（如 Kafka）发布一个“数据变更”消息，所有订阅了该消息的服务实例都会去**删除**自己的 L1 缓存。

这个方案复杂度较高，只适用于对性能要求极为苛刻的核心服务。对大部分业务场景来说，一个设计良好的 Redis 缓存层已经足够。

## 总结

回顾我们的缓存架构演进之路，其实就是一部不断解决问题的历史。从最初的 `map+RWMutex`，到考虑 TTL 和并发问题的 `singleflight`，再到微服务时代的 Redis 分布式缓存，以及最终的多级缓存，每一步都是为了让我们的临床医疗系统更快、更稳定。

我的几点核心体会：
1.  **没有银弹，按需选择**：不要一开始就上 Redis，简单的本地缓存能解决问题时，它是最快的。
2.  **缓存的核心是“失效”**：如何设计缓存的淘汰和更新策略，远比选择用什么工具重要。
3.  **防御性编程**：永远要考虑缓存击穿、雪崩等异常情况，为你的系统加上保险。
4.  **一致性与性能的权衡**：在分布式系统中，追求强一致性往往会牺牲性能。想清楚你的业务场景能否接受最终一致性。

希望我的这些一线经验，能对正在构建 Go 系统的你有所启发。缓存设计是门艺术，更是门工程学，需要在实践中不断打磨。