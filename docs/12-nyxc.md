# 12. NyxClient - gRPC客户端

## 12.1 概述

NyxClient 是专门为 Nyx 框架设计的 gRPC 客户端库，它提供了完整的 RPC 客户端功能，包括连接池管理、本地缓存、认证机制、流式请求支持等。NyxClient 通过标准化的接口简化了客户端与服务端的交互，同时提供了丰富的配置选项来满足不同的使用场景。

## 12.2 核心架构

NyxClient 采用分层架构设计，主要包含以下核心组件：

### 12.2.1 客户端核心 (NyxClient)

```go
type NyxClient struct {
    Address string    // 服务端地址
    appid   string    // 应用ID
    secret  string    // 应用密钥
    rpc     *rpc.RpcPool  // 连接池
    logger  *log.Logger   // 日志记录器
    authFn  func(appid, secret string) map[string]string  // 认证函数
    cache   cache.Cache   // 本地缓存
}
```

### 12.2.2 连接池管理 (RpcPool)

连接池负责管理客户端与服务端的连接，包含以下关键配置：

- **MaxIdleConns**: 最大空闲连接数
- **MaxOpenConns**: 最大打开连接数
- **连接超时**: 默认 3 秒

### 12.2.3 响应处理 (Response & ResponseIter)

- **Response**: 处理单次请求响应
- **ResponseIter**: 处理流式响应迭代

## 12.3 客户端初始化

### 12.3.1 基本初始化

```go
// 使用默认配置创建客户端
client, err := NewNyxClient("localhost:8080", "your_appid", "your_secret")
if err != nil {
    log.Fatal(err)
}
defer client.Close()
```

### 12.3.2 自定义配置

```go
// 创建自定义配置的客户端
client, err := NewNyxClient(
    "localhost:8080",
    "your_appid", 
    "your_secret",
    UseCache(1000),                    // 启用缓存，设置缓存大小为1000
    WithMaxIdleConns(10),             // 设置最大空闲连接数为10
    WithMaxOpenConns(50),             // 设置最大打开连接数为50
    WithDialTimeout(5*time.Second),   // 设置连接超时时间为5秒
)
```

## 12.4 连接池配置

### 12.4.1 空闲连接管理

```go
// 设置连接池参数
client, err := NewNyxClient(
    "localhost:8080",
    "your_appid", 
    "your_secret",
    WithMaxIdleConns(20),   // 最大空闲连接数：20
    WithMaxOpenConns(100),  // 最大打开连接数：100
)
```

**连接池工作原理**：
- 当连接池满时，新的请求会等待最多 3 秒
- 如果连接失败次数过多，该连接会被标记为失败状态并关闭
- 空闲连接数超过最大空闲数时，多余的连接会被关闭

### 12.4.2 连接超时配置

```go
// 自定义连接超时时间
client, err := NewNyxClient(
    "localhost:8080",
    "your_appid", 
    "your_secret",
    WithDialTimeout(10*time.Second),  // 连接超时10秒
)
```

## 12.5 缓存机制

### 12.5.1 启用缓存

```go
// 启用缓存，使用默认配置
client, err := NewNyxClient(
    "localhost:8080",
    "your_appid", 
    "your_secret",
    UseCache(),  // 使用默认缓存大小(1000)
)
```

### 12.5.2 自定义缓存大小

```go
// 设置自定义缓存大小
client, err := NewNyxClient(
    "localhost:8080",
    "your_appid", 
    "your_secret",
    UseCache(5000),  // 设置缓存大小为5000
)
```

### 12.5.3 请求级缓存配置

#### 12.5.3.1 固定TTL缓存

```go
// 请求使用60秒TTL的缓存
response, err := client.Request(
    "user.getInfo",
    params,
    WithCache(60),  // 缓存60秒
)
```

#### 12.5.3.2 自动刷新缓存

```go
// 请求使用自动刷新缓存，刷新间隔30秒
response, err := client.Request(
    "user.getInfo", 
    params,
    WithRefreshCache(30),  // 每30秒自动刷新一次
)
```

