---
title: 领域驱动ABP框架入门基础
date: 2022-07-08 14:02:25
tags: 
    - ABP
    - 领域驱动
    - DDD
    - ASP.NET Core
---
# ABP框架入门基础
ABP是一个开源且文档友好的应用程序框架。ABP不仅仅是一个框架，它还提供了一个最徍实践的基于领域驱动设计(DDD)的体系结构模型，可以支持 .net framework和 .net core两种技术流派。
## 概念
### 领域驱动
基于它的设计理念,在介绍ABP之前,先简单介绍下领域驱动概念设计.  
领域驱动(Domain-Driven Design ),并不是一种技术架构，而是一种划分业务领域范围的方法论。具体来说它是一种拆解业务、划分业务、确定业务边界的方法， 是一种高度复杂的领域设计思想，将问题拆分成一个个的域，试图分离技术实现的复杂性，主要解决的是软件难以理解难以演进的问题，目的就是将复杂问题领域简单化，帮助我们设计出清晰的领域和边界，可以很好的实现技术架构的演进。  

以保险业务为例来进行编程实践，一个高度抽象的保险领域划分如图所示。通过用例分析，我们把整个业务划分成产品域、承保、核保、理赔等多个领域（Bounded-Context），每个领域又可以根据业务发展情况拆分子域。

![](../imgs/领域驱动DDD/20220815141237.png)  
### 分层
在每个领域内部，相对于 MVC 对应用三层架构的拆分，领域驱动的设计将应用模块内部分为如图示的四层。
![](../imgs/领域驱动DDD/20220815141500.png)  
- 用户接口(表示层): 为用户提供接口. 使用应用层实现与用户交互.
- 应用层: 表示层与领域层的中介,编排业务对象执行特定的应用程序任务. 使用应用程序逻辑实现用例.
- 领域层: 包含业务对象以及业务规则. 是应用程序的核心.
- 基础设施层: 提供通用的技术功能,支持更高的层,主要使用第三方类库.
### 内容
- 领域层:
  - 实体:DDD中要求实体是唯一的且可持续变化的。业务中最常见的唯一标识的用户就是实体,比如普通人通过身份证对其进行唯一标识,不管你年龄住址等信息如何变动,你依然是你.
  - 聚合与聚合根:我们把一些关联性极强、生命周期一致的实体、值对象放到一个聚合里。聚合是领域对象的显式分组，旨在支持领域模型的行为和不变性，同时充当一致性和事务性边界。如果聚合根比喻成一个小组,那么聚合根就是组长,通过他可以快速定位到这个小组. 以上面所属的保险行业为例定义的聚合和聚合根:  
  ![](../imgs/领域驱动DDD/20220815144413.png)  
  - 值对象:当你只关心某个对象的属性时，该对象便可作为一个值对象。 我们需要将值对象看成不变对象，不要给它任何身份标识，还应该尽量避免像实体对象一样的复杂性。比如上面保单的客户实体,他的地址就可以看成一个值对象.
  - 仓储:仓储介于领域模型和数据模型之间，主要用于聚合的持久化和检索。它隔离了领域模型和数据模型.这点其实和我们以往在EF Core或者FreeSql上使用的仓储模式区别不大.
  - 领域服务:领域中的一些概念不太适合建模为对象，即归类到实体对象或值对象，因为它们本质上就是一些操作，一些动作，而不是事物。  
    - 可以使用领域服务的情况：
        - 执行一个显著的业务操作
        - 对领域对象进行转换
        - 以多个领域对象作为输入参数进行计算，结果产生一个值对象
  - 规约:顾名思义,是一些规则/约束条件,比如银行在贷款的时候会考虑对方的还款能力和信用记录,这里的考虑条件就是规约.

