# 05. 配置系统

Nyx 框架提供了灵活的配置管理系统，支持 YAML 格式的配置文件，并具有模块化配置和全局访问能力。

## 配置加载原理

### 初始化流程

配置系统的加载过程如下：

1. **入口点**：在应用启动时，`nyx.Init()` 函数会调用 `x.NewConfig(conf_file)` 来初始化配置
2. **解析过程**：
   - 使用 `gopkg.in/yaml.v3` 库解析配置文件（`.conf` 文件实际为 YAML 格式）
   - 通过自定义的 YAML 封装器 (`x/yaml/yaml.go`) 进行解析
   - 支持 `include` 指令，允许将多个配置文件合并
3. **存储**：解析后的配置数据存储在全局变量 `x.Conf` 中


## 05.1 配置系统概述

### 05.1.1 Nyx 配置特点

Nyx 框架的配置系统具有以下特点：

- **YAML 格式**：使用人类可读的 YAML 格式定义配置
- **环境隔离**：支持开发、测试、生产环境的独立配置
- **Include 支持**：支持配置文件的模块化和复用
- **热重载**：支持配置文件的动态更新
- **类型安全**：提供类型安全的配置获取方法
- **默认值支持**：支持配置的默认值设置

### 05.1.2 配置获取方式

Nyx 框架通过 `x.Conf` 提供统一的配置访问接口：

```go
// 获取字符串配置
conf := x.Conf.GetStringMap("cookie_conf")

// 获取子配置项
secret := x.Conf.GetString("cookie_conf.secret")
maxlife := x.Conf.GetInt("cookie_conf.maxlife")

// 获取布尔配置
debug := x.Conf.GetBool("db_master.debug")
```

## 05.2 配置文件结构

### 05.2.1 基础配置文件

在示例应用中，配置文件采用模块化的设计：

```yaml
# app.conf - 环境特定配置
!include ../conf/app.common.conf

##### 环境模式, 开发:dev  测试:test 生产:prod
env_mode: dev  

db_master:
    type: mysql 
    host: localhost:3306
    user: root 
    password: 123456 
    database: apptest 
    charset: utf8mb4
    max_open_conns: 800
    max_idle_conns: 200
    debug: true

cookie_conf:
     secret:        "test secret"
     maxlife:       "86400" #强制退出时间
     expire:        "3600"  #无操作过期时间(公网)
     expire_lan:    "86400" #无操作过期时间(局域网)
     domain:        ""
     path:          "/"
     samesite:      0
     secure:        false
     httponly:      false
```

### 05.2.2 公共配置文件

```yaml
# app.common.conf - 公共配置
#####################################
#                                   
# 1. 支持 yaml 语法
# 2. 支持 include 形式, include 中的配置可以被外层覆盖  
# 格式: !include ../conf/app.common.conf
#                                   
#####################################

##### 环境模式, 开发:dev  测试:test 生产:prod
env_mode: prod 

######## 基础配置项 ######## 
#时区
#time_zone: UTC #未指定时使用:Local

#进程pid的文件
#app_pid_file: ./pid #默认:./pid

#guid参数使用的key
#guid_key: guid

#lang参数使用的key, 可通过 lang 动态配置语言
#lang_key: lang 

#数值类型转换时使用的舍入类型
#round_type: 0 #0:舍去 1:银行家算法 2:向上取整 3:向下取整 4:四舍五入, 默认舍去
```

### 05.2.3 环境特定配置

#### 开发环境 (app.conf)

```yaml
env_mode: dev  
db_master:
    type: mysql 
    host: localhost:3306
    user: root 
    password: 123456 
    database: apptest 
    charset: utf8mb4
    max_open_conns: 800
    max_idle_conns: 200
    debug: true
```

#### 生产环境 (app.conf.prod)

```yaml
env_mode: prod 
db_master:
    type: mysql 
    host: newexample.org
    user: root 
    password: zsdffeH5anY
    database: app 
    charset: utf8mb4
    max_open_conns: 800
    max_idle_conns: 200
    debug: false 
```

#### 测试环境 (app.conf.test)

```yaml
env_mode: test 
db_master:
    type: mysql 
    host: newexample.org
    user: root 
    password: zsdffeH5anY
    database: apptest 
    charset: utf8mb4
    max_open_conns: 800
    max_idle_conns: 200
    debug: false 
```