#### 12.5.3.3 缓存回调函数

```go
// 定义缓存更新回调函数
cacheCallback := func(data map[string]any) error {
    fmt.Printf("缓存已更新: %+v\n", data)
    return nil
}

// 使用回调函数的缓存请求
response, err := client.Request(
    "user.getInfo",
    params, 
    WithCache(60, cacheCallback),
)
```

### 12.5.4 缓存工作原理

1. **缓存键生成**: 基于方法名和参数生成唯一键
2. **缓存存储**: 使用 Gob 编码存储响应数据、头部信息、尾部信息和缓存时间
3. **缓存命中**: 优先返回缓存数据，减少服务端请求
4. **自动刷新**: 支持定时刷新缓存，保持数据新鲜度

### 12.5.5 检查缓存状态

```go
response, err := client.Request("user.getInfo", params)

// 检查是否命中缓存
if response.HitCache {
    cacheTime, _ := response.GetCacheTime()
    fmt.Printf("命中缓存，缓存时间: %v\n", cacheTime)
}
```

## 12.6 请求处理

### 12.6.1 同步请求

```go
// 基本同步请求
response, err := client.Request("user.getInfo", map[string]any{
    "user_id": 123,
})
if err != nil {
    log.Printf("请求失败: %v", err)
    return
}

// 检查响应状态
if response.GetCode() != 0 {
    log.Printf("业务错误: %s", response.GetMsg())
    return
}

// 获取响应数据
data := response.GetData()
userName := data["name"].(string)
```

### 12.6.2 流式请求

```go
// 流式请求处理
iter, err := client.RequestStream("user.streamData", params)
if err != nil {
    log.Printf("流式请求失败: %v", err)
    return
}

// 方法1: 逐行处理流数据
err = iter.Foreach(func(res *rpc.Response) error {
    data := res.GetData()
    fmt.Printf("接收到数据: %+v\n", data)
    return nil
})

// 方法2: 收集所有流数据
results, err := iter.Collect()
for _, res := range results {
    data := res.GetData()
    fmt.Printf("收集到的数据: %+v\n", data)
}
```

### 12.6.3 请求参数处理

NyxClient 支持多种参数类型：

```go
// 结构体参数
type UserParams struct {
    UserID   int64  `json:"user_id"`
    UserName string `json:"user_name"`
}

// 结构体参数
params := UserParams{
    UserID:   123,
    UserName: "张三",
}
response, err := client.Request("user.update", params)

// Map参数
params := map[string]any{
    "user_id":   123,
    "user_name": "张三",
}
response, err := client.Request("user.update", params)
```

## 12.7 请求选项配置

### 12.7.1 超时配置

```go
// 设置请求超时时间
response, err := client.Request(
    "user.getInfo",
    params,
    WithTimeout(5*time.Second),  // 5秒超时
)
```

### 12.7.2 上下文传递

```go
// 使用自定义上下文
ctx := context.Background()
ctx = context.WithValue(ctx, "guid", "your-guid")

response, err := client.Request(
    "user.getInfo",
    params,
    WithContext(ctx),
)
```

### 12.7.3 自定义头部信息

```go
// 添加自定义头部
headers := map[string]string{
    "X-Client-Version": "1.0.0",
    "X-Request-ID":     "req-123",
}

response, err := client.Request(
    "user.getInfo", 
    params,
    WithHeaders(headers),
)
```

## 12.8 认证机制

### 12.8.1 默认认证

NyxClient 自动处理认证，在每个请求中包含以下头部信息：

- `Authorization`: 认证令牌
- `X-Appid`: 应用ID
- `guid`: 全局唯一标识符

### 12.8.2 自定义认证函数

```go
// 定义自定义认证函数
customAuthFn := func(appid, secret string) map[string]string {
    // 生成自定义token逻辑
    token := generateCustomToken(appid, secret)
    return map[string]string{
        "Authorization": "Bearer " + token,
        "X-Custom-Header": "custom-value",
    }
}

// 使用自定义认证函数创建客户端
client, err := NewNyxClient(
    "localhost:8080",
    "your_appid",
    "your_secret",
    WithAuthFn(customAuthFn),
)
```

