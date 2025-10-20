### Golang WebSocket高可用：彻底解决连接安全与Goroutine泄漏 (Context/Gin实战)### 好的，各位同学，我是阿亮。

在咱们临床医疗软件这个行业里，稳定性和安全性是压倒一切的。想象一下，我们正在开发一个“临床研究智能监测系统”，研究护士（CRC）和监查员（CRA）需要实时看到患者数据的录入状态和质疑（Query）提醒。这种实时性，WebSocket 是不二之un。但它就像一把双刃剑，用好了是神器，用不好，半夜三点被运维的电话叫起来救火，那滋味可不好受。

今天，我就结合我们实际踩过的一些坑，跟大家聊聊怎么给我们的 Go WebSocket 服务穿上“防弹衣”，不仅能抵御外部的 DDoS 攻击，还能防止内部的内存泄漏导致服务“猝死”。

---

### **一、地基要稳：连接建立前的“三道岗哨”**

任何一个 WebSocket 服务，连接的建立阶段都是最脆弱的。我们不能让任何人随随便便就和我们的服务器“攀上关系”。在我们的“电子患者自报告结局系统（ePRO）”里，一个患者App要连上后台报数据，我们必须设下三道岗哨。

#### **第一道岗：身份验证（Authentication）**

这是最重要，也最容易被忽视的一步。很多新手写 WebSocket，习惯是先升级成 WebSocket 连接，然后再在连接里收发消息做身份验证。**这是大错特错的！**

一个未经身份验证的连接，本身就在消耗服务器资源（一个 goroutine，一个文件描述符，一块内存）。如果攻击者海量地发起这种“匿名”连接，你的服务器瞬间就会被拖垮。

**正确做法：在 HTTP 升级（Upgrade）请求阶段，就完成身份验证。**

在我们的项目中，所有客户端（App、Web端）登录后都会拿到一个 JWT (JSON Web Token)。后续所有请求，包括这个想要升级成 WebSocket 的 HTTP GET 请求，都必须在 Header 里带上这个 Token。

我们用 `gin` 框架来演示这个过程，这在做单体服务或者网关时非常常见。

