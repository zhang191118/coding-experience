### Go语言DDD实战：构建临床试验系统的端到端测试与CI/CD流水线(含Testcontainers)### # Go中DDD端到端测试实战：从临床试验系统到CI/CD就绪的测试流水线

## 第一章：在医疗临床系统中理解DDD端到端测试的核心理念

在我负责的临床试验电子数据采集系统（EDC）项目中，端到端测试不是可选项，而是法规要求。国家药监局（NMPA）要求所有临床试验数据必须可追溯、可验证，这意味着我们的测试不仅要验证代码逻辑，更要验证整个业务流程是否符合GCP（药物临床试验质量管理规范）。

### 测试边界与职责划分：以患者访视流程为例

在临床试验中，一个典型的患者访视流程涉及多个业务域：
- 患者管理域：患者筛选、知情同意
- 访视计划域：安排访视时间、检查项
- 数据采集域：收集实验室检查结果、不良事件
- 监查域：数据核查、质疑管理

我们的端到端测试必须覆盖从患者入组到数据锁定的完整流程：

```go
// 临床试验访视端到端测试场景
func TestPatientVisitE2E(t *testing.T) {
    // 1. 患者筛选与入组
    patient := domain.NewPatient("P001", "试验组", []domain.InclusionCriteria{...})
    
    // 2. 安排基线访视
    visit := domain.ScheduleVisit(patient.ID, domain.BaselineVisit, time.Now())
    
    // 3. 采集实验室数据
    labData := domain.LabData{
        PatientID: patient.ID,
        VisitID:   visit.ID,
        Tests:     []domain.LabTest{...},
    }
    
    // 4. 记录不良事件
    ae := domain.AdverseEvent{
        PatientID: patient.ID,
        Severity:  domain.SeverityModerate,
        Relationship: domain.RelationshipPossible,
    }
    
    // 验证：所有数据必须通过医学监查
    err := monitor.ReviewVisitData(patient.ID, visit.ID)
    require.NoError(t, err)
    
    // 验证：数据质疑必须被及时处理
    queries := queryRepo.GetOpenQueries(patient.ID)
    require.Empty(t, queries)
}
```

### 使用测试数据库隔离状态：法规合规要求

在医疗系统中，测试数据必须与生产数据完全隔离，但又要保持数据结构的一致性。我们使用Docker Compose启动完整的测试环境：

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  test-postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: edc_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5433:5432"
    
  test-redis:
    image: redis:7-alpine
    ports:
      - "6380:6379"
    
  test-minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: test
      MINIO_ROOT_PASSWORD: test123
    ports:
      - "9000:9000"
      - "9001:9001"
```

对应的Go测试初始化代码：

```go
// internal/test/setup.go
package test

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "os"
    "testing"
    "time"
    
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
)

type TestEnvironment struct {
    PostgresContainer *postgres.PostgresContainer
    RedisContainer    testcontainers.Container
    DB                *sql.DB
    RedisAddr         string
}

func SetupTestEnvironment(t *testing.T) *TestEnvironment {
    ctx := context.Background()
    
    // 启动PostgreSQL容器（临床试验数据存储）
    pgContainer, err := postgres.Run(ctx,
        "postgres:15-alpine",
        postgres.WithDatabase("edc_test"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second)),
    )
    if err != nil {
        t.Fatalf("failed to start postgres container: %v", err)
    }
    
    connStr, err := pgContainer.ConnectionString(ctx)
    if err != nil {
        t.Fatalf("failed to get connection string: %v", err)
    }
    
    // 连接数据库并执行迁移
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        t.Fatalf("failed to connect to test database: %v", err)
    }
    
    // 执行数据库迁移（包含临床试验特定的表结构）
    if err := runClinicalMigrations(db); err != nil {
        t.Fatalf("failed to run migrations: %v", err)
    }
    
    // 插入基础测试数据（符合CDISC标准）
    if err := seedTestData(db); err != nil {
        t.Fatalf("failed to seed test data: %v", err)
    }
    
    t.Cleanup(func() {
        db.Close()
        if err := pgContainer.Terminate(ctx); err != nil {
            t.Logf("failed to terminate postgres container: %v", err)
        }
    })
    
    return &TestEnvironment{
        PostgresContainer: pgContainer,
        DB:                db,
    }
}

