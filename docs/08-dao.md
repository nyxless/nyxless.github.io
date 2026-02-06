# 08. 数据访问层详解 (DAO)

在现代应用程序架构中，数据访问层（Data Access Object, DAO）是连接业务逻辑与数据库的核心桥梁。Nyx 框架提供了完整的数据访问层实现，支持多种数据库、事务管理、读写分离以及缓存机制。本章将详细介绍 Nyx 框架中 DAO 层的实现原理、使用方法和最佳实践。

## 8.1 DAO 概述

### 8.1.1 什么是 DAO

DAO 是 MVC 架构中的 M 层的重要组成部分，它封装了所有对数据库的访问操作，为上层业务逻辑提供服务。DAO 层的主要职责包括：

- **数据库连接管理**：管理数据库连接的生命周期
- **CRUD 操作**：提供对数据库的增删改查操作
- **事务管理**：确保数据操作的原子性和一致性
- **数据映射**：将数据库记录映射为业务对象
- **性能优化**：实现读写分离、连接池、缓存等优化策略

### 8.1.2 Nyx 框架 DAO 特点

Nyx 框架的 DAO 层具有以下特点：

- **多数据库支持**：支持 MySQL、PostgreSQL、SQLite 等主流数据库
- **内置 ORM**：简化数据操作和对象映射
- **读写分离**：支持主从数据库的读写分离策略
- **事务支持**：提供完整的事务管理功能
- **智能缓存**：内置查询缓存和自动刷新机制
- **灵活查询**：支持复杂的 JOIN 操作和条件查询
- **原生 SQL**：提供底层 SQL 客户端支持
- **配置驱动**：通过配置文件灵活配置数据库连接参数

## 8.2 DAO 核心结构

### 8.2.1 Dao 结构体

Nyx 框架的 DAO 核心是 `Dao` 结构体，它是所有数据库操作的入口：

```go
type Dao struct {
    DBWriter, DBReader   db.DBClient
    table                string
    primary              string
    defaultFields        string //默认字段,逗号分隔
    fields               string //通过setFields方法指定的字段,逗号分隔,只能通过getFields使用一次
    countField           string //getCount方法使用的字段
    index                string //查询使用的索引
    limit                string
    autoOrder            bool //是否自动排序(默认按自动主键倒序排序)
    order                string
    group                string
    filter               string //过滤条件
    filterValues         []any  //过滤条件的值
    forceMaster          bool   //强制使用主库读，只能通过useMaster 使用一次
    ctx                  context.Context
    alias                string //表别名
    leftJoin             []*JoinOn
    innerJoin            []*JoinOn
    cnt                  *int //存放查询总数的变量指针
    useCache             bool //使用缓存
    cacheTtl             int  //缓存时长，单位秒
    cacheRefreshInterval int  //缓存自动刷新间隔，单位秒
    cacheCallbackFn      CacheCallbackFn
    hit                  bool      // 是否命中缓存
    cacheTime            time.Time // 临时存放缓存时间
}
```

### 8.2.2 初始化 DAO

#### 基本初始化

```go
// 使用默认配置名
func NewUserDao() *UserDao {
    ins := &UserDao{}
    ins.Init() // 使用默认配置名 "db_master"
    return ins
}

// 自定义配置名
func NewUserDaoWithConfig(confName string) *UserDao {
    ins := &UserDao{}
    ins.Init(confName) // 使用指定的配置名
    return ins
}
```

#### 事务初始化

```go
// 使用现有事务创建 DAO
func NewUserDaoWithTx(tx db.DBClient) *UserDao {
    ins := &UserDao{}
    ins.InitTx(tx) // 初始化事务 DAO
    return ins
}
```

## 8.3 基础配置方法

### 8.3.1 表和主键设置

```go
// 设置表名
dao.SetTable("user")

// 设置主键
dao.SetPrimary("uid")

// 获取表名
tableName := dao.GetTable()

// 获取主键
primaryKey := dao.GetPrimary()
```

### 8.3.2 字段选择

