### Golang设计模式：告别面条代码！6大生产级实战助你打造卓越系统 (含Go-Zero/Kafka)### 你好，我是阿亮。在临床医疗软件这个行业摸爬滚打了 8 年多，我深知我们写的每一行代码，背后都可能关系到一次临床试验的成败，甚至影响到患者的健康。我们面对的系统，比如电子数据采集系统（EDC）、临床试验项目管理系统（CTMS），逻辑异常复杂，而且对数据的准确性、安全性和系统的稳定性要求极高。

刚入行那会儿，我也曾堆砌过大量的 `if-else`，写出过牵一发而动全身的“面条代码”。随着项目越来越大，新需求不断涌入，旧代码的维护成本高得吓人。那段痛苦的经历让我明白，仅仅会写 Go 的语法是远远不够的，**掌握设计模式，用工程化的思维去构建代码，才是从“能用”到“好用”再到“卓越”的关键。**

今天，我想结合我们医疗SaaS领域的实际业务场景，聊聊我沉淀下来的 6 个核心设计模式。这篇文章不谈空泛的理论，只讲我们如何在真实的项目中，用这些模式解决棘手的问题，希望能帮到正在成长路上的你。

---

### 开篇之前：先统一思想，Go 的设计哲学

在深入模式之前，我们必须先理解 Go 的两个核心设计哲学，它们是我们应用所有模式的基石：

1.  **面向接口编程，而不是面向实现编程**：Go 的接口是隐式实现的（非侵入式）。一个结构体只要实现了接口的所有方法，它就自动成为了该接口的类型。这给了我们极大的灵活性，也是解耦的关键。我们后续聊的很多模式，本质上都是在利用接口来隔离变化。
2.  **组合优于继承**：Go 没有传统意义上的 `class` 和 `extends`。它鼓励你通过在一个结构体中嵌入另一个结构体（组合）来复用代码。这避免了继承带来的复杂层级和紧耦合问题，让代码结构更扁平、更清晰。

记住这两点，你会发现 Go 实现设计模式的方式，比很多传统面向对象语言更加简洁和优雅。

---

### 模式一：单例模式（Singleton）- 全局资源的安全港

**1. 我遇到的问题**

在我们的“临床试验机构项目管理系统”中，几乎所有服务都需要访问数据库和读取全局配置。如果每个请求都创建一个新的数据库连接，或者每次都重新加载一次配置文件，系统的性能开销会非常巨大，高并发下甚至会导致数据库连接数耗尽而崩溃。我们需要一个机制，确保像数据库连接池、全局配置这类资源，在整个应用的生命周期中，**有且仅有一个实例**。

**2. 单例模式如何解决？**

单例模式的核心就是保证一个类型只有一个实例，并提供一个全局访问点。在并发环境下，我们必须确保这个实例的创建过程是线程安全的。Go 的 `sync` 包为我们提供了完美的工具：`sync.Once`。

`sync.Once` 能保证其内部的 `Do` 方法传入的函数，在多 Goroutine 并发调用下，也只会被执行一次。这正是我们实现线程安全单例所需要的。

**3. Go 实战代码 (基于 Gin 框架)**

假设我们需要一个全局的数据库连接池。下面是如何用 `sync.Once` 实现一个安全的数据库连接单例，并在 Gin 的 HTTP Handler 中使用它。

