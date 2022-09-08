---
title: EF Core导航属性那点事
date: 2022-08-23 11:43:33
tags:
    - Entity Framework
    - 备忘录
    - EF Core
    - 导航属性
---
### 依赖关系
EF默认情况下的依赖推测已经相当敏捷了,在大部分情况下都不用去指定依赖关系,比如在下面这种的博客-用户-标签关系中:
```csharp
class Blog{
    public int Id { get; set; }
    public List<Tag> Tags { get; set; }
    public User Author { get; set; }
    public int AuthorId { get; set; }
}

class User{
    public int  Id
    public string Name { get; set; }
    public List<Blog> Blog { get;set; }
}

class Tag {
    public int Id { get; set; }
    public List<Blog> Blogs { get;set; }
}
```
上面这种依赖关系非常常见,博客跟用户是一对多,博客跟标签是多对多.这种情况下不需要做额外配置,包括中间表BlogTag EF也会隐式帮我们生成.包括Blog如果不显式设置UserId也会生成一个对应的外键(EF里面叫影子外键)

这种情况下依赖关系是很明确的,但是在新增的时候有一个问题,比如我只指定了Blog的Auhtor,而没有去指定UserId,然后用DbConetxt去Add呢?比如:
```csharp
var blog = new Blog{
    Author = await dbContext.FindAsync(1);
}
await dbContext.AddAsync(blog);
await dbContext.SaveChangesAsync();
```
这种情况下EF会尝试把Author当新值去插入,但是由于此情况下User是已经存在的项,插入的时候自然会报错.在这个[回答](https://stackoverflow.com/a/50906962/15117498)中解释了这一点,并给出了三种解决方案:

-  修改实体状态为Added而不是使用Add方法:
    ```csharp
    _context.Entry(Product).State = EntityState.Added;
    await _context.SaveChangesAsync();
    ```
- 在Add之前附加导航属性对象
    ```csharp
    if (Product.Shop != null) _context.Attach(Product.Shop);
    _context.Products.Add(Product);
    await _context.SaveChangesAsync();
    ```
- 使用Update代替Add
    ```chsarp
    _context.Products.Update(Product);
    await _context.SaveChangesAsync();
    ```
在实际中,~~我选择第三个方法会导致第二次Update会出现主键冲突错误,而第一种方案没有这个问题~~.答案末尾也针对不推荐使用这种方法的原因说明了:
>The last technique is explained in Saving Data - Disconnected Entities - Mix of new and existing entities:
>> With auto-generated keys, Update can again be used for both inserts and updates, even if the graph contains a mix of entities that require inserting and those that require updating  
>
>Since it works only when all entities use auto-generated PKs, and also produces unnecessary updates of the related entities, I don't recommend it.

**主键冲突错误是因为在Blazor中使用DbContextFacotry产生的DbContext不一致的原因,与Update无关**
### 复杂导航属性
比如上述场景,如果文章多出一个联合发布人的情况:
```csharp
class Blog{
    public int Id { get; set; }
    public List<Tag> Tags { get; set; }
    public User Author { get; set; }
    public List<User> UnionAuthor {get;set;}
    public int AuthorId { get; set; }
}

class User{
    public int  Id
    public string Name { get; set; }
    public List<Blog> Blog { get; set; }
    public List<Blog> UnionBlog {get; set;}
}
```
这种情况下EF没法识别依赖关系,需要在OnModelCreating中指定这点
```csharp
modelBuilder.Entity<Blog>()
            .HasOne(t => t.Author)
            .WithMany(t => t.Blog)
            .HasForeignKey(t=>t.AuthorId);
modelBuilder.Entity<Blog>()
            .HasMany(t => t.UnionAuthor)
            .WithMany(t => t.UnionBlog);
```
### 转换器
转换器是EF Core2.1带来的功能,按理来说转换器不属于导航属性的内容,但是它是反其道而行,可以在一定程度上避免使用导航属性,比如在Blog中添加一个`List<string> Imgs`这种情况,在以前情况无法直接去定义这种属性,不得不去额外定义一个表,而值转换器解决了这个问题:
```csharp
 modelBuilder.Entity<Blog>()
            .Property(e => e.Imgs)
            .HasConversion(
                v => string.Join(",", v != null ? v.ToArray() : Enumerable.Empty<string>()),
                v => v.Split(',', StringSplitOptions.RemoveEmptyEntries),
                new ValueComparer<ICollection<string>>(
                    (c1, c2) => c1 != null && c2 != null && c1.SequenceEqual(c2),
                    c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.GetHashCode())),
                    c => c));
```
`HasConversion`第一个参数为序列化方法,第二个参数为反序列方法,第三个则为值比较方法(用于实体追踪)
不仅仅是List,也可以转换各种结构体或者类
```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }

    public IList<AnnualFinance> Finances { get; set; }
}

modelBuilder.Entity<Blog>()
    .Property(e => e.Finances)
    .HasConversion(
        v => JsonSerializer.Serialize(v, (JsonSerializerOptions)null),
        v => JsonSerializer.Deserialize<List<AnnualFinance>>(v, (JsonSerializerOptions)null),
        new ValueComparer<IList<AnnualFinance>>(
            (c1, c2) => c1.SequenceEqual(c2),
            c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.GetHashCode())),
            c => (IList<AnnualFinance>)c.ToList()));
```
这种序列号也就相当于EF自动处理了以前在业务层写的序列化转换部分.除了自定义转换器外,EF Core也内置了一些默认转换器,比如最简单的Bool to Int(在Sqlite中就会默认执行此处理):
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder
        .Entity<User>()
        .Property(e => e.IsActive)
        .HasConversion<int>();
}
```
## 引用
- [EF文档-值转换](https://docs.microsoft.com/zh-cn/ef/core/modeling/value-conversions?tabs=data-annotations)
- [ef core one to many relationship throw exception Cannot add or update a child row](https://stackoverflow.com/questions/50889676/ef-core-one-to-many-relationship-throw-exception-cannot-add-or-update-a-child-ro)