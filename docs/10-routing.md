# 10. 路由系统

Nyx 框架提供了强大而智能的路由系统(HTTP)，支持自动化路由生成和灵活的配置化路由规则。框架能够根据 URL 路径自动映射到控制器文件和方法，同时支持通过配置文件进行路由重写。

## 自动化路由规则

### 核心原理

框架遵循约定优于配置的原则，通过 URL 路径自动映射到控制器文件和方法：**URL 路径直接对应控制器文件的目录结构和控制器方法**。

### 映射规则

#### 基础映射
```
URL路径: admin/user/getUserInfo
映射到: example_app/controller/api/admin/user_controller.go
调用方法: UserController.GetUserInfoAction()
```

#### 完整映射示例

| URL 路径 | 文件路径 | 控制器方法 |
|---------|----------|-----------|
| `index/index` | `controller/api/index_controller.go` | `IndexController.IndexAction()` |
| `user/getUserInfo` | `controller/api/user_controller.go` | `UserController.GetUserInfoAction()` |
| `admin/user/getUserInfo` | `controller/api/admin/user_controller.go` | `UserController.GetUserInfoAction()` |

#### 方法命名规则
- **控制器方法**：控制器名 + `Action`（如 `UserController` → `GetUserInfoAction`）

### 分组支持

#### 单级分组
```
URL: admin/user/getUserInfo
分组: admin
控制器: UserController
方法: GetUserInfoAction
```

#### 多级分组
```
URL: v2/admin/user/getUserInfo
分组: v2/admin
控制器: UserController  
方法: GetUserInfoAction

对应文件: controller/api/v2/admin/user_controller.go
```

#### 分组优先级规则
当 URL 路径的分组部分和控制器名称存在歧义时，**优先匹配已存在的分组目录**。

### 自动化路由示例

#### 项目结构
```
example_app/
├── controller/
│   └── api/
│       ├── index_controller.go      # IndexController (默认分组)
│       ├── user_controller.go       # UserController  (默认分组)
│       └── admin/
│           ├── user_controller.go    # UserController (admin分组)
│           ├── post_controller.go    # PostController (admin分组)
│           └── v2/
│               └── user_controller.go     # UserController (admin/v2分组)
```

#### 对应路由映射

**基础路由**
- `GET /index` → `IndexController.IndexAction()`
- `GET /user/getUserInfo` → `UserController.GetUserInfoAction()`
- `POST /user/createUser` → `UserController.CreateUserAction()`

**分组路由**
- `GET /admin/user/getUserInfo` → `api/admin/UserController.GetUserInfoAction()`
- `GET /admin/dashboard` → `admin/DashboardController.DashboardAction()`
- `GET /v2/user/list` → `v2/UserController.ListAction()`

### 参数传递

查询参数自动绑定到控制器属性：

```
URL: /user/list?page=1&limit=10
映射: UserController.ListAction()

在方法中接收:
func (this *UserController) ListAction() {
    page := this.GetParam("page")    // "1"
    limit := this.GetParam("limit")  // "10"
}
```

## 配置路由规则

除了自动化路由，框架还支持通过配置文件进行路由重写和自定义映射。

### 配置格式

路由配置通过 `url_route` 字段在配置文件中定义：

```yaml
url_route:
  - prefix: ""                    # 前缀匹配规则
    group_rule:                   # 组规则配置
      - from: ""                 # 源路径前缀
        to: ""                   # 目标路径前缀
    path_rule:                    # 路径规则配置
      - from: ""                 # 源路径
        to: ""                   # 目标处理程序
        method: []               # 允许的 HTTP 方法
```

### 配置元素说明

#### prefix（前缀匹配）
- **作用**：定义需要匹配的前缀路径
- **行为**：只有 URL 匹配到此前缀时，才执行后续的路由规则

#### group_rule（组规则）
- **作用**：进行路径前缀替换
- **逻辑**：`prefix + from/*` => `prefix + to/*`

#### path_rule（路径规则）
- **作用**：进行完整的路径匹配替换
- **逻辑**：`prefix + from` => `to`