// 临床试验特定的数据库迁移
func runClinicalMigrations(db *sql.DB) error {
    migrations := []string{
        // 患者域
        `CREATE TABLE patients (
            id VARCHAR(36) PRIMARY KEY,
            subject_id VARCHAR(50) UNIQUE NOT NULL,
            site_id VARCHAR(50) NOT NULL,
            randomization_number INT,
            treatment_group VARCHAR(50),
            status VARCHAR(20) CHECK (status IN ('SCREENING', 'ENROLLED', 'COMPLETED', 'WITHDRAWN')),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )`,
        
        // 访视域
        `CREATE TABLE visits (
            id VARCHAR(36) PRIMARY KEY,
            patient_id VARCHAR(36) REFERENCES patients(id),
            visit_name VARCHAR(100) NOT NULL,
            visit_date DATE NOT NULL,
            visit_window_start DATE,
            visit_window_end DATE,
            status VARCHAR(20) CHECK (status IN ('SCHEDULED', 'IN_PROGRESS', 'COMPLETED', 'MISSED')),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )`,
        
        // 表单数据域（符合CDISC ODM标准）
        `CREATE TABLE form_data (
            id VARCHAR(36) PRIMARY KEY,
            patient_id VARCHAR(36) REFERENCES patients(id),
            visit_id VARCHAR(36) REFERENCES visits(id),
            form_oid VARCHAR(100) NOT NULL,
            item_group_oid VARCHAR(100),
            item_oid VARCHAR(100) NOT NULL,
            value TEXT,
            status VARCHAR(20) CHECK (status IN ('DATA_ENTRY', 'VERIFIED', 'LOCKED')),
            query_status VARCHAR(20) CHECK (query_status IN ('NONE', 'OPEN', 'ANSWERED', 'CLOSED')),
            created_by VARCHAR(100),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            verified_at TIMESTAMP,
            locked_at TIMESTAMP
        )`,
        
        // 数据质疑域
        `CREATE TABLE data_queries (
            id VARCHAR(36) PRIMARY KEY,
            form_data_id VARCHAR(36) REFERENCES form_data(id),
            query_text TEXT NOT NULL,
            status VARCHAR(20) CHECK (status IN ('OPEN', 'ANSWERED', 'CLOSED')),
            created_by VARCHAR(100),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            answered_at TIMESTAMP,
            closed_at TIMESTAMP
        )`,
    }
    
    for _, migration := range migrations {
        if _, err := db.Exec(migration); err != nil {
            return fmt.Errorf("failed to execute migration: %v", err)
        }
    }
    
    return nil
}
```

### 模拟外部依赖的策略：与医院HIS/LIS系统集成

在真实的临床试验中，我们的系统需要与医院信息系统（HIS）、实验室信息系统（LIS）集成。在测试中，我们使用WireMock来模拟这些外部系统：

```go
// internal/test/external_mocks.go
package test

import (
    "net/http"
    "testing"
    
    "github.com/wiremock/go-wiremock"
)

type ExternalSystemMocks struct {
    WireMockClient *wiremock.Client
    HISMock        *HISServiceMock
    LISMock        *LISServiceMock
}