## 05.3 服务器配置详解

### 05.3.1 HTTP 服务器配置

```yaml
######## http server 配置 ######## 
http_server:
    addr: #监听地址
    port: 8080 #http请求监听端口
    read_timeout: 60000 #读超时ms
    write_timeout: 0 #写超时ms 
    use_graceful: true #平滑重启
    #max_post_size: 128 #设置请求postsize, 单位: M
    #pprof_enable: false #是否打开pprof
    #static_files: #静态资源
    #    enabled: false #开关, 默认关
    #    path: static #url路由
    #    root:   /www/demo/web #资源目录
    #template: # 模板
    #    enabled: false  #开关, 默认关 
    #    root: /www/demo/src/templates #模板路径
    #    recursion_limit: 3 #引用层数限制
    method_rule: #http METHOD 匹配规则
        - path: [test/hello,user]
          allow: [POST,GET]
        - path: [index]
          forbid: [GET]
```

### 05.3.2 RPC 服务器配置

```yaml
######## rpc server 配置 ######## 
rpc_server: 
    addr: #监听地址
    port: 8081 #监听端口
    timeout: 3000 #rpc 连接超时时间ms
    use_graceful: true #平滑重启
    #monitor_port: 9001 #状态监听端口
```

### 05.3.3 TCP 服务器配置

```yaml
######## tcp server 配置 ######## 
tcp_server: 
    addr: #监听地址
    port: 8082 #监听端口
    use_graceful: true #平滑重启
    #monitor_port: 9001 #状态监听端口
```

### 05.3.4 WebSocket 服务器配置

```yaml
######## websocket server 配置 ######## 
ws_server: 
    addr: #监听地址
    port: 8083 #监听端口
    read_timeout: 30000 #读超时ms
    write_timeout: 30000 #写超时ms 
    use_graceful: true #平滑重启
    #monitor_port: 9001 #状态监听端口
```

## 05.4 路由配置系统

### 05.4.1 方法规则配置

```yaml
method_rule: #http METHOD 匹配规则
    - path: [test/hello,user]
      allow: [POST,GET]
    - path: [index]
      forbid: [GET]
```

### 05.4.2 URL 路由配置

```yaml
######## 路由配置 ######## 
#实际的 [group/]controller/action 是和目录结构对应的
url_route:
  - prefix: #匹配到此前缀则执行后面的规则
    group_rule:
      - from: 
        to: 
  - prefix: /api/v1 #匹配到此前缀则执行后面的规则
    #enabled: true #开关, 默认开
    #allow_snake: false #自动转换蛇形命名
    group_rule: #前缀替换, prefix + from/* => to/*
      - from: /  
        to: /   #实际的 [group/]controller/action 的前缀
    path_rule: # path 的完整匹配替换， prefix + from => to
      - from: /info
        to: user/GetUserInfo #实际的 [group/]controller/action
      - from: /users/@user_id
        to: user/GetUserInfo
        method: [GET,POST] 
      - from: /users/@uid/@name
        to: user/GetUserInfo
        method: GET 
      - from: /audience/export/@code
        to: audience/export
        method: GET 
```

### 05.4.3 路由规则处理

根据配置文件，Nyx 框架在初始化阶段会解析路由规则：

1. **方法规则处理**：`method_rule` 配置指定路径允许或禁止的 HTTP 方法
2. **分组规则处理**：`group_rule` 配置处理 URL 前缀替换
3. **路径规则处理**：`path_rule` 配置处理完整的路径映射

## 05.5 中间件配置

### 05.5.1 CORS 配置

```yaml
######## http 跨域中间件配置 ########
cors:
    enabled: true 
    allowed_origins: ["*"]
    #allow_credentials: false
    #allowed_methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"]
    #allowed_headers: ["Content-Type", "Authorization", "Accept", "Origin", "X-Requested-With"]
    #max_age: 86400
    #exposed_headers: []
    allowed_groups: [] #指定路由分组
```

### 05.5.2 压缩配置

```yaml
######## http 压缩中间件配置 ########
compress:
    enabled: true 
    #min_size: 1024 #小于 1 k 不压缩
    #gzip_level: 6 #gzip压缩等级
    #deflate_level: #deflate压缩等级
    allowed_groups: [] #指定路由分组
```