```go
// 设置默认字段（所有查询都会包含）
dao.SetDefaultFields("uid", "username", "email", "status")

// 设置本次查询的特定字段
dao.SetFields("uid", "username", "email")

// 设置统计字段
dao.SetCountField("uid")
```

### 8.3.3 索引和限制

```go
// 使用指定索引
dao.UseIndex("idx_user_status")

// 设置限制条件（limit）
dao.Limit(10, 20) // LIMIT 10,20

// 设置自动排序
dao.SetAutoOrder(true) // 按主键倒序排序
```

## 8.4 查询条件构建

### 8.4.1 灵活的条件解析

Nyx DAO 提供了强大的条件解析功能，支持多种参数格式：

```go
// 字符串条件
dao.SetFilter("status=? AND created_at > ?", 1, "2024-01-01")

// 数组参数
dao.SetFilter("status=? AND name IN ?", []any{1, "admin"})

// Map 条件
dao.SetFilter(map[string]any{
    "status": 1,
    "age":    25,
})

// 复杂条件（IN 查询）
dao.SetFilter(map[string]any{
    "status": 1,
    "name":   []any{"张三", "李四"}, // 会被转换为 IN 条件
})
```

### 8.4.2 链式条件构建

```go
// 链式设置多个条件
dao.SetFilter("status=?", 1).
    SetFilter("created_at > ?", "2024-01-01").
    Order("created_at DESC").
    Limit(10, 20)
```

### 8.4.3 排序和分组

```go
// 设置排序
dao.Order("created_at DESC", "username ASC")

// 设置分组
dao.Group("status", "department")

// 获取要查询的字段
fields := dao.GetFields()
```

## 8.5 JOIN 操作详解

### 8.5.1 左连接查询

```go
// 简单的左连接
func GetUsersWithOrders() ([]map[string]interface{}, error) {
    userDao := dao.NewUserDao()
    orderDao := dao.NewOrderDao()
    
    // 设置别名
    userDao.Alias("u")
    orderDao.Alias("o")
    
    // 构建连接条件
    joinCondition := userDao.On("u.uid", "o.user_id")
    
    return userDao.
        LeftJoin(joinCondition).
        SetFields("u.uid", "u.username", "u.email", "o.order_id", "o.amount").
        GetRecords()
}

// 多表左连接
func GetUserOrderDetails(userID int) ([]map[string]interface{}, error) {
    dao := dao.NewUserDao()
    orderDao := dao.NewOrderDao()
    productDao := dao.NewProductDao()
    
    // 多层连接
    orderJoin := dao.On("u.uid", "o.user_id")
    productJoin := orderDao.On("o.order_id", "p.order_id")
    
    return dao.Alias("u").
        LeftJoin(orderJoin, productJoin).
        SetFields("u.uid", "u.username", "o.order_id", "o.amount", "p.product_name").
        GetRecords("u.uid=?", userID)
}
```

### 8.5.2 内连接查询

```go
// 内连接查询
func GetActiveUsersWithOrders() ([]map[string]interface{}, error) {
    dao := dao.NewUserDao()
    orderDao := dao.NewOrderDao()
    
    joinCondition := dao.On("u.uid", "o.user_id")
    
    return dao.Alias("u").
        InnerJoin(joinCondition).
        SetFields("u.uid", "u.username", "u.email", "o.order_id", "o.amount").
        SetFilter("u.status=?", 1).
        GetRecords()
}
```

### 8.5.3 自连接和复杂条件

```go
// 自连接（获取用户推荐关系）
func GetUserReferrals(referrerID int) ([]map[string]interface{}, error) {
    dao := dao.NewUserDao()
    refDao := dao.NewUserDao()
    
    // 自连接
    joinCondition := dao.On("u.uid", "ref.referrer_id")
    
    return dao.Alias("u").
        InnerJoin(joinCondition).
        SetFields("u.uid", "u.username", "u.email", "ref.uid as referee_id").
        GetRecords("ref.referrer_id=?", referrerID)
}

// 自定义比较条件
func GetUserDataWithCustomCompare() ([]map[string]interface{}, error) {
    dao := dao.NewUserDao()
    
    // 不等于连接
    joinCondition := dao.CompareOn("!=", "u.department_id", "d.department_id")
    
    return dao.Alias("u").
        LeftJoin(joinCondition).
        SetFields("u.uid", "u.username", "u.department_id", "d.department_name").
        GetRecords()
}
```

