# 09. 模型系统

Nyx 框架提供了完整的模型系统，通过脚手架自动生成和手工定义两种方式，支持数据建模、类型转换和与数据库的无缝集成。

## 基础模型结构

### Model 结构体

框架定义了一个基础的 `Model` 结构体，所有模型都继承自它：

```go
type Model struct {
    Ext map[string]any `json:"-"` //扩展属性
}
```

这个基础结构提供了扩展属性存储能力，允许在运行时为模型添加额外的属性信息。

## 脚手架自动生成的模型

### 生成特点

Nyx 框架提供了强大的脚手架工具，能够根据数据库表结构自动生成完整的模型代码：

#### 1. **智能类型映射**
脚手架根据数据库字段类型自动映射到 Go 类型：

```go
// 数据库类型 -> Go 类型映射
int        -> int
varchar    -> string  
text       -> string
datetime   -> time.Time
decimal    -> float64
bool       -> bool
blob       -> []byte
```

#### 2. **完整的结构体定义**
生成的模型结构体包含：
- 所有表字段的 Go 类型定义
- 完整的 JSON 标签 (`json:"字段名"`)
- 详细的结构标签 (`orm:"column:字段名;type:类型;primaryKey;not null;default:默认值"`)
- 字段注释信息

#### 3. **标准化的命名规范**
- 结构体名使用 PascalCase（如 `UserModel`）
- 字段名使用 PascalCase（如 `UserName`）
- JSON 标签使用 snake_case（如 `"user_name"`）
- 数据库列名保持原始格式

#### 4. **继承基础 Model**
所有生成的模型都嵌入基础的 `model.Model` 结构体，获得扩展属性能力。

### 生成示例

基于您的脚手架模板，实际生成的代码如下：

```go
package model

import (
    "github.com/nyxless/nyx/model"
    "github.com/nyxless/nyx/x"
    "time"
)

// 用户模型 - 此文件是由 nyx 脚手架自动生成, 可按需要修改
type UserModel struct {
    model.Model
    
    Uid        int64     `json:"uid" orm:"column:uid;type:integer;primaryKey;not null"` // 用户ID
    Name       string    `json:"name" orm:"column:name;type:string;not null"`           // 用户名
    Gender     string    `json:"gender" orm:"column:gender;type:string"`                // 性别
    Email      string    `json:"email" orm:"column:email;type:string"`                  // 邮箱
    CreatedAt  time.Time `json:"created_at" orm:"column:created_at;type:datetime;default:CURRENT_TIMESTAMP"` // 创建时间
    UpdatedAt  time.Time `json:"updated_at" orm:"column:updated_at;type:datetime;default:CURRENT_TIMESTAMP"` // 更新时间
}

// 创建空的 UserModel
func User() *UserModel {
    return &UserModel{}
}

// 从 map 初始化 UserModel
func NewUserModel(m x.MAP) *UserModel {
    c := User()
    c.Fill(m)
    return c
}

// Fill 方法 - 将数据库数据格式化为模型对象
func (this *UserModel) Fill(m x.MAP) {
    if uid, ok := m["uid"]; ok {
        this.Uid = x.AsInt64(uid)
    }
    if name, ok := m["name"]; ok {
        this.Name = x.AsString(name)
    }
    if gender, ok := m["gender"]; ok {
        this.Gender = x.AsString(gender)
    }
    if email, ok := m["email"]; ok {
        this.Email = x.AsString(email)
    }
    if createdAt, ok := m["created_at"]; ok {
        this.CreatedAt = x.AsTime(createdAt)
    }
    if updatedAt, ok := m["updated_at"]; ok {
        this.UpdatedAt = x.AsTime(updatedAt)
    }
    
    // 初始化扩展属性
    this.Model.Ext = x.MAP{}
}

// ToMap 方法 - 将模型对象转换为 map
func (this *UserModel) ToMap() x.MAP {
    return x.MAP{
        "uid":         this.Uid,
        "name":        this.Name,
        "gender":      this.Gender,
        "email":       this.Email,
        "created_at":  this.CreatedAt,
        "updated_at":  this.UpdatedAt,
    }
}
```

### 自动生成的方法

每个生成的模型都包含以下标准方法：