### 配置示例

#### 1. 基础路由重写

```yaml
url_route:
  - prefix: ""                    # 影响所有路径
    group_rule:
      - from: ""                 # 源前缀为空
        to: ""                   # 目标前缀为空
    path_rule:
      - from: /                  # 根路径
        to: index/index          # 重写到 index 控制器的 index 方法
```

#### 2. API 版本控制

```yaml
url_route:
  - prefix: /api/v1              # 匹配 /api/v1 前缀
    group_rule:
      - from: /                  # 将 /api/v1/ 开头的路径
        to: /                    # 映射到无前缀的控制器路径
    path_rule:
      - from: /info             # /api/v1/info
        to: user/GetUserInfo    # 映射到 user 组的 GetUserInfo 方法
      - from: /users/@user_id   # 支持路径参数
        to: user/GetUserInfo    
        method: [GET,POST]      # 允许 GET 和 POST 方法
```

#### 3. 复杂路由映射

```yaml
url_route:
  # 管理后台路由
  - prefix: /admin
    group_rule:
      - from: /                  # /admin/*
        to: /                   # 映射到根路径
    path_rule:
      - from: /dashboard         # /admin/dashboard
        to: admin/Dashboard      # admin 分组的 Dashboard 控制器
        method: GET
      - from: /users/@user_id    # 路径参数
        to: admin/GetUserInfo
        method: [GET,POST,PUT]
        
  # 公开 API 路由
  - prefix: /public
    group_rule:
      - from: /
        to: /
    path_rule:
      - from: /health
        to: public/HealthCheck
        method: GET
```

## 路由优先级

### 优先级顺序
1. **配置路由规则**：优先匹配配置文件中的路由规则
2. **自动化路由**：如果没有配置规则或未匹配，则使用自动化路由

### 冲突处理
- 如果配置路由和自动化路由产生冲突，配置路由优先
- 可以通过禁用配置规则来强制使用自动化路由

```yaml
url_route:
  - prefix: /api/v1
    enabled: false             # 禁用此规则，使用自动化路由
    path_rule:
      - from: /users
        to: custom/UserList
```

## 路径参数支持

### 参数语法
- **单参数**：`@参数名`
- **多参数**：连续使用多个 `@参数名`

### 参数传递
路径参数通过 `this.GetParam("参数名")` 在控制器方法中获取：

```yaml
url_route:
  - prefix: /api/v1
    path_rule:
      - from: /users/@user_id          # 单参数
        to: user/GetUserInfo
      - from: /users/@uid/@name       # 多参数
        to: user/GetUserDetail
```

```go
func (this *UserController) GetUserInfoAction() {
    // 获取路径参数
    userId := this.GetParam("user_id")
    
    // 参数类型转换
    userIdInt := this.GetParamInt("user_id", 0)
    userIdStr := this.GetParamString("user_id", "")
}

func (this *UserController) GetUserDetailAction() {
    // 获取多个参数
    uid := this.GetParam("uid")
    name := this.GetParam("name")
}
```

## HTTP 方法限制

### 配置方法限制
```yaml
url_route:
  - prefix: /api/v1
    path_rule:
      - from: /users/@user_id
        to: user/GetUserInfo
        method: [GET]              # 只允许 GET 方法
      
      - from: /users
        to: user/CreateUser
        method: [POST]             # 只允许 POST 方法
      
      - from: /users/@user_id
        to: user/UpdateUser
        method: [PUT,PATCH]       # 允许 PUT 和 PATCH 方法
```

### 支持的 HTTP 方法
- `GET` - 查询数据
- `POST` - 创建数据
- `PUT` - 更新数据（完整更新）
- `PATCH` - 更新数据（部分更新）
- `DELETE` - 删除数据
- `HEAD` - 获取头部信息
- `OPTIONS` - 获取支持的 HTTP 方法

## 路由处理流程

