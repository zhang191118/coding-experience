今天，我想把这些年在项目中踩过的坑、总结的经验，梳理成一份 Go 项目的 Code Review 实战清单。这不只是一套规则，更是我们团队协作的共识和保障系统稳定运行的“压舱石”。希望能帮助正在使用 Go 的你，无论你是刚入门还是有几年经验，都能有所收获。

---

### **第一章：地基篇 - 代码结构与可维护性审查**

一个项目的结构，就像一座医院的建筑蓝图。蓝图清晰，各个科室（模块）才能高效协作；蓝图混乱，未来要加个新科室（功能）就得“伤筋动骨”。

#### **1.1 包（Package）的设计：是“科室分离”而非“大杂烩”**

刚接触项目的同学很容易把代码都堆在一个 `main` 包或者几个模糊的包里。随着我们业务越来越复杂，比如“临床试验项目管理系统”（CTMS）和“电子患者自报告结局系统”（ePRO）需要数据联动，混乱的包结构会变成一场灾难。

**审查要点：**

*   **高内聚，低耦合：** 每个包都应该有一个明确的“职责”。比如，`trial` 包只负责临床试验的核心业务逻辑，`patient` 包只管患者信息，`common` 包存放全局通用的工具类（如统一的日志、错误码定义）。
*   **拒绝循环依赖：** 这是 Go 编译器的强制要求，但在设计层面就要避免。如果 `trial` 依赖 `patient`，那么 `patient` 绝不能反过来依赖 `trial`。这通常意味着你的抽象层次出了问题。
*   **清晰的目录结构：** 我们团队全面采用 `go-zero` 框架，它推荐的 `api`, `internal` 结构就是很好的实践。

**实战场景：**

在我们的“临床试验电子数据采集系统”中，有一个模块负责处理患者通过 App 提交的问卷（ePRO 数据）。我们是这样划分的：

```plaintext
ePRO-service/
├── api/                  # .api 文件，定义了 HTTP 接口
│   └── epro.api
├── internal/
│   ├── config/           # 配置
│   │   └── config.go
│   ├── handler/          # HTTP Handler 层，负责请求的接收和校验
│   │   └── submithandler.go
│   ├── logic/            # 核心业务逻辑层
│   │   └── submitlogic.go
│   ├── svc/              # ServiceContext，服务依赖的上下文
│   │   └── servicecontext.go
│   └── types/            # 请求和响应的结构体定义
│       └── types.go
└── epro.go               # main 函数入口
```

**Review 时我会问：**

> “`submitlogic.go` 里的逻辑，是不是都和‘提交问卷’这个单一动作相关？如果里面混入了查询患者基本信息的逻辑，那这部分是不是应该通过 RPC 调用 `patient-service` 来实现，而不是在这里直接操作患者的数据库表？”

这能确保每个微服务的职责边界清晰，便于独立迭代和维护。

#### **1.2 接口（Interface）的抽象：定义“契约”而非“实现”**

Go 的接口是隐式实现的，这非常强大。一个好的接口应该只定义“需要做什么”，而不是“应该怎么做”。

**审查要点：**

*   **小接口原则：** 接口应该小而专注。`io.Reader` 就是典范，它只有一个 `Read` 方法。我们应该避免定义一个包含十几个方法的“上帝接口”。
*   **面向消费方定义：** 接口应该由使用它的一方来定义，而不是实现方。这能更好地满足消费方的需求。

**实战场景：**

我们的“临床研究智能监测系统”需要从不同来源采集数据，比如医院的 HIS 系统、患者穿戴设备、实验室数据等。每种数据源的接入方式都不同。

**反面教材：** 定义一个大而全的 `DataService` 接口。

**正确做法：** 定义一个最小化的采集器接口。