### 05.5.3 访问日志配置

```yaml
######## (http/rpc)访问日志中间件配置 ######## 
http_log:
    enabled: true #http接口访问日志中间件开关
    info_level_name: ACCESS #接口正常返回日志(errno == 0)
    warn_level_name: WARN #接口异常返回日志(errno != 0)
    error_level_name: ERROR #系统错误日志
    omit_params: [password,token] #需要在日志中隐藏掉的参数
    method: [] #需要记录日志的方法列表
    except: [] #需要在 method 列表中排除的方法
    req_method: [] #需要记录请求数据的方法列表
    req_except: [] #需要在 req_method 列表中排除的方法
    res_method: [] #需要记录返回数据的方法列表
    res_except: [] #需要在 res_method 列表中排除的方法
    allowed_groups: [] #指定路由分组
```

## 05.6 日志系统配置

### 05.6.1 基础日志配置

```yaml
######## 日志配置 ######## 
log:
    enabled: true # 日志开关
    level: 0xFF # 日志级别 
    #show_level: true #显示当前日志级别 
    #trace_file: false # 打印文件名
    #time_format: "" #时间 Layout 格式
    #prefix: "" #统一前缀值
    #use_sync: true    # 异步队列开关, 默认 true
    #queue_size: 1024 # 异步队列缓冲长度, 默认 1024
    #bulk_size: 64      # 每次提交队列的条数, 默认 32
    file_enabled: true # 使用 FileWriter
    file_rule: # FileWriter时 日志规则
        path: "logs"
        buffer_size: 8192 #文件IO 缓冲区大小
        file_size: 104851000 # 单个文件最大字节
        compress: false #是否压缩历史日志
        compress_before: 2 #压缩时间N分钟以前日志
        remove: false #是否删除历史日志
        remove_before: 3 #删除时间N分钟以前日志
        naming_format: "app-2006-01-02_15.log"
```

### 05.6.2 分级日志配置

```yaml
file_level_rule: # 不同级别规则，继承默认规则
    ERROR: #级别名
        naming_format: "app-2006-01-02_15.error.log"
    WARN: #级别名
        naming_format: "access-2006-01-02_15.warn.log"
    ACCESS:  # 自定义级别: 成功访问日志
        file_size: 104851000  #500MB
        naming_format: "access-2006-01-02_15.log"
    nyxclient:
        naming_format: "nyxclient-2006-01-02_15.log"
```

## 05.7 鉴权配置

### 05.7.1 API 鉴权配置

```yaml
######## (http/rpc)鉴权中间件配置 ######## 
auth:
    api_check:
        enabled: true #是否开启api鉴权
        ttl: 300 #token 有效期(秒)，默认 5 分钟 
        method: [] #需要鉴权的方法列表
        except: [] #需要在 method 列表中排除的方法
        check_nonce: false #防重入检查
        allowed_groups: [admin, aaa] #指定中间件路由分组
    rpc_check:
        enabled: true #是否开启rpc鉴权
        ttl: 300 #token 有效期(秒)，默认 5 分钟
        method:
        except:
        allowed_groups: [admin, aaa]
```

### 05.7.2 应用授权配置

```yaml
    app: #授权接入方
       - appid: 1000
         secret: test
         name: 测试 
         api_allow: [] #允许访问的api方法
         api_forbid: [] #禁止访问的api方法
         rpc_allow: [] #允许访问的rpc方法
         rpc_forbid: [] #允许访问的rpc方法
```

## 05.8 缓存配置

### 05.8.1 本地缓存配置

```yaml
######## 本地缓存配置 ######## 
localcache:
    enabled: true
    size: 536870912
```

### 05.8.2 RPC 客户端配置

```yaml
######## nyxclient 配置 ######## 
#示例: 使用 nyxclient 连接 message 服务配置
rpc_client_message:
    host: 05.0.0.1:9002
    appid: test
    secret: test
```

## 05.9 配置获取方法

### 05.9.1 基础配置获取

在 Nyx 框架中，配置通过 `x.Conf` 对象获取：

```go
// Cookie 配置获取（示例应用中的实际使用）
func (this *Auth) getCookieConfig() map[string]string {
    return x.Conf.GetStringMap("cookie_conf")
}

// 获取具体配置项
secret := x.Conf.GetString("cookie_conf.secret")
maxlife := x.Conf.GetInt("cookie_conf.maxlife")
expire := x.Conf.GetString("cookie_conf.expire")
```

