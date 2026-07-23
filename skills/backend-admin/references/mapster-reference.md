# Mapster 对象映射参考

## 核心原则

Mapster 是本项目的**唯一对象映射方式**。所有 DTO ↔ Entity 转换必须通过 Mapster 完成，禁止逐字段手写赋值。

## 基础用法

### 单对象映射

```csharp
// Input → Entity（新增）
var entity = input.Adapt<XxxEntity>();

// Entity → Output（查询返回）
return entity.Adapt<XxxOutput>();

// 集合映射
var outputs = entities.Adapt<List<XxxOutput>>();
```

### 更新场景：先查后映射

```csharp
// 修改操作：先从数据库查出实体，再映射覆盖
var entity = await _repo.GetByIdAsync(input.Id);
input.Adapt(entity);  // 将 input 属性映射到已有 entity 上
await _repo.UpdateAsync(entity);
```

> ⚠️ **关键区别**：
> - `.Adapt<T>()` 创建新对象 → 用于新增
> - `.Adapt(entity)` 映射到已有对象 → 用于修改

## 映射配置（仅当名称不一致时使用）

当 DTO 属性名和 Entity 属性名不一致，才需要配置：

```csharp
// 全局配置（在 Startup / AppStartup 中）
TypeAdapterConfig<XxxInput, XxxEntity>
    .NewConfig()
    .Map(dest => dest.RealName, src => src.Name);

// 或内联配置（临时使用）
var entity = input.Adapt<XxxEntity>(config =>
    config.Map(dest => dest.RealName, src => src.Name));
```

## 审计与框架字段（禁止手动赋值）

AOP 在 SqlSugar 的 **Insert/Update** 拦截中自动赋值审计字段，**禁止在 Service 中手动赋值**：

| 来源基类 | AOP 赋值字段 | 触发时机 |
|----------|-------------|----------|
| `EntityBase` | `CreateTime` / `CreateUserId` / `CreateUserName` | Insert 时赋值，Update 时忽略 |
| `EntityBase` | `UpdateTime` / `UpdateUserId` / `UpdateUserName` | Update 时赋值 |
| `EntityBaseOrg` 系列 | `OrgId` | Insert/Update 时按当前用户机构赋值 |
| 租户基类系列 | `TenantId` | Insert/Update 时按当前租户赋值 |

**以下字段不在 Insert/Update AOP 范围**，机制不同：

| 字段 | 机制 |
|------|------|
| `IsDelete` | 默认值 `= false`；软删除操作时设为 `true` |
| `DeleteTime` | 软删除操作时赋值 |

→ Input DTO **不包含**这些字段；`.Adapt<T>()` 会自动跳过目标对象中源 DTO 没有的属性。

```csharp
// ✅ 正确：AOP 字段由框架处理，无需手动赋值
var entity = input.Adapt<XxxEntity>();
await _repo.InsertAsync(entity);  // AOP 自动填充审计字段（CreateTime/CreateUserId 等）

// ❌ 错误：手动赋值 AOP 管理的字段
entity.CreateTime = DateTime.Now;       // 禁止！AOP 会处理
entity.CreateUserId = _userManager.UserId; // 禁止！AOP 会处理
entity.OrgId = _userManager.OrgId;     // 禁止！（EntityBaseOrg 的 AOP 会处理）
```

## 允许手动补充的场景

仅在以下场景允许在 `.Adapt<T>()` 后补充少量赋值（1-3 行）：

```csharp
// ✅ 允许：需要额外查询才能获得的关联字段
entity.OrgName = await _orgRepo.GetNameById(input.OrgId);

// ✅ 允许：属性名不一致且未配置 TypeAdapterConfig 时（优先配置 Config）
entity.RealName = input.Name;
```

## 反模式（绝对禁止）

### ❌ 逐字段手写赋值

```csharp
// 禁止！
entity.Name = input.Name;
entity.Phone = input.Phone;
entity.Email = input.Email;
entity.Address = input.Address;
entity.Remark = input.Remark;
```

→ 应改为：`var entity = input.Adapt<XxxEntity>();`

### ❌ 全手动 DTO → Entity

```csharp
// 禁止！
var output = new XxxOutput
{
    Id = entity.Id,
    Name = entity.Name,
    Phone = entity.Phone
};
```

→ 应改为：`return entity.Adapt<XxxOutput>();`

### ❌ 手动赋值审计字段

```csharp
// 禁止！审计字段由 AOP 自动填充
entity.CreateTime = DateTime.Now;
entity.CreateUserId = _userManager.UserId;
```

→ 直接 `input.Adapt(entity)` 后 `UpdateAsync`，AOP 会自动处理

### ❌ 查询结果逐字段拼接

```csharp
// 禁止！
return new XxxOutput
{
    Id = item.Id,
    Name = item.Name,
    DeptName = dept.Name
};
```

→ 应改为：配置 Mapster 映射 DeptName，然后 `.Adapt<XxxOutput>()`

## 常见场景速查

| 场景 | 用法 |
|------|------|
| 新增 | `input.Adapt<Entity>()` |
| 修改 | `input.Adapt(existingEntity)` |
| 查询返回 | `entity.Adapt<Output>()` |
| 集合返回 | `list.Adapt<List<Output>>()` |
| 分页返回 | `query.ToPageAsync<Entity, Output>(input)` |
| 属性名不一致 | `TypeAdapterConfig` 配置映射 |
| 需要额外字段 | `.Adapt<T>()` 后补充关联查询赋值 |
