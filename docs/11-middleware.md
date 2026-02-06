# 11. 中间件系统

Nyx 框架提供了强大的中间件系统，支持 HTTP 和 RPC 两种协议的横切功能。通过中间件，开发者可以在请求处理链中插入自定义逻辑，实现认证授权、日志记录、跨域处理、数据压缩等功能。

## 11.1 中间件概述

### 11.1.1 什么是中间件

中间件（Middleware）是位于应用程序和底层服务之间的软件层，用于处理横切关注点。在 Nyx 框架中，中间件主要用于：

- **认证授权**：API 密钥验证、权限检查
- **日志记录**：请求日志、操作审计
- **跨域处理**：CORS 头部设置
- **数据压缩**：响应内容压缩
- **性能监控**：请求时间和资源监控
- **错误处理**：统一异常处理

### 11.1.2 Nyx 中间件特点

- **双协议支持**：同时支持 HTTP 和 RPC 中间件
- **分组应用**：可按路由分组选择性应用中间件
- **配置驱动**：通过配置文件灵活开启/关闭
- **高性能**：优化的中间件链执行机制
- **易于扩展**：支持自定义中间件开发

## 11.2 中间件接口

### 11.2.1 HTTP 中间件接口

```go
type HttpMiddleware func(http.Handler) http.Handler
```

HTTP 中间件接收一个 `http.Handler`，返回一个包装后的 `http.Handler`。

### 11.2.2 RPC 中间件接口

```go
type RpcMiddleware func(RpcHandler) RpcHandler
```

RPC 中间件接收一个 `RpcHandler`，返回一个包装后的 `RpcHandler`。

### 11.2.3 中间件注册

```go
// 注册 HTTP 中间件，支持按分组应用
func UseHttpMiddleware(middleware HttpMiddleware, groups ...string)

// 注册 RPC 中间件，支持按分组应用
func UseRpcMiddleware(middleware RpcMiddleware, groups ...string)
```

## 11.3 内置中间件

### 11.3.1 认证中间件

#### HTTP 认证中间件

```go
func ApiAuth(config *AuthConfig) x.HttpMiddleware
```

支持 API 密钥认证，验证请求头中的认证信息。

#### RPC 认证中间件

```go
func RpcAuth(config *AuthConfig) x.RpcMiddleware
```

支持 gRPC metadata 认证，验证 RPC 调用权限。

#### 认证配置

```go
type AuthConfig struct {
    AppSecrets  map[string]string // appid:secret 映射
    CheckTTL    int               // 时间戳有效期检查
    CheckNonce  bool              // 是否检查 nonce 防重放
    CheckMethod []string          // 需要认证的方法列表
    CheckExcept []string          // 排除认证的方法列表
    CheckAllow  map[string][]string // appid 允许访问的方法
    CheckForbid map[string][]string // appid 禁止访问的方法
    LocalCache  cache.LocalCache  // 本地缓存（用于 nonce 检查）
}
```

#### 认证中间件配置示例

```yaml
# HTTP 认证配置
auth:
  enabled: true
  http_check:
    enabled: true
    ttl: 300                    # 5分钟有效期
    check_nonce: true           # 启用防重放
    allowed_groups: ["api", "admin"]  # 应用分组

# 配置应用密钥
auth:
  app:
    - appid: "app001"
      secret: "secret123"
      rpc_allow: ["user/*", "order/*"]  # RPC 允许访问的方法
      rpc_forbid: ["admin/*"]            # RPC 禁止访问的方法
```

### 11.3.2 日志中间件

#### HTTP 日志中间件

```go
func HttpLog(config *LogConfig) x.HttpMiddleware
```

记录 HTTP 请求的详细信息，包括请求参数、响应状态、执行时间等。

#### RPC 日志中间件

```go
func RpcLog(config *LogConfig) x.RpcMiddleware
```

记录 RPC 调用的详细信息，包括调用参数、响应数据、执行时间等。

#### 日志配置