## 12.9 错误处理

### 12.9.1 网络错误处理

```go
response, err := client.Request("user.getInfo", params)
if err != nil {
    // 网络或连接错误
    log.Printf("网络错误: %v", err)
    
    // 检查是否为连接池满错误
    if strings.Contains(err.Error(), "pool is full") {
        // 处理连接池满的情况
        time.Sleep(time.Second)
        // 重试逻辑
        response, err = client.Request("user.getInfo", params)
    }
}
```

### 12.9.2 业务错误处理

```go
response, err := client.Request("user.getInfo", params)
if err == nil {
    // 网络请求成功，检查业务状态
    code := response.GetCode()
    if code != 0 {
        // 业务逻辑错误
        msg := response.GetMsg()
        log.Printf("业务错误 [%d]: %s", code, msg)
        return
    }
    
    // 处理正常响应
    data := response.GetData()
    // ...
}
```

### 12.9.3 流式请求错误处理

```go
iter, err := client.RequestStream("user.streamData", params)
if err != nil {
    log.Printf("流式请求失败: %v", err)
    return
}

err = iter.Foreach(func(res *rpc.Response) error {
    if res.GetCode() != 0 {
        // 流中的业务错误
        log.Printf("流式业务错误: %s", res.GetMsg())
        return nil  // 继续处理下一条
    }
    
    // 处理正常数据
    data := res.GetData()
    // ...
    return nil
})
```

## 12.10 日志记录

### 12.10.1 日志级别和输出

NyxClient 支持多级别日志记录：

1. **请求日志**: 记录所有请求的详细信息
2. **错误日志**: 专门记录错误信息

### 12.10.2 启用调试模式

```go
client, err := NewNyxClient(
    "localhost:8080",
    "your_appid",
    "your_secret",
    WithDebug(true),  // 启用调试模式
)
```

### 12.10.3 日志文件

默认情况下，日志文件会输出到：
- `logs/nyxclient-YYYY-MM-DD.log`: 一般日志
- `logs/nyxclient-err-YYYY-MM-DD.log`: 错误日志

### 12.10.4 集成 Nyx 框架日志

如果在 Nyx 框架项目中使用，NyxClient 会自动使用 `x.Logger` 的配置：

```go
// 在 Nyx 框架项目中
if x.Logger != nil {
    nyxclient.cache = x.LocalCache
    x.Info("nyxclient use x.LocalCache")
}
```

## 12.11 性能优化

### 12.11.1 连接池优化

```go
// 高并发场景的连接池配置
client, err := NewNyxClient(
    "localhost:8080",
    "your_appid",
    "your_secret",
    WithMaxIdleConns(50),   // 增加空闲连接
    WithMaxOpenConns(200),  // 增加最大连接
    WithDialTimeout(1*time.Second),  // 减少连接超时
)
```

### 12.11.2 缓存策略

```go
// 热点数据缓存策略
response, err := client.Request(
    "config.getSettings",
    params,
    WithRefreshCache(300),  // 5分钟自动刷新
)
```

### 12.11.3 并发请求

```go
// 并发请求示例
var wg sync.WaitGroup
responses := make(chan *rpc.Response, len(requests))

for i, req := range requests {
    wg.Add(1)
    go func(r Request, idx int) {
        defer wg.Done()
        
        resp, err := client.Request(r.Method, r.Params)
        if err != nil {
            log.Printf("请求 %d 失败: %v", idx, err)
            return
        }
        
        responses <- resp
    }(req, i)
}

wg.Wait()
close(responses)

// 处理所有响应
for resp := range responses {
    // 处理响应
}
```

## 12.12 最佳实践

### 12.12.1 客户端重用

```go
// 在应用启动时创建客户端，整个生命周期重用
var globalClient *NyxClient

func init() {
    client, err := NewNyxClient("localhost:8080", "appid", "secret")
    if err != nil {
        log.Fatal("Failed to create NyxClient:", err)
    }
    globalClient = client
}

func getUserInfo(userID int64) (*rpc.Response, error) {
    return globalClient.Request("user.getInfo", map[string]any{
        "user_id": userID,
    })
}
```