## 8.6 缓存机制

### 8.6.1 基本缓存使用

```go
// 简单缓存查询
func GetCachedUser(id int) (*model.UserModel, error) {
    dao := dao.NewUserDao()
    
    // 缓存 5 分钟
    data, err := dao.WithCache(300).GetRecord(id)
    if err != nil {
        return nil, err
    }
    
    return model.NewUserModel(data), nil
}

// 带回调的缓存
func GetUserWithAdvancedCache(id int) (*model.UserModel, error) {
    dao := dao.NewUserDao()
    
    // 缓存 10 分钟，带回调
    callback := func(data interface{}) error {
        // 缓存更新时的回调处理
        log.Printf("缓存已更新: %+v", data)
        return nil
    }
    
    data, err := dao.WithCache(600, callback).GetRecord(id)
    if err != nil {
        return nil, err
    }
    
    return model.NewUserModel(data), nil
}
```

### 8.6.2 自动刷新缓存

```go
// 自动刷新缓存
func GetAutoRefreshUserStats() (map[string]interface{}, error) {
    dao := dao.NewUserDao()
    
    // 每 10 分钟自动刷新一次缓存
    data, err := dao.WithRefreshCache(600).GetRecords()
    if err != nil {
        return nil, err
    }
    
    stats := make(map[string]interface{})
    stats["total_users"] = len(data)
    
    return stats, nil
}

// 检查缓存命中情况
func GetUserWithCacheStats(id int) (*model.UserModel, bool, time.Time, error) {
    dao := dao.NewUserDao()
    
    data, err := dao.WithCache(300).GetRecord(id)
    if err != nil {
        return nil, false, time.Time{}, err
    }
    
    // 获取缓存信息
    cacheTime, hit := dao.GetCacheTime()
    
    return model.NewUserModel(data), hit, cacheTime, nil
}
```

## 8.7 读写分离

### 8.7.1 自动读写分离

Nyx DAO 支持自动的读写分离：

```go
// 强制使用主库
dao.UseMaster()
```

### 8.7.2 读写分离使用示例

```go
// 读操作（自动使用从库）
func GetUser(id int) (*model.UserModel, error) {
    dao := dao.NewUserDao()
    data, err := dao.GetRecord(id)
    if err != nil {
        return nil, err
    }
    return model.NewUserModel(data), nil
}

// 写操作（使用主库）
func CreateUser(user *model.UserModel) (int, error) {
    dao := dao.NewUserDao()
    return dao.AddRecord(user.Map())
}

// 强制使用主库读操作
func GetUserFromMaster(id int) (*model.UserModel, error) {
    dao := dao.NewUserDao()
    data, err := dao.UseMaster().GetRecord(id)
    if err != nil {
        return nil, err
    }
    return model.NewUserModel(data), nil
}
```

## 8.8 CRUD 操作详解

### 8.8.1 插入操作

```go
// 插入单条记录
func AddUser(user *model.UserModel) (int, error) {
    dao := dao.NewUserDao()
    return dao.AddRecord(user.Map())
}

// 插入多条记录
func AddUsers(users []*model.UserModel) (int, error) {
    dao := dao.NewUserDao()
    
    // 准备数据
    vals := make([]map[string]interface{}, len(users))
    for i, user := range users {
        vals[i] = user.Map()
    }
    
    return dao.AddRecord(vals...)
}
```

### 8.8.2 查询操作

