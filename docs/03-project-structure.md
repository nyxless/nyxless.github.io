# 03. 项目结构与开发规范

在开始深入学习 Nyx 框架的具体功能之前，建立良好的项目结构和遵循统一的开发规范是至关重要的。本章将详细介绍 Nyx 项目的标准目录结构、代码组织方式以及开发过程中的最佳实践，帮助您构建规范、可维护的项目架构。

## 3.1 标准项目结构

### 3.1.1 完整的项目目录结构

一个规范的 Nyx 项目应该遵循以下目录结构：

```
my-nyx-project/
├── main.go                    # 程序入口点
├── go.mod                     # Go 模块定义
├── conf/                      # 配置文件目录
│   ├── app.common.conf       # 通用配置文件
│   ├── app.conf.dev          # 开发环境配置
│   ├── app.conf.test         # 测试环境配置
│   └── app.conf.prod         # 生产环境配置
├── controller/                # 控制器层
│   ├── api/                   # HTTP API 控制器
│   │   ├── base_controller.go # 基础控制器
│   │   └── user_controller.go # 用户控制器
│   └── rpc/                   # gRPC 控制器
│       └── user_controller.go # 用户 RPC 接口
├── svc/                      # 服务层 (业务逻辑)
│   └── user_svc.go           # 用户服务
├── dao/                      # 数据访问层
│   └── user_dao.go          # 用户数据层
├── model/                   # 数据模型
│   └── user.go              # 用户模型
├── middleware/              # 中间件
│   ├── auth.go              # 认证中间件
│   └── rate.go              # 限流中间件
├── utils/                   # 工具函数
│   ├── crypto.go            # 加密工具
│   ├── time.go              # 时间处理
│   └── validation.go        # 验证工具
├── tools/                   # 开发工具
│   ├── gen_orm.go          # ORM 代码生成器
│   ├── gen_dao             # DAO 生成器
│   └── test_rpc.go         # RPC 测试工具
├── static/                  # 静态资源
│   ├── css/                 # 样式文件
│   ├── js/                  # JavaScript 文件
│   └── images/              # 图片资源
├── docs/                    # 项目文档
│   ├── api/                 # API 文档
│   └── development.md      # 开发文档
├── tests/                   # 测试文件
│   ├── unit/               # 单元测试
│   ├── integration/        # 集成测试
│   └── fixtures/           # 测试数据
├── logs/                    # 日志文件 (运行时生成)
├── backup/                  # 自动备份目录
├── Dockerfile              # Docker 构建文件
├── docker-compose.yml      # Docker Compose 配置
├── Makefile               # 构建脚本
└── README.md              # 项目说明
```

### 3.1.2 核心目录详解

#### `main.go` - 程序入口
```go
package main

import (
    "github.com/nyxless/nyx"
)

func main() {
    app := nyx.NewNyx()
    app.Run()
}
```

#### `conf/` - 配置管理
- **app.conf**: 环境特定配置，包含环境标识、端口设置等
- **app.common.conf**: 通用配置，所有环境共享的设置