func SetupExternalMocks(t *testing.T) *ExternalSystemMocks {
    // 启动WireMock服务器
    wmClient := wiremock.NewClient("http://localhost:8080")
    
    // 模拟HIS系统患者信息接口
    err := wmClient.StubFor(wiremock.Get(wiremock.URLPathEqualTo("/his/patients/123")).
        WillReturnResponse(
            wiremock.NewResponse().
                WithStatus(http.StatusOK).
                WithHeader("Content-Type", "application/json").
                WithBody(`{
                    "patientId": "123",
                    "name": "张三",
                    "gender": "M",
                    "birthDate": "1980-01-01",
                    "medicalRecordNo": "MRN001"
                }`),
        ))
    if err != nil {
        t.Fatalf("failed to stub HIS patient API: %v", err)
    }
    
    // 模拟LIS系统检验结果接口
    err = wmClient.StubFor(wiremock.Post(wiremock.URLPathEqualTo("/lis/results")).
        WithRequestBody(wiremock.EqualToJson(`{
            "patientId": "123",
            "tests": ["WBC", "RBC", "PLT"]
        }`)).
        WillReturnResponse(
            wiremock.NewResponse().
                WithStatus(http.StatusOK).
                WithHeader("Content-Type", "application/json").
                WithBody(`{
                    "results": [
                        {"testCode": "WBC", "value": "6.5", "unit": "10^9/L", "referenceRange": "4.0-10.0"},
                        {"testCode": "RBC", "value": "4.8", "unit": "10^12/L", "referenceRange": "4.0-5.5"},
                        {"testCode": "PLT", "value": "220", "unit": "10^9/L", "referenceRange": "100-300"}
                    ]
                }`),
        ))
    if err != nil {
        t.Fatalf("failed to stub LIS results API: %v", err)
    }
    
    t.Cleanup(func() {
        // 清理所有Stub
        if err := wmClient.Reset(); err != nil {
            t.Logf("failed to reset wiremock: %v", err)
        }
    })
    
    return &ExternalSystemMocks{
        WireMockClient: wmClient,
    }
}
```

## 第二章：医疗领域DDD测试基础构建

### 2.1 理解医疗DDD分层架构与测试边界

在临床试验系统中，我们的分层架构必须符合医疗行业的特殊性：

```
┌─────────────────────────────────────────────────────────────┐
│                   展现层 (Presentation)                      │
│  • 患者ePRO界面 (React/Vue)                                 │
│  • 研究者工作界面                                           │
│  • 监查员数据核查界面                                       │
│  • 统计师数据导出接口                                       │
└─────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────┐
│                   应用层 (Application)                       │
│  • 患者访视用例服务                                         │
│  • 数据质疑管理服务                                         │
│  • 医学编码服务 (MedDRA/WHO-DD)                             │
│  • 数据锁定服务                                             │
└─────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────┐
│                   领域层 (Domain)                            │
│  • 患者聚合根 (Patient)                                     │
│  • 访视值对象 (Visit)                                       │
│  • 实验室检验实体 (LabTest)                                 │
│  • 不良事件实体 (AdverseEvent)                              │
│  • 数据质疑领域服务 (QueryService)                          │
│  • 业务规则：访视窗口期、数据完整性检查                     │
└─────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────┐
│                基础设施层 (Infrastructure)                   │
│  • PostgreSQL仓储 (临床试验数据)                            │
│  • Redis缓存 (访视计划缓存)                                 │
│  • MinIO对象存储 (影像文件)                                 │
│  • Kafka消息队列 (领域事件)                                 │
│  • 外部系统适配器 (HIS/LIS集成)                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 领域模型的单元测试实践：以不良事件为例

在临床试验中，不良事件（Adverse Event）的报告有严格的业务规则。让我们看看如何测试这些规则：