```go
// 按主键查询
func GetUser(id any) (*model.UserModel, error) {
    dao := dao.NewUserDao()
    data, err := dao.GetRecord(id)
    if err != nil {
        return nil, err
    }
    
    if len(data) == 0 {
        return nil, fmt.Errorf("数据不存在: %v", id)
    }
    
    return model.NewUserModel(data), nil
}

// 条件查询单条
func GetUserByEmail(email string) (*model.UserModel, error) {
    dao := dao.NewUserDao()
    data, err := dao.GetRecordBy("email=?", email)
    if err != nil {
        return nil, err
    }
    
    if data == nil {
        return nil, fmt.Errorf("用户不存在: %s", email)
    }
    
    return model.NewUserModel(data), nil
}

// 查询多条记录
func GetUsers(params ...any) ([]*model.UserModel, error) {
    dao := dao.NewUserDao()
    data, err := dao.GetRecords(params...)
    if err != nil {
        return nil, err
    }
    
    var list []*model.UserModel
    for _, row := range data {
        list = append(list, model.NewUserModel(row))
    }
    
    return list, nil
}

// 分页查询
func GetUserList(page, num int, params ...any) (int, []*model.UserModel, error) {
    var total int
    dao := dao.NewUserDao()
    
    data, err := dao.WithCount(&total).Limit((page-1)*num, num).GetRecords(params...)
    if err != nil {
        return 0, nil, err
    }
    
    var list []*model.UserModel
    for _, row := range data {
        list = append(list, model.NewUserModel(row))
    }
    
    return total, list, nil
}
```

### 8.8.3 更新操作

```go
// 按主键更新
func UpdateUser(user *model.UserModel) (int, error) {
    dao := dao.NewUserDao()
    return dao.SetRecord(user.Map(), user.Uid)
}

// 按条件更新
func UpdateUserByEmail(email string, updates map[string]interface{}) (int, error) {
    dao := dao.NewUserDao()
    updates["updated_at"] = time.Now()
    return dao.SetRecordBy(updates, "email=?", email)
}

// Upsert 操作
func UpsertUser(user *model.UserModel) (int, error) {
    dao := dao.NewUserDao()
    user.UpdatedAt = time.Now()
    return dao.ResetRecord(user.Map())
}
```

### 8.8.4 删除操作

```go
// 按主键删除
func DelUser(id any) (int, error) {
    dao := dao.NewUserDao()
    return dao.DelRecord(id)
}

// 按条件删除
func DelUserByEmail(email string) (int, error) {
    dao := dao.NewUserDao()
    return dao.DelRecordBy("email=?", email)
}

// 删除所有符合条件的记录（危险操作）
func DelUsersByStatus(status int) (int, error) {
    dao := dao.NewUserDao()
    return dao.DelRecords("status=?", status)
}
```

## 8.9 高级查询功能

### 8.9.1 单字段查询

```go
// 获取单字段值
func GetUserEmail(id int) (interface{}, error) {
    dao := dao.NewUserDao()
    return dao.GetValue("email", "uid=?", id)
}

// 获取多个字段值
func GetUserEmails(status int) ([]interface{}, error) {
    dao := dao.NewUserDao()
    return dao.GetValues("email", "status=?", status)
}

// 获取键值对映射
func GetUserStatusMap() (map[interface{}]interface{}, error) {
    dao := dao.NewUserDao()
    return dao.GetValuesMap("uid", "status", "status=?", 1)
}
```

### 8.9.2 聚合查询

```go
// 统计查询
func GetUserCount() (int, error) {
    dao := dao.NewUserDao()
    return dao.GetCount()
}

// 按条件统计
func GetUserCountByStatus(status int) (int, error) {
    dao := dao.NewUserDao()
    return dao.GetCount("status=?", status)
}

// 统计查询
func GetUserStatistics() (map[string]interface{}, error) {
    dao := dao.NewUserDao()
    
    // 总用户数
    total, _ := dao.GetCount()
    
    // 活跃用户数
    active, _ := dao.GetCount("status=?", 1)
    
    // 今日新增
    today := time.Now().Format("2006-01-02")
    todayNew, _ := dao.GetCount("DATE(created_at)=?", today)
    
    return map[string]interface{}{
        "total_users": total,
        "active_users": active,
        "inactive_users": total - active,
        "today_new": todayNew,
    }, nil
}
```

