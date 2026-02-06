# 02. Nyx 脚手架工具

## 概述

Nyx 脚手架是一套完整的开发工具集，提供了从项目初始化、代码生成、运行调试到打包及服务管理的全流程工具。安装后脚手架后，开发者可以使用这些工具快速构建和生成项目所需的各类代码文件。

Nyx 脚手架的主要功能包括：
- **代码自动生成**
- **运行调试**
- **服务管理**

## 安装和配置

### 安装脚手架工具

```bash
# 安装最新版本的 Nyx 脚手架
go install github.com/nyxless/nyx/cmd/nyx@latest
```

### 环境要求

- Go 1.23+

## 核心功能

### 1. 项目启动和配置管理

#### 主入口文件 (nyx.go)

`nyx.go` 是 Nyx 框架的入口文件，包含了项目启动的所有核心逻辑：

#### 支持的运行模式

```go
const (
    SERVER_HTTP = "http"  // HTTP 服务器
    SERVER_RPC  = "rpc"  // gRPC 服务器
    SERVER_TCP  = "tcp"  // TCP 服务器
    SERVER_WS   = "ws"   // WebSocket 服务器
    SERVER_CLI  = "cli"  // CLI 模式
)
```

#### 启动参数配置

```bash
# 基础启动命令
nyx run -c config.conf -m http,rpc -d

# 参数说明：
# -c: 配置文件路径
# -m: 运行模式 (http,rpc,tcp,ws,cli)
# -d: 启用调试模式
# -p: CLI 模式下的路径参数
# -q: CLI 模式下的查询参数
```

### 2. 快速代码生成

#### ORM 代码自动生成工具

`nyx gen orm` 是强大的 ORM 代码生成工具，支持从 SQL 文件自动生成完整的MVC代码框架。

**快速使用示例**：

```bash
# 基本用法 - 生成完整项目框架
nyx gen orm -s doc/test.sql

# 指定表生成
nyx gen orm -s doc/test.sql -t user,product,order

# 哈希分表支持
nyx gen orm -s doc/test.sql -t user,product,order -n "user:10,order:5"

# 强制覆盖现有文件
nyx gen orm -s doc/test.sql -f
```

#### DAO 生成脚本
`nyx gen dao` 是轻量级的 DAO 生成脚本：

```bash
# 快速生成 DAO
nyx gen dao -t user

# 带分表功能
nyx gen dao -t user -n 10 -f
```

#### RPC 生成脚本

`nyx gen rpc` 用于生成 RPC sdk 代码(封装 nyxclient 对rpc 接口的请求)：

```bash
# 生成用户 RPC sdk 代码 
nyx gen rpc -m user -r "user/getInfo,user/updateProfile"
```

### 3. 开发工作流程

#### 推荐的项目开发流程

1. **设计数据库表结构**
2. **使用脚手架生成基础代码**
3. **定制和优化生成的代码**
4. **使用测试工具验证功能**

#### 示例：从零开始创建用户管理模块

```bash
# 1. 创建用户表 SQL 文件(生成的项目中默认会有一个 doc/test.sql)
cat > test.sql << 'EOF'
CREATE TABLE `user` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `username` varchar(50) NOT NULL COMMENT '用户名',
    `email` varchar(100) NOT NULL COMMENT '邮箱',
    `password` varchar(255) NOT NULL COMMENT '密码',
    `status` tinyint(1) DEFAULT 1 COMMENT '状态',
    `created_at` timestamp DEFAULT CURRENT_TIMESTAMP,
    `updated_at` timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `username` (`username`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
EOF

# 2. 生成完整代码框架
nyx gen orm -s doc/test.sql

```

## 最佳实践

####  编码建议

生成的代码是高质量的起点，建议：
- 保持 DAO 层的通用性
- 在 Service 层实现具体业务逻辑
- 根据需要调整 Controller 的验证规则