```go
package collector

// DataCollector 定义了数据采集器的通用行为。
// 任何数据源只要实现了这个接口，就能被我们的系统集成。
type DataCollector interface {
    // Collect 方法负责从特定数据源采集数据，并返回标准格式的试验数据切片。
    // ctx 用于控制采集的超时和取消。
    Collect(ctx context.Context, trialID string) ([]TrialData, error)
}

// TrialData 是我们系统内部流转的标准数据结构。
type TrialData struct {
    PatientID string
    DataType  string // e.g., "BloodPressure", "HeartRate"
    Value     json.RawMessage
    Timestamp time.Time
}
```

这样，无论是接入一个新的蓝牙血压计，还是对接一套新的医院信息系统，我们只需要写一个新的 `struct` 并实现 `Collect` 方法即可，系统核心逻辑完全不用动。

#### **1.3 错误处理：是“传递线索”而非“丢弃现场”**

在医疗系统中，一个被忽略的 `error` 可能会导致数据错漏，后果不堪设想。Go 的 `error` 返回值设计，就是在强制我们正视每一个可能出错的地方。

**审查要点：**

*   **绝不丢弃 error：** 永远不要使用 `_` 来忽略一个 `error`，除非你 100% 确定它永远不会发生（比如 `bytes.Buffer` 的 `Write` 方法）。
*   **错误包装（Wrapping）：** 从 Go 1.13 开始，使用 `fmt.Errorf` 配合 `%w` 动词来包装错误。这能形成一个错误链，方便我们追根溯源。
*   **定义业务错误码：** 对于 API 服务，需要给前端返回明确的业务状态码，而不是笼统的 `500 Internal Server Error`。

**实战场景：**

在 `go-zero` 项目中，我们会定义统一的业务错误码，并通过 `xerr` 包来处理。

**1. 定义业务错误码 (`internal/errorx/bizerror.go`)**

```go
package errorx

import "github.com/zeromicro/x/errors"

// 定义业务相关的错误码
var (
    ErrPatientNotFound = errors.New(1001, "患者信息不存在")
    ErrTrialDataInvalid = errors.New(1002, "临床试验数据格式无效")
    // ... 更多业务错误
)
```

**2. 在 Logic 层使用 (`internal/logic/submitlogic.go`)**

```go
package logic

import (
    "context"
    "your-project/internal/errorx" // 引入自定义错误包
    // ...
)

func (l *SubmitLogic) Submit(req *types.SubmitRequest) (*types.SubmitResponse, error) {
    // 1. 检查患者是否存在
    patient, err := l.svcCtx.PatientModel.FindOne(l.ctx, req.PatientID)
    if err != nil {
        if errors.Is(err, model.ErrNotFound) {
            // 关键：将数据库层的错误转换为清晰的业务错误
            return nil, errorx.ErrPatientNotFound
        }
        // 对于其他未知数据库错误，包装并记录日志
        return nil, errors.Wrapf(err, "数据库查询患者失败, patientID: %s", req.PatientID)
    }

    // 2. 校验数据格式
    if !isValid(req.Data) {
        return nil, errorx.ErrTrialDataInvalid
    }

    // ... 业务逻辑 ...

    return &types.SubmitResponse{Success: true}, nil
}
```

`go-zero` 的 `httpx` 中间件会自动捕获这些 `errors.New` 创建的业务错误，并将其转换为对应的 HTTP 状态码和 JSON 响应体，前端就能根据 `code` 字段（如 1001）做相应的处理（比如提示“患者信息不存在”）。对于被 `errors.Wrapf` 包装的系统级错误，则会返回 500 错误并记录详细日志，保护了系统内部细节不被暴露。

---

### **第二章：并发篇 - Goroutine 与 Channel 的“交通规则”**

Go 的并发能力是其核心优势，但也是最容易出问题的地方。“智能开放平台”经常需要并发处理来自多个第三方系统的 API 请求，如果并发控制不当，很容易导致服务雪崩。

#### **2.1 Goroutine 的生命周期管理：谁创建，谁负责**