- 应用服务层
  - 应用服务:应用服务实现应用程序的用例, 将领域层逻辑公开给表示层,从表示层(可选)调用应用服务,DTO (数据传输对象) 作为参数. 返回(可选)DTO给表示层.这有点类似Controller.
  - 数据传输对象(DTO):用于在应用层和表示层或其他类型的客户端之间传输数据,这在ABP中是可选但是推荐使用的,其实就是一直在用的视图模型(Req/Rsp)
  - 工作单元(UOW):提供了对应用程序中的数据库连接和事务范围的抽象和控制.ABP的工作单元按约定工作, 所以大部分情况下你不需要处理UOW,一旦一个新的UOW启动,它将创建一个环境作用域,当前作用域中执行的所有数据库操作都将参与该作用域并将其视为单个事务边界. 操作一起提交(成功时)或回滚(异常时).

## 开发环境
以官方文档的Web MVC Razor应用程序为例:
需要先安装Node.js 12/14  
安装ABP CLI:  
`dotnet tool install -g Volo.Abp.Cli`  
更新:  
`dotnet tool update -g Volo.Abp.Cli`  
创建项目模板:  
- 默认MVC: `abp new Acme.BookStore`  
- WebAPI: `abp new Acme.BookStore -u none`
>**如果运行时提示找不到库文件,请使用PowerShell执行以下命令安装Npm包:**  
>
>1. Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process  
>2. npm install -g npm-windows-upgrade  
>3. npm-windows-upgrade  

可以得到下图的项目结构:  
![](../imgs/领域驱动DDD/20220815154934.png)  
其中,Acme.BookStore.DbMigrator项目为数据库迁移项目,初次运行它,会自动安装ef tool(如果使用ef core)进行数据库迁移并设置种子数据.
项目分层及依赖结构:
![](../imgs/领域驱动DDD/20220815155252.png)  

- Domain：主要包含实体和域服务
- Domain.Shared：可以与客户端共享的其他与域相关的对象（枚举或其它与实体相关的用于引用
的类）,比如本地化文件.
- EntityFrameworkCore：EF Core的集成项目
- Application.Contracts：包含应用服务(Service)的接口(IService)以及应用服务层(.Application)的
DTO(Data Transfer Objects)
- Application：包含应用服务(Service)，是.Application.Contracts中的IService接口实现
- HttpApi: 用于定义API Controllers
- HttpApi.Client：定义C#客户端代理以使用解决方案的HTTP API的项目，可以将此库共享给第三方
客户端以便在其他DotNet应用程序中使用该项目HTTP API
其中,运行项目Web根据创建模板也有命名差异,比如为Blazor模板时即为Blazor,如果是使用WebAPI模板,则为HttpApi.Host,其中除了program.cs外,还有  
![](../imgs/领域驱动DDD/20220816105806.png)  
`{项目模板名}Module` 、 `{项目模板名}BrandingProvider` 、 `{项目模板名}AutoMapperProfile`  
三个类,其中最重要的`Module`是整个项目的配置文件类,大部分配置项都在其中定义,比如路由、I18N、中间件等等;`AutoMapperProfile`则是自动映射工具的配置规则(默认集成了AutoMapper);`BrandingProvider`则是MVC/Razor下UI层的视图显示配置(比如AppName和LogoUrl之类的).