```go
type LogConfig struct {
    InfoLogName    string   // 信息日志名称
    WarnLogName    string   // 警告日志名称
    ErrorLogName   string   // 错误日志名称
    CheckMethod    []string // 需要记录日志的方法
    CheckExcept    []string // 排除记录的方法
    CheckReqMethod []string // 需要记录请求体的方法
    CheckReqExcept []string // 排除记录请求体的方法
    CheckResMethod []string // 需要记录响应体的方法
    CheckResExcept []string // 排除记录响应体的方法
    Logger         *log.Logger // 日志记录器
}
```

#### 日志中间件配置示例

```yaml
# HTTP 日志配置
http_log:
  enabled: true
  info_level_name: "http_info"
  warn_level_name: "http_warn" 
  error_level_name: "http_error"
  method: ["api", "admin"]    # 记录日志的分组
  req_method: ["POST", "PUT"] # 记录请求体的HTTP方法
  res_method: []              # 不记录响应体（节省空间）

# RPC 日志配置
rpc_log:
  enabled: true
  info_level_name: "rpc_info"
  warn_level_name: "rpc_warn"
  error_level_name: "rpc_error"
  method: ["user", "order"]  # 记录日志的分组
  req_method: ["*"]          # 记录所有请求参数
  res_method: ["user/*"]     # 只记录用户相关响应
```

### 11.3.3 CORS 中间件

```go
func Cors(opts *CorsOptions) x.HttpMiddleware
```

处理跨域资源共享（CORS）头部，支持预检请求处理。

#### CORS 配置

```go
type CorsOptions struct {
    AllowedOrigins   []string // 允许的来源域名
    AllowedMethods   []string // 允许的HTTP方法
    AllowedHeaders   []string // 允许的请求头
    ExposedHeaders   []string // 暴露的响应头
    AllowCredentials bool     // 是否允许凭证
    MaxAge          int      // 预检缓存时间
}
```

#### CORS 配置示例

```yaml
# CORS 配置
cors:
  enabled: true
  allowed_origins: 
    - "https://example.com"
    - "https://app.example.com"
    - "*.example.com"          # 支持通配符
  allowed_methods:
    - "GET"
    - "POST" 
    - "PUT"
    - "DELETE"
  allowed_headers:
    - "Content-Type"
    - "Authorization"
    - "X-Requested-With"
  allow_credentials: true      # 允许携带凭证
  max_age: 86400              # 24小时预检缓存
```

### 11.3.4 压缩中间件

```go
func Compress(config *CompressConfig) x.HttpMiddleware
```

对响应内容进行 gzip 或 deflate 压缩，减少网络传输量。

#### 压缩配置

```go
type CompressConfig struct {
    MinSize      int // 压缩数据大小阈值（字节）
    GzipLevel    int // gzip 压缩等级 (1-9)
    DeflateLevel int // deflate 压缩等级 (1-9)
}
```

#### 压缩配置示例

```yaml
# 压缩配置
compress:
  enabled: true
  min_size: 1024             # 1KB 以上才压缩
  gzip_level: 6                # gzip 压缩等级
  deflate_level: 6             # deflate 压缩等级
```

## 11.4 中间件使用

### 11.4.1 在应用初始化中注册中间件

```go
func main() {
    app := nyx.New()
    
    // 注册 HTTP 中间件
    app.UseHttpMiddleware(middleware.ApiAuth(authConfig), "api", "admin")
    app.UseHttpMiddleware(middleware.HttpLog(logConfig))
    app.UseHttpMiddleware(middleware.Cors(corsOptions))
    app.UseHttpMiddleware(middleware.Compress(compressConfig))
    
    // 注册 RPC 中间件
    app.UseRpcMiddleware(middleware.RpcAuth(authConfig), "user", "order")
    app.UseRpcMiddleware(middleware.RpcLog(logConfig))
    
    app.Run()
}
```

### 11.4.2 按分组应用中间件

```go
// 只对 admin 分组应用认证中间件
app.UseHttpMiddleware(middleware.ApiAuth(authConfig), "admin")

// 对所有分组应用日志中间件
app.UseHttpMiddleware(middleware.HttpLog(logConfig))

// 只对 api 分组应用 CORS 中间件
app.UseHttpMiddleware(middleware.Cors(corsOptions), "api")
```

### 11.4.3 配置文件启用中间件