```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"sync"

	"github.com/gin-gonic/gin"
	_ "github.com/go-sql-driver/mysql" // 导入MySQL驱动
)

// DBManager 结构体用于管理数据库连接
type DBManager struct {
	*sql.DB
}

var (
	instance *DBManager
	once     sync.Once
)

// GetDBInstance 是获取数据库连接单例的全局方法
func GetDBInstance() *DBManager {
	once.Do(func() {
		// Do方法中的函数只会在第一次调用时执行
		log.Println("Initializing database connection...")

		// 这里替换成你的数据库连接信息
		// 格式: "user:password@tcp(host:port)/dbname"
		dsn := "root:your_password@tcp(127.0.0.1:3306)/clinical_trials?charset=utf8mb4&parseTime=True&loc=Local"
		
		db, err := sql.Open("mysql", dsn)
		if err != nil {
			// 在实际项目中，这里应该用 panic 或者更优雅的方式处理，
			// 因为数据库连接失败，程序基本无法正常工作。
			log.Fatalf("Failed to open database connection: %v", err)
		}

		// 设置数据库连接池参数
		db.SetMaxOpenConns(100) // 最大打开连接数
		db.SetMaxIdleConns(10)  // 最大空闲连接数

		if err = db.Ping(); err != nil {
			log.Fatalf("Failed to ping database: %v", err)
		}

		instance = &DBManager{DB: db}
		log.Println("Database connection initialized successfully.")
	})

	return instance
}

// GetPatientDataHandler 是一个示例API处理器，用于获取患者数据
func GetPatientDataHandler(c *gin.Context) {
	patientID := c.Param("id")

	// 从单例获取数据库连接
	db := GetDBInstance()

	var patientName string
	// 使用单例连接执行查询
	err := db.QueryRow("SELECT name FROM patients WHERE id = ?", patientID).Scan(&patientName)
	if err != nil {
		if err == sql.ErrNoRows {
			c.JSON(404, gin.H{"error": "Patient not found"})
			return
		}
		c.JSON(500, gin.H{"error": "Database query failed"})
		log.Printf("Error querying patient data: %v", err)
		return
	}

	c.JSON(200, gin.H{
		"patientId":   patientID,
		"patientName": patientName,
	})
}

func main() {
	router := gin.Default()

	// 定义一个API路由
	router.GET("/patient/:id", GetPatientDataHandler)

	fmt.Println("Server is running on port 8080...")
	router.Run(":8080")
}
```

**阿亮剖析：**

*   **`once sync.Once`**: 这是核心。`once.Do()` 会保证里面的匿名函数在程序运行期间，无论被多少个 Goroutine 调用，都只执行一次。这就完美解决了并发初始化时可能创建多个实例的竞态问题。
*   **懒加载（Lazy Loading）**: 我们的数据库连接是在 `GetDBInstance()` 第一次被调用时才初始化的，而不是在程序启动时。这对于那些不一定会被用到的重资源来说，可以优化启动速度。
*   **为什么返回指针 `*DBManager`**：返回指针可以确保所有使用者都操作同一个实例的内存地址，而不是它的副本。
*   **要点**：单例虽好，但不要滥用。它会引入全局状态，使得单元测试变得困难。只在那些真正需要全局唯一的资源上使用，如数据库连接、配置、日志器等。

---

### 模式二：工厂模式（Factory）- 创建对象的专业车间

**1. 我遇到的问题**

在我们的“临床研究智能监测系统”中，有一个核心功能是导出监测报告。客户的需求五花八门，有的要导出成 CSV 格式方便用 Excel 分析，有的要 PDF 格式用于正式归档，未来可能还要支持 Word 或 Excel 格式。

如果把创建不同格式报告的逻辑都写在一个函数里，会变成一个巨大的 `switch-case` 结构。每当要增加一种新的报告格式，我都得去修改这个核心函数，这违反了“对修改关闭，对扩展开放”的开闭原则，代码会越来越臃肿和脆弱。

**2. 工厂模式如何解决？**

工厂模式通过定义一个用于创建对象的接口（或者在 Go 中，一个函数），让子类（或者具体的实现）来决定实例化哪一个类。这使得一个类的实例化延迟到其子类。在 Go 中，我们通常用一个返回接口类型的函数来实现，这被称为“简单工厂”。

**3. Go 实战代码**

我们来定义一个 `ReportExporter` 接口，然后用一个工厂函数根据传入的类型来创建具体的导出器。

