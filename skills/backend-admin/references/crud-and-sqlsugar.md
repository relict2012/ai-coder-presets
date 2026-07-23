# CRUD 模板与 SqlSugar 规范

## 标准 CRUD 方法结构

请求 CRUD 功能时必须生成完整结构，所有方法在 Service 内，使用 Mapster 映射。

```csharp
public class XxxService : IDynamicApiController, ITransient
{
    private readonly SqlSugarRepository<Xxx> _repo;

    // 1. 分页查询
    public async Task<PageResult<XxxOutput>> QueryXxx(QueryXxxInput input)
    {
        var query = _repo.AsQueryable()
            .WhereIF(!string.IsNullOrWhiteSpace(input.Keyword),
                x => x.Name.Contains(input.Keyword));

        return await query.ToPageAsync<Xxx, XxxOutput>(input);
    }

    // 2. 详情
    public async Task<XxxOutput> GetXxx(GetXxxInput input)
    {
        var entity = await _repo.GetByIdAsync(input.Id);
        return entity.Adapt<XxxOutput>();
    }

    // 3. 新增
    public async Task AddXxx(AddXxxInput input)
    {
        var entity = input.Adapt<Xxx>();
        await _repo.InsertAsync(entity);
    }

    // 4. 修改
    public async Task UpdateXxx(UpdateXxxInput input)
    {
        var entity = input.Adapt<Xxx>();
        await _repo.UpdateAsync(entity);
    }

    // 5. 删除
    public async Task DeleteXxx(DeleteXxxInput input)
    {
        await _repo.DeleteByIdAsync(input.Id);
    }

    // 6. 状态修改（如适用）
    public async Task SetXxxStatus(SetXxxStatusInput input)
    {
        var entity = await _repo.GetByIdAsync(input.Id);
        entity.Status = input.Status;
        await _repo.UpdateAsync(entity);
    }
}
```

## 对应 DTO 结构

```csharp
// 分页查询 - 继承 BasePageInput
public class QueryXxxInput : BasePageInput
{
    public string Name { get; set; }
}

// 详情/删除 - 继承 BaseIdInput
public class GetXxxInput : BaseIdInput { }
public class DeleteXxxInput : BaseIdInput { }

// 新增 - 自定义 Input
public class AddXxxInput
{
    public string Name { get; set; }
}

// 修改 - 自定义 Input，必须包含 Id
public class UpdateXxxInput
{
    public long Id { get; set; }
    public string Name { get; set; }
}

// 状态修改 - 继承 BaseStatusInput
public class SetXxxStatusInput : BaseStatusInput { }
```

## SqlSugar 查询规范

1. 查询必须使用 `AsQueryable()`
2. 分页必须使用 `ToPageAsync()` / `ToPagedListAsync()`
3. 禁止字符串拼接 SQL
4. 优先使用 Lambda 表达式
5. 条件查询用 `WhereIF` 避免空值干扰

复杂查询示例：

```csharp
var query = _repository.AsQueryable()
    .WhereIF(input.OrgId > 0, u => orgList.Contains(u.OrgId))
    .WhereIF(!_userManager.SuperAdmin, u => u.AccountType != AccountTypeEnum.SysAdmin)
    .WhereIF(!string.IsNullOrWhiteSpace(input.Keyword), u => u.Name.Contains(input.Keyword));

return await query.ToPagedListAsync(input.Page, input.PageSize);
```

## 对象映射

所有映射必须用 Mapster `.Adapt<T>()`，详见 `@references/mapster-reference.md`。

审计字段（CreateTime/CreateUserId 等）由 AOP 自动填充，禁止手动赋值。仅允许补充关联查询字段。