#### 1. **构造函数**
```go
// 创建空模型实例
func ModelName() *ModelNameModel

// 从 map 创建模型实例
func NewModelName(m x.MAP) *ModelNameModel
```

#### 2. **Fill 方法** - 数据库数据格式化
```go
func (this *ModelName) Fill(m x.MAP)
```

**设计初衷和作用：**
- **数据格式化**：将数据库读取的原始 map 数据进行格式化和类型转换
- **类型转换**：自动处理数据类型转换，将任意类型转换为对应的 Go 类型
- **安全性**：支持可选字段的存在性检查，避免 panic

#### 3. **ToMap 方法** - 数据转换
```go
func (this *ModelName) ToMap() x.MAP
```

**作用：**
- 将结构化数据转换为 map 格式，便于序列化和其他处理
- 保持字段名与数据库列名的一致性

## 核心方法详解

### Fill 方法详解

`Fill` 方法是模型系统中的关键组件，负责将数据库查询结果转换为结构化的 Go 对象。

#### 功能特点

1. **类型转换**
```go
func (this *UserModel) Fill(m x.MAP) {
    if uid, ok := m["uid"]; ok {
        this.Uid = x.AsInt64(uid)  // 自动转换为 int64
    }
    if name, ok := m["name"]; ok {
        this.Name = x.AsString(name)  // 自动转换为 string
    }
    if createAt, ok := m["created_at"]; ok {
        this.CreatedAt = x.AsTime(createAt)  // 自动转换为 time.Time
    }
    // 初始化扩展属性
    this.Model.Ext = x.MAP{}
}
```

2. **类型转换**
- **字符串转换**：`x.AsString()` 确保字段为字符串类型
- **数值转换**：`x.AsInt64()`, `x.AsInt()` 等确保数值类型正确
- **时间转换**：`x.AsTime()` 处理各种时间格式
- **布尔转换**：`x.AsBool()` 处理布尔类型

3. **使用说明**
- **字符串转换**：`x.AsString()` 确保字段为字符串类型
- **数值转换**：`x.AsInt64()`, `x.AsInt()` 等确保数值类型正确
- **时间转换**：`x.AsTime()` 处理各种时间格式
- **布尔转换**：`x.AsBool()` 处理布尔类型

#### 设计初衷

**将数据库读取的原始 map 数据格式化后生成模型对象**，具体原因包括：

1. **数据标准化**：数据库返回的数据往往是 `map[string]any` 格式，需要转换为结构化的 Go 对象
2. **类型转换**：通过类型转换函数将任意类型转换为对应的 Go 类型
3. **操作便利**：提供业务代码更容易使用的结构化数据格式

#### 使用场景

```go
// DAO 层从数据库获取数据
func (this *UserDao) GetUser(id int64) (*model.UserModel, error) {
    data, err := this.GetRecord(id)
    if err != nil {
        return nil, err
    }
    
    // 使用 Fill 方法将数据库数据转换为结构化对象
    return model.NewUserModel(data), nil
}

// 批量数据处理
func (this *UserDao) GetUsers(params ...any) ([]*model.UserModel, error) {
    data, err := this.GetRecords(params...)
    if err != nil {
        return nil, err
    }
    
    var users []*model.UserModel
    for _, row := range data {
        users = append(users, model.NewUserModel(row))
    }
    return users, nil
}
```

### Map() 方法详解

`Map()` 方法是 `x.Mapper` 接口的实现，用于将模型对象快速转换为 map 格式。

#### Mapper 接口定义

```go
type Mapper interface {
    Map() map[string]any
}
```

#### 设计初衷

**在 DAO 底层的 sqlClient 操作数据时快速将模型对象转换为 map 数据**，具体原因包括：

1. **SQL 参数要求**：底层数据库操作通常需要 `map[string]any` 格式的参数
2. **性能优化**：避免手动逐个字段转换的性能开销
3. **统一接口**：提供标准化的数据转换接口
4. **简化操作**：在插入、更新等操作中简化数据准备过程

#### 实现方式

模型对象需要实现 `Mapper` 接口：

```go
func (this *UserModel) Map() map[string]any {
    return x.MAP{
        "uid":         this.Uid,
        "name":        this.Name,
        "gender":      this.Gender,
        "email":       this.Email,
        "created_at":  this.CreatedAt,
        "updated_at":  this.UpdatedAt,
    }
}
```