**`main.go`**
```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/golang-jwt/jwt/v5"
	"github.com/gorilla/websocket"
)

// 模拟的JWT密钥，实际项目中请从配置中心或环境变量加载
var jwtSecret = []byte("your-very-secret-key")

// Claims 定义了JWT中存储的数据
type Claims struct {
	UserID   string `json:"userID"`
	UserRole string `json:"userRole"`
	jwt.RegisteredClaims
}

// AuthMiddleware 是一个Gin中间件，用于在升级连接前校验JWT
func AuthMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		// 实际项目中，token可能在Header, Query参数或Cookie中
		// 这里以Header为例: "Authorization: Bearer <token>"
		authHeader := c.GetHeader("Authorization")
		if authHeader == "" {
			// 为了兼容某些客户端可能把token放在query参数里
			authHeader = c.Query("token")
			if authHeader == "" {
				log.Println("认证失败: 缺少Token")
				c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Authorization token required"})
				return
			}
		} else {
			// 如果是 "Bearer <token>" 格式，提取token部分
			const bearerPrefix = "Bearer "
			if len(authHeader) > len(bearerPrefix) && authHeader[:len(bearerPrefix)] == bearerPrefix {
				authHeader = authHeader[len(bearerPrefix):]
			}
		}

		token, err := jwt.ParseWithClaims(authHeader, &Claims{}, func(token *jwt.Token) (interface{}, error) {
			// 校验签名算法
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
			}
			return jwtSecret, nil
		})

		if err != nil {
			log.Printf("认证失败: Token解析错误: %v\n", err)
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
			return
		}

		if claims, ok := token.Claims.(*Claims); ok && token.Valid {
			// 认证通过，将用户信息存入Gin的Context，方便后续Handler使用
			c.Set("userID", claims.UserID)
			c.Set("userRole", claims.UserRole)
			log.Printf("用户 %s 认证通过", claims.UserID)
			c.Next()
		} else {
			log.Println("认证失败: Token无效")
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
			return
		}
	}
}

// WebSocket 处理器
var upgrader = websocket.Upgrader{
	// CheckOrigin 解决跨域问题
	CheckOrigin: func(r *http.Request) bool {
		// 在这里可以加入白名单校验，比如只允许我们自己的前端域名访问
		// log.Printf("Origin: %s", r.Header.Get("Origin"))
		return true // 为了演示方便，这里允许所有来源
	},
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

func websocketHandler(c *gin.Context) {
	// 从中间件设置的Context中获取用户信息
	userID, _ := c.Get("userID")

	conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
	if err != nil {
		log.Printf("用户 %s WebSocket升级失败: %v\n", userID, err)
		return
	}
	defer conn.Close()

	log.Printf("用户 %s 连接成功!", userID)

	// 这里是每个连接的核心处理逻辑
	handleConnection(conn, userID.(string))
}

func handleConnection(conn *websocket.Conn, userID string) {
	// 简单实现：服务器每秒给客户端发一个消息
	ticker := time.NewTicker(1 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case t := <-ticker.C:
			message := fmt.Sprintf("你好 %s, 现在是 %s", userID, t.Format(time.RFC3339))
			err := conn.WriteMessage(websocket.TextMessage, []byte(message))
			if err != nil {
				log.Printf("用户 %s 写入消息失败: %v, 连接即将关闭", userID, err)
				return // 写入失败，通常意味着连接已断开，退出循环
			}
		}
	}
}

func main() {
	r := gin.Default()

	// 生成一个测试用的JWT Token
	// 实际场景中，这是在用户登录接口生成的
	claims := &Claims{
		UserID:   "patient-007",
		UserRole: "patient",
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
			Issuer:    "my-hospital-platform",
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	tokenString, _ := token.SignedString(jwtSecret)
	fmt.Printf("--- 测试用JWT Token ---\n%s\n----------------------\n", tokenString)
	fmt.Println("请使用ws工具连接 ws://localhost:8080/ws?token=<上面生成的token>")


	// WebSocket路由，应用AuthMiddleware中间件
	wsGroup := r.Group("/ws")
	wsGroup.Use(AuthMiddleware())
	{
		wsGroup.GET("", websocketHandler)
	}

	r.Run(":8080")
}
```
**怎么运行和测试？**
1.  把代码保存为 `main.go`。
2.  安装依赖：`go get github.com/gin-gonic/gin`、`go get github.com/gorilla/websocket`、`go get github.com/golang-jwt/jwt/v5`。
3.  运行 `go run main.go`，终端会打印出一个测试用的 JWT Token。
4.  使用任何 WebSocket 测试工具（比如 Postman 或在线工具），连接地址填 `ws://localhost:8080/ws?token=<你复制的Token>`。
5.  如果你不带 `token` 或者带一个错误的 `token` 访问 `http://localhost:8080/ws`，会看到 `401 Unauthorized` 错误，连接根本不会升级。

**这道岗哨的意义：** 将绝大多数无效和恶意的连接请求挡在门外，极大地节省了服务器资源。

#### **第二道岗：来源检查（Origin Check）**

这个检查是防止“跨站 WebSocket 劫持”（CSWSH）攻击。简单说，就是防止其他不相关的网站（比如一个钓鱼网站）伪造请求，利用你已经登录我们系统的浏览器身份，来和我们的 WebSocket 服务器建立连接。

在 `gorilla/websocket` 库里，`Upgrader` 的 `CheckOrigin` 字段就是干这个的。

```go
var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        // 从配置中读取允许的域名列表
        allowedOrigins := []string{"https://epro.my-hospital.com", "https://cra.my-hospital.com"}
        origin := r.Header.Get("Origin")
        for _, o := range allowedOrigins {
            if o == origin {
                return true
            }
        }
        log.Printf("拒绝了来自非法Origin的连接: %s", origin)
        return false
    },
    // ...
}
```
**关键点：** `CheckOrigin` 绝对不能图省事直接返回 `true`。必须严格校验请求来源是不是我们自己的前端应用。

#### **第三道岗：连接频率限制（Rate Limiting）**

即使是通过了身份验证的合法用户，也可能因为客户端 Bug 或恶意行为，在短时间内疯狂地建立连接。这同样会耗尽服务器资源。

我们需要一个限流器，限制同一个 IP 地址在单位时间内的连接次数。这里我们继续用 `gin` 中间件来实现，非常方便。