```go
// domain/adverse_event_test.go
package domain_test

import (
    "testing"
    "time"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    
    "clinical-trial-system/domain"
)

func TestAdverseEvent_Validation(t *testing.T) {
    t.Run("严重不良事件必须24小时内报告", func(t *testing.T) {
        ae := domain.AdverseEvent{
            PatientID:     "P001",
            EventTerm:     "急性肝损伤",
            StartDate:     time.Now().Add(-12 * time.Hour), // 12小时前发生
            Severity:      domain.SeveritySevere,
            Relationship:  domain.RelationshipProbable,
        }
        
        // 验证：严重不良事件必须设置紧急报告标志
        err := ae.Validate()
        require.NoError(t, err)
        assert.True(t, ae.RequiresUrgentReporting())
        
        // 验证：报告时限必须在24小时内
        assert.True(t, ae.ReportingDeadline().Before(time.Now().Add(24*time.Hour)))
    })
    
    t.Run("导致停药的AE必须记录详细原因", func(t *testing.T) {
        ae := domain.AdverseEvent{
            PatientID:    "P002",
            EventTerm:    "皮疹",
            Severity:     domain.SeverityModerate,
            ActionTaken:  domain.ActionDrugWithdrawn,
        }
        
        // 应该失败：停药但未记录原因
        err := ae.Validate()
        assert.Error(t, err)
        assert.Contains(t, err.Error(), "停药原因必须记录")
        
        // 添加原因后应该通过
        ae.ActionTakenReason = "患者出现3级皮疹，根据方案要求停药"
        err = ae.Validate()
        assert.NoError(t, err)
    })
    
    t.Run("SAE必须关联研究者评估", func(t *testing.T) {
        ae := domain.NewAdverseEvent(
            "P003",
            "过敏性休克",
            domain.SeveritySevere,
            domain.RelationshipDefinite,
        )
        
        // SAE需要研究者评估
        require.True(t, ae.IsSerious())
        require.True(t, ae.RequiresInvestigatorAssessment())
        
        // 模拟研究者评估
        assessment := domain.InvestigatorAssessment{
            AssessorID:   "INV001",
            AssessedAt:   time.Now(),
            Causality:    domain.CausalityRelated,
            Expectedness: domain.ExpectedUnexpected,
            Comments:     "符合方案定义的SAE标准",
        }
        
        ae.RecordInvestigatorAssessment(assessment)
        
        assert.NotNil(t, ae.InvestigatorAssessment)
        assert.Equal(t, domain.CausalityRelated, ae.InvestigatorAssessment.Causality)
    })
}

func TestAdverseEvent_DomainEvents(t *testing.T) {
    t.Run("严重不良事件应触发SAE通知事件", func(t *testing.T) {
        ae := domain.NewAdverseEvent(
            "P004",
            "心肌梗死",
            domain.SeveritySevere,
            domain.RelationshipPossible,
        )
        
        // 获取领域事件
        events := ae.DomainEvents()
        require.Len(t, events, 1)
        
        event, ok := events[0].(domain.AdverseEventReported)
        require.True(t, ok)
        
        assert.Equal(t, ae.PatientID, event.PatientID)
        assert.Equal(t, ae.EventTerm, event.EventTerm)
        assert.True(t, event.IsSerious)
        assert.True(t, event.RequiresUrgentReporting)
    })
    
    t.Run("AE严重程度升级应触发事件", func(t *testing.T) {
        ae := domain.NewAdverseEvent(
            "P005",
            "头痛",
            domain.SeverityMild,
            domain.RelationshipUnlikely,
        )
        
        // 初始为轻度
        assert.False(t, ae.IsSerious())
        
        // 升级为严重
        ae.UpgradeSeverity(domain.SeveritySevere, "患者出现意识障碍")
        
        events := ae.DomainEvents()
        require.Len(t, events, 2) // 创建事件 + 升级事件
        
        // 查找严重程度升级事件
        var upgradeEvent domain.AdverseEventSeverityUpgraded
        for _, e := range events {
            if evt, ok := e.(domain.AdverseEventSeverityUpgraded); ok {
                upgradeEvent = evt
                break
            }
        }
        
        assert.Equal(t, ae.PatientID, upgradeEvent.PatientID)
        assert.Equal(t, domain.SeverityMild, upgradeEvent.PreviousSeverity)
        assert.Equal(t, domain.SeveritySevere, upgradeEvent.NewSeverity)
        assert.Equal(t, "患者出现意识障碍", upgradeEvent.Reason)
    })
}
```

### 2.3 应用服务层的集成测试策略：使用go-zero框架

在我们的临床试验系统中，我们使用go-zero框架构建微服务。以下是患者管理服务的集成测试示例：

```go
// internal/logic/patientlogic_test.go
package logic_test

import (
    "context"
    "testing"
    
    "github