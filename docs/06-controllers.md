# 06. 控制器详解
## 统一接口处理多种协议

控制器是 Nyx 框架的核心组件, 本章将深入讲解控制器的设计思想、实现原理和使用方法，让您掌握如何用统一的代码处理 HTTP、gRPC 请求。

## 目录结构
```
controller/     # 控制器基类实现
  controller.go # 控制器核心逻辑
  httpContainer.go # HTTP 参数容器
  rpcContainer.go # RPC 参数容器
  http.go       # HTTP 控制器实现
  rpc.go       # RPC 控制器实现
```

## 核心概念

### 1. 控制器基类 (Controller)
控制器基类是所有控制器的父类，提供了以下核心功能：

#### 1.1 控制器生命周期
```go
// 初始化方法，可在子控制器中重写, 框架自动调用
func (c *Controller) Init() {}

// 结束方法, 可在子控制器中重写, 框架自动调用
func (c *Controller) Final() {}
```

#### 1.2 上下文管理
```go
// 获取上下文值
func (c *Controller) GetCtx(key string) any

// 设置上下文值
func (c *Controller) SetCtx(key, value any)

// 设置请求标识符
func (c *Controller) SetGuid(guid string)

// 设置语言环境
func (c *Controller) SetLang(lang string)

// 在业务日志中添加自定义字段
func (c *Controller) AddLog(k string, v any)

// 在业务日志中删除字段(比如密码等敏感字段)
func (c *Controller) OmitLog(v ...string)
```

#### 1.3 响应处理
```go
// 根据捕获的错误获取需要返回的错误码、错误信息及数据
func (c *Controller) GetErrorResponse(err any) (int32, string, x.MAP)

// 格式化输出
func (c *Controller) RenderResponser(errno int32, errmsg string, retdata any) *x.ResponseData

// 获取响应数据
func (c *Controller) GetResponseData() (context.Context, *x.ResponseData, error)
```

### 2. HTTP 控制器 (HTTP)
HTTP 控制器扩展了基础控制器，专门处理 HTTP 请求：

#### 2.1 请求准备
```go
func (h *HTTP) Prepare(w http.ResponseWriter, r *http.Request, controller, action, group string)
```
该方法会：
- 初始化 HTTP 相关对象
- 设置模板引擎（如果启用）
- 设置 GUID 用于日志追踪
- 设置语言环境

#### 2.2 参数获取接口
控制器实现了 `requestContainer` 接口，提供了丰富的参数获取方法：

```go
// 获取所有参数
GetParams() x.MAP

// 获取单个参数
GetParam(key string) any

// 获取字符串参数
GetString(key string, defaultValues ...string) string

// 获取整数参数
GetInt(key string, defaultValues ...int) int
GetInt8(key string, defaultValues ...int8) int8
GetInt16(key string, defaultValues ...int16) int16
GetInt32(key string, defaultValues ...int32) int32
GetInt64(key string, defaultValues ...int64) int64

// 获取布尔参数
GetBool(key string, defaultValues ...bool) bool

// 获取浮点数参数
GetFloat(key string, defaultValues ...float64) float64
GetFloat32(key string, defaultValues ...float32) float32
GetFloat64(key string, defaultValues ...float64) float64

// 获取 JSON 参数
GetJsonMap(key string) x.MAP

// 获取数组参数
GetSlice(key string, separators ...string) []any
GetStringSlice(key string, separators ...string) []string
GetIntSlice(key string, separators ...string) []int
GetInt32Slice(key string, separators ...string) []int32
GetInt64Slice(key string, separators ...string) []int64

// 获取 Map 参数
GetMap(key string) x.MAP
GetStringMap(key string) x.MAPS
GetIntMap(key string) x.MAPI
GetMapSlice(key string) []x.MAP

// 获取其他类型参数
GetBytes(key string, defaultValues ...[]byte) []byte
GetTime(key string) time.Time

// 获取网络信息
GetIp() (ip string)
GetHeader(key string, defaultValues ...string) (ret string)
GetHeaders() x.MAPS
SetHeader(key, val string)
SetHeaders(headers x.MAPS)

// 响应输出
Render(data ...any)
RenderError(err any)
RenderStream(data any) error
```

#### 2.3 文件上传
```go
// 获取上传文件
func (h *HTTP) GetFile(key string) (multipart.File, *multipart.FileHeader, error)

// 获取请求体
func (h *HTTP) GetRequestBody() (rbody []byte, err error)

// 设置 POST 表单大小, 应该在 Init 方法中调用
func (h *HTTP) SetMaxPostSize(m int64)
```

#### 2.4 响应渲染
```go
// 渲染文本
func (h *HTTP) RenderText(res any)

// 渲染 HTTP 状态码
func (h *HTTP) RenderStatus(code int)

// 渲染文件下载
func (h *HTTP) RenderFile(rs io.ReadSeeker, filename string)

// 渲染 HTML 模板
func (h *HTTP) RenderHtml(files ...string)

// 重定向
func (h *HTTP) Redirect(url string, codes ...int)
```

#### 2.5 模板操作
```go
// 模板变量赋值
func (h *HTTP) Assign(vals ...any)
```