### 8.9.3 存在性检查

```go
// 检查记录是否存在
func UserExists(id int) (bool, error) {
    dao := dao.NewUserDao()
    return dao.Exists(id)
}

// 按条件检查存在
func UserExistsByEmail(email string) (bool, error) {
    dao := dao.NewUserDao()
    return dao.ExistsBy("email=?", email)
}
```

## 8.10 事务管理

### 8.10.1 事务初始化

```go
// 开始事务
func BeginTransaction() (db.DBClient, error) {
    return dao.TransBegin()
}

// 开始只读事务
func BeginReadOnlyTransaction() (db.DBClient, error) {
    return dao.ReadTransBegin()
}
```

### 8.10.2 事务中的操作

```go
// 事务中创建用户
func CreateUserWithTx(user *model.UserModel, profile *model.UserProfile) error {
    // 开始事务
    txClient := dao.TransBegin()
    if txClient == nil {
        return fmt.Errorf("无法开始事务")
    }
    defer txClient.Rollback() // 确保回滚
    
    // 在事务中操作
    dao := dao.NewUserDaoWithTx(txClient)
    
    // 插入用户
    userID, err := dao.AddRecord(user.Map())
    if err != nil {
        return err
    }
    
    // 插入用户详情
    profile.Uid = userID
    profileDao := dao.NewUserProfileDaoWithTx(txClient)
    _, err = profileDao.AddRecord(profile.Map())
    if err != nil {
        return err
    }
    
    // 提交事务
    return txClient.Commit()
}

// 只读事务查询
func GetUserStatsInReadTx() (map[string]interface{}, error) {
    txClient := dao.ReadTransBegin()
    if txClient == nil {
        return nil, fmt.Errorf("无法开始只读事务")
    }
    defer txClient.Rollback()
    
    dao := dao.NewUserDaoWithTx(txClient)
    
    // 在只读事务中执行查询
    total, _ := dao.GetCount()
    active, _ := dao.GetCount("status=?", 1)
    
    return map[string]interface{}{
        "total_users": total,
        "active_users": active,
    }, nil
}
```

## 8.11 上下文管理

### 8.11.1 上下文传递

```go
// 设置上下文
func GetUserWithContext(id int, ctx context.Context) (*model.UserModel, error) {
    dao := dao.NewUserDao().WithContext(ctx)
    data, err := dao.GetRecord(id)
    if err != nil {
        return nil, err
    }
    return model.NewUserModel(data), nil
}

// 在控制器中使用
func (c *UserController) GetUser(rw http.ResponseWriter, req *http.Request) nyx.Response {
    params := c.GetParams(req)
    id := params.GetInt("id", 0)
    
    // 传递请求上下文
    ctx := req.Context()
    user, err := service.GetUserWithContext(id, ctx)
    if err != nil {
        return c.NotFound("用户不存在")
    }
    
    return c.Success(user)
}
```

## 8.12 原生 SQL 支持

### 8.12.1 直接执行 SQL

```go
// 执行原生 SQL
func ExecuteRawSQL() (int, error) {
    dao := dao.NewUserDao()
    return dao.Execute("UPDATE users SET status = ? WHERE created_at < ?", 
        0, time.Now().AddDate(0, -1, 0))
}

// 查询单行
func QueryUserByEmail(email string) (map[string]interface{}, error) {
    dao := dao.NewUserDao()
    return dao.QueryRow("SELECT * FROM users WHERE email = ? LIMIT 1", email)
}

// 查询多行
func QueryUsersByStatus(status int) ([]map[string]interface{}, error) {
    dao := dao.NewUserDao()
    return dao.Query("SELECT uid, username, email FROM users WHERE status = ? ORDER BY created_at DESC", status)
}

// 查询单字段
func QueryUserCount() (interface{}, error) {
    dao := dao.NewUserDao()
    return dao.QueryOne("SELECT COUNT(*) FROM users")
}
```

