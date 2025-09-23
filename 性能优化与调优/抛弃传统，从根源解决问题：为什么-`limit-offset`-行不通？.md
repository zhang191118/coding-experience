### 抛弃传统，从根源解决问题：为什么 `LIMIT OFFSET` 行不通？

刚入行的同学，甚至一些有经验的开发者，实现分页时第一个想到的就是 SQL 的 `LIMIT` 和 `OFFSET`。

```sql
-- 查询第 101 页，每页 20 条患者记录
SELECT * FROM patient_records ORDER BY created_at DESC LIMIT 20 OFFSET 2000;
```

这个方案在数据量小的时候跑得很好。但在我们的场景下，当 `OFFSET` 达到几十万甚至上百万时，数据库的性能会急剧下降。原因很简单：数据库需要先找到并跳过 `OFFSET` 指定的 2000 条记录，然后才取那 20 条。这个“跳过”的动作，在底层其实是把数据都读出来再扔掉，`OFFSET` 越大，扔掉的数据越多，查询时间越长。

在我们一个早期的项目中，有个研究报告页面就用了这种方式。当一个中心的试验数据超过 50 万条后，查询第 200 页之后的报告，API 响应时间直接飙到 30 秒以上，导致前端频繁超时。这是生产事故，必须解决。

---

### 策略一：游标分页（Cursor/Keyset Pagination）—— 我们现在的标配方案

这是我们目前在绝大多数读多写多、需要连续浏览的场景下采用的方案。它的核心思想是：**不再告诉数据库“跳过多少条”，而是告诉它“从哪一条记录开始”**。

为了实现这个，我们需要一个能唯一且有序标识每条记录的“游标”。通常，我们会用自增 ID 或者一个高精度的时间戳字段。

#### 场景：加载患者填报的 ePRO 问卷列表

在我们的 ePRO 系统中，患者会持续提交健康问卷。我们需要一个无限滚动的列表来展示这些提交记录。

**实现思路：**
客户端第一次请求时不带任何游标参数，我们返回按时间倒序的第一页数据。同时，把这页数据里最后一条记录的 `ID` 和 `created_at`（创建时间）作为 `next_cursor` 返回给客户端。客户端加载下一页时，就把这个 `next_cursor` 带上。

**后端 Go (Gin 框架) 代码示例：**

```go
package main

import (
	"database/sql"
	"fmt"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	_ "github.com/go-sql-driver/mysql"
)

// PatientReport ePRO 记录
type PatientReport struct {
	ID        int64     `json:"id"`
	PatientID string    `json:"patient_id"`
	Content   string    `json:"content"`
	CreatedAt time.Time `json:"created_at"`
}

// DB 是一个模拟的数据库连接
var DB *sql.DB

// GetPatientReportsHandler 处理分页请求
func GetPatientReportsHandler(c *gin.Context) {
	limitStr := c.DefaultQuery("limit", "20")
	limit, _ := strconv.Atoi(limitStr)

	// lastID 是上一页的最后一条记录 ID，初次加载时为 0
	lastIDStr := c.DefaultQuery("last_id", "0")
	lastID, _ := strconv.ParseInt(lastIDStr, 10, 64)

	var rows *sql.Rows
	var err error

	// 构建查询
	// 关键点：WHERE id < ? (如果是倒序的话)
	// 我们总是取比上一页最后一条记录ID更小的数据
	if lastID == 0 {
		// 首页查询
		query := "SELECT id, patient_id, content, created_at FROM patient_reports ORDER BY id DESC LIMIT ?"
		rows, err = DB.Query(query, limit)
	} else {
		// 后续页查询
		query := "SELECT id, patient_id, content, created_at FROM patient_reports WHERE id < ? ORDER BY id DESC LIMIT ?"
		rows, err = DB.Query(query, lastID, limit)
	}

	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "Database query failed"})
		return
	}
	defer rows.Close()

	reports := make([]PatientReport, 0, limit)
	for rows.Next() {
		var r PatientReport
		if err := rows.Scan(&r.ID, &r.PatientID, &r.Content, &r.CreatedAt); err != nil {
			// 在生产环境中，这里应该记录日志
			continue
		}
		reports = append(reports, r)
	}

	var nextLastID int64 = 0
	if len(reports) > 0 {
		nextLastID = reports[len(reports)-1].ID
	}

	c.JSON(http.StatusOK, gin.H{
		"data":       reports,
		"next_last_id": nextLastID, // 返回给前端，用于下一次请求
	})
}


func main() {
    // 此处应为真实的数据库初始化
	// DB, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/database")
	// ...

	r := gin.Default()
	r.GET("/reports", GetPatientReportsHandler)
	r.Run(":8080")
}
```