```go
package main

import (
	"fmt"
	"log"
)

// ReportData 代表报告的核心数据结构
type ReportData struct {
	Title   string
	Content string
}

// ReportExporter 定义了报告导出器的行为契约
type ReportExporter interface {
	Export(data ReportData) ([]byte, error)
}

// CSVExporter 实现了导出CSV格式报告的逻辑
type CSVExporter struct{}

func (e *CSVExporter) Export(data ReportData) ([]byte, error) {
	log.Println("Exporting report to CSV...")
	csvContent := fmt.Sprintf("Title,%s\nContent,%s", data.Title, data.Content)
	return []byte(csvContent), nil
}

// PDFExporter 实现了导出PDF格式报告的逻辑
type PDFExporter struct{}

func (e *PDFExporter) Export(data ReportData) ([]byte, error) {
	log.Println("Exporting report to PDF...")
	// 实际项目中会使用第三方库来生成PDF
	pdfContent := fmt.Sprintf("PDF Report\nTitle: %s\nContent: %s", data.Title, data.Content)
	return []byte(pdfContent), nil
}

// NewReportExporter 是我们的工厂函数
// 它根据指定的格式返回一个实现了 ReportExporter 接口的实例
func NewReportExporter(format string) (ReportExporter, error) {
	switch format {
	case "csv":
		return &CSVExporter{}, nil
	case "pdf":
		return &PDFExporter{}, nil
	default:
		return nil, fmt.Errorf("unsupported report format: %s", format)
	}
}

func main() {
	report := ReportData{
		Title:   "Q1 Clinical Trial Progress",
		Content: "Patient recruitment is on track.",
	}

	// 场景1: 导出CSV报告
	csvExporter, err := NewReportExporter("csv")
	if err != nil {
		log.Fatal(err)
	}
	csvData, _ := csvExporter.Export(report)
	fmt.Println("CSV Report:\n", string(csvData))
	
	fmt.Println("--------------------")

	// 场景2: 导出PDF报告
	pdfExporter, err := NewReportExporter("pdf")
	if err != nil {
		log.Fatal(err)
	}
	pdfData, _ := pdfExporter.Export(report)
	fmt.Println("PDF Report:\n", string(pdfData))

	// 场景3: 尝试不支持的格式
	_, err = NewReportExporter("word")
	if err != nil {
		fmt.Println("Error:", err)
	}
}
```

**阿亮剖析：**

*   **核心是接口 `ReportExporter`**: 这是解耦的关键。`main` 函数（客户端代码）只依赖于这个接口，它根本不关心具体的导出器是 `CSVExporter` 还是 `PDFExporter`。
*   **工厂函数 `NewReportExporter`**: 它封装了创建对象的复杂逻辑。客户端代码从 “我需要自己 `new` 一个具体的导出器” 变成了 “我告诉工厂我想要什么，工厂给我一个能用的东西”。
*   **易于扩展**: 现在，如果需要增加一个 Word 导出功能，我只需要：
    1.  创建一个 `WordExporter` 结构体，并实现 `Export` 方法。
    2.  在 `NewReportExporter` 函数的 `switch` 中增加一个 `case "word"`。
    3.  客户端代码完全不需要任何改动！这就是开闭原则的体现。

---

### 模式三：策略模式（Strategy）- 算法的自由切换

**1. 我遇到的问题**

在我们的“电子患者自报告结局（ePRO）系统”中，患者提交的数据需要进行校验。但不同的临床试验项目，其校验规则千差万别。

*   **项目A**：可能只需要简单的非空校验。
*   **项目B**：可能需要复杂的逻辑校验，比如“如果患者年龄小于18岁，则某个问题必须回答‘否’”。
*   **项目C**：可能需要调用外部AI模型进行语义校验。

如果把所有这些校验逻辑都写在一起，代码会变得极其混乱，充满了 `if-else if`，每次修改一个项目的校验规则都像在雷区排雷。

**2. 策略模式如何解决？**

策略模式定义了一系列的算法，并将每一个算法封装起来，使它们可以相互替换。策略模式让算法的变化独立于使用算法的客户。

在我们的场景中，每一种校验规则就是一种“策略”。我们可以把这些策略封装成不同的对象，然后在运行时根据具体项目的配置，动态地选择并使用相应的校验策略。

**3. Go 实战代码**