一个常见的错误是启动了一个 Goroutine 后就“撒手不管”了，这被称为“Goroutine 泄漏”。当泄漏的 Goroutine 越来越多，会耗尽内存和 CPU 资源。

**审查要点：**

*   **明确的退出机制：** 任何长时间运行的 Goroutine 都必须有明确的退出机制。`context.Context` 是实现这一点的最佳工具。
*   **使用 `sync.WaitGroup` 等待 Goroutine 结束：** 如果主协程需要等待一组子协程完成工作，必须使用 `WaitGroup`。

**实战场景：**

我们的“学术推广平台”有一个功能，需要向订阅了某场学术会议的用户批量发送短信提醒。我们会为每个发送任务启动一个 Goroutine。

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// sendSMS 模拟发送短信，可能会耗时较长
func sendSMS(ctx context.Context, wg *sync.WaitGroup, phone string) {
	defer wg.Done()

	fmt.Printf("准备向 %s 发送短信...\n", phone)
	select {
	case <-time.After(2 * time.Second): // 模拟耗时操作
		fmt.Printf("成功向 %s 发送短信\n", phone)
	case <-ctx.Done(): // 接收取消信号
		// ctx.Done() 返回一个 channel，当 context 被取消或超时，这个 channel 会被关闭
		// 从一个关闭的 channel 读数据会立即返回零值，所以 select 会选中这个 case
		fmt.Printf("任务取消，停止向 %s 发送短信, 原因: %v\n", phone, ctx.Err())
	}
}

func main() {
	// 创建一个可取消的 context，并设置 3 秒后超时
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	// defer cancel() 是必须的，确保在函数退出时释放与 context 相关的资源
	defer cancel()

	var wg sync.WaitGroup
	phones := []string{"13800138000", "13900139000", "15000150000", "15100151000"}

	for _, phone := range phones {
		wg.Add(1)
		go sendSMS(ctx, &wg, phone)
	}

	// 等待所有 Goroutine 完成
	wg.Wait()
	fmt.Println("所有短信发送任务结束。")
}
```

**Review 时我会检查：**

1.  `go` 关键字后面调用的函数，第一个参数是不是 `context.Context`？
2.  有没有使用 `select` 语句监听 `ctx.Done()`？
3.  启动 Goroutine 的地方，`sync.WaitGroup` 的 `Add` 和 Goroutine 内部的 `Done` 是不是成对出现的？
4.  主流程中，有没有调用 `wg.Wait()`？

#### **2.2 Channel 的使用：是“管道”而非“容器”**

Channel 是 Goroutine 之间通信的桥梁。新手常犯的错误是把它当成一个线程安全队列来用，而忽略了它阻塞的特性。

**审查要点：**

*   **理解缓冲与非缓冲：**
    *   **非缓冲 (`make(chan T)`)：** 发送和接收必须同时准备好，否则一方会阻塞。适合做“信号通知”或强同步。
    *   **缓冲 (`make(chan T, size)`)：** 在缓冲区满之前，发送不会阻塞。适合做任务队列，解耦生产者和消费者。
*   **死锁（Deadlock）的预防：**
    *   不要在一个 Goroutine 中对同一个非缓冲 Channel 同时进行读和写。
    *   向一个没有接收者的非缓冲 Channel 写数据会永久阻塞。
    *   从一个空的 Channel 读数据会永久阻塞。
*   **优雅地关闭 Channel：**
    *   由发送方负责关闭 Channel。
    *   接收方可以通过 `v, ok := <-ch` 的语法判断 Channel 是否已关闭。`ok` 为 `false` 表示已关闭。

**实战场景：**

在我们的 AI 系统中，有一个服务需要接收大量的影像数据进行预处理，然后交给算法模型。生产者（接收请求的 Handler）和消费者（处理数据的 Worker）的速度可能不匹配。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type ImageData struct {
	ID   int
	Data string
}

// worker 是消费者，从任务通道中获取数据并处理
func worker(id int, tasks <-chan ImageData, wg *sync.WaitGroup) {
	defer wg.Done()
	// 使用 for-range 遍历 channel 是一个优雅的接收方式。
	// 当 channel被关闭后，循环会自动结束。
	for task := range tasks {
		fmt.Printf("工人 %d 正在处理影像 %d\n", id, task.ID)
		time.Sleep(1 * time.Second) // 模拟处理耗时
		fmt.Printf("工人 %d 完成处理影像 %d\n", id, task.ID)
	}
	fmt.Printf("工人 %d 退出，因为任务通道已关闭\n", id)
}

func main() {
	// 创建一个带缓冲的 channel 作为任务队列，容量为 10
	// 这样，生产者可以快速地向队列中放入 10 个任务而不会被阻塞
	tasks := make(chan ImageData, 10)

	var wg sync.WaitGroup

	// 启动 3 个 worker Goroutine
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go worker(i, tasks, &wg)
	}

	// 生产者：向 tasks channel 发送 15 个任务
	for i := 1; i <= 15; i++ {
		task := ImageData{ID: i, Data: "这是一些影像数据"}
		fmt.Printf("生产者放入任务 %d\n", i)
		tasks <- task
	}

	// 关键：所有任务都发送完毕后，由生产者关闭 channel
	// 这会通知所有 worker，不会再有新任务了
	close(tasks)

	// 等待所有 worker 处理完 channel 中剩余的任务并退出
	wg.Wait()

	fmt.Println("所有影像数据处理完毕。")
}
```