### 3. RPC 控制器 (RPC)
RPC 控制器专门处理 gRPC 请求：

#### 3.1 请求准备
```go
func (r *RPC) Prepare(ctx context.Context, params x.MAP, controller, action, group string, stream x.Stream)
```

#### 3.2 流式处理
```go
// 输出流式数据
func (r *RPC) RenderStream(data any) error
```

### 4. CLI 控制器 (CLI)
CLI 控制器用于命令行接口处理：

#### 4.1 请求准备
```go
func (c *CLI) Prepare(params url.Values, controller, action, group string)
```

#### 4.2 参数获取
CLI 控制器实现了 `requestContainer` 接口的子集，专注于基本的参数获取功能，支持所有数据类型转换方法。

## 快速开始

### 1. 创建控制器
在 `api` 包中创建新的控制器：

```go
package api

import (
    "github.com/nyxless/nyx/controller"
)

type UserController struct {
    controller.HTTP
}
```

### 2. 实现 Action 方法
Action 方法命名规则：`{ActionName}Action`

```go
func (this *UserController) ListAction() {
    // 获取请求参数
    page := this.GetInt("page", 1)
    size := this.GetInt("size", 20)
    
    // 在接口访问日志中追加字段
    this.AddLog("page", page)
    
    // 忽略敏感字段（如密码）
    this.OmitLog("password")
    
    // 业务逻辑...
    
    // 返回响应
    data := map[string]any{
        "list":  userList,
        "total": total,
        "page":  page,
    }
    
    // 返回 json 数据
    this.Render(data)
}

func (this *UserController) CreateAction() {
    // 获取 JSON 参数
    userData := this.GetJsonMap("user")
    
    // 获取文件
    file, header, err := this.GetFile("avatar")
    if err == nil {
        defer file.Close()
        // 处理文件...
    }
    
    // 业务逻辑...
   
    this.Render()
}
```

### 3. 错误处理示例
```go
func (this *UserController) ErrorAction() {
    // 抛出业务错误
    var err error

    // 不推荐 ❌
    // 错误信息应该在配置预定义的变量中
    if (errCondition) {
        err = x.NewErr(ERR_CODE, "参数无效")
    }
 
    // 不推荐 ❌
    if err != nil {
        this.RenderError(err)
        return
    }

    // 推荐的优雅实现方式，使用拦截器 + 预定义错误信息 ✅ 
    x.Interceptor(err == nil, x.ERR_SYSTEM, err)
}

// 完整的错误处理模式
func (this *UserController) CompleteAction() {
    // 获取请求参数
    userId := this.GetInt("user_id")
    name := this.GetString("name")
    
    // 参数验证
    if userId <= 0 {
        this.RenderError("用户ID无效")
        return
    }
    
    if name == "" {
        this.RenderError("用户名不能为空")
        return
    }
    
    // 在日志中添加额外信息
    this.AddLog("user_id", userId)
    this.AddLog("operation", "update_profile")
    
    // 业务逻辑...
    
    // 成功响应
    this.Render(map[string]any{
        "status": "success",
        "user_id": userId,
    })
}
```

### 4. 使用模板渲染
```go
func (this *UserController) ProfileAction() {
    // 准备模板数据
    user := getUserProfile()
    
    // 赋值给模板
    this.Assign("user", user)
    this.Assign("title", "用户资料")
    
    // 渲染模板
    // 默认使用：ControllerName_ActionName.html
    this.RenderHtml()
    
    // 或指定模板文件
    this.RenderHtml("user/profile.html")
}
```

## 注意事项

1. **Action 方法必须公开**（首字母大写）
2. **新增控制器或方法后，需要执行 nyx init 创建路由**
3. **参数获取的优先级**：JSON参数 > URL参数 > 默认值**
4. **数组参数支持多种格式**：
   - `key=1&key=2&key=3`（重复参数）
   - `key=1,2,3`（逗号分隔）
   - `key[]=1&key[]=2&key[]=3`（数组标记）
5. **模板渲染时的文件路径规则**：
   - 默认：`ControllerName_ActionName.html`
   - 指定文件：支持相对路径和绝对路径
   - 分组控制器：自动添加分组前缀
6. **文件上传注意事项**：
   - 需要设置合适的 `MaxPostSize`
   - 记得关闭文件句柄
   - 支持多文件上传
7. **日志记录最佳实践**：
   - 使用 `AddLog` 添加业务相关日志
   - 使用 `OmitLog` 隐藏敏感信息
   - 避免在日志中记录密码、Token等敏感数据

## 最佳实践

### 1. 控制器设计原则
- **单一职责**：每个控制器只处理特定的业务领域
- **方法命名**：遵循 `{ActionName}Action` 命名规范
- **参数验证**：在 Action 方法开始处进行参数验证

### 2. 错误处理最佳实践
- 使用预定义的错误码和错误信息
- 利用拦截器进行优雅的错误处理
- 在开发环境中保留详细的错误信息，生产环境隐藏敏感信息

### 3. 性能优化建议
- 合理使用缓存机制
- 避免在控制器中进行复杂的业务逻辑
- 及时释放资源（如文件句柄、数据库连接）

**下一步**：继续学习 [服务层详解](./07-services.md)