#### 默认集成类库
列举一些ABP自带和集成的一些常用、可选的工具类库:
| 用途 | 类库 |
| - | -
| 日志 | Serilog
| 依赖注入 | AutoFac
| 对象映射 | AutoMapper
| 模型验证 | FluentValidation
| 权限控制 | IdentityServer4
| 后台作业 | Hangfile/RabbitMQ/Quartz
| 虚拟文件 | VirtualFileSystem
| 分布式锁 | DistributedLock
| 单元测试 | xunit
### 模板应用
以官网教程为例,对一本书的最简单的增删改查操作流程如下:
创建实体模型:
在Domain下添加实体模型
```csharp
public class Book:AuditedAggregateRoot<Guid>
{
    public Book(Guid id):base(id){

    }
    public string Name { get; set; }
    [MaxLength(120)]
    public string ConverUrl { get; set; }
    public string Author { get; set; }
    public string Publisher { get; set; }
    public decimal Price { get; set; }
    public DateTime PublicationDate { get; set; }
}
```
ABP为实体提供了两个基本的基类: AggregateRoot和Entity.也就是我们上文提到的聚合根和实体——聚合根一定定义的是实体
`BasicAggregateRoot` 是创建聚合根的最简单的基础类,AggregateRoot在其之上添加了一些基础审计功能,比如(CreationTime、CreatorId、 LastModificationTime、LastModifierId等). `Guid` 是实体的主键 (Id).添加默认有参构造函数来确保主键不会在误操作下被无序生成(比如对象映射时),在插入时,通过IGuidGenerator.Create()(ApplicationService中默认包含)来产生有序Guid.**永远不要使用Guid.New()来生成主键**  
然后在`EntityFrameworkCore`项目下的`{项目名}DbContext`类中添加DbSet
```csharp
 public DbSet<Book> Books { get; set; }
```
添加完成后,分别使用ef tool的`add-migrattion up`和`update-database`将添加迁移并更更新到数据库中.  
![](../imgs/领域驱动DDD/20220815170248.png)  
然后在`Application.Contract`中添加`IBookAppService`接口和`BookDto`类
```csharp
public class BookAddReq
{
    public class BookAddReq
{
    [Required]
    public string Name { get; set; }
    [Required]
    public string ConverUrl { get; set; }
    [Required]
    public string Author { get; set; }
    [Required]
    public string Publisher { get; set; }
    [Required]
    public decimal Price { get; set; }
    [Required]
    [DataType(DataType.Date)]
    public DateTime PublicationDate { get; set; }
}
}

public class BookRsp
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string ConverUrl { get; set; }
    public string Author { get; set; }
    public string Publisher { get; set; }
    public decimal Price { get; set; }
    public DateTime PublicationDate { get; set; }
}

public interface IBookAppService : IApplicationService
{
    Task<List<BookRsp>> GetListAsync();
    Task<BookRsp> CreateAsync(BookAddReq req);
    Task DeleteAsync(Guid id);
}
```
IApplicationService并没有定义任何操作方法,也可以通过实现ICrudAppService来定义最基础的CURD方法  
在AutoMapperProfile中设置对象映射关系(如果使用AutoMapper的话):
```csharp  
CreateMap<BookAddReq, Book>();
CreateMap<Book, BookRsp>();
```
然后在`Application`中实现`IBookService`:
```csharp
//应用服务需要继承ApplicationService
public class BookAppService : ApplicationService,IBookAppService
{
    private readonly IRepository<Book, Guid> _bookRepository;
    public BookAppService(IRepository<Book, Guid> repository)
    {
        _bookRepository = repository;
    }
    public async Task<BookRsp> CreateAsync(BookAddReq req)
    {
        var book = ObjectMapper.Map(req,new Book(GuidGenerator.Create()));
        await _bookRepository.InsertAsync(book);
        return ObjectMapper.Map<Book, BookRsp>(book);
    }

    public async Task DeleteAsync(Guid id)
    {
        await _bookRepository.DeleteAsync(id);
    }

    public async Task<List<BookRsp>> GetListAsync()
    {
        return (await _bookRepository.GetListAsync())
            .ToList()
            .Adapt<List<BookRsp>>();
    }
}
```
同时,默认的仓储支持通过仓储对象获取dbcontext和Dbset:
```csharp
var dbContext = await _bookRepository.GetDbContextAsync();
var dbSet = await _bookRepository.GetDbSetAsync();
```

Service创建完成后,启动项目打开`/Swagger`,即可看到的Restful API,这部分我们并没有在Controller里面去定义,这是ABP通过动态API默认实现的.
![](../imgs/领域驱动DDD/20220816101032.png)  
| Service方法名称 | HttpMethod | 路由 
| ---------| --------- | ----
| GetAsync(Guid id) |GET|	/api/app/book/{id}
GetListAsync()|	GET|/api/app/book
CreateAsync(CreateBookDto input)|POST|/api/app/book
UpdateAsync(Guid id, UpdateBookDto input)|PUT|/api/app/book/{id}
DeleteAsync(Guid id)|DELETE|/api/app/book/{id}
GetEditorsAsync(Guid id)|GET|/api/app/book/{id}/editors
CreateEditorAsync(Guid id, BookEditorCreateDto input)	|POST|	/api/app/book/{id}/editor