**Review 时我会强调：** `close(tasks)` 这个操作至关重要。如果没有它，`worker` 中的 `for-range` 会在处理完 15 个任务后因为 channel 中再无数据而永久阻塞，导致死锁。

---

### **第三章：工程篇 - 代码的健壮性与质量保障**

代码能跑起来只是第一步，能在各种压力和异常情况下稳定运行，才是生产级代码的标准。

#### **3.1 单元测试：是“健康体检”而非“形式主义”**

在医疗行业，软件的正确性至关重要。我们要求核心业务逻辑的单元测试覆盖率必须达到 90% 以上。

**审查要点：**

*   **测试覆盖率：** 使用 `go test -cover` 命令检查覆盖率。但不要盲目追求 100%，重点是覆盖核心逻辑和所有分支。
*   **表格驱动测试（Table-Driven Tests）：** 对于一个函数有多种输入和预期输出的场景，这是 Go 中非常优雅的测试方式。
*   **关注边界条件：** `nil`、空字符串、零值、最大/最小值等边界情况是 Bug 的重灾区，必须测试。

**实战场景：**

在“临床试验机构项目管理系统”中，有一个函数用于判断某个试验项目是否已到期。

**被测试函数 (`internal/biz/trial.go`)**

```go
package biz

import "time"

// IsExpired 判断试验是否到期
func IsExpired(trialEndTime time.Time, now time.Time) bool {
    if trialEndTime.IsZero() {
        return false // 永不到期
    }
    return now.After(trialEndTime)
}
```

**测试代码 (`internal/biz/trial_test.go`)**