### 1. 接收请求
```go
// 框架接收 HTTP 请求
func (n *Nyx) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 解析路由
    group, controller, action, params := x.ParseRoute(r.URL.Path, r.Method)
    
    // 匹配控制器
    controllerInstance := n.getController(group, controller)
    
    // 调用方法
    controllerInstance.CallAction(action, params)
}
```

### 2. 路由解析
```go
// ParseRoute 函数逻辑
func ParseRoute(uri, method string) (group, controller, action string, params MAPS) {
    // 1. 检查配置路由规则
    if configRoute := matchConfigRoute(uri, method); configRoute != nil {
        return configRoute.group, configRoute.controller, configRoute.action, configRoute.params
    }
    
    // 2. 使用自动化路由
    return parseAutoRoute(uri)
}
```

### 3. 控制器调用
```go
// 控制器方法调用
func (c *Controller) CallAction(action string, params MAPS) {
    methodName := action + "Action"
    
    // 反射调用方法
    method := reflect.ValueOf(c).MethodByName(methodName)
    if method.IsValid() {
        method.Call([]reflect.Value{})
    }
}
```

## 最佳实践

### 1. 目录结构规范
```
controller/
 └─ api/
   ├── user/                 # 用户相关控制器
   │   ├── user_controller.go
   │   └── profile_controller.go
   ├── admin/                # 管理后台控制器
   │   ├── dashboard_controller.go
   │   └── user_controller.go
   └── v1/              # API 版本控制
       ├── user_controller.go
       └── order_controller.go
```

### 2. 命名规范
- **控制器文件**：使用下划线命名，如 `user_controller.go`
- **控制器结构**：使用驼峰命名，如 `UserController`
- **方法名**：使用驼峰命名， Action 结尾, 如 `GetUserInfoAction`

### 3. 分组策略
```yaml
# 推荐的分组策略
url_route:
  # API 版本控制
  - prefix: /api/v1
    path_rule:
      - from: /users
        to: api/v1/user/ListUser
        
  # 功能分组
  - prefix: /admin
    path_rule:
      - from: /users
        to: admin/user/ManageUser
```

### 4. 安全性考虑
```yaml
# 限制敏感操作的 HTTP 方法
url_route:
  - prefix: /admin
    path_rule:
      - from: /users/@user_id
        to: admin/DeleteUser
        method: [DELETE]          # 只允许删除方法
      - from: /settings
        to: admin/UpdateSettings
        method: [POST,PUT]        # 只允许修改方法
```

## 路由测试
```go
// 测试路由解析
func TestRouteParsing() {
    testCases := []struct {
        uri      string
        method   string
        expected string
    }{
        {"admin/user/getUserInfo", "GET", "admin.UserController.GetUserInfoAction"},
        {"api/v2/user/list", "GET", "api.v2.UserController.ListAction"},
    }
    
    for _, tc := range testCases {
        group, controller, action, _ := x.ParseRoute(tc.uri, tc.method)
        result := fmt.Sprintf("%s.%s.%s", group, controller, action+"Action")
        
        if result == tc.expected {
            fmt.Printf("✅ %s: %s\n", tc.uri, result)
        } else {
            fmt.Printf("❌ %s: 期望 %s, 实际 %s\n", tc.uri, tc.expected, result)
        }
    }
}
```

## 性能优化

### 1. 路由缓存
- 框架会自动缓存路由解析结果
- 相同 URL 的后续请求会直接使用缓存

### 2. 控制器实例池
- 控制器实例会被复用
- 避免频繁创建和销毁对象

### 3. 反射优化
- 预编译控制器方法映射
- 减少运行时反射开销

## 总结

Nyx 框架的路由系统提供了：

1. **智能化自动路由**：URL 路径直接映射到控制器文件和方法，开箱即用
2. **灵活的配置路由**：支持复杂的路由重写和自定义映射
3. **强大的分组支持**：支持多级分组目录，自动处理命名冲突
4. **参数化路由**：原生支持路径参数和查询参数绑定
5. **高性能解析**：优化的路由查找和匹配算法

这套路由系统既保持了约定优于配置的简洁性，又提供了足够的灵活性来满足复杂应用的需求。