**`rate_limiter.go`**
```go
package main

import (
    "log"
	"net/http"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
	"golang.org/x/time/rate"
)

// IP限流器
var (
	// mu保护下面的ips map
	mu  sync.Mutex
	ips = make(map[string]*rate.Limiter)
)

// IPRateLimiterMiddleware 创建一个基于IP的限流中间件
// r: 每秒可以产生的令牌数
// b: 令牌桶的大小
func IPRateLimiterMiddleware(r rate.Limit, b int) gin.HandlerFunc {
	return func(c *gin.Context) {
		mu.Lock()
		// 从map中获取当前IP的限流器，如果不存在则创建一个新的
		limiter, exists := ips[c.ClientIP()]
		if !exists {
			limiter = rate.NewLimiter(r, b)
			ips[c.ClientIP()] = limiter
            log.Printf("为新IP %s 创建限流器", c.ClientIP())
		}
        mu.Unlock()

		// Allow()会消耗一个令牌，如果桶里没有可用的令牌，则返回false
		if !limiter.Allow() {
            log.Printf("IP %s 请求过于频繁，已被限流", c.ClientIP())
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{"error": "Too many requests"})
			return
		}
		c.Next()
	}
}

// 清理长期不活跃的IP限流器，防止内存泄漏
func init() {
    go func() {
        for {
            time.Sleep(10 * time.Minute)
            mu.Lock()
            // 简单实现：如果一个limiter在过去10分钟没有被使用过，就清理掉
            // 更优的实现可以使用LRU Cache
            for ip, limiter := range ips {
                // rate.Limiter没有提供上次访问时间，这里简化处理
                // 在实际项目中，可以封装一个带时间戳的结构体
                // 这里为了演示，我们假设长时间存在的limiter就是不活跃的
                // 这是一个简化的清理策略，生产环境需要更精细的控制
                if len(ips) > 10000 { // 设定一个阈值，防止map无限增长
                    delete(ips, ip)
                }
            }
            log.Printf("清理IP限流器，当前数量: %d", len(ips))
            mu.Unlock()
        }
    }()
}

```

**如何在 `main.go` 中使用它？**
```go
// 在main.go的main函数中
func main() {
    r := gin.Default()
    
    // ... (JWT token 生成代码) ...
    
    wsGroup := r.Group("/ws")
    // 先限流，再认证，顺序很重要！
    // 限制每个IP每秒最多创建2个连接，桶大小为5，允许一定的突发
    wsGroup.Use(IPRateLimiterMiddleware(2, 5)) 
    wsGroup.Use(AuthMiddleware())
    {
        wsGroup.GET("", websocketHandler)
    }

    r.Run(":8080")
}
```
**这道岗哨的意义：** 这是抵御连接型 DDoS 攻击和客户端滥用的有效防线。它能保证即使是合法用户，也无法通过高频重连来攻击服务器。

### **二、内部强健：Goroutine 的生命周期管理**

连接建立起来后，真正的挑战才刚刚开始。每个 WebSocket 连接，我们通常会启动至少一个 `goroutine` 来处理它的读写。如果这些 `goroutine` 因为各种原因无法正常退出，就会像僵尸一样永远留在内存里，最终把内存吃光，这就是 **Goroutine 泄漏**。

在我们“临床试验机构项目管理系统”里，一个PM（项目经理）打开 dashboard，会建立一个 WebSocket 连接，实时接收他所负责的所有项目的动态。如果他直接关了浏览器，我们必须确保服务器端的 `goroutine` 能被干净地回收。

#### **核心武器：`context.Context`**

`context` 是 Go 语言里管理 `goroutine` 生命周期的神器。把它想象成一个“指令官”，当它说“撤退” (`cancel()`被调用)时，所有听命于它的 `goroutine` 都应该立刻停止工作，清理资源，然后退出。

**改造后的 `handleConnection`**

我们将使用 `context` 来优雅地关闭一个连接的所有 `goroutine`。