### 05.9.2 复杂配置获取

```go
// 获取 RPC 客户端配置
conf := x.Conf.GetStringMap("rpc_client_${service}")
client, err := nyxc.NewNyxClient(conf["host"], conf["appid"], conf["secret"])

// 获取数据库配置
dbConfig := x.Conf.GetStringMap("db_master")
host := dbConfig["host"]
user := dbConfig["user"]
password := dbConfig["password"]
database := dbConfig["database"]
```

### 05.9.3 控制器中的配置使用

```go
// 在控制器中使用配置
func (this *LoginController) LoginAction() {
    // 参数获取
    username := this.GetString("username")
    password := this.GetString("password")

    // 参数验证
    x.Interceptor(username != "", x.ERR_PARAMS, "username")
    x.Interceptor(password != "", x.ERR_PARAMS, "password")

    // 业务逻辑
    daoUser := dao.NewAdvUserDao()
    userinfo, err := daoUser.GetAdvUserByEmail(username)

    x.Interceptor(err == nil, common.ERR_PASSWORD)
    x.Interceptor(lib.VerifyPassword(password, userinfo.Password), common.ERR_PASSWORD)
    x.Interceptor(model.USER_STATUS_ACTIVE == userinfo.Status, common.ERR_PASSWORD)

    // 认证配置使用
    this.Auth.Login(userinfo.AdvUserId, username)
}
```

## 05.10 应用启动配置

### 05.05.1 Main 函数配置

```go
// main.go - 应用启动入口
package main

import (
    //_ "midas-exter/embed"
    "github.com/nyxless/nyx"
    "github.com/nyxless/nyx/x"
    _ "midas-exter/autoload"
    "midas-exter/svc"
    "net/http"
)

func main() {
    // 创建 Nyx 应用实例
    n := nyx.NewNyx()

    // 使用 http middleware 示例
    x.UseHttpMiddleware(test)

    // 初始化会话服务
    err := svc.NewSession().UpdateSessionIds()
    if err != nil {
        panic(err)
    }

    // 启动应用
    n.Run()
}

// HTTP 中间件示例
func test(next http.Handler) http.Handler {
    return http.HandlerFunc(func(rw http.ResponseWriter, r *http.Request) {
        x.Println("test1")

        x.Println(r.Context().Value("controller"))
        x.Println(r.Context().Value("action"))
        x.Println(r.Context().Value("aa"))
        next.ServeHTTP(rw, r)
        x.Println(r.Context().Value("controller"))
        x.Println(r.Context().Value("action"))
        x.Println(r.Context().Value("aa"))
        x.Println("test2")
    })
}
```

### 05.05.2 配置初始化流程

1. **配置文件加载**：Nyx 框架在启动时加载配置文件
2. **环境检测**：根据环境模式选择相应的配置
3. **配置解析**：解析并验证配置的有效性
4. **服务初始化**：基于配置初始化各个服务组件
5. **路由注册**：根据路由配置注册路由规则
6. **中间件加载**：根据配置加载中间件组件

## 05.11 环境管理与部署

### 05.11.1 环境模式配置

Nyx 框架支持三种环境模式：

| 环境模式 | 配置项 | 说明 |
|----------|--------|------|
| `dev` | `env_mode: dev` | 开发环境，开启调试功能 |
| `test` | `env_mode: test` | 测试环境，用于测试验证 |
| `prod` | `env_mode: prod` | 生产环境，优化性能 |

### 05.11.2 数据库配置差异

#### 开发环境
```yaml
db_master:
    type: mysql 
    host: localhost:3306
    user: root 
    password: 123456 
    database: apptest 
    charset: utf8mb4
    max_open_conns: 800
    max_idle_conns: 200
    debug: true  # 开启调试
```

#### 生产环境
```yaml
db_master:
    type: mysql 
    host: newexample.org
    user: root 
    password: zsdffeH5anY
    database: app 
    charset: utf8mb4
    max_open_conns: 800
    max_idle_conns: 200
    debug: false  # 关闭调试
```

### 05.11.3 部署策略

#### 开发环境部署
1. 使用 `app.conf` 配置文件
2. 连接本地开发数据库
3. 开启调试模式和详细日志
4. 启用开发特定的中间件