如果不想某个Service或者方法自动生成动态API,可以使用`[RemoteService(IsEnabled = false)]`特性来禁止生成行为
同时,即便不是在Controller中,service依然支持使用标准ASP.NET Core的特性(`[HttpPost]`, `[HttpGet]`, `[HttpPut]`... 等等.)来设置其路由生成行为.但是**无法同时支持`[Route]`路由特性**;
如果需要设置路由,则需要在`Module`下进行配置  
```csharp
private void ConfigureAutoApiControllers()
{
    Configure<AbpAspNetCoreMvcOptions>(options =>
    {
        options.ConventionalControllers.Create(typeof(BookStoreApplicationModule).Assembly, opt =>
        {
            //do somethings
            // opt.RootPath 设置根路由
            // opt.UrlControllerNameNormalizer 设置控制器生成
        });
    });
}
```

动态API并非ApplicationService的专利,只要实现了IRemoteService,那么就会成为APIController,事实上,ApplicationService就是实现了IRemoteService.

#### 依赖注入
上面过程中,可能会注意到在写完Service后,并没有像在Asp.Net Core中使用Service.AddScope()来注入,这是ABP引入了依照约定的服务注册.依照约定你无需做任何事,它会自动完成.  

固有的注册类型:
- 模块类注册为singleton.
- MVC控制器（继承Controller或AbpController）被注册为transient.
- MVC页面模型（继承PageModel或AbpPageModel）被注册为transient.
- MVC视图组件（继承ViewComponent或AbpViewComponent）被注册为transient.
- 应用程序服务（实现IApplicationService接口或继承ApplicationService类）注册为transient.
- 仓储库（实现IRepository接口）注册为transient.
- 域服务（实现IDomainService接口）注册为transient.
  

使用接口或者特性进行注册:
- 接口注册
  - ITransientDependency 注册为transient生命周期.
  - ISingletonDependency 注册为singleton生命周期.
  - IScopedDependency 注册为scoped生命周期.
- 特性注册,使用`[Dependency]`特性,它具有以下属性:
    - Lifetime: 注册的生命周期:Singleton,Transient或Scoped.
    - TryRegister: 设置true则只注册以前未注册的服务.使用IServiceCollection的TryAdd ... 扩展方法.
    - ReplaceServices: 设置true则替换之前已经注册过的服务.使用IServiceCollection的Replace扩展方法.
  
当然.你也可以禁用它并使用手动注册,由于其集成了AutoFac,所以也支持属性注入.  

#### 权限控制
ABP将Asp.Net Core的Authorize带到了ApplicationService,使得应用服务仍旧支持`[Authorize]`和`[AllowAnonymous]`特性.
```csharp
 [Authorize]
public class AuthorAppService : ApplicationService, IAuthorAppService
{
    public Task<List<AuthorDto>> GetListAsync()
    {
        ...
    }

    [AllowAnonymous]
    public Task<AuthorDto> GetAsync(Guid id)
    {
        ...
    }
}
```
如果需要自定义权限策略,可以按照ASP.NET Core文档进行实施策略授权,但对于简单的 true/false 条件(比如是否授予了用户策略) ABP定义了权限系统,可以为特定用户,角色或客户端授权或禁止的简单策略.
在Application.Contract下的Permissions文件夹中包含了`{项目名}PermissionDefinitionProvider`和`{项目名}Permission`  

在`Permission`中可以定义一些权限的硬编码,比如角色名,权限值名之类.  
在`PermissionDefinitionProvider`的Define中自定义权限组和权限:
```csharp
public override void Define(IPermissionDefinitionContext context)
{
    var myGroup = context.AddGroup(BookStorePermissions.GroupName);
    var permission = myGroup.AddPermission("Books");
    permission.AddChild("CreateBook");
    permission.AddChild("DeleteBook");
    permission.AddChild("GetBook");
}
```