```go
package main

import (
	"fmt"
	"log"
)

// PatientReport 代表患者提交的数据
type PatientReport struct {
	PatientID int
	Age       int
	Answers   map[string]string
}

// ValidationStrategy 定义了数据校验策略的接口
type ValidationStrategy interface {
	Validate(report PatientReport) error
}

// SimpleValidation 实现了简单的非空校验策略
type SimpleValidation struct{}

func (s *SimpleValidation) Validate(report PatientReport) error {
	log.Println("Applying Simple Validation Strategy...")
	for key, value := range report.Answers {
		if value == "" {
			return fmt.Errorf("answer for '%s' cannot be empty", key)
		}
	}
	return nil
}

// AgeBasedValidation 实现了基于年龄的复杂逻辑校验策略
type AgeBasedValidation struct{}

func (a *AgeBasedValidation) Validate(report PatientReport) error {
	log.Println("Applying Age-Based Validation Strategy...")
	if report.Age < 18 {
		if consentAnswer, ok := report.Answers["parental_consent"]; !ok || consentAnswer != "yes" {
			return fmt.Errorf("patients under 18 require parental consent")
		}
	}
	// ...可以加入更多其他校验
	return nil
}

// ReportProcessor 是使用校验策略的上下文环境
type ReportProcessor struct {
	validator ValidationStrategy
}

// NewReportProcessor 创建一个新的报告处理器，并设置校验策略
func NewReportProcessor(validator ValidationStrategy) *ReportProcessor {
	return &ReportProcessor{validator: validator}
}

// SetValidator 允许在运行时动态更换校验策略
func (rp *ReportProcessor) SetValidator(validator ValidationStrategy) {
	rp.validator = validator
}

// Process 接收并处理报告
func (rp *ReportProcessor) Process(report PatientReport) {
	err := rp.validator.Validate(report)
	if err != nil {
		log.Printf("Validation failed for patient %d: %v", report.PatientID, err)
		// ... 记录校验失败，通知相关人员等
		return
	}
	log.Printf("Report for patient %d processed successfully.", report.PatientID)
	// ... 存入数据库等后续操作
}

func main() {
	// 报告1，来自项目A，使用简单校验
	report1 := PatientReport{
		PatientID: 101,
		Age:       35,
		Answers:   map[string]string{"pain_level": "5", "mood": "good"},
	}
	simpleProcessor := NewReportProcessor(&SimpleValidation{})
	simpleProcessor.Process(report1)

	fmt.Println("--------------------")

	// 报告2，来自项目B，患者未成年，使用基于年龄的校验
	report2 := PatientReport{
		PatientID: 102,
		Age:       16,
		Answers:   map[string]string{"parental_consent": "no"},
	}
	ageBasedProcessor := NewReportProcessor(&AgeBasedValidation{})
	ageBasedProcessor.Process(report2)

	// 报告3，同样来自项目B，但数据合规
	report3 := PatientReport{
		PatientID: 103,
		Age:       17,
		Answers:   map[string]string{"parental_consent": "yes"},
	}
	ageBasedProcessor.Process(report3)
}
```
**阿亮剖析：**

*   **分离关注点**：`ReportProcessor` 的职责是“处理报告的流程”，而各种 `ValidationStrategy` 实现的职责是“具体的校验算法”。两者通过接口解耦，各自独立演化。
*   **运行时切换**：通过 `NewReportProcessor` 注入或 `SetValidator` 方法，我们可以在程序运行时动态地改变一个 `ReportProcessor` 的行为，这非常灵活。例如，我们可以从数据库或配置文件中读取项目配置，来决定到底该用哪种校验策略。
*   **消除条件语句**：策略模式把 `if-else` 或者 `switch` 的分支逻辑，转化为了不同策略类的多态实现。这让代码更清晰，也更符合开闭原则。

---

### 模式四：装饰器模式（Decorator）- 功能的动态叠加

**1. 我遇到的问题**

在我们的“智能开放平台”中，我们提供了大量的 API 接口供第三方系统调用，比如获取脱敏后的研究数据。这些 API 有一些通用的横切关注点（cross-cutting concerns）：

1.  所有请求都需要记录访问日志。
2.  所有请求都需要进行身份认证和权限校验。
3.  对于一些耗时较长的请求，我们需要监控其执行时间。

最笨的办法是在每个 API Handler 函数里都重复写一遍日志、认证和监控的代码。这显然是不可接受的，代码冗余且难以维护。

**2. 装饰器模式如何解决？**

装饰器模式允许向一个现有的对象添加新的功能，同时又不改变其结构。这种模式创建了一个装饰器类，用来包装原有的类，并在保持类方法签名完整性的前提下，提供了额外的功能。

在 Go 的 Web 开发中，这个模式最经典的体现就是 **中间件（Middleware）**。一个中间件就是一个函数，它接收一个 `http.Handler`，并返回一个新的 `http.Handler`。新的 Handler 在调用原始 Handler 前后，执行一些额外的操作。

**3. Go 实战代码 (基于 Gin 框架)**

Gin 框架的中间件机制就是装饰器模式的完美实践。