### 8.12.2 流式查询

```go
// 流式查询大数据集
func QueryUsersStream(callback func(map[string]interface{}) error) error {
    dao := dao.NewUserDao()
    iter, err := dao.QueryStream("SELECT * FROM users ORDER BY created_at DESC")
    if err != nil {
        return err
    }
    defer iter.Close()
    
    for {
        row, err := iter.Next()
        if err != nil {
            if err == io.EOF {
                break
            }
            return err
        }
        
        if err := callback(row); err != nil {
            return err
        }
    }
    
    return nil
}
```

## 8.13 链式调用的重要说明

### 8.13.1 链式调用的优势

Nyx DAO 的核心优势之一是**链式调用**，它让查询构建更加直观和简洁：

```go
// ✅ 链式调用的优势示例
dao.SetFilter("status=?", 1).
    SetFilter("created_at > ?", "2024-01-01").
    Order("created_at DESC").
    Limit(10, 20).
    WithCache(300).
    UseMaster()

data, err := dao.GetRecords()
```

### 8.13.2 用户 DAO 的链式调用局限性

**重要说明**：Nyx DAO 设计了一个重要的限制，用户层 DAO 只能继承 Dao 并重写 `WithContext` 方法：

```go
// 用户 DAO 结构
type UserDao struct {
    dao.Dao  // 嵌套 Dao
}

// 只允许重写 WithContext 方法
func (u *UserDao) WithContext(ctx context.Context) *UserDao {
    u.Dao.WithContext(ctx)
    return u
}
```

### 8.13.3 链式调用的实际行为

**✅ 有效的链式调用**（调用 Dao 原生方法）：
```go
dao := dao.NewUserDao()
data, err := dao.
    SetFilter("status=?", 1).     // 有效：Dao 原生方法
    Order("created_at DESC").    // 有效：Dao 原生方法
    WithCache(300).              // 有效：Dao 原生方法
    GetRecords()                 // 有效：Dao 原生方法
```

**❌ 无效的链式调用**（调用用户 DAO 自定义方法）：
```go
dao := dao.NewUserDao()
data, err := dao.
    SetFilter("status=?", 1).     // 有效：调用的是 Dao 方法
    SearchUsers("keyword", nil).  // ❌ 无效：SearchUsers 是用户 DAO 方法
    GetRecords()                  // ❌ 无效：前面方法已破坏链式调用
```

### 8.13.4 设计理念和最佳实践

这种设计有几个重要考虑：

1. **防止 DAO 滥用**：避免用户过度依赖链式调用，导致复杂的业务逻辑分散在 DAO 层
2. **职责分离**：用户 DAO 专注于业务方法，原生方法专注于数据操作
3. **代码清晰性**：业务查询应该通过专门的方法暴露，而不是通过链式构建

**推荐的使用方式**：

```go
// ✅ 推荐：用户 DAO 暴露明确的业务方法
func (u *UserDao) SearchActiveUsers(keyword string, page, num int) ([]*model.UserModel, error) {
    // 在方法内部使用链式调用
    data, err := u.
        SetFilter("status=?", 1).
        SetFilter("username LIKE ?", "%"+keyword+"%").
        Order("created_at DESC").
        Limit((page-1)*num, num).
        GetRecords()
    
    if err != nil {
        return nil, err
    }
    
    var list []*model.UserModel
    for _, row := range data {
        list = append(list, model.NewUserModel(row))
    }
    
    return list, nil
}

// ✅ 使用：调用专门的业务方法
users, err := dao.SearchActiveUsers("admin", 1, 10)
```

## 8.14 实际项目示例

### 8.14.1 完整的用户 DAO 实现