在ABP自带的Web端的权限管理中(如果是带UI的模板)可以看到自定义的权限并将其分配给角色和用户  
![仅当拥有父级权限时子权限才可被选中](../imgs/领域驱动DDD/20220816174421.png)  
ABP会将其进行持久化存入数据库内  
然后便可以在ApplicationService中设置对应的权限:
```csharp
[Authorize("Books")]
public class BookAppService : ApplicationService,IBookAppService
{
    public Task<List<AuthorDto>> GetListAsync()
    {
        ...
    }

    [Authorize("CreateBook")]
    public async Task<BookRsp> CreateAsync(BookAddReq req)
    {
        ...
    }
}
```
WebAPI下,ABP在Swagger中默认启用Oauth2授权码模式
![ Oauth2授权码模式与第三方平台例如微信扫码登录一致,登录后跳转返回jwt](../imgs/领域驱动DDD/20220817135244.png) 

> **由于ABP默认集成了IdentityServer,所以未登录的情况下请求接口会跳转至`/Account/Login`而非返回401,这会给前端带来困扰.需要在AddAuthentication()中使用JwtBearerDefaults.AuthenticationScheme参数来指定默认的认证方案(IdentityServer默认会使用Cookie方案).**

ApplicationService中封装了CurrentUser代替Controller中的User,同时,你也可以使用AuthorizationService来检查权限.

#### 异常处理

 ABP了集成常用的异常处理,大部分情况下都不用自己来自定义业务异常.ABP会自动处理所有异常 .如果是API/AJAX请求,会向客户端返回一个标准格式化后的错误消息,默认情况下ABP会执行以下处理:
- 自动隐藏内部详细错误 并返回标准错误消息.
- 为异常消息的本地化 提供一种可配置的方式.
- 自动为标准异常设置 HTTP状态代码 ,并提供可配置选项,以映射自定义异常

当满足下面任意一个条件时,AbpExceptionFilter 会处理此异常:

- 当controller action方法返回类型是object result(而不是view result)并有异常抛出时.
- 当一个请求为AJAX(Http请求头中X-Requested-With为XMLHttpRequest)时.
- 当客户端接受的返回类型为application/json(Http请求头中accept 为application/json)时.

如果异常被处理过,则会自动记录日志并将格式化的JSON消息返回给客户端.
绝大部分异常都是业务异常,可以直接使用BusinessException抛出并记录,BusinessException 除了实现IHasErrorCode,IHasErrorDetails ,IHasLogLevel 接口外,还实现了IBusinessException 接口.其默认日志级别为Warning.
```csharp
public BusinessException(string code = null, string message = null, string details = null, Exception innerException = null, LogLevel logLevel = LogLevel.Warning)
        : base(message, innerException)
    {
        Code = code;
        Details = details;
        LogLevel = logLevel;
    }
```
如果要直接显示具体错误原因,可以使用UserFriendlyException来抛出,不同于BusinessException只有Code被返回,UserFriendlyException不会对msg和的detail做任何处理.  
如果需要在异常返回中带上data,可以使用withData或者直接使用设置Data属性:
```csharp
 throw new UserFriendlyException("10001", "对不起,该id未找到对应的实体")
            .WithData("BookId",123)
            .WithData("RequestId",10086);
```
返回Data:
```json
{
  "error": {
    "code": "对不起,该id未找到对应的实体",
    "message": "10001",
    "details": null,
    "data": {
      "BookId": 123,
      "RequestId": 10086
    },
    "validationErrors": null
  }
}
```
同时,ABP会按照以下规则,自动映射常见的异常类型的HTTP状态代码:

- 对于 AbpAuthorizationException:
  - 用户没有登录,返回 401 (未认证).
  - 用户已登录,但是当前访问未授权,返回 403 (未授权).
- 对于 AbpValidationException 返回 400 (错误的请求) .
- 对于 EntityNotFoundException返回 404 (未找到).
- 对于 IBusinessException 和 IUserFriendlyException (它是IBusinessException的扩展) - 返回403 (未授权) .
- 对于 NotImplementedException 返回 501 (未实现) .
- 对于其他异常 (基础架构中未定义的) 返回 500 (服务器内部错误) .

#### 领域服务
在前面我们定义了Book聚合根,而最开始我们已经知道聚合根可以找到聚合内的多个实体,而书作为聚合根,理应可以直接定位它的作者,所以我们这里来定义一个作者实体.