**优点：**
*   **性能稳定：** 无论翻到第几页，查询速度几乎不变。因为 `WHERE id < ?` 可以高效地利用主键索引。
*   **数据一致性好：** 在高频写入的场景下，即使有新数据插入，也不会导致分页数据重复或遗漏，因为它总是从一个固定的“锚点”向后查找。

**缺点：**
*   **无法直接跳页：** 不能直接跳转到“第 50 页”，只能一页一页地向后翻。但在我们的无限滚动、加载更多的场景下，这完全不是问题。

---

### 策略二：时间范围分片 + 游标 —— 对付海量日志和时序数据

在我们的“临床研究智能监测系统”中，我们需要处理海量的操作日志和系统事件流。这些数据有强烈的时序性，而且查询通常也带着时间范围。

直接在单一巨型日志表上用游标分页也会遇到问题，比如索引会变得非常大，维护成本高。我们的做法是**在数据库层面进行分区（Partitioning）**。

以 PostgreSQL 为例，我们可以按月创建分区表：

```sql
-- 主表
CREATE TABLE system_logs (
    id BIGSERIAL,
    event_type VARCHAR(50),
    log_time TIMESTAMPTZ NOT NULL,
    details JSONB
) PARTITION BY RANGE (log_time);

-- 2024 年 10 月的分区
CREATE TABLE system_logs_2024_10 PARTITION OF system_logs
    FOR VALUES FROM ('2024-10-01 00:00:00+00') TO ('2024-11-01 00:00:00+00');
```

**查询流程：**
当用户查询一个时间段的日志时，数据库的查询优化器会自动定位到对应的分区表，而不是扫描整个主表。比如查询 `2024-10-15` 到 `2024-10-20` 的数据，只会扫描 `system_logs_2024_10` 这个分区。

在这个基础上，我们再结合游标分页，性能就非常理想了。查询语句会变成这样：

```sql
SELECT id, event_type, log_time, details FROM system_logs
WHERE
    log_time >= '2024-10-15T00:00:00Z' AND log_time < '2024-10-21T00:00:00Z'
    AND (log_time, id) < ('2024-10-18T10:30:00Z', 1234567) -- (上一页的最后一条记录的 log_time 和 id)
ORDER BY log_time DESC, id DESC
LIMIT 100;
```
**关键点：** 这里我们用了复合游标 `(log_time, id)`，因为 `log_time` 可能不唯一。这种方式保证了排序的绝对稳定。

---

### 策略三：并行查询与数据聚合 —— 微服务架构下的分页挑战

我们的很多新系统，比如“临床试验机构项目管理系统”，都是基于微服务架构。数据被分散在不同的服务中，比如 `project-service` 管理项目信息，`site-service` 管理机构（医院）信息。

**场景：** 在一个页面上，需要展示一个集团下所有医院（site）的正在进行的项目列表，并支持分页。

这意味着我们需要从多个 `site-service` 实例（或者一个 `site-service` 访问按 `site_id` 分片的数据库）中拉取数据，然后聚合成一个统一的列表。

**实现思路（The Scatter-Gather Pattern）：**
1.  **请求分发 (Scatter):** API 网关收到分页请求（比如第 2 页，每页 10 条）。它不能简单地让每个 `site-service` 都返回第 2 页的数据，因为我们不知道数据在各个服务中的分布情况。正确的做法是，为了获取最终的 10 条数据，网关需要向每个 `site-service` 请求更多的数据，比如请求 `page * size = 2 * 10 = 20` 条数据。
2.  **结果聚合 (Gather):** 网关并发地向所有相关的 `site-service` 发出请求。
3.  **内存排序与截取:** 等所有服务都返回数据后，在网关层将所有结果合并，进行全局排序，然后根据分页参数 `(page-1)*size` 和 `size` 在内存中进行最终的截取，返回给客户端。

**后端 Go (go-zero 框架) 逻辑示例：**