```go
// 用户数据访问层
type UserDao struct {
    dao.Dao
}

// 创建用户 DAO 实例
func NewUserDao(tx ...db.DBClient) *UserDao {
    ins := &UserDao{}
    ins.Init(tx...)
    return ins
}

// 初始化
func (u *UserDao) Init(tx ...db.DBClient) {
    if len(tx) > 0 {
        u.Dao.InitTx(tx[0])
    } else {
        u.Dao.Init()
    }
    u.SetTable("user")
    u.SetPrimary("uid")
    u.SetDefaultFields("uid", "username", "email", "status", "created_at", "updated_at")
}

// 设置上下文
func (u *UserDao) WithContext(ctx context.Context) *UserDao {
    u.Dao.WithContext(ctx)
    return u
}

// 业务方法
func (u *UserDao) AddUser(t *model.UserModel) (int, error) {
    return u.AddRecord(t.Map())
}

func (u *UserDao) SetUser(t *model.UserModel) (int, error) {
    return u.SetRecord(t.Map(), t.Uid)
}

func (u *UserDao) ResetUser(t *model.UserModel) (int, error) {
    return u.ResetRecord(t.Map())
}

func (u *UserDao) DelUser(id any) (int, error) {
    return u.DelRecord(id)
}

func (u *UserDao) GetUser(id any) (*model.UserModel, error) {
    data, err := u.GetRecord(id)
    if err != nil {
        return nil, err
    }
    
    if len(data) == 0 {
        return nil, fmt.Errorf("数据不存在: %v", id)
    }
    
    return model.NewUserModel(data), nil
}

func (u *UserDao) GetUsers(params ...any) ([]*model.UserModel, error) {
    data, err := u.GetRecords(params...)
    if err != nil {
        return nil, err
    }
    
    var list []*model.UserModel
    for _, row := range data {
        list = append(list, model.NewUserModel(row))
    }
    
    return list, nil
}

func (u *UserDao) GetUserList(page, num int, params ...any) (int, []*model.UserModel, error) {
    var total int
    data, err := u.WithCount(&total).Limit((page-1)*num, num).GetRecords(params...)
    if err != nil {
        return 0, nil, err
    }
    
    var list []*model.UserModel
    for _, row := range data {
        list = append(list, model.NewUserModel(row))
    }
    
    return total, list, nil
}
```

### 8.14.2 复杂的业务查询

```go
// 获取用户统计信息
func (u *UserDao) GetUserStatistics() (map[string]interface{}, error) {
    // 总用户数
    total, _ := u.GetCount()
    
    // 活跃用户数
    active, _ := u.GetCount("status=?", 1)
    
    // 今日新增
    today := time.Now().Format("2006-01-02")
    todayNew, _ := u.GetCount("DATE(created_at)=?", today)
    
    return map[string]interface{}{
        "total_users": total,
        "active_users": active,
        "inactive_users": total - active,
        "today_new": todayNew,
    }, nil
}

// 搜索用户
func (u *UserDao) SearchUsers(keyword string, status *int, page, num int) ([]*model.UserModel, int, error) {
    var total int
    
    // 构建查询条件
    dao := u.WithContext(u.ctx)
    
    if keyword != "" {
        dao.SetFilter("username LIKE ? OR email LIKE ?", "%"+keyword+"%", "%"+keyword+"%")
    }
    
    if status != nil {
        dao.SetFilter("status=?", *status)
    }
    
    data, err := dao.WithCount(&total).Limit((page-1)*num, num).Order("created_at DESC").GetRecords()
    if err != nil {
        return nil, 0, err
    }
    
    var list []*model.UserModel
    for _, row := range data {
        list = append(list, model.NewUserModel(row))
    }
    
    return list, total, nil
}
```

## 8.15 最佳实践

### 8.15.1 DAO 层设计原则

1. **单一职责**：每个 DAO 只负责一种数据实体的操作
2. **接口隔离**：根据业务需求定义不同的 DAO 接口
3. **事务边界**：明确 DAO 方法的事务边界
4. **异常处理**：统一的错误处理和日志记录
5. **性能优化**：合理使用索引、缓存和批量操作

### 8.15.2 使用建议

1. **使用链式调用**：Nyx DAO 支持链式调用，使代码更简洁
2. **合理使用缓存**：对热点查询使用缓存，提升性能
3. **避免 N+1 查询**：使用 JOIN 或批量查询替代循环查询
4. **事务使用场景**：涉及多表操作的业务逻辑使用事务
5. **读写分离**：读多写少的场景充分利用读写分离