#### 生产环境部署
1. 使用 `app.conf.prod` 配置文件
2. 连接生产数据库
3. 关闭调试模式
4. 优化日志级别和文件大小
5. 配置监控和健康检查

#### 测试环境部署
1. 使用 `app.conf.test` 配置文件
2. 连接测试数据库
3. 启用测试相关的配置

## 05.12 最佳实践

### 05.12.1 配置组织建议

1. **模块化设计**：将通用配置放在 `app.common.conf` 中
2. **环境隔离**：为不同环境创建独立的配置文件
3. **敏感信息**：使用环境变量或密钥管理服务
4. **文档注释**：为每个配置项添加清晰的注释说明

### 05.12.2 安全配置建议

```yaml
# 安全的 Cookie 配置示例
cookie_conf:
     secret: "${COOKIE_SECRET:default_secret}"  # 使用环境变量
     maxlife:       "${COOKIE_MAXLIFE:86400}"
     expire:        "${COOKIE_EXPIRE:3600}"
     expire_lan:    "${COOKIE_EXPIRE_LAN:86400}"
     domain:        "${COOKIE_DOMAIN:}"
     path:          "/"
     samesite:      0
     secure:        true  # 生产环境设为 true
     httponly:      true  # 防止 XSS 攻击
```

### 05.12.3 性能优化建议

```yaml
# 优化的服务器配置
http_server:
    port: 8080
    read_timeout: 60000
    write_timeout: 0
    use_graceful: true

# 优化的数据库配置
db_master:
    max_open_conns: 800   # 根据并发需求调整
    max_idle_conns: 200   # 合理设置空闲连接数

# 优化的日志配置
log:
    file_size: 104851000   # 100MB 日志文件大小
    compress: true         # 压缩历史日志
    compress_before: 1440   # 24小时前的日志进行压缩
```

### 05.12.4 监控配置建议

```yaml
# 监控和健康检查
monitor:
    enabled: true
    port: 9001
    endpoints:
        - /health
        - /metrics
        - /status
```

## 05.13 错误处理配置

### 05.13.1 错误信息配置

```yaml
######## 错误信息配置 ######## 
#默认语言
#default_lang: EN

#支持在配置文件中配置语言及错误信息，code 必须在代码中定义
#err_msg:
#   106:    
#       CN: "认证失败"
#       EN: "token is invalid" 
```

### 05.13.2 自定义错误处理

在控制器中处理配置相关的错误：

```go
func (this *Auth) CheckLogin() bool {
    if !this.logined {
        userstr := this.GetCookie("U", true)
        signstr := this.GetCookie("S", false)

        if "" == userstr || "" == signstr {
            this.logined = false
        } else {
            // 使用配置验证签名
            conf := this.getCookieConfig()
            secret := conf["secret"]
            
            // 签名验证逻辑
            if !verifySignature(userstr, signstr, secret) {
                this.Logout()
                return false
            }
            
            this.logined = true
        }
    }

    return this.logined
}
```

## 05.14 小结

配置与部署是 Nyx 框架的重要组成部分，主要包括：

### 🎯 **核心功能**
- **YAML 配置系统**：人类可读的配置文件格式
- **环境隔离**：支持开发、测试、生产环境的独立配置
- **模块化设计**：通过 Include 功能实现配置复用
- **类型安全**：提供类型安全的配置获取方法

### 🛠️ **配置类别**
- **服务器配置**：HTTP、RPC、TCP、WebSocket 服务器配置
- **路由配置**：方法规则、URL 路由、路径映射配置
- **中间件配置**：CORS、压缩、日志、鉴权等中间件配置
- **业务配置**：数据库、缓存、Cookie 等业务相关配置

### 💼 **实际应用**
- **环境管理**：基于环境模式的配置选择
- **部署策略**：针对不同环境的部署配置
- **安全配置**：生产环境的安全配置建议
- **性能优化**：基于配置的性能调优建议

### 🚀 **最佳实践**
- **配置组织**：合理的配置文件结构和命名规范
- **安全考虑**：敏感信息的环境变量管理
- **监控配置**：生产环境的监控和健康检查
- **错误处理**：自定义错误信息和处理机制

通过合理配置 Nyx 框架，可以构建出既灵活又稳定的企业级应用程序。

