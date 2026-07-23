# Entity 基类体系参考

所有业务 Entity 必须继承合适的基类，禁止重复定义基类已有的字段。

## 继承关系

```
EntityBaseId (雪花Id)
    ├── EntityBase (审计字段)
    │       ├── EntityBaseDel (软删除)
    │       │       └── EntityBaseOrgDel (软删除 + 机构)
    │       ├── EntityBaseOrg (机构数据权限)
    │       │       └── EntityBaseTenantOrg (机构 + 租户)
    │       ├── EntityBaseTenant (租户)
    │       │       ├── EntityBaseTenantDel (租户 + 软删除)
    │       │       └── EntityBaseTenantOrg (租户 + 机构) ─→ 已重复，见上
    │       └── EntityBaseTenantOrgDel (租户 + 软删除 + 机构)
    └── EntityBaseTenantId (租户但无审计字段)
```

## 各基类字段明细

### EntityBaseId

所有 Entity 的根基类，仅提供主键。

```csharp
public abstract class EntityBaseId
{
    [SugarColumn(ColumnName = "Id", ColumnDescription = "主键Id",
        IsPrimaryKey = true, IsIdentity = false)]
    public virtual long Id { get; set; }
}
```

### EntityBase（最常用）

继承 `EntityBaseId`，包含审计字段（AOP 自动赋值）。

```csharp
public abstract class EntityBase : EntityBaseId
{
    [SugarColumn(ColumnDescription = "创建时间", IsNullable = true, IsOnlyIgnoreUpdate = true)]
    public virtual DateTime CreateTime { get; set; }

    [SugarColumn(ColumnDescription = "更新时间")]
    public virtual DateTime? UpdateTime { get; set; }

    [OwnerUser]
    [SugarColumn(ColumnDescription = "创建者Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? CreateUserId { get; set; }

    [SugarColumn(ColumnDescription = "创建者姓名", Length = 64, IsOnlyIgnoreUpdate = true)]
    public virtual string? CreateUserName { get; set; }

    [SugarColumn(ColumnDescription = "修改者Id")]
    public virtual long? UpdateUserId { get; set; }

    [SugarColumn(ColumnDescription = "修改者姓名", Length = 64)]
    public virtual string? UpdateUserName { get; set; }
}
```

> 注：`IsOnlyIgnoreUpdate = true` 表示新增时 AOP 赋值，更新时忽略（不会覆盖原值）。

### EntityBaseDel（软删除）

继承 `EntityBase`，增加软删除字段。**注意：这些字段不是 Insert/Update AOP 赋值的**。

- `IsDelete`：字段默认值 `= false`，Insert 时自带；软删除操作时设为 `true`
- `DeleteTime`：软删除操作时赋值，不在 Insert/Update AOP 范围
- `IDeletedFilter`：使 SqlSugar SELECT 自动加 `WHERE IsDelete = false`，无需手动过滤

```csharp
public abstract class EntityBaseDel : EntityBase, IDeletedFilter
{
    [SugarColumn(ColumnDescription = "软删除")]
    public virtual bool IsDelete { get; set; } = false;

    [SugarColumn(ColumnDescription = "软删除时间")]
    public virtual DateTime? DeleteTime { get; set; }
}
```

> 使用 `EntityBaseDel` 时，SqlSugar 查询会自动过滤 `IsDelete == false`，无需手动加 Where 条件。

### EntityBaseOrg（机构数据权限）

继承 `EntityBase`，增加机构字段（AOP 按当前用户机构自动赋值）。

```csharp
public abstract class EntityBaseOrg : EntityBase, IOrgIdFilter
{
    [SugarColumn(ColumnDescription = "机构Id", IsNullable = true)]
    public virtual long OrgId { get; set; }
}
```

> 使用 `EntityBaseOrg` 时，SqlSugar 查询会自动按当前用户的机构过滤数据。

### EntityBaseOrgDel（机构 + 软删除）

```csharp
public abstract class EntityBaseOrgDel : EntityBaseDel, IOrgIdFilter
{
    [SugarColumn(ColumnDescription = "机构Id", IsNullable = true)]
    public virtual long OrgId { get; set; }
}
```

### EntityBaseTenant（多租户）

继承 `EntityBase`，增加租户字段（AOP 按当前租户自动赋值）。

```csharp
public abstract class EntityBaseTenant : EntityBase, ITenantIdFilter
{
    [SugarColumn(ColumnDescription = "租户Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? TenantId { get; set; }
}
```

> 使用租户基类时，SqlSugar 查询会自动按 TenantId 过滤，无需手动加 Where 条件。

### EntityBaseTenantDel（多租户 + 软删除）

```csharp
public abstract class EntityBaseTenantDel : EntityBaseDel, ITenantIdFilter
{
    [SugarColumn(ColumnDescription = "租户Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? TenantId { get; set; }
}
```

### EntityBaseTenantOrg（多租户 + 机构）

```csharp
public abstract class EntityBaseTenantOrg : EntityBaseOrg, ITenantIdFilter
{
    [SugarColumn(ColumnDescription = "租户Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? TenantId { get; set; }
}
```

### EntityBaseTenantOrgDel（多租户 + 软删除 + 机构）

```csharp
public abstract class EntityBaseTenantOrgDel : EntityBaseOrgDel, ITenantIdFilter
{
    [SugarColumn(ColumnDescription = "租户Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? TenantId { get; set; }
}
```

### EntityBaseTenantId（租户但无审计字段）

继承 `EntityBaseId`（而非 EntityBase），适用于不需要审计字段的租户场景。

```csharp
public abstract class EntityBaseTenantId : EntityBaseId, ITenantIdFilter
{
    [SugarColumn(ColumnDescription = "租户Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? TenantId { get; set; }
}
```

## 选择规则

1. **默认选择 `EntityBase`** — 大多数业务实体
2. **需要软删除？** → 加 `Del`（如 `EntityBaseDel`）
3. **需要机构数据权限？** → 加 `Org`（如 `EntityBaseOrg`）
4. **多租户系统？** → 加 `Tenant`（如 `EntityBaseTenant`）
5. **组合需求？** → 按组合选择（如 `EntityBaseTenantOrgDel`）
6. **不需要审计字段但需要租户？** → `EntityBaseTenantId`

禁止：
- ❌ 在 Entity 子类中重复定义 Id / CreateTime / IsDelete / OrgId / TenantId 等基类字段
- ❌ 手动赋值 AOP 管理的字段
- ❌ 在查询中手动加 `Where(x => !x.IsDelete)` 等过滤条件（基类已自动处理）