```go
package main

import (
	"log"
	"time"

	"github.com/gin-gonic/gin"
)

// LoggingMiddleware 记录每个请求的耗时和路径
func LoggingMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		
		// 在请求处理之前...
		log.Printf("Request received: %s %s", c.Request.Method, c.Request.URL.Path)
		
		// 调用链中的下一个处理函数（可以是另一个中间件，也可以是最终的Handler）
		c.Next() 
		
		// 在请求处理之后...
		latency := time.Since(start)
		log.Printf("Request completed in %v, Status: %d", latency, c.Writer.Status())
	}
}

// AuthMiddleware 模拟身份认证
func AuthMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		// 从Header中获取Token
		token := c.GetHeader("X-Auth-Token")
		
		// 实际项目中会进行复杂的Token校验
		if token != "valid-token-for-clinical-study" {
			// 认证失败，中止请求链
			c.AbortWithStatusJSON(401, gin.H{"error": "Unauthorized"})
			return
		}
		
		// 认证成功，可以在Context中设置用户信息，供后续Handler使用
		c.Set("userID", "user123")
		
		c.Next()
	}
}

// GetClinicalDataHandler 是受保护的API，处理获取临床数据的请求
func GetClinicalDataHandler(c *gin.Context) {
	// 从中间件设置的Context中获取用户信息
	userID, _ := c.Get("userID")
	
	studyID := c.Param("studyId")
	
	log.Printf("User '%s' is fetching data for study '%s'", userID, studyID)
	
	// 模拟耗时的数据查询操作
	time.Sleep(100 * time.Millisecond)
	
	c.JSON(200, gin.H{
		"studyId": studyID,
		"data":    "some sensitive clinical data...",
	})
}

func main() {
	router := gin.Default()

	// 使用 LoggingMiddleware 应用于所有路由
	router.Use(LoggingMiddleware())

	// 创建一个API组，这个组下的所有路由都需要经过身份认证
	authorized := router.Group("/api/v1")
	authorized.Use(AuthMiddleware())
	{
		authorized.GET("/studies/:studyId/data", GetClinicalDataHandler)
		// 还可以定义更多受保护的路由
	}

	// 一个公开的路由，不需要认证
	router.GET("/health", func(c *gin.Context) {
		c.JSON(200, gin.H{"status": "UP"})
	})

	router.Run(":8080")
}
```
**阿亮剖析：**

*   **`gin.HandlerFunc`**：这是 Gin 中处理 HTTP 请求的函数类型，它就是被装饰的“核心组件”。
*   **中间件函数**：`LoggingMiddleware` 和 `AuthMiddleware` 就是“装饰器”。它们返回一个 `gin.HandlerFunc`，在这个函数内部包装了对下一个处理函数的调用。
*   **`c.Next()`**: 这是调用链的核心。它将控制权传递给下一个中间件或最终的 Handler。代码写在 `c.Next()` 之前，就是在请求到达业务逻辑前执行；写在之后，就是在业务逻辑执行完毕后执行。
*   **`c.AbortWithStatusJSON()`**: 如果认证失败，这个函数会立即中止请求处理链，后续的中间件和 Handler 都不会被执行，保证了安全性。
*   **组合使用**：我们可以像搭积木一样，通过 `router.Use()` 或 `group.Use()` 把多个中间件组合起来，形成一个处理链。这种方式非常灵活，功能可以按需插拔。

---

### 模式五：观察者模式（Observer）- 事件驱动的优雅解耦

**1. 我遇到的问题**

在我们的“电子患者自报告结局（ePRO）系统”中，当一个患者成功提交一份问卷后，需要触发一系列的后续操作：

1.  **数据存储**：将问卷数据写入主数据库。
2.  **通知医生**：如果问卷中某些指标超过警戒值（比如疼痛评分过高），需要立即通过系统消息或短信通知负责的医生。
3.  **更新统计**：实时更新该临床试验的整体数据统计看板。
4.  **记录审计**：在审计日志中记录这次提交事件。

如果把这四个操作都写在提交问卷的那个 API Handler 里，会产生严重的问题：
*   **紧耦合**：提交问卷的逻辑和通知、统计、审计的逻辑耦合在一起，任何一个模块的修改都可能影响主流程。
*   **性能瓶颈**：如果通知医生或更新统计是耗时操作，会拖慢整个 API 的响应时间，影响用户体验。
*   **扩展性差**：未来如果需要增加一个新的操作，比如“触发AI分析”，又得去修改那个核心的 Handler。

**2. 观察者模式如何解决？**