```go
// 在main.go中，重写websocketHandler和handleConnection

func websocketHandler(c *gin.Context) {
	userID, _ := c.Get("userID")

	conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
	if err != nil {
		log.Printf("用户 %s WebSocket升级失败: %v\n", userID, err)
		return
	}
	// 注意：defer conn.Close() 仍然重要，作为最后保障
	defer conn.Close()

	log.Printf("用户 %s 连接成功!", userID)

	// 为这个连接创建一个独立的上下文
	ctx, cancel := context.WithCancel(context.Background())
	// 当函数退出时，调用cancel()通知所有子goroutine结束
	defer cancel()

	// 启动一个goroutine来读取客户端消息
	go readLoop(ctx, conn, userID.(string))
	
	// 主goroutine负责写入数据（比如心跳和业务推送）
	writeLoop(ctx, conn, userID.(string))

	log.Printf("用户 %s 的处理函数已结束", userID.(string))
}

// readLoop 负责从客户端读取消息
func readLoop(ctx context.Context, conn *websocket.Conn, userID string) {
	defer func() {
		log.Printf("用户 %s 的 readLoop 退出", userID)
	}()

	// 设置消息大小限制，防止客户端发送超大消息耗尽内存
	// 比如，我们规定一条消息最大不能超过4KB
	conn.SetReadLimit(4096)
	
	// 设置读超时，用于心跳检测
	// 如果在 30s 内没有收到任何消息（包括pong），则认为连接断开
	// 这个时间需要比客户端ping的间隔稍长一些
	conn.SetReadDeadline(time.Now().Add(30 * time.Second))
	conn.SetPongHandler(func(string) error {
		// 收到pong消息，说明连接是健康的，重置读超时
		log.Printf("用户 %s 收到Pong", userID)
		conn.SetReadDeadline(time.Now().Add(30 * time.Second))
		return nil
	})

	for {
		// ReadMessage是一个阻塞操作
		messageType, p, err := conn.ReadMessage()
		if err != nil {
			// 检查错误类型，websocket.IsCloseError可以判断是否是正常的关闭消息
			if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
				log.Printf("用户 %s 读取消息时发生意外错误: %v", userID, err)
			} else {
				log.Printf("用户 %s 连接正常关闭", userID)
			}
			// 无论是何种错误，都意味着连接已经结束，应该退出goroutine
			// 这里我们通过调用main handler里的cancel函数来通知writeLoop也退出
            // 不过因为main handler在readLoop结束后也会退出从而调用cancel，这里可以省略
			return
		}
		
		log.Printf("收到用户 %s 的消息 (类型: %d): %s", userID, messageType, string(p))
		// 在这里处理收到的业务消息...
	}
}


// writeLoop 负责向客户端写入数据（心跳和业务消息）
func writeLoop(ctx context.Context, conn *websocket.Conn, userID string) {
	// Ping Ticker, 每15秒发一个ping包
	pingTicker := time.NewTicker(15 * time.Second)
	// 业务消息Ticker（模拟）
	businessTicker := time.NewTicker(5 * time.Second)

	defer func() {
		pingTicker.Stop()
		businessTicker.Stop()
		log.Printf("用户 %s 的 writeLoop 退出", userID)
	}()

	for {
		select {
		case <-pingTicker.C:
			// 发送Ping消息
			// 设置写超时，保证写入操作不会永远阻塞
			if err := conn.WriteControl(websocket.PingMessage, []byte{}, time.Now().Add(5*time.Second)); err != nil {
				log.Printf("用户 %s 发送Ping失败: %v", userID, err)
				return // 发送失败，连接可能已断开
			}
			log.Printf("向用户 %s 发送Ping", userID)

		case t := <-businessTicker.C:
			// 模拟业务推送
			message := fmt.Sprintf("这是给 %s 的业务推送, 时间: %s", userID, t.Format("15:04:05"))
			if err := conn.WriteMessage(websocket.TextMessage, []byte(message)); err != nil {
				log.Printf("用户 %s 推送业务消息失败: %v", userID, err)
				return
			}
		
		case <-ctx.Done():
			// 上下文被取消，意味着连接应该关闭
			log.Printf("收到上下文取消信号，为用户 %s 关闭连接", userID)
			// 发送一个关闭帧给客户端
			err := conn.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""))
			if err != nil {
				log.Printf("用户 %s 发送关闭帧失败: %v", userID, err)
			}
			return
		}
	}
}
```

**代码解析与关键点：**