```csharp
public class Author:FullAuditedAggregateRoot<Guid>
{
    public string Name { get; private set; }
    public DateTime BirthDate { get; set; }
    public string ShortBio { get; set; }

    private Author()
    {
        /* This constructor is for deserialization / ORM purpose */
    }

    internal Author(
        Guid id,
        [NotNull] string name,
        DateTime birthDate,
        [CanBeNull] string shortBio = null)
        : base(id)
    {
        SetName(name);
        BirthDate = birthDate;
        ShortBio = shortBio;
    }

    internal Author ChangeName([NotNull] string name)
    {
        SetName(name);
        return this;
    }

    private void SetName([NotNull] string name)
    {
        Name = Check.NotNullOrWhiteSpace(
            name,
            nameof(name),
            maxLength: AuthorConsts.MaxNameLength
        );
    }
}
```
`FullAuditedAggregateRoot`由`AuditedAggregateRoot`继承而来,在此基础上添加了
```csharp
public virtual bool IsDeleted { get; set; }
public virtual Guid? DeleterId { get; set; }
public virtual DateTime? DeletionTime { get; set; }
```
这三个属性用于实现软删除,完整的FullAudited就相当于封装了这部分审计功能.
![](../imgs/领域驱动DDD/20220818134513.png)  
Author的`Name`和`SetName()`限制为私有,构造方法和`ChangeName()`限制为仅项目内访问,强制其只能使用领域服务设置名字.`Check()`是ABP所提供的检查类,会用来校验方法是否合法并设置值.
```csharp
public class AuthorManager : DomainService
{
    private readonly IRepository<Author,Guid> _authorRepository;

    public AuthorManager(IRepository<Author, Guid> authorRepository)
    {
        _authorRepository = authorRepository;
    }

    public async Task<Author> CreateAsync(
        [NotNull] string name,
        DateTime birthDate,
        [CanBeNull] string shortBio = null)
    {
        Check.NotNullOrWhiteSpace(name, nameof(name));

        var existingAuthor = await _authorRepository.SingleOrDefaultAsync(t=>t.Name == name);
        if (existingAuthor != null)
        {
            throw new BusinessException($"{name} already exits");
        }

        return new Author(
            GuidGenerator.Create(),
            name,
            birthDate,
            shortBio
        );
    }

    public async Task ChangeNameAsync(
        [NotNull] Author author,
        [NotNull] string newName)
    {
        Check.NotNull(author, nameof(author));
        Check.NotNullOrWhiteSpace(newName, nameof(newName));

        var existingAuthor = await _authorRepository.SingleOrDefaultAsync(t=>t.Name == newName);
        if (existingAuthor != null && existingAuthor.Id != author.Id)
        {
            throw new BusinessException($"{newName} already exits");
        }

        author.ChangeName(newName);
    }
}

```
然后仍然是在DbContext中添加`DbSet<Author>`并执行迁移和更新,这点和Book部分一样.

接着定义DTO和ApplicationService
DTO:
```csharp
public class GetAuthorListDto : PagedAndSortedResultRequestDto
{
    public string Filter { get; set; }
}

public class CreateAuthorDto
{
    [Required]
    [StringLength(35)]
    public string Name { get; set; }

    [Required]
    public DateTime BirthDate { get; set; }

    public string ShortBio { get; set; }
}

public class UpdateAuthorDto
{
    [Required]
    [StringLength(35)]
    public string Name { get; set; }

    [Required]
    public DateTime BirthDate { get; set; }

    public string ShortBio { get; set; }
}

public class AuthorDto : EntityDto<Guid>
{
    public string Name { get; set; }

    public DateTime BirthDate { get; set; }

    public string ShortBio { get; set; }
}
```
`其中PagedAndSortedResultRequestDto` 具有标准分页和排序属性: `int MaxResultCount`, `int SkipCount`和 `string Sorting`

```csharp
public interface IAuthorAppService : IApplicationService
{
    Task<AuthorDto> GetAsync(Guid id);

    Task<PagedResultDto<AuthorDto>> GetListAsync(GetAuthorListDto input);

    Task<AuthorDto> CreateAsync(CreateAuthorDto input);

    Task UpdateAsync(Guid id, UpdateAuthorDto input);

    Task DeleteAsync(Guid id);
}
```
> *实现类略*