观察者模式定义了一种一对多的依赖关系，让多个观察者对象同时监听某一个主题对象。当主题对象状态发生变化时，会通知所有观察者对象，使它们能够自动更新自己。

在微服务架构中，这种模式通常通过**消息队列（Message Queue）**来实现。

*   **主题（Subject）**：就是提交问卷的 API 服务，它在完成核心操作后，会发布一个“问卷已提交”的事件（消息）到消息队列。
*   **观察者（Observers）**：是多个独立的、订阅了这个事件的后台服务（或消费者），比如通知服务、统计服务、审计服务。

当事件发布后，所有观察者都会收到消息并执行各自的逻辑。发布者根本不关心有多少观察者，也不关心它们具体做什么。

**3. Go 实战代码 (基于 go-zero 和 Kafka)**

`go-zero` 框架内置了对 Kafka 的支持（通过 `kq` 组件），非常适合实现这种事件驱动的架构。

**项目结构假设：**

*   `epros-api`: 接收患者提交问卷的 API 服务 (发布者/主题)。
*   `notification-svc`: 监听问卷提交事件，并发送通知的服务 (观察者1)。
*   `statistics-svc`: 监听事件，更新统计的服务 (观察者2)。

**第一步：在 `epros-api` (发布者) 中发送消息**

```go
// epros-api/internal/logic/submiteprologic.go

package logic

import (
    "context"
    "encoding/json"
    "fmt"
    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/service"
    "github.com/zeromicro/go-zero/core/stores/kq"
    "epros-api/internal/svc"
    "epros-api/internal/types"
)

type SubmitEproLogic struct {
    logx.Logger
    ctx    context.Context
    svcCtx *svc.ServiceContext
}

// ... NewSubmitEproLogic and other methods ...

// PatientReportSubmittedEvent 定义了事件消息的结构
type PatientReportSubmittedEvent struct {
	ReportID  string `json:"reportId"`
	PatientID string `json:"patientId"`
	StudyID   string `json:"studyId"`
	IsUrgent  bool   `json:"isUrgent"`
}

func (l *SubmitEproLogic) SubmitEpro(req *types.SubmitEproRequest) (resp *types.SubmitEproResponse, err error) {
    // 1. 核心业务：将问卷数据存入数据库
    reportID, isUrgent, err := l.svcCtx.EproModel.Insert(l.ctx, req.Data)
    if err != nil {
        return nil, err
    }
    
    // 2. 状态变化，发布事件！
    event := PatientReportSubmittedEvent{
        ReportID:  reportID,
        PatientID: req.PatientID,
        StudyID:   req.StudyID,
        IsUrgent:  isUrgent,
    }
    
    // 序列化消息体
    body, err := json.Marshal(event)
    if err != nil {
        // 记录错误，但不应阻塞主流程
        l.Errorf("failed to marshal event: %v", err)
    } else {
        // 使用 svcCtx 中配置的 kq Pusher 发送消息
        if err := l.svcCtx.KqPusher.Push(string(body)); err != nil {
            l.Errorf("failed to push message to kafka: %v", err)
        } else {
            l.Infof("event published successfully: %s", string(body))
        }
    }

    return &types.SubmitEproResponse{Success: true, ReportID: reportID}, nil
}
```

**第二步：在 `notification-svc` (观察者) 中消费消息**

`go-zero` 可以通过 `goctl` 工具快速创建一个 `kq` 消费者服务。

```bash
# 在 notification-svc 项目下执行
goctl kq new notification
```

这会生成消费者的基本框架。我们只需填写业务逻辑。