1.  **`context.WithCancel`**: 在 `websocketHandler` 的开始，我们创建了一个 `context` 和它的 `cancel` 函数。`defer cancel()` 保证了无论这个 handler 怎么退出（正常结束、panic），`cancel` 函数都会被调用。
2.  **双 Goroutine 模型**: 我们启动了一个 `readLoop` 专门负责读，`writeLoop` 在主 goroutine 里负责写。这是很常见的模式，避免了读写相互阻塞。
3.  **`ctx.Done()`**: 在 `writeLoop` 的 `select` 语句中，我们监听了 `ctx.Done()` channel。一旦 `cancel()` 被调用，这个 channel 就会关闭，`case <-ctx.Done()` 就会被选中，`writeLoop` 就能优雅退出。
4.  **`readLoop` 的退出**: `readLoop` 的退出通常是被动的，当 `conn.ReadMessage()` 返回错误时（比如连接被客户端关闭，或者网络中断），循环就会结束，`goroutine` 也就退出了。当`readLoop`退出后，`websocketHandler`函数也会返回，从而触发`defer cancel()`，通知`writeLoop`退出。
5.  **心跳机制 (Ping/Pong)**:
    *   `writeLoop` 定时发送 `PingMessage`。
    *   `readLoop` 设置了 `SetReadDeadline`。如果长时间没收到任何数据（包括客户端自动回复的 Pong），`ReadMessage` 就会超时返回错误，从而关闭这个“僵尸连接”。
    *   `SetPongHandler` 里的逻辑是：只要收到了 Pong，就刷新 `ReadDeadline`，证明连接还活着。
6.  **`SetReadLimit`**: 这是防止内存攻击的关键。我们限制了单条消息的最大尺寸。如果攻击者发送一个1GB的消息，`ReadMessage` 会立刻报错，而不是傻傻地去分配1GB内存然后OOM。

### **三、微服务实战：在 `go-zero` 中构建健壮的 WebSocket 服务**

在我们复杂的“智能开放平台”中，各个功能都是微服务。WebSocket 服务通常也是一个独立的微服务，专门负责长连接管理。它从消息队列（如 Kafka）消费业务消息，再精准地推送给对应的在线用户。

这里我们用 `go-zero` 来演示一个生产级的 WebSocket 服务结构。

**1. 定义 API 文件 (`ws.api`)**

```api
syntax = "v1"

info(
    title: "WebSocket Service"
    desc: "负责临床试验项目实时消息推送"
    author: "阿亮"
    version: "1.0"
)

// WebSocket 连接请求，通过 HTTP GET 发起
// 使用 go-zero 的 jwt 中间件进行认证
@server(
    jwt: Auth
    handler: WebSocketHandler
    path: /ws/project/:projectId
    method: get
)
```
*   我们定义了一个 GET 路由，并且用 `@server(jwt: Auth)` 声明了这个接口需要 JWT 认证。
*   路径中带了 `projectId`，这样我们就知道这个连接是为了监听哪个项目的动态。

**2. 生成代码**

执行 `goctl api go -api ws.api -dir .`，`go-zero` 会帮我们生成好骨架代码。

**3. 编写 `websocket_handler.go`**

Handler 的职责很简单，就是把请求交给 Logic 处理，它只负责 HTTP 层面的事情。

```go
// internal/handler/websocket_handler.go

package handler

import (
	"net/http"

	"github.com/zeromicro/go-zero/rest/httpx"
	"my-project/internal/logic"
	"my-project/internal/svc"
	"my-project/internal/types"
)

func WebSocketHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// go-zero v1.3.0 之后 httpx.ParsePath أصبح deprecated, use httpx.Vars instead
		// var req types.Request
		// if err := httpx.Parse(r, &req); err != nil {
		// 	httpx.Error(w, err)
		// 	return
		// }
        // 注意：go-zero的httpx.Parse不直接支持解析路径参数到结构体，我们手动获取
        projectId := r.URL.Query().Get("projectId") // 假设从query获取，或从路径

		l := logic.NewWebSocketLogic(r.Context(), svcCtx)
		// Logic层不返回resp和err，因为它接管了连接
		l.HandleWebSocket(w, r, projectId)
	}
}
```

**4. 核心逻辑 `websocket_logic.go`**

这里才是我们实现所有安全和生命周期管理的地方。