### 8.15.3 链式调用最佳实践

1. **在业务方法内部使用**：将复杂的链式调用封装在用户 DAO 的专门方法中
2. **避免过长的链式调用**：如果一个链式调用超过 5-7 步，考虑拆分为多个方法
3. **利用 WithContext**：通过重写 WithContext 方法传递上下文信息
4. **组合使用缓存和分页**：充分利用 DAO 的缓存和分页功能

```go
// 推荐的复杂查询模式
func (u *UserDao) GetActiveUsersWithCache(page, num int) ([]*model.UserModel, int, error) {
    var total int
    
    data, err := u.
        WithCount(&total).
        WithCache(300). // 使用缓存
        Limit((page-1)*num, num).
        Order("created_at DESC").
        GetRecords("status=?", 1)
    
    if err != nil {
        return nil, 0, err
    }
    
    var list []*model.UserModel
    for _, row := range data {
        list = append(list, model.NewUserModel(row))
    }
    
    return list, total, nil
}
```

### 8.15.4 错误处理策略

```go
// 自定义错误类型
var (
    ErrUserNotFound    = errors.New("user not found")
    ErrUserAlreadyExists = errors.New("user already exists")
)

// 统一的错误处理
func (u *UserDao) GetUserByEmail(email string) (*model.UserModel, error) {
    data, err := u.GetRecordBy("email=?", email)
    if err != nil {
        return nil, fmt.Errorf("database error: %w", err)
    }
    
    if data == nil {
        return nil, ErrUserNotFound
    }
    
    return model.NewUserModel(data), nil
}
```

## 8.16 小结

DAO 层是 Nyx 框架数据持久化的核心组件，它提供了：

- **完整的基础功能**：CRUD 操作、条件查询、分页等
- **高级查询能力**：JOIN 操作、聚合查询、流式查询
- **性能优化特性**：智能缓存、读写分离、连接池管理
- **事务管理**：支持事务和只读事务
- **灵活的配置**：支持多种数据库和配置方式
- **原生 SQL 支持**：满足复杂查询需求

### 核心特性亮点

#### 1. 灵活的查询构建
```go
dao.SetFilter("status=?", 1).
    SetFilter("created_at > ?", "2024-01-01").
    Order("created_at DESC").
    Limit(10, 20)
```

#### 2. 智能缓存机制
```go
data, err := dao.WithCache(300).GetRecords("status=?", 1)
```

#### 3. 强大的 JOIN 支持
```go
userDao.LeftJoin(userDao.On("u.uid", "o.user_id"))
```

#### 4. 高效的批量操作
```go
affected, _ := dao.AddRecord(userDataSlice...)
```

#### 5. **链式调用与设计约束**
Nyx DAO 的链式调用是一个强大的特性，但需要理解其设计约束：

```go
// ✅ 正确的使用方式
dao := dao.NewUserDao()
data, err := dao.
    SetFilter("status=?", 1).
    Order("created_at DESC").
    GetRecords()

// ✅ 通过用户 DAO 的专门方法
users, err := dao.SearchActiveUsers("admin", 1, 10)

// ❌ 不推荐：在用户层进行复杂链式构建
// 业务逻辑应该通过专门方法暴露
```

这种设计既保证了链式调用的便利性，又避免了 DAO 层的滥用，体现了 Nyx 框架在易用性和架构清晰性之间的平衡。

通过合理使用 DAO 层的各种功能，开发者可以构建高效、可靠的数据访问层，为上层业务逻辑提供坚实的数据基础。在实际项目中，建议根据具体业务需求选择合适的 DAO 设计模式，并充分利用 Nyx 提供的缓存、JOIN、事务等高级特性来优化性能和开发效率。

---

**下一章预告**：第九章将详细介绍模型层（Models）的使用，包括数据模型定义、验证规则、序列化等功能的实现和应用场景。