#### 使用场景

```go
// DAO 插入操作
func (this *UserDao) AddUser(user *model.UserModel) (int, error) {
    // 使用 Map() 方法快速转换为 map 格式
    return this.AddRecord(user.Map())
}

// DAO 更新操作  
func (this *UserDao) UpdateUser(user *model.UserModel) (int, error) {
    // 使用 Map() 方法获取更新数据
    return this.SetRecord(user.Map(), user.Uid)
}

// DAO 批量插入操作
func (this *UserDao) AddUsers(users []*model.UserModel) (int, error) {
    var maps []map[string]any
    for _, user := range users {
        maps = append(maps, user.Map())
    }
    return this.AddRecords(maps)
}
```

## 数据类型转换工具

框架提供了完整的数据类型转换工具函数：

### 基础转换函数

```go
// 字符串转换
x.AsString(value any) string

// 数值转换
x.AsInt(value any) int
x.AsInt64(value any) int64
x.AsFloat64(value any) float64

// 时间转换
x.AsTime(value any) time.Time

// 布尔转换
x.AsBool(value any) bool

// 切片转换
x.AsStringSlice(value any) []string
x.AsIntSlice(value any) []int
```

### 转换示例

```go
// 处理各种数据源
dbData := map[string]any{
    "uid":        123,           // int -> int64
    "name":       "张三",        // string -> string  
    "email":      "test@example.com",
    "created_at": "2024-01-01", // string -> time.Time
    "is_active":  1,            // int -> bool
}

// 使用类型转换函数
user := model.NewUserModel(dbData)
```

## 模型扩展属性

### Ext 字段的使用

基础的 `Model` 结构体包含 `Ext` 字段，用于存储扩展属性：

```go
type Model struct {
    Ext map[string]any `json:"-"` // 扩展属性，不参与 JSON 序列化
}
```

#### 使用场景

1. **临时数据存储**
```go
// 在业务逻辑中添加临时属性
user.Model.Ext["session_id"] = "abc123"
user.Model.Ext["permissions"] = []string{"read", "write"}
```

2. **动态属性**
```go
// 根据业务需要添加动态属性
user.Model.Ext["computed_field"] = user.Balance * 0.05 // 计算字段
```

3. **关联数据缓存**
```go
// 存储关联数据，避免重复查询
user.Model.Ext["user_roles"] = roles
user.Model.Ext["user_permissions"] = permissions
```

## 最佳实践

### 1. 模型设计原则

- **单一职责**：每个模型专注于单一业务实体
- **类型转换**：充分利用框架提供的类型转换功能
- **命名规范**：遵循 Go 和数据库的最佳命名习惯

### 2. 性能优化

- **避免不必要的转换**：在 DAO 层进行批量转换
- **合理使用 Ext 字段**：存储临时或动态数据
- **类型转换缓存**：对于大量数据处理，考虑缓存转换结果

### 3. 错误处理

模型创建过程中需要注意空值处理：

```go
// 模型创建时的空值检查
func NewUserModel(data map[string]any) (*UserModel, error) {
    if data == nil {
        return nil, errors.New("数据不能为空")
    }
    
    user := User()
    user.Fill(data)
    return user, nil
}
```

### 4. 测试建议

```go
// 模型单元测试示例
func TestUserModel_Fill(t *testing.T) {
    testData := map[string]any{
        "uid":         int64(123),
        "name":        "测试用户", 
        "email":       "test@example.com",
        "created_at":  "2024-01-01T00:00:00Z",
    }
    
    user := model.NewUserModel(testData)
    
    assert.Equal(t, int64(123), user.Uid)
    assert.Equal(t, "测试用户", user.Name)
    assert.Equal(t, "test@example.com", user.Email)
    assert.NotNil(t, user.CreatedAt)
}
```

## 总结

Nyx 框架的模型系统提供了：

1. **自动化生成**：通过脚手架快速生成标准化的模型代码
2. **类型转换**：完整的类型转换和格式化机制
3. **灵活扩展**：通过 Ext 字段支持动态属性
4. **性能优化**：Mapper 接口优化数据库操作性能
5. **开发效率**：标准化的 CRUD 操作和工具方法

这套模型系统显著提高了开发效率，同时保证了代码质量和性能表现。