```yaml
# 在 nyx.go 的 useHttpMiddlewares() 中会自动加载以下配置

# 认证中间件
auth:
  enabled: true
  http_check:
    enabled: true
    ttl: 300
    allowed_groups: ["api", "admin"]

# 日志中间件
http_log:
  enabled: true
  info_level_name: "http_access"
  allowed_groups: ["api"]

# CORS 中间件
cors:
  enabled: true
  allowed_origins: ["*"]
  allowed_methods: ["GET", "POST", "PUT", "DELETE"]

# 压缩中间件
compress:
  enabled: true
  min_size: 1024
  gzip_level: 6
```

## 11.5 自定义中间件

### 11.5.1 自定义 HTTP 中间件

```go
// 自定义请求时长监控中间件
func RequestDuration() x.HttpMiddleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            
            // 包装 ResponseWriter 来获取状态码
            statusWriter := &statusResponseWriter{ResponseWriter: w, statusCode: 200}
            
            // 调用下一个处理器
            next.ServeHTTP(statusWriter, r)
            
            // 记录执行时长
            duration := time.Since(start)
            if duration > time.Second {
                x.Warn("Slow request:", r.URL.Path, "duration:", duration)
            }
        })
    }
}

// 辅助结构体
type statusResponseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (w *statusResponseWriter) WriteHeader(statusCode int) {
    w.statusCode = statusCode
    w.ResponseWriter.WriteHeader(statusCode)
}
```

### 11.5.2 自定义 RPC 中间件

```go
// 自定义 RPC 限流中间件
func RateLimit(limit int) x.RpcMiddleware {
    tokens := make(chan struct{}, limit)
    
    for i := 0; i < limit; i++ {
        tokens <- struct{}{}
    }
    
    return func(next x.RpcHandler) x.RpcHandler {
        return func(ctx context.Context, params x.MAP, stream x.Stream) (context.Context, *x.ResponseData, error) {
            select {
            case <-tokens:
                defer func() { tokens <- struct{}{} }()
                return next(ctx, params, stream)
            default:
                return ctx, &x.ResponseData{
                    Code: 429,
                    Msg:  "Too Many Requests",
                }, nil
            }
        }
    }
}
```

### 11.5.3 注册自定义中间件

```go
func main() {
    app := nyx.New()
    
    // 注册自定义中间件
    app.UseHttpMiddleware(RequestDuration(), "api")
    app.UseRpcMiddleware(RateLimit(100), "user")
    
    app.Run()
}
```

## 11.6 中间件执行顺序

### 11.6.1 执行顺序规则

中间件按照注册顺序执行，外层中间件先执行，内层中间件后执行：

```go
app.UseHttpMiddleware(middleware1)  // 最外层，先执行
app.UseHttpMiddleware(middleware2)  // 中间层，第二个执行  
app.UseHttpMiddleware(middleware3)  // 最内层，最后执行
```

执行顺序：`middleware1 → middleware2 → middleware3 → 控制器 → middleware3 → middleware2 → middleware1`

### 11.6.2 分组中间件执行

```go
// 注册分组中间件
app.UseHttpMiddleware(A(), "group1")
app.UseHttpMiddleware(B())           // 全局中间件
app.UseHttpMiddleware(C(), "group2")

// 执行时：
// group1: A → B → 控制器 → B → A
// group2: B → C → 控制器 → C → B
// 其他组: B → 控制器 → B
```

## 11.7 中间件最佳实践

### 11.7.1 中间件设计原则

1. **单一职责**：每个中间件只负责一个功能
2. **高性能**：避免复杂的计算和阻塞操作
3. **错误处理**：妥善处理异常情况
4. **资源管理**：及时释放占用的资源
5. **可配置性**：支持通过配置文件自定义行为

### 11.7.2 性能优化建议

```go
// ✅ 推荐的实现：使用对象池减少分配
func EfficientMiddleware() x.HttpMiddleware {
    pool := &sync.Pool{
        New: func() interface{} {
            return &ContextData{}
        },
    }
    
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := pool.Get().(*ContextData)
            defer pool.Put(ctx)
            
            // 使用 ctx...
            next.ServeHTTP(w, r)
        })
    }
}

// ❌ 不推荐：频繁创建对象
func InefficientMiddleware() x.HttpMiddleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            data := &ContextData{}  // 每次请求都创建新对象
            // 使用 data...
            next.ServeHTTP(w, r)
        })
    }
}
```