```go
// 在 API 网关服务的 logic 文件中

type (
    // 假设这是下游 site-service 的客户端
	SiteServiceClient interface {
		GetProjects(ctx context.Context, in *site.GetProjectsReq) (*site.GetProjectsResp, error)
	}
)

func (l *ListProjectsLogic) ListProjects(req *types.ListProjectsReq) (*types.ListProjectsResp, error) {
	// 假设我们有3个下游服务/分片
	siteServices := []string{"site1", "site2", "site3"}
	
	var wg sync.WaitGroup
	resultsChan := make(chan []*types.Project, len(siteServices))
	errChan := make(chan error, len(siteServices))

	// 为了获取第 page 页，每页 size 条数据，我们需要从每个分片中获取 page*size 条
	// 这是一个简化的策略，更精确的策略需要更复杂的计算
	fetchLimit := req.Page * req.PageSize

	for _, serviceName := range siteServices {
		wg.Add(1)
		go func(name string) {
			defer wg.Done()
			
			// 使用 go-zero 的 rpc 客户端调用下游服务
			// client := l.svcCtx.SiteRpcClients[name] 
			// resp, err := client.GetProjects(l.ctx, &site.GetProjectsReq{Limit: fetchLimit})
			
			// --- 以下是模拟调用 ---
			// 模拟调用，实际项目中这里是 RPC 调用
			fmt.Printf("Querying service %s for %d projects\n", name, fetchLimit)
			time.Sleep(100 * time.Millisecond) // 模拟网络延迟
			// 模拟返回数据
			projects := make([]*types.Project, 0)
			// ... 填充模拟数据
			// --- 模拟结束 ---
			
			if err != nil {
				errChan <- err
				return
			}
			resultsChan <- projects // resp.Projects
		}(serviceName)
	}

	wg.Wait()
	close(resultsChan)
    close(errChan)

	// 检查是否有错误发生
	if len(errChan) > 0 {
		return nil, <-errChan // 返回第一个遇到的错误
	}

	// 合并所有结果
	allProjects := make([]*types.Project, 0)
	for projects := range resultsChan {
		allProjects = append(allProjects, projects...)
	}

	// 在内存中进行全局排序
	sort.Slice(allProjects, func(i, j int) bool {
		return allProjects[i].CreatedAt.After(allProjects[j].CreatedAt)
	})

	// 在内存中进行分页
	start := (req.Page - 1) * req.PageSize
	end := start + req.PageSize
	if start > len(allProjects) {
		return &types.ListProjectsResp{Projects: []*types.Project{}, Total: int64(len(allProjects))}, nil
	}
	if end > len(allProjects) {
		end = len(allProjects)
	}

	finalProjects := allProjects[start:end]

	return &types.ListProjectsResp{Projects: finalProjects, Total: int64(len(allProjects))}, nil
}
```

**这种方式的权衡：**
*   **优点：** 解决了分布式数据的分页问题，对客户端透明。
*   **缺点：**
    *   **放大查询：** 向下游请求了比最终需要多得多的数据，增加了网络开销和下游服务的压力。
    *   **越往后翻页，性能越差：** 与 `LIMIT OFFSET` 问题类似，翻的页数越深，需要从下游拉取和在内存中排序的数据就越多。
    *   **不适合海量数据：** 如果聚合后的数据量太大，网关会成为内存瓶颈。

所以，这个方案只适用于**分片不多、总数据量可控、且深度分页需求较少**的场景。

---

### 总结与选型建议

在我们的临床医疗IT实践中，没有一种分页方案是万能的。我们需要根据具体的业务场景、数据特点和架构来选择。

| 策略 | 最适用场景 | 关键挑战 | 我们在哪个系统用？ |
| :--- | :--- | :--- | :--- |
| **游标分页** | 无限滚动列表、实时数据流、高频写入场景 | 不支持随机跳页 | ePRO 系统、患者日记、EDC 数据录入列表 |
| **时间范围分区** | 海量日志、时序数据查询与归档 | 数据库分区规则的维护 | 智能监测系统日志、系统操作审计追踪 |
| **并行聚合** | 微服务架构下跨服务/分片的数据列表 | 越深分页性能越差，网关内存瓶颈 | 临床试验机构项目管理平台（查询前几页） |

作为架构师，我的建议是：
*   **首选游标分页**。它能从根本上解决深度分页的性能问题，是构建高性能、可扩展系统的基石。
*   对于时序数据，**不要犹豫，上分区表**。这是数据库层面最高效的优化。
*   谨慎使用并行聚合分页。清晰地认识到它的局限性，**只用于浅分页场景**，并做好监控，防止网关被打爆。

希望我这些在一线项目中总结的经验，能帮助你更好地应对大数据分页的挑战。