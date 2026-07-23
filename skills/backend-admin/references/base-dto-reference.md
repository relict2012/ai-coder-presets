# 基础 DTO 体系参考

所有业务 DTO 必须合理继承以下基础 DTO，禁止重复定义基础字段。

## 继承关系

```
BaseIdInput (主键操作)
    └── BaseStatusInput (状态更新)

BaseFilter (筛选基类 - 抽象)
    └── BasePageInput (分页查询)
            └── BasePageTimeInput (带时间分页)

BaseImportInput (数据导入)
```

## BaseIdInput

单 ID 操作（查询、删除、详情等）。

```csharp
public class BaseIdInput
{
    [Required(ErrorMessage = "Id不能为空")]
    [DataValidation(ValidationTypes.Numeric)]
    public virtual long Id { get; set; }
}
```

示例：

```csharp
public class GetUserInput : BaseIdInput { }

public async Task<UserOutput> GetUser(GetUserInput input)
{
    var user = await _userRepository.GetByIdAsync(input.Id);
    return user.Adapt<UserOutput>();
}
```

## BaseImportInput

Excel 导入、批量数据导入。

```csharp
public class BaseImportInput
{
    [ImporterHeader(IsIgnore = true)]
    [ExporterHeader(IsIgnore = true)]
    public virtual long Id { get; set; }

    [ImporterHeader(IsIgnore = true)]
    [ExporterHeader("错误信息", ColumnIndex = 9999, IsBold = true, IsAutoFit = true)]
    public virtual string Error { get; set; }
}
```

示例：

```csharp
public class ImportUserInput : BaseImportInput
{
    [ImporterHeader(Name = "用户名")]
    [Required(ErrorMessage = "用户名不能为空")]
    public string UserName { get; set; }

    [ImporterHeader(Name = "邮箱")]
    public string Email { get; set; }
}
```

## BaseFilter

筛选过滤条件抽象基类。

```csharp
public abstract class BaseFilter
{
    public Search? Search { get; set; }
    public string? Keyword { get; set; }
    public Filter? Filter { get; set; }
}
```

辅助类：

```csharp
public class Search
{
    public List<string> Fields { get; set; }
    public string? Keyword { get; set; }
}

public class Filter
{
    public FilterLogicEnum? Logic { get; set; }
    public IEnumerable<Filter>? Filters { get; set; }
    public string? Field { get; set; }
    public FilterOperatorEnum? Operator { get; set; }
    public object? Value { get; set; }
}
```

## BasePageInput

分页查询（继承 BaseFilter）。

```csharp
public class BasePageInput : BaseFilter
{
    [DataValidation(ValidationTypes.Numeric)]
    public virtual int Page { get; set; } = 1;

    [DataValidation(ValidationTypes.Numeric)]
    public virtual int PageSize { get; set; } = 20;

    public virtual string Field { get; set; }
    public virtual string Order { get; set; }
    public virtual string DescStr { get; set; } = "descending";
}
```

## BasePageTimeInput

带时间范围的分页查询（继承 BasePageInput）。

```csharp
public class BasePageTimeInput : BasePageInput
{
    public DateTime? StartTime { get; set; }
    public DateTime? EndTime { get; set; }
}
```

示例：

```csharp
public class QueryUserInput : BasePageInput
{
    public string UserName { get; set; }
    public int? Status { get; set; }
}

public async Task<PageResult<UserOutput>> QueryUser(QueryUserInput input)
{
    var query = _userRepository.AsQueryable();

    if (!string.IsNullOrEmpty(input.Keyword))
        query = query.Where(u => u.UserName.Contains(input.Keyword));

    return await query.ToPageAsync<User, UserOutput>(input);
}
```

## BaseStatusInput

状态更新（继承 BaseIdInput）。

```csharp
public class BaseStatusInput : BaseIdInput
{
    [Enum]
    public StatusEnum Status { get; set; }
}
```

示例：

```csharp
public class SetUserStatusInput : BaseStatusInput
{
    public string Remark { get; set; }
}

public async Task SetUserStatus(SetUserStatusInput input)
{
    var user = await _userRepository.GetByIdAsync(input.Id);
    user.Status = input.Status;
    await _userRepository.UpdateAsync(user);
}
```

## 使用规则

1. 优先继承基础 DTO，根据业务场景选择
2. 禁止在子类重复定义 Id、Page、PageSize 等字段
3. 基础 DTO 属性都是 virtual，子类可按需重写
4. 基础 DTO 已包含验证，子类可补充业务验证
5. Input DTO **不包含**审计字段（CreateTime/CreateUserId/CreateUserName/UpdateTime/UpdateUserId/UpdateUserName），这些由框架 AOP 自动赋值；Output DTO 可包含审计字段用于展示