```go
// notification-svc/internal/mqs/notification.go (这是 goctl 生成的模板，我们填充逻辑)

package mqs

import (
    "context"
    "encoding/json"
    "github.com/zeromicro/go-zero/core/logx"
    "notification-svc/internal/svc"
)

type NotificationMq struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
}

// PatientReportSubmittedEvent 消息结构需要和发布者一致
type PatientReportSubmittedEvent struct {
	ReportID  string `json:"reportId"`
	PatientID string `json:"patientId"`
	StudyID   string `json:"studyId"`
	IsUrgent  bool   `json:"isUrgent"`
}


func NewNotificationMq(ctx context.Context, svcCtx *svc.ServiceContext) *NotificationMq {
    return &NotificationMq{
        ctx:    ctx,
        svcCtx: svcCtx,
    }
}

// Consume 是处理消息的核心方法
func (l *NotificationMq) Consume(key, val string) error {
    logx.Infof("Received Kafka message: key=%s, value=%s", key, val)

    var event PatientReportSubmittedEvent
    if err := json.Unmarshal([]byte(val), &event); err != nil {
        logx.Errorf("failed to unmarshal message: %v", err)
        return err // 返回错误，消息会根据配置重试
    }

    // 观察者的核心逻辑
    if event.IsUrgent {
        logx.Infof("Urgent report %s received for patient %s! Notifying doctor...", event.ReportID, event.PatientID)
        // 调用短信、邮件或App推送服务...
        // l.svcCtx.NotificationProvider.SendAlert(...)
    }

    return nil // 正常处理完成，返回 nil
}
```

**阿亮剖析：**

*   **异步和解耦**：`epros-api` 服务在发布消息后，它的任务就结束了，可以立刻响应客户端。它完全不知道也不关心后续的通知、统计等操作。这些操作由下游的消费者服务异步执行。
*   **高可用和削峰填谷**：如果瞬间有大量患者提交问卷，消息队列可以作为缓冲。消费者服务可以按照自己的处理能力，平稳地消费消息，避免了对下游系统（如短信网关）的冲击。
*   **强大的扩展性**：现在要增加“AI分析”功能，我只需要再创建一个新的 `ai-analysis-svc` 服务，让它也订阅 `PatientReportSubmitted` 这个主题即可。原有的 `epros-api` 和 `notification-svc` 等服务一行代码都不用改。这就是观察者模式在分布式系统中的威力。

---

### 模式六：建造者模式 (Builder) 与 Go 惯用法：函数式选项 (Functional Options)

**1. 我遇到的问题**

在我们的“临床试验项目管理系统”中，有一个查询患者列表的功能，查询条件非常复杂且灵活。研究员可能需要根据以下任意组合进行筛选：

*   年龄范围（例如 30-50 岁）
*   性别
*   入组的试验项目ID
*   特定的诊断结果（例如“高血压”）
*   是否已签署知情同意书
*   ...等等十几个可选条件

如果用一个函数来处理，这个函数的参数列表会变得非常长，例如 `FindPatients(ageMin, ageMax, gender, studyID, diagnosis, ...)`。很多参数可能都是可选的，调用时需要传入很多默认值（如 `0`, `""`, `nil`），代码非常丑陋且容易出错。

另一种方式是创建一个 `PatientQuery` 结构体，然后让调用者自己去填充。但这会导致一个问题：结构体的字段是公开的，我们无法控制哪些字段组合是合法的，也无法在创建时进行统一的校验。

**2. 建造者模式如何解决？**

建造者模式旨在将一个复杂对象的构建与其表示分离，使得同样的构建过程可以创建不同的表示。它一步一步地创建一个复杂对象，非常适合于对象属性多且部分可选的场景。

然而，在 Go 中，传统的建造者模式（创建一个 `Builder` 对象，链式调用 `SetXXX()`，最后调用 `Build()`）显得有些笨重，因为它模仿了 Java/C# 的风格。Go 社区演化出了一种更简洁、更惯用的实现方式，叫做**函数式选项模式（Functional Options Pattern）**。

**3. Go 实战代码 (函数式选项模式)**