#### `controller/` - 请求处理层
- **api/**: 处理 HTTP 请求的控制器
- **rpc/**: 处理 gRPC 请求的控制器

#### `service/` - 业务逻辑层
- 包含核心业务逻辑实现
- 协调 DAO 和模型层

#### `dao/` - 数据访问层
- 数据库操作封装
- 支持读写分离和事务

## 3.2 命名规范

### 3.2.1 文件命名规范

#### Go 文件命名
- **控制器文件**: `{name}_controller.go`
  - `user_controller.go`
  - `order_controller.go`

- **服务文件**: `{name}.go`
  - `user_service.go`
  - `auth_service.go`

- **DAO 文件**: `{name}_dao.go`
  - `user_dao.go`
  - `order_dao.go`

- **模型文件**: `{name}.go`
  - `user.go`
  - `order.go`

#### 目录命名
- **复数形式**: `controller/`, `service/`, `dao/`
- **小写字母**: `middleware/`, `utils/`

### 3.2.2 变量和函数命名

#### 变量命名
```go
// 推荐使用驼峰命名法
var UserID int64
var UserName string
var IsActive bool

// 避免缩写
var UID int64          // 不推荐
var UserIdentifier int64 // 推荐
```

#### 函数命名
```go
// 控制器方法命名：{操作}Action
func (c *UserController) CreateUserAction() {}
func (c *UserController) UpdateUserAction() {}
func (c *UserController) DeleteUserAction() {}

// 服务方法命名：驼峰命名
func (s *UserService) CreateUser(user *model.User) error {}
func (s *UserService) GetUserByID(id int64) (*model.User, error) {}

// DAO 方法命名：驼峰命名 + 描述性后缀
func (d *UserDao) Insert(user *model.User) error {}
func (d *UserDao) FindByID(id int64) (*model.User, error) {}
func (d *UserDao) UpdateByID(id int64, updates map[string]interface{}) error {}
```

## 3.3 代码组织规范

### 3.3.1 控制器层组织

#### 基础控制器结构
```go
type BaseController struct {
    controller.HTTP  // HTTP 基础功能
    Auth *service.Auth // 认证服务
    UserID int64    // 当前用户 ID
}

// Init 方法初始化控制器
func (c *BaseController) Init() {
    c.Auth = service.NewAuth(c.W, c.R)
    c.Interceptor(c.Auth.CheckLogin(), common.ERR_TOKEN)
    c.UserID = c.Auth.GetUserID()
}
```

#### 业务控制器实现
```go
type UserController struct {
    BaseController    // 继承基础控制器
    UserService *service.UserService
}

func (c *UserController) Init() {
    c.BaseController.Init()
    c.UserService = service.NewUserService()
}

func (c *UserController) CreateUserAction() {
    // 参数验证
    username := c.GetParam("username")
    email := c.GetParam("email")
    
    if username == "" || email == "" {
        c.RenderJSON(common.ERR_PARAM, "用户名和邮箱不能为空")
        return
    }
    
    // 业务逻辑
    user := &model.User{
        Username: username,
        Email:    email,
    }
    
    err := c.UserService.CreateUser(user)
    if err != nil {
        c.RenderJSON(common.ERR_SYSTEM, err.Error())
        return
    }
    
    c.RenderJSON(common.SUCCESS, user)
}
```

### 3.3.2 服务层组织

#### 服务接口定义
```go
type UserService interface {
    CreateUser(user *model.User) error
    GetUserByID(id int64) (*model.User, error)
    UpdateUser(id int64, updates map[string]interface{}) error
    DeleteUser(id int64) error
}
```

#### 服务实现
```go
type UserServiceImpl struct {
    UserDao   *dao.UserDao
    Validator *validation.Validator
}

func NewUserService() UserService {
    return &UserServiceImpl{
        UserDao:   dao.NewUserDao(),
        Validator: validation.NewValidator(),
    }
}

func (s *UserServiceImpl) CreateUser(user *model.User) error {
    // 数据验证
    if err := s.Validator.Validate(user); err != nil {
        return err
    }
    
    // 执行业务逻辑
    return s.UserDao.Insert(user)
}
```

### 3.3.3 DAO 层组织

#### 基础 DAO 结构
```go
type UserDao struct {
    dao.Dao
}

func NewUserDao() *UserDao {
    dao := &UserDao{}
    dao.Init()
    dao.SetTable("users")
    dao.SetPrimary("id")
    return dao
}

// 插入用户
func (d *UserDao) Insert(user *model.User) error {
    return d.InsertRecord(user)
}

// 根据 ID 查询用户
func (d *UserDao) FindByID(id int64) (*model.User, error) {
    user := &model.User{}
    err := d.FindByPrimary(id, user)
    return user, err
}
```

## 3.4 配置管理规范

### 3.4.1 配置文件组织

#### app.common.conf - 通用配置
```toml
# 数据库配置
[db_master]
type = "mysql"
host = "${DB_HOST:localhost:3306}"
username = "${DB_USERNAME:root}"
password = "${DB_PASSWORD:password}"
database = "${DB_DATABASE:myapp}"

[db_slave]
type = "mysql"
host = "${DB_SLAVE_HOST:localhost:3306}"
username = "${DB_USERNAME:root}"
password = "${DB_PASSWORD:password}"
database = "${DB_DATABASE:myapp}"

# Redis 配置
[redis]
host = "${REDIS_HOST:localhost:6379}"
password = "${REDIS_PASSWORD:}"
db = 0

# 日志配置
[log]
level = "${LOG_LEVEL:info}"
file = "logs/app.log"
max_size = 100
max_age = 30
max_backups = 10
```

#### app.conf - 环境特定配置
```toml
!include ../conf/app.common.conf

# 环境标识
env_mode = "dev"  # dev, test, prod

# 应用配置
app_name = "myapp"
debug = true
port = 8080

# 认证配置
[jwt]
secret = "${JWT_SECRET:your-secret-key}"
expire = 86400

# RPC 客户端配置
[rpc_client_user_service]
host = "${USER_SERVICE_HOST:localhost:8081}"
appid = "${USER_SERVICE_APPID:user_service}"
secret = "${USER_SERVICE_SECRET:secret123}"
```

### 3.4.2 环境变量使用

#### 配置优先级
1. **命令行参数**: 最高优先级
2. **环境变量**: 中等优先级
3. **配置文件**: 默认值

#### 示例配置使用
```go
// 在代码中读取配置
func GetDatabaseConfig() {
    host := x.Conf.GetString("db_master.host")
    username := x.Conf.GetString("db_master.username")
    password := x.Conf.GetString("db_master.password")
    
    // 环境变量示例
    dbHost := os.Getenv("DB_HOST")
    if dbHost != "" {
        host = dbHost
    }
}
```

## 3.5 数据库开发规范

### 3.5.1 表设计规范

#### 通用字段
```sql
-- 所有业务表都应该包含这些字段
CREATE TABLE `users` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
    `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    `deleted_at` TIMESTAMP NULL DEFAULT NULL COMMENT '删除时间 (软删除)',
    
    -- 业务字段
    `username` VARCHAR(50) NOT NULL COMMENT '用户名',
    `email` VARCHAR(100) NOT NULL COMMENT '邮箱',
    `status` TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1-正常，0-禁用',
    
    PRIMARY KEY (`id`),
    UNIQUE KEY `username` (`username`),
    UNIQUE KEY `email` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

#### 索引规范
```sql
-- 主键索引 (自动创建)
-- 唯一索引：确保数据唯一性
-- 普通索引：提高查询性能
-- 复合索引：多字段查询优化

-- 示例：用户表索引
CREATE INDEX `idx_status_created` ON `users` (`status`, `created_at`);
CREATE INDEX `idx_email_status` ON `users` (`email`, `status`);
```

### 3.5.2 ORM 使用规范

#### 模型定义
```go
type User struct {
    ID        int64     `json:"id" xorm:"id"`
    Username  string    `json:"username" xorm:"username"`
    Email     string    `json:"email" xorm:"email"`
    Status    int       `json:"status" xorm:"status"`
    CreatedAt time.Time `json:"created_at" xorm:"created_at"`
    UpdatedAt time.Time `json:"updated_at" xorm:"updated_at"`
    DeletedAt *time.Time `json:"deleted_at" xorm:"deleted_at"`
}

func (User) TableName() string {
    return "users"
}

// 软删除查询
func (d *UserDao) FindActiveUsers() ([]*User, error) {
    var users []*User
    err := d.Where("deleted_at IS NULL").Find(&users)
    return users, err
}
```

## 3.6 API 设计规范

### 3.6.1 RESTful API 设计

#### 路由命名规范
```
GET    /api/users           # 获取用户列表
GET    /api/users/{id}      # 获取单个用户
POST   /api/users           # 创建用户
PUT    /api/users/{id}      # 更新用户
DELETE /api/users/{id}      # 删除用户
```

#### 响应格式规范
```go
// 统一响应结构
type APIResponse struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
    Meta    *Meta       `json:"meta,omitempty"`
}

type Meta struct {
    Page     int `json:"page"`
    PerPage  int `json:"per_page"`
    Total    int `json:"total"`
    LastPage int `json:"last_page"`
}
```

#### 控制器实现
```go
func (c *UserController) GetUsersAction() {
    // 分页参数
    page := c.GetInt("page", 1)
    perPage := c.GetInt("per_page", 20)
    
    // 业务逻辑
    users, total, err := c.UserService.GetUsers(page, perPage)
    if err != nil {
        c.RenderJSON(common.ERR_SYSTEM, err.Error())
        return
    }
    
    // 响应数据
    meta := &common.Meta{
        Page:     page,
        PerPage:  perPage,
        Total:    total,
        LastPage: (total + perPage - 1) / perPage,
    }
    
    c.RenderJSON(common.SUCCESS, users, meta)
}
```

## 3.7 测试规范

### 3.7.1 测试文件组织

```
tests/
├── unit/                    # 单元测试
│   ├── controller/         # 控制器测试
│   ├── service/           # 服务测试
│   └── dao/               # DAO 测试
├── integration/            # 集成测试
│   ├── api/               # API 测试
│   └── db/                # 数据库测试
├── fixtures/               # 测试数据
│   └── users.json         # 用户测试数据
└── helpers/                # 测试辅助函数
    └── test_helper.go     # 测试工具函数
```

### 3.7.2 单元测试示例

#### 服务层测试
```go
func TestUserService_CreateUser(t *testing.T) {
    // 准备测试数据
    user := &model.User{
        Username: "testuser",
        Email:    "test@example.com",
    }
    
    // 创建模拟 DAO
    mockDAO := &MockUserDao{}
    service := NewUserServiceWithDAO(mockDAO)
    
    // 执行测试
    err := service.CreateUser(user)
    
    // 断言结果
    assert.NoError(t, err)
    assert.True(t, mockDAO.InsertCalled)
    assert.Equal(t, user.Username, mockDAO.InsertedUser.Username)
}
```

#### 控制器测试
```go
func TestUserController_CreateUserAction(t *testing.T) {
    // 创建测试控制器
    controller := NewTestUserController()
    controller.SetParam("username", "testuser")
    controller.SetParam("email", "test@example.com")
    
    // 执行测试
    controller.CreateUserAction()
    
    // 检查响应
    assert.Equal(t, common.SUCCESS, controller.GetResponseCode())
    assert.NotNil(t, controller.GetResponseData())
}
```

## 3.8 版本控制规范

### 3.8.1 Git 提交规范

#### 提交信息格式
```
<type>(<scope>): <subject>

<body>

<footer>
```

#### 类型定义
- **feat**: 新功能
- **fix**: 修复 bug
- **docs**: 文档更新
- **style**: 代码格式调整
- **refactor**: 代码重构
- **test**: 测试相关
- **chore**: 构建工具或辅助工具

#### 提交示例
```bash
feat(user): 添加用户头像上传功能

1. 实现头像上传接口
2. 添加图片格式验证
3. 集成 CDN 存储

关闭 issue #123
```

### 3.8.2 分支管理

#### 分支命名规范
- **master**: 主分支，生产环境代码
- **develop**: 开发分支，集成功能
- **feature/**: 功能分支，新功能开发
- **hotfix/**: 紧急修复分支
- **release/**: 发布分支

#### 工作流程
```bash
# 开发新功能
git checkout develop
git pull origin develop
git checkout -b feature/user-management

# 开发完成
git add .
git commit -m "feat(user): 完成用户管理功能"
git push origin feature/user-management

# 合并到开发分支
git checkout develop
git merge feature/user-management
git branch -d feature/user-management
```

## 3.9 文档规范

### 3.9.1 代码注释

#### 函数注释
```go
// GetUserByID 根据用户 ID 获取用户信息
// 
// 参数:
//   id - 用户 ID
//
// 返回:
//   *model.User - 用户信息
//   error - 错误信息
func (s *UserService) GetUserByID(id int64) (*model.User, error) {
    // 实现逻辑
}
```

#### 结构体注释
```go
// User 用户模型
// 
// 包含用户基本信息，如用户名、邮箱、状态等
type User struct {
    ID       int64  `json:"id" xorm:"id"`
    Username string `json:"username" xorm:"username"`
    Email    string `json:"email" xorm:"email"`
    Status   int    `json:"status" xorm:"status"`
}
```

### 3.9.2 API 文档

#### 文档注释
```go
// CreateUser 创建新用户
// 
// @Summary 创建新用户
// @Description 创建新的用户账户
// @Tags users
// @Accept json
// @Produce json
// @Param user body CreateUserRequest true "用户信息"
// @Success 200 {object} User "创建成功"
// @Failure 400 {object} ErrorResponse "参数错误"
// @Failure 500 {object} ErrorResponse "系统错误"
// @Router /api/users [post]
func (c *UserController) CreateUserAction() {
    // 实现逻辑
}
```

## 3.10 部署规范

### 3.10.1 Docker 化

#### Dockerfile
```dockerfile
# 多阶段构建
FROM golang:1.20-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# 生产环境镜像
FROM alpine:latest

RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /app/main .
COPY --from=builder /app/conf ./conf

EXPOSE 8080
CMD ["./main"]
```

#### docker-compose.yml
```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=mysql
      - REDIS_HOST=redis
    depends_on:
      - mysql
      - redis
    volumes:
      - ./logs:/app/logs

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: myapp
    volumes:
      - mysql_data:/var/lib/mysql
      - ./sql/init:/docker-entrypoint-initdb.d

  redis:
    image: redis:7-alpine

volumes:
  mysql_data:
```

### 3.10.2 环境配置

#### 生产环境变量
```bash
# .env.production
APP_ENV=production
APP_DEBUG=false
APP_PORT=8080

# 数据库配置
DB_HOST=prod-db.example.com:3306
DB_USERNAME=app_user
DB_PASSWORD=secure_password
DB_DATABASE=myapp_prod

# Redis 配置
REDIS_HOST=prod-redis.example.com:6379
REDIS_PASSWORD=redis_password

# JWT 配置
JWT_SECRET=super_secret_jwt_key
JWT_EXPIRE=86400

# 日志配置
LOG_LEVEL=warn
LOG_FILE=/app/logs/app.log
```

## 总结

建立良好的项目结构和遵循统一的开发规范是成功项目的基础。通过本章的介绍，您应该能够：

1. **建立标准项目结构**: 了解完整的目录组织和文件命名规范
2. **遵循代码规范**: 掌握变量、函数、文件的命名约定
3. **组织代码层次**: 理解 MVC 架构各层的职责分工
4. **管理配置**: 熟练使用多环境配置管理
5. **设计数据库**: 遵循表设计和 ORM 使用规范
6. **构建 API**: 设计规范的 RESTful API
7. **编写测试**: 建立完整的测试体系
8. **版本控制**: 使用规范的 Git 工作流
9. **编写文档**: 为代码和 API 添加清晰文档
10. **部署应用**: 容器化部署和环境配置

这些规范和最佳实践将帮助您构建高质量、可维护的 Nyx 项目。建议在项目开始时就建立这些规范，确保整个开发过程的一致性和规范性。