```go
package biz

import (
	"testing"
	"time"
)

func TestIsExpired(t *testing.T) {
	// 定义当前时间为一个固定值，让测试结果可预测
	now := time.Date(2023, 10, 26, 10, 0, 0, 0, time.UTC)

	// 表格驱动测试
	testCases := []struct {
		name         string    // 测试用例名称
		trialEndTime time.Time // 输入：试验结束时间
		expected     bool      // 期望输出
	}{
		{
			name:         "已到期",
			trialEndTime: time.Date(2023, 10, 25, 10, 0, 0, 0, time.UTC),
			expected:     true,
		},
		{
			name:         "未到期",
			trialEndTime: time.Date(2023, 10, 27, 10, 0, 0, 0, time.UTC),
			expected:     false,
		},
		{
			name:         "正好在到期时间点",
			trialEndTime: time.Date(2023, 10, 26, 10, 0, 0, 0, time.UTC),
			expected:     false, // After 判断是不包含等于的
		},
		{
			name:         "边界条件：零值时间（永不到期）",
			trialEndTime: time.Time{},
			expected:     false,
		},
	}

	for _, tc := range testCases {
		// t.Run 让每个测试用例在测试报告中都有独立的名字
		t.Run(tc.name, func(t *testing.T) {
			actual := IsExpired(tc.trialEndTime, now)
			if actual != tc.expected {
				t.Errorf("期望得到 %v, 但实际得到 %v", tc.expected, actual)
			}
		})
	}
}
```

**Review 时我会看：** 是否覆盖了正常、异常和边界情况？测试用例的命名是否清晰地描述了测试场景？

#### **3.2 日志记录：是“飞行记录仪”而非“打印垃圾”**

当线上系统出问题时，日志是我们唯一的线索。混乱的日志不仅没用，还会干扰排查。

**审查要点：**

*   **结构化日志：** 必须使用结构化日志库（如 `zerolog`, `zap`），输出 JSON 格式。这便于 Logstash、Fluentd 等工具采集和分析。
*   **日志级别：** 正确使用 `Debug`, `Info`, `Warn`, `Error`, `Fatal` 等级别。`Info` 记录关键业务流程，`Error` 记录需要人工介入的错误。
*   **包含上下文：** 日志中必须包含关键的上下文信息，如 `trace_id`, `user_id`, `patient_id` 等，方便串联起一个完整的请求链路。

**实战场景：**

在 `go-zero` 中，我们通过中间件注入带上下文的 logger，业务代码中可以直接使用。

```go
// 在 ServiceContext 中初始化带 trace id 的 logger
// go-zero 默认已集成

// 在 logic 中使用
func (l *SubmitLogic) Submit(req *types.SubmitRequest) (*types.SubmitResponse, error) {
    // 使用 logx 记录日志，它会自动从 context 中读取 trace_id 等信息
    logx.WithContext(l.ctx).Infof("开始处理患者 %s 的问卷提交请求", req.PatientID)

    _, err := l.svcCtx.PatientModel.FindOne(l.ctx, req.PatientID)
    if err != nil {
        // 记录错误日志，并附加上下文信息
        logx.WithContext(l.ctx).Errorf("查询患者信息失败, patientID: %s, error: %v", req.PatientID, err)
        return nil, errorx.ErrPatientNotFound
    }
    
    // ...
    logx.WithContext(l.ctx).Infof("患者 %s 的问卷提交成功", req.PatientID)
    return &types.SubmitResponse{Success: true}, nil
}
```

**输出的日志（JSON 格式）：**

```json
{"@timestamp":"2023-10-26T12:00:00.000+08:00", "level":"info", "trace":"a1b2c3d4e5f6", "content":"开始处理患者 P12345 的问卷提交请求"}
{"@timestamp":"2023-10-26T12:00:01.000+08:00", "level":"error", "trace":"a1b2c3d4e5f6", "content":"查询患者信息失败, patientID: P12345, error: record not found"}
```

通过 `trace` 字段，我们可以轻松地在 Kibana 或 Grafana Loki 中筛选出某一次请求的所有相关日志。

### **结语**

Code Review 远不止是找 Bug。它是一种知识传递、标准统一和团队文化建设的过程。在我们这个行业，严谨的 Code Review 更是对项目、对用户负责的体现。

这份清单是我个人和团队多年实践的沉淀，它肯定不是最全的，但每一条都经过了实战的检验。希望这些来自一线的“炮火声”，能让你在编写 Go 代码时多一份思考，多一份从容。记住，好的代码是“长”出来的，不是一蹴而就的。让我们一起，用高质量的代码，构建更可靠的系统。