```go
// internal/logic/websocket_logic.go

package logic

import (
	"context"
	"log"
	"net/http"
	"time"

	"github.com/gorilla/websocket"
	"github.com/zeromicro/go-zero/core/logx"
	"my-project/internal/svc"
)

type WebSocketLogic struct {
	ctx    context.Context
	svcCtx *svc.ServiceContext
	logx.Logger
}

func NewWebSocketLogic(ctx context.Context, svcCtx *svc.ServiceContext) *WebSocketLogic {
	return &WebSocketLogic{
		ctx:    ctx,
		svcCtx: svcCtx,
		Logger: logx.WithContext(ctx),
	}
}

var upgrader = websocket.Upgrader{
	CheckOrigin: func(r *http.Request) bool {
		// 从svcCtx.Config中读取配置的白名单
		// return isOriginAllowed(r.Header.Get("Origin"), svcCtx.Config.AllowedOrigins)
		return true // 演示用
	},
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

func (l *WebSocketLogic) HandleWebSocket(w http.ResponseWriter, r *http.Request, projectId string) {
	// 从JWT中间件注入的上下文里获取用户ID
	// go-zero v1.6.0 及以后版本中，jwt解析后的claims存储在固定的key中
	userID := l.ctx.Value("userId").(string) // 注意：key和类型取决于你的jwt配置
	
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		l.Errorf("用户 %s 升级WebSocket失败: %v", userID, err)
		return
	}
	defer conn.Close()

	l.Infof("用户 %s 已连接, 监听项目: %s", userID, projectId)
	
	// 这里可以把该用户的连接实例注册到一个全局的Manager中，
	// 方便其他地方根据userID或projectId找到这个连接并推送消息。
	// client := NewClient(conn, userID, projectId)
	// GetClientManager().Register(client)
	// defer GetClientManager().Unregister(client)

	// 后续的读写循环逻辑和上面Gin的例子完全一样
	// 同样使用 context.WithCancel, 双goroutine模型，心跳，读限制等

	// 比如，我们可以创建一个子上下文来管理这个连接的生命周期
	connCtx, cancel := context.WithCancel(l.ctx)
	defer cancel()
	
	go l.readLoop(connCtx, conn, userID)
	l.writeLoop(connCtx, conn, userID)

	l.Infof("用户 %s 连接处理结束", userID)
}

// readLoop 和 writeLoop 方法的实现和Gin示例中的几乎一样，
// 只是把 log.Printf 换成 l.Infof / l.Errorf
func (l *WebSocketLogic) readLoop(ctx context.Context, conn *websocket.Conn, userID string) {
	// ... (代码同上)
}

func (l *WebSocketLogic) writeLoop(ctx context.Context, conn *websocket.Conn, userID string) {
	// ... (代码同上)
	// 真实的微服务场景，这里不是用 ticker 模拟业务消息
	// 而是会启动一个goroutine去订阅Kafka/Nats等消息队列
	// go l.subscribeFromMQ(ctx, conn, userID)
	// ...
}
```
**`go-zero` 架构的优势:**

*   **配置驱动:** 像 `AllowedOrigins`, JWT密钥等都可以统一写在 `etc/ws-api.yaml` 配置文件中，代码更干净。
*   **自带中间件:** JWT认证、日志、链路追踪等开箱即用，我们只需关注业务逻辑。
*   **逻辑分离:** Handler 和 Logic 的分离非常清晰，符合微服务的设计哲学。

### **总结**

好了，今天我们从实际的医疗项目场景出发，把一个 Go WebSocket 服务从“裸奔”状态武装到了“牙齿”。总结一下核心要点：

1.  **“先盘查，后迎客”**: 永远在 HTTP 升级阶段完成**身份认证、来源检查、频率限制**，把风险挡在门外。
2.  **“指令官带队”**: 为每个连接创建独立的 `context.Context`，用它来统一号令所有相关 `goroutine` 的生死存亡，这是防止 `goroutine` 泄漏的根本。
3.  **“保持心跳，及时清理”**: 实现严格的 Ping/Pong 心跳机制和读超时，果断踢掉“僵尸连接”，释放宝贵的服务器资源。
4.  **“不贪多，设上限”**: 永远用 `SetReadLimit` 限制消息大小，这是防止恶意大数据包攻击导致内存溢出的救命稻草。

这些策略不是什么高深的理论，而是我们在一次次压力测试、一次次线上问题复盘中总结出来的血泪经验。特别是在我们这个行业，系统的稳定性和数据的安全性容不得半点马虎。希望今天的分享能帮助大家在构建自己的高并发 WebSocket 服务时，少走一些弯路。

我是阿亮，我们下次再聊。