### 12.12.2 超时设置

```go
// 根据业务场景设置合适的超时时间
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

response, err := client.Request(
    "user.getInfo",
    params,
    WithContext(ctx),
    WithTimeout(2*time.Second),  // 请求超时2秒
)
```

### 12.12.3 错误重试机制

```go
// 带重试的请求函数
func requestWithRetry(client *NyxClient, method string, params any, maxRetries int) (*rpc.Response, error) {
    for i := 0; i < maxRetries; i++ {
        response, err := client.Request(method, params)
        if err == nil {
            return response, nil
        }
        
        // 网络错误才重试
        if isNetworkError(err) {
            time.Sleep(time.Duration(i+1) * time.Second)
            continue
        }
        
        // 业务错误不重试
        return nil, err
    }
    
    return nil, fmt.Errorf("max retries exceeded")
}
```

### 12.12.4 监控和指标

```go
// 记录请求性能指标
func requestWithMetrics(client *NyxClient, method string, params any) (*rpc.Response, error) {
    start := time.Now()
    
    response, err := client.Request(method, params)
    
    duration := time.Since(start)
    status := "success"
    if err != nil {
        status = "error"
    }
    
    // 记录性能指标
    log.Printf("method=%s duration=%v status=%s", method, duration, status)
    
    return response, err
}
```

## 12.13 完整示例

### 12.13.1 基础使用示例

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"
    
    "github.com/nyxless/nyx/x"
    "github.com/nyxless/nyxc"
    "github.com/nyxless/nyxc/rpc"
)

func main() {
    // 1. 创建客户端
    client, err := grpc.NewNyxClient(
        "localhost:8080",
        "demo_app",
        "demo_secret",
        grpc.UseCache(1000),                    // 启用缓存
        grpc.WithMaxIdleConns(10),             // 连接池配置
        grpc.WithMaxOpenConns(50),
        grpc.WithTimeout(5*time.Second),        // 请求超时
        grpc.WithDebug(true),                   // 启用调试
    )
    if err != nil {
        log.Fatal("创建客户端失败:", err)
    }
    defer client.Close()
    
    // 2. 同步请求示例
    fmt.Println("=== 同步请求示例 ===")
    
    // 简单参数请求
    response, err := client.Request("user.getInfo", map[string]any{
        "user_id": 123,
    })
    if err != nil {
        log.Printf("请求失败: %v", err)
    } else {
        fmt.Printf("响应代码: %d\n", response.GetCode())
        fmt.Printf("响应消息: %s\n", response.GetMsg())
        if response.HitCache {
            cacheTime, _ := response.GetCacheTime()
            fmt.Printf("命中缓存，时间: %v\n", cacheTime)
        }
        
        data := response.GetData()
        fmt.Printf("响应数据: %+v\n", data)
    }
    
    // 3. 带缓存的请求示例
    fmt.Println("\n=== 缓存请求示例 ===")
    
    response, err = client.Request(
        "config.getSettings",
        map[string]any{"config_key": "app_settings"},
        rpc.WithCache(60),  // 60秒缓存
    )
    if err != nil {
        log.Printf("配置获取失败: %v", err)
    } else {
        fmt.Printf("配置数据: %+v\n", response.GetData())
    }
    
    // 4. 流式请求示例
    fmt.Println("\n=== 流式请求示例 ===")
    
    streamIter, err := client.RequestStream(
        "data.streamProcessor",
        map[string]any{
            "batch_size": 100,
            "filter":     "active_users",
        },
    )
    if err != nil {
        log.Printf("流式请求失败: %v", err)
    } else {
        // 处理流数据
        count := 0
        err = streamIter.Foreach(func(res *rpc.Response) error {
            if res.GetCode() == 0 {
                data := res.GetData()
                fmt.Printf("流数据 %d: %+v\n", count+1, data)
                count++
            }
            return nil
        })
        
        if err != nil {
            log.Printf("流处理错误: %v", err)
        } else {
            fmt.Printf("共处理 %d 条流数据\n", count)
        }
    }
    
    // 5. 高级配置示例
    fmt.Println("\n=== 高级配置示例 ===")
    
    // 自定义上下文的请求
    ctx := context.Background()
    ctx = context.WithValue(ctx, "guid", x.GetUUID())
    
    response, err = client.Request(
        "user.updateProfile",
        map[string]any{
            "user_id":   123,
            "user_name": "新用户名",
            "email":     "newemail@example.com",
        },
        rpc.WithContext(ctx),
        rpc.WithTimeout(10*time.Second),
        rpc.WithHeaders(map[string]string{
            "X-Client-Version": "1.0.0",
            "X-Request-ID":     "req-456",
        }),
    )
    
    if err != nil {
        log.Printf("更新请求失败: %v", err)
    } else {
        fmt.Printf("更新结果: %+v\n", response.GetData())
    }
    
    fmt.Println("=== NyxClient 示例完成 ===")
}
```

### 12.13.2 错误处理和重试示例

```go
package retry