Auhtor功能完成后,可能会觉得领域服务和应用服务有点重叠,二者在实际开发中也确实不好区分,但是大体上而言应用服务是和**用户/使用侧相关的交互服务逻辑**，关注的是应用场景,领域服务是和**业务/物体侧相关的内在服务逻辑**，关注的是核心逻辑,比如电梯服务，应用服务偏用户使用侧包括按键、刷卡、高低层、奇偶层分流等，领域服务偏电梯自身属性和方法包括电梯上行、下行、开门、关门等。

#### 聚合关系
Auhtor类定义完成后,我们在Book中添加对应的实体外键:`Guid AuthorId`,虽然DDD最佳实践中要求仅通过id引用其它聚合对象. 但是, 你可以添加这样的导航属性
```csharp
[ForeignKey(nameof(AuthorId))]
public Author Author { get; set; }
````
并为EF Core配置它. 这样, 你在获取图书和它们的作者时就不需要写join查询了, 这会使代码简洁且逻辑清晰很多.

大部分情况下EF Core都会自动识别依赖关系,但是如果定义了复杂导航属性,可能需要配置EF Core映射关系(这部分只是演示,这种简单映射其实不用处理):
```csharp
builder.Entity<Book>(b =>
{
    b.ToTable(BookStoreConsts.DbTablePrefix + "Books", BookStoreConsts.DbSchema);
    b.ConfigureByConvention(); //auto configure for the base class props
    b.Property(x => x.Name).IsRequired().HasMaxLength(128);

    // ADD THE MAPPING FOR THE RELATION
    b.HasOne<Author>().WithMany().HasForeignKey(x => x.AuthorId).IsRequired();
});
```
这样,在需要引用Book作者的时候,如果开启了EF Core预先加载,可以直接使用EBook.Author,或者使用`.Include(t=>t.Author)`来显式加载.同时,在ABP的默认仓储中也封装了子对象加载,在获取导航属性的时候相当方便:
```csharp
//包含子对象,单查询时默认开启
public async Task TestWithDetails(Guid id)
{
    var book = await _bookRepository.GetAsync(id);
}

//不含子对象
public async Task TestWithoutDetails(Guid id)
{
    var book = await _bookRepository.GetAsync(id, includeDetails: false);
}

//集合中包含子对象,默认关闭
public async Task TestWithDetails()
{
    var books = await _bookRepository.GetListAsync(includeDetails: true);
}
```
你也可以先查询不包含子对象的结果,再稍后获得其子对象:
```csharp
var order = await _orderRepository.GetAsync(id, includeDetails: false);
//order.Lines 此时是空的

await _orderRepository.EnsureCollectionLoadedAsync(order, x => x.Lines);
//order.Lines 被填充
```

#### 模块化
最上面说的`Module`配置类,其实就是ABP模块化的一个具体体现,定义模块由继承AbpModule实现:
```csharp
public class BlogModule : AbpModule
{
            
}
```
然后可以重写ConfigureServices方法,入参ServiceConfigurationContext中可以拿到IServiceCollection,由此可以实现将配置项注入ASP.NET Core管道中.
```csharp
public class BlogModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        //...
    }
}
```
同时,Modoule中定义了PreConfigureServices和PostConfigureServices方法用来在ConfigureServices之前或之后执行配置.
如果模块之间具有相互依赖的逻辑,可以使用`[DependsOn(typeof(AbpAspNetCoreMvcModule))]`进行引用,ABP在启动时会调查应用程序的依赖关系,并以正确的顺序初始化/关闭模块.
## 参考
- [领域驱动设计在互联网业务开发中的实践](https://tech.meituan.com/2017/12/22/ddd-in-practice.html)
- [ABP-Web应用程序开发教程](https://docs.abp.io/zh-Hans/abp/latest/Tutorials/Part-1)
- [ABP-快速入门](https://docs.abp.io/zh-Hans/abp/latest/Tutorials/Todo/Index?UI=BlazorServer&DB=EF)