```go
package main

import (
	"fmt"
	"strings"
)

// PatientQuery 封装了复杂的患者查询条件
// 注意，字段都是小写，外部无法直接修改，保证了封装性
type PatientQuery struct {
	ageMin      int
	ageMax      int
	gender      string
	studyIDs    []string
	diagnoses   []string
	hasConsented bool
}

// Option 是一个函数类型，用于修改 PatientQuery 对象
// 这就是函数式选项模式的核心
type Option func(*PatientQuery)

// NewPatientQuery 是我们的“建造者”，它接收一系列的 Option
func NewPatientQuery(opts ...Option) *PatientQuery {
	// 1. 创建一个带有默认值的查询对象
	query := &PatientQuery{
		ageMin:      0,
		ageMax:      120, // 默认查询所有年龄
		hasConsented: true, // 默认只查询已签署同意书的
	}

	// 2. 遍历并应用所有传入的 Option 函数
	for _, opt := range opts {
		opt(query)
	}

	return query
}

// 下面是具体的 Option 实现函数，它们都返回一个 Option 类型的闭包
// 这种以 "With" 开头的命名是 Go 的惯例

func WithAgeRange(min, max int) Option {
	return func(q *PatientQuery) {
		if min > 0 {
			q.ageMin = min
		}
		if max > 0 {
			q.ageMax = max
		}
	}
}

func WithGender(gender string) Option {
	return func(q *PatientQuery) {
		q.gender = gender
	}
}

func WithStudyIDs(ids ...string) Option {
	return func(q *PatientQuery) {
		q.studyIDs = append(q.studyIDs, ids...)
	}
}

func WithDiagnoses(diagnoses ...string) Option {
	return func(q *PatientQuery) {
		q.diagnoses = append(q.diagnoses, diagnoses...)
	}
}

// ToSQL 方法根据查询条件生成SQL语句（仅为示例）
func (q *PatientQuery) ToSQL() string {
	var conditions []string
	conditions = append(conditions, fmt.Sprintf("age >= %d AND age <= %d", q.ageMin, q.ageMax))
	conditions = append(conditions, fmt.Sprintf("has_consented = %t", q.hasConsented))

	if q.gender != "" {
		conditions = append(conditions, fmt.Sprintf("gender = '%s'", q.gender))
	}
	if len(q.studyIDs) > 0 {
		conditions = append(conditions, fmt.Sprintf("study_id IN ('%s')", strings.Join(q.studyIDs, "','")))
	}
	if len(q.diagnoses) > 0 {
		conditions = append(conditions, fmt.Sprintf("diagnosis IN ('%s')", strings.Join(q.diagnoses, "','")))
	}

	return "SELECT * FROM patients WHERE " + strings.Join(conditions, " AND ")
}


func main() {
	// 场景1: 查询 40-60岁 参与了 S001 试验的男性患者
	query1 := NewPatientQuery(
		WithAgeRange(40, 60),
		WithGender("male"),
		WithStudyIDs("S001"),
	)
	fmt.Println("Query 1 SQL:", query1.ToSQL())

	// 场景2: 查询所有被诊断为 "高血压" 或 "糖尿病" 的患者
	query2 := NewPatientQuery(
		WithDiagnoses("Hypertension", "Diabetes"),
	)
	fmt.Println("Query 2 SQL:", query2.ToSQL())

	// 场景3: 一个非常复杂的查询
	query3 := NewPatientQuery(
		WithAgeRange(25, 45),
		WithStudyIDs("S002", "S003"),
		WithDiagnoses("Asthma"),
	)
	fmt.Println("Query 3 SQL:", query3.ToSQL())
}
```

**阿亮剖析：**

*   **可读性极高**：调用代码 `NewPatientQuery(WithAgeRange(40, 60), WithGender("male"))` 像读句子一样，非常清晰地描述了要构建一个什么样的查询。
*   **灵活性和可扩展性**：增加一个新的查询条件，只需要增加一个新的 `WithXXX` 函数即可，对现有代码零侵入。
*   **默认值处理**：`NewPatientQuery` 函数内部首先创建了一个包含默认值的对象，然后才应用选项。这使得调用者只需关心他们想要覆盖的那些条件。
*   **封装性**：`PatientQuery` 的字段是私有的，外部只能通过我们提供的 `WithXXX` 选项来修改，这给了我们在选项函数内部添加校验逻辑的机会，保证了构建过程的健壮性。

这个模式在 Go 开源社区中被广泛使用，例如 `gRPC` 的客户端创建、`go-kit` 的服务构建等。它完美体现了 Go 语言简洁、灵活的函数式编程思想。

---

### 总结

从单例的资源管理，到工厂的对象创建，再到策略的算法切换、装饰器的功能增强、观察者的事件解耦，以及函数式选项的复杂对象构建，这 6 个设计模式几乎涵盖了我们日常后端开发中遇到的大部分复杂场景。

它们不是什么“银弹”，也不是需要生搬硬套的教条。它们是前人智慧的结晶，是在无数次代码重构的痛苦中总结出的“良方”。希望通过我结合医疗行业真实场景的分享，能让你对这些模式有更具体、更深刻的理解。

真正的成长，始于在下一次面对复杂需求时，你脑海里能浮现出这些模式的影子，并思考：“嗯，这个问题，也许用策略模式会更优雅。”

我是阿亮，我们下次再聊。