### 11.7.3 错误处理模式

```go
func RobustMiddleware() x.HttpMiddleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if err := recover(); err != nil {
                    x.Error("Middleware panic:", err)
                    http.Error(w, "Internal Server Error", 500)
                }
            }()
            
            next.ServeHTTP(w, r)
        })
    }
}
```

### 11.7.4 配置化中间件

```go
type ConfigurableMiddleware struct {
    Enabled    bool
    Threshold  int
    LogLevel   string
}

func (cm *ConfigurableMiddleware) Middleware() x.HttpMiddleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !cm.Enabled {
                next.ServeHTTP(w, r)
                return
            }
            
            // 根据配置执行逻辑...
            start := time.Now()
            next.ServeHTTP(w, r)
            duration := time.Since(start)
            
            if duration > time.Duration(cm.Threshold)*time.Millisecond {
                x.Log(cm.LogLevel, "Slow request:", r.URL.Path, "duration:", duration)
            }
        })
    }
}
```

## 11.8 监控和调试

### 11.8.1 中间件监控

```go
type MiddlewareMetrics struct {
    TotalRequests    int64
    TotalErrors      int64
    AverageDuration  time.Duration
    SlowRequests     int64
}

func MonitorMiddleware(metrics *MiddlewareMetrics) x.HttpMiddleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            atomic.AddInt64(&metrics.TotalRequests, 1)
            
            defer func() {
                duration := time.Since(start)
                atomic.AddInt64(&metrics.TotalErrors, 1)
                
                // 更新平均时长
                oldAvg := atomic.LoadInt64(&metrics.AverageDuration)
                newAvg := time.Duration((int64(duration) + oldAvg) / 2)
                atomic.StoreInt64(&metrics.AverageDuration, int64(newAvg))
                
                // 记录慢请求
                if duration > time.Second {
                    atomic.AddInt64(&metrics.SlowRequests, 1)
                }
            }()
            
            next.ServeHTTP(w, r)
        })
    }
}
```

### 11.8.2 中间件调试

```go
func DebugMiddleware(name string) x.HttpMiddleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            x.Info("Middleware enter:", name, r.URL.Path)
            
            start := time.Now()
            next.ServeHTTP(w, r)
            duration := time.Since(start)
            
            x.Info("Middleware exit:", name, r.URL.Path, "duration:", duration)
        })
    }
}
```

## 11.9 常见问题

### 11.9.1 中间件执行顺序问题

**问题**：中间件执行顺序不符合预期
**解决**：理解注册顺序与执行顺序的关系，外层先执行，内层后执行

### 11.9.2 分组中间件冲突

**问题**：同一分组多个中间件冲突
**解决**：合理规划分组策略，避免重叠

### 11.9.3 性能问题

**问题**：中间件影响响应速度
**解决**：优化中间件实现，使用缓存和对象池

## 11.10 总结

Nyx 框架的中间件系统提供了：

### 🎯 **核心特性**
- **双协议支持**：HTTP 和 RPC 统一中间件接口
- **分组应用**：支持按路由分组选择性应用
- **配置驱动**：灵活的配置文件控制
- **高性能执行**：优化的中间件链机制

### 🛠️ **内置中间件**
- **认证中间件**：API 密钥和权限验证
- **日志中间件**：详细的请求响应记录
- **CORS 中间件**：跨域资源共享处理
- **压缩中间件**：响应数据压缩

### 💼 **应用价值**
- **横切关注点**：统一处理认证、日志、监控等
- **代码复用**：避免重复的功能代码
- **性能优化**：内置性能优化中间件
- **安全加固**：多层安全验证机制

通过合理使用 Nyx 框架的中间件系统，开发者可以构建安全、高性能、易维护的 Web 应用程序。

---

**下一章预告**：第十二章将详细介绍"高级配置"，包括环境变量管理、热重载机制、生产部署配置等高级功能。
