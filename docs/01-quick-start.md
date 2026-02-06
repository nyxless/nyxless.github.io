# 01. 快速上手
## 5分钟创建你的第一个 Nyx 项目

欢迎使用 Nyx 框架！本指南将带你通过 Nyx 脚手架工具，在 5 分钟内创建并运行你的第一个 "Hello World" 应用。我们将创建 HTTP 服务和 RPC 服务，让你体验 Nyx 框架的强大功能。

## 环境准备

### 系统要求

- Go 1.23+ 
- 支持的操作系统：Linux、Windows、macOS

## 第一步：安装 Nyx 脚手架

Nyx 提供了一个强大的脚手架工具，可以自动生成项目结构和基础代码。

### 安装命令

```bash
go install github.com/nyxless/nyx/cmd/nyx@latest
```

### 验证安装

```bash
nyx version
```

你应该看到类似以下的版本信息：

```
Nyx 脚手架版本: 1.0.0
```

## 第二步：创建你的第一个项目

现在让我们使用脚手架创建第一个 Nyx 项目：

### 创建项目

创建名为 demo 的项目
```bash
nyx create demo
```

脚手架将自动为你生成完整的项目结构：

```
demo/
├── main.go                 # 应用入口文件
├── go.mod                  # Go 模块文件
├── conf/                   # 配置文件目录
│   ├── app.common.conf     # 通用配置
│   ├── app.conf.dev        # 开发环境配置
│   └── app.conf.prod       # 生产环境配置
├── controller/             # 控制器层
│   │── api/
│   │    └── test_controller.go # 测试控制器（包含 hello 接口）
│   └── rpc/
│        └── test_controller.go # 测试控制器（包含 hello 接口）
├── svc/                # 服务层
│   └── test_service.go    # 测试服务
├── dao/                    # 数据访问层
│   └── test_dao.go        # 测试 DAO
└── model/                  # 数据模型
    └── test_model.go      # 测试模型
```

### 初始化 Go 模块

进入项目目录并初始化 Go 模块：

```bash
cd demo 
go mod init demo 
```

### 安装依赖

运行以下命令安装：

```bash
go mod tidy
```

## 第三步：运行 HTTP 服务

### 启动 HTTP 服务

使用 `nyx run` 命令启动 HTTP 服务：

```bash
nyx run
```

或者明确指定 HTTP 模式：

```bash
nyx run -m http
```

### 测试 Hello World 接口

打开浏览器或使用 curl 访问测试接口：

```bash
curl http://localhost:8080/test/hello
```

你将看到：

```
Hello, World!
```

## 第四步：运行 RPC 服务

### 启动 RPC 服务

在新的终端窗口中，进入项目目录，启动 RPC 服务：

```bash
nyx run -m rpc
```

RPC 服务默认运行在端口 8081：

### 测试 RPC 服务

在新的终端窗口中，进入项目目录下的 tools 目录：

```bash
cd tools
go run test_rpc.go
```
 你将看到 grpc 服务返回的接口信息


## 核心概念

### 1. MVC 架构

Nyx 采用经典的 MVC 架构：

- **Model（模型）**：在 `model/` 目录中定义数据结构
- **Controller（控制器）**：在 `controller/` 目录中处理请求和响应
- **Service（服务）**：在 `service/` 目录中封装业务逻辑
- **DAO（数据访问对象）**：在 `dao/` 目录中处理数据持久化

### 2. 统一协议支持

Nyx 框架支持多种协议，统一处理：

- **HTTP**：Web API 和 RESTful 服务
- **RPC**：gRPC 微服务通信
- **WebSocket**：实时双向通信
- **TCP**：自定义协议通信
- **CLI**：命令行界面

### 3. 配置系统

Nyx 使用 YAML 格式配置文件，支持：

- 环境分离（dev、prod、test）
- 配置包含（`!include` 指令）
- 使用环境变量


### 4. 中间件机制

Nyx 框架默认提供了以下中间件：

- 日志记录
- 身份验证
- CORS 支持
- http压缩

同时，提供了自定义中间件扩展支持

## 下一步

恭喜！你已经成功使用 Nyx 脚手架创建并运行了第一个应用。接下来你可以：

1. **学习脚手架工具**：了解 [脚手架工具详解](./scaffolding)
2. **探索项目结构**：了解 [项目结构详解](./project-structure)
3. **深入核心概念**：学习 [核心概念详解](./core-concepts)
4. **开发自定义功能**：扩展你的应用功能

## 常见问题

### Q: 运行 go mod tidy 时遇到 grpc 引用冲突问题, 如何 解决？
A: 运行 go get google.golang.org/grpc 升级 grpc 版本即可。

## 小结

通过本指南，你已经：

- ✅ 安装了 Nyx 脚手架工具
- ✅ 使用脚手架创建了第一个项目
- ✅ 运行了 HTTP "Hello World" 服务
- ✅ 运行了 RPC "Hello World" 服务
- ✅ 了解了项目结构和核心概念

现在你已经具备了使用 Nyx 框架开发应用的基础。开始你的 Nyx 开发之旅吧！

---

**下一步**：学习 [脚手架工具详解](./02-scaffolding.md) 了解更高级的项目生成功能