import (
    "context"
    "fmt"
    "log"
    "net"
    "strings"
    "time"
    
    "github.com/nyxless/nyxc/rpc"
    "google.golang.org/grpc/status"
    "google.golang.org/grpc/codes"
)

type RetryConfig struct {
    MaxRetries int
    BaseDelay time.Duration
    MaxDelay  time.Duration
}

func RequestWithRetry(client *rpc.NyxClient, method string, params any, config RetryConfig) (*rpc.Response, error) {
    var lastErr error
    
    for attempt := 0; attempt <= config.MaxRetries; attempt++ {
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        
        response, err := client.Request(
            method,
            params,
            rpc.WithContext(ctx),
            rpc.WithTimeout(25*time.Second),
        )
        
        if err == nil {
            // 请求成功
            if attempt > 0 {
                fmt.Printf("请求成功，尝试次数: %d\n", attempt+1)
            }
            return response, nil
        }
        
        lastErr = err
        
        // 判断是否为可重试的错误
        if !isRetryableError(err) {
            fmt.Printf("不可重试的错误: %v\n", err)
            return nil, err
        }
        
        // 最后一次尝试失败
        if attempt == config.MaxRetries {
            fmt.Printf("达到最大重试次数 (%d)，最后错误: %v\n", config.MaxRetries, lastErr)
            return nil, lastErr
        }
        
        // 计算延迟时间 (指数退避)
        delay := time.Duration(1<<uint(attempt)) * config.BaseDelay
        if delay > config.MaxDelay {
            delay = config.MaxDelay
        }
        
        fmt.Printf("请求失败 (尝试 %d/%d): %v，%v 后重试\n", attempt+1, config.MaxRetries+1, err, delay)
        time.Sleep(delay)
    }
    
    return nil, lastErr
}

func isRetryableError(err error) bool {
    // 网络相关错误可重试
    if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
        return true
    }
    
    // gRPC 状态错误
    if grpcStatus, ok := status.FromError(err); ok {
        switch grpcStatus.Code() {
        case codes.Unavailable, codes.DeadlineExceeded, codes.Internal:
            return true
        }
    }
    
    // 连接池满错误
    if strings.Contains(err.Error(), "pool is full") {
        return true
    }
    
    return false
}
```

## 12.14 总结

NyxClient 是一个功能完整、高性能的 gRPC 客户端库，它提供了：

1. **连接池管理**: 高效的连接复用和资源管理
2. **本地缓存**: 支持多种缓存策略，减少服务端压力
3. **认证机制**: 自动处理请求认证，简化客户端开发
4. **流式支持**: 完整支持 gRPC 流式请求
5. **错误处理**: 完善的错误处理和重试机制
6. **性能优化**: 针对高并发场景的优化配置
7. **日志监控**: 详细的日志记录便于问题排查

通过合理配置和使用 NyxClient，开发者可以构建稳定、高性能的 RPC 客户端应用。
