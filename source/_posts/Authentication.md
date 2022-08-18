---
title: ASP.NET Core中的认证与授权
date: 2022-08-17 16:08:41
tags:
    - asp.net core
    - 认证
    - 鉴权
---
## 引言
无论做什么应用,除非是完全公开的静态官网,总是会接触到认证与授权这两个概念.也许在小项目中这二者经常被混淆甚至误用,但是在现代化应用中,二者职责已相当分明,前者判断你是否合法,比如你登录爱奇艺时,认证中间件只判断账号密码是否正确,而当你点进一部新剧时,授权中间件则会对你账号的进行鉴权,比如是否有会员,是否有试看卷,通俗点来说,认证是鉴别用户是谁,而授权则是判断用户能做什么.

在ASP.NET core中认证是Authentication,授权是Authorization.也就是在项目入口中经常添加的
```csharp
app.UseAuthentication()
app.UseAuthorization();
```

不同于以前在PHP中随便写写Cookie login,现在对安全的提高和权限的细分,已经使传统的基于Cookie和Session的认证方式无法完成部分环境要求(比如前后端分离后,部分接口认证转向JWT,或者Oauth2的授权码模式),传统ASP.NET MVC应用中,通常使用IdentityServier或者直接在Controller校验后HttpClient.Sigin(),现在这种方式仍然予以保留.在ASP.NET Core中由于框架被细分(例如Blazor、WebAPI、grpc),对这块权限跨分极为精密,能做的更广也更为精确.  
## Cliam(标识)和ClaimsIdentity(证件)
首先,ASP.NET Core中定义的最小身份信息是`Cliam`,原意是声明或者主张,不必在意翻译是否准确,可以将它理解为证件上的某一项标识,比如身份证上的姓名:张三、性别:男或者身份证号:101XXXXXXXXXXXX等等,这些都可以看成一个个键值对,Key是类型(CliamType/string),Value是具体值(string),并且这种键值对是可以重用的,比如你身份证和你驾照上的身份证上就有相同的标识,我们就不必去重复定义标识,不同标识可以组合成不同的证件,而这个"证件"在ASP.NET Core中是`ClaimsIdentity`,如果把前者理解为键值对,后者可以理解为`SortedList`(可重复键值对).有时候一个证件不足以证明一个人的身份,一个人也可以拥有多张证件.而这个证件持有人就是`ClaimsPrincipal `.  

想象一下,写字楼大门保安只看你的工牌是否属于写字楼里面某个楼层的公司,那他就没有必要在乎你上面的名字,而等你进入公司后,公司又需要判断你的级别,不至于把普通员工安排到总经理办公室.等你下班后,你小区的保安又需要你小区的门禁卡,即便你们公司的工牌上和门禁卡上的姓名标识是一样的,保安也不会看你的工牌就放你进去,但是如果你恰巧没带门禁卡,却带着房产证的话,看房产证的信息判断你是小区业主,依然可以放行.  

这几个场景下,要求的验证信息和方式各不相同不同,写字楼保安只在乎你的工牌上是否标识了某个公司,不管你是男是女;而公司需要你的工牌属于公司且需要查看工牌的身份信息;小区保安则要求你出示的是小区的门禁卡,至于你进入小区后怎么刷卡坐电梯那就是另一回事.应用到上面的`Cliam`和`ClaimsIdentity`,写字楼保安需要你的`Cliam`的Type是公司,Key是写字楼的某一项公司,并不关心你的ClaimsIdentity.而公司则需要你的`ClaimsIdentity`是XXX公司的工牌,并会查看你的`ClaimsIdentity`中`Cliam Type`为部门的值.而小区保安则判断你的`ClaimsPrincipal`是否为业主,而不在乎你拿什么证件来证明.

## 身份认证
想象一下,如果需要自定义一个从数据库中验证并且有状态的Token认证,这和使用jwt或者其它token方案都不相同,那么需要如何操作呢?
在ASP.NET Core中,无论是认证还是授权,都是方案对应一个处理程序.其中,认证时的认证方案由`Authorize`特性的`AuthenticationSchemes`指定,所以,我们先需要添加一个认证方案:
```csharp
builder.Services.AddAuthentication(opt =>
{
    opt.AddScheme<TokenAuthentication>("CustomToken", "Token");
});
```
认证方案添加了,在定义认证处理程序,认证方案需要实现IAuthenticationHandler:
```csharp
public class TokenAuthentication : IAuthenticationHandler
{
    private HttpContext _context = default!;
    private AppDbContext _dbContext = default!;
    private AuthenticationScheme _scheme = default!;
    public async Task InitializeAsync(AuthenticationScheme scheme, HttpContext context)
    {
        _context = context;
        _scheme = scheme;
        await Task.FromResult(_dbContext = context.RequestServices.GetRequiredService<AppDbContext>());
    }

    public async Task<AuthenticateResult> AuthenticateAsync()
    {
        bool haveToken = _context.Request.Headers.TryGetValue("Authorization", out var tokenHeader);
        string token = haveToken ? tokenHeader.ToString()["Bearer ".Length..] : string.Empty;
        if (haveToken && await _dbContext.Users.SingleOrDefaultAsync(t => t.Token == token) is User user)//验证token是否正确
        {
            //这里的ClaimsIdentity构参scheme一定要和注入服务的scheme一样,除非此认证处理程序是默认的情况
            ClaimsIdentity claimsIdentity = new ClaimsIdentity(nameof(TokenAuthentication));
            claimsIdentity.AddClaims(new Claim[]
            {
                new (ClaimTypes.Name, user.Name!),
                new (ClaimTypes.Sid, user.Id.ToString()!)
            });
            //鉴权成功，写入用户信息
            return AuthenticateResult.Success(new AuthenticationTicket(new ClaimsPrincipal(claimsIdentity), _scheme.Name));
    }
        return AuthenticateResult.NoResult();
    }

public async Task ChallengeAsync(AuthenticationProperties? properties)
{
    await Task.FromResult(_context.Response.StatusCode = 401);
}

public async Task ForbidAsync(AuthenticationProperties? properties)
{
    await Task.FromResult(_context.Response.StatusCode = 403);
}
}

```
完事在控制器或者Action中指定Schemes即可:
```csharp
[Authorize(AuthenticationSchemes = "CustomToken")]
```
## 角色授权
在传统应用中RBAC(基于角色的访问控制)大行其道,在现如今的后台管理中依然适用,比如上面励志中,只需要定义公司员工和小区业主身份,并对应添加允许通行、坐电梯、停地下停车场等权限,就可以根据不同身份控制权限,并且同一用户可以同时拥有多个角色.具有一定的灵活性.

在ASP.NET Core中仍旧支持此种授权方式,添加认证和鉴权中间件后,我们可以在控制器或者具体方法上使用`[Authorize]`特性的Role来控制其角色要求.
```csharp
//二者都可访问
[Authorize(Roles = "Administrator, PowerUser")]
public class ControlAllPanelController : Controller
{
    public IActionResult SetTime() =>
        Content("Administrator || PowerUser");

    //仅支持Administrator访问
    [Authorize(Roles = "Administrator")]
    public IActionResult ShutDown() =>
        Content("Administrator only");
}
```
登录的时候在ClaimsPrincipal中写入角色claim:
```csharp
//这里的ClaimsIdentity构参scheme一定要和注入服务的scheme一样,无论是默认还是自定义认证服务
var identity = new ClaimsIdentity(new ClaimsIdentity(CookieAuthenticationDefaults.AuthenticationScheme));
//自定义的claim信息
identity.AddClaim(new Claim(ClaimTypes.Role, "Administrator"));
identity.AddClaim(new Claim(ClaimTypes.Role, "PowerUser"));
AuthenticationProperties properties = new AuthenticationProperties()
{
    //设置cookie票证的过期时间
    ExpiresUtc = DateTime.Now.AddDays(7),
    RedirectUri = model.ReturnUrl
};
await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, new ClaimsPrincipal(identity), properties);
```
> 由于ClaimsIdentity内的键值对是可重复的,所以可以直接Add多个同Type的Claim(`ClaimTypes.Role`).

RBAC大行其道,但是在某些情况下它缺乏一定的灵活性,比如现在新开了个网吧,需要18岁以上的成年人才可进入,如果在这里加一个可进入酒吧的权限,那后面再多一个只准12岁以下进入的游乐场或者只准60岁以下进入的鬼屋,难不成都要逐个加么?

基于此种应用需求,微软引入了声明和策略授权.

## 声明授权
回到上面所述的,小区保安只关心你有没有小区业主的身份,也不关心具体在几栋在几楼,我们可以做如下定义:
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages();
builder.Services.AddControllersWithViews();

builder.Services.AddAuthorization(options =>
{
   options.AddPolicy("Owner", policy => policy.RequireClaim("Residential"));
});

var app = builder.Build();

app.UseStaticFiles();

app.UseAuthentication();

app.UseAuthorization();

app.MapDefaultControllerRoute();

app.MapRazorPages();

app.Run();
```
在控制器或者Action上方标注授权要求:
```csharp
[Authorize(Policy = "EnsureSafety")]
public IActionResult EnterCommunity()
{
    return View();
}
```
当然,也是支持多重策略应用的,比如进入小区保安只管你业主身份,但是单元楼楼下有个大妈防着小偷,一定要熟面孔才让进:
```csharp
//业主策略
[Authorize(Policy = "Owner")]
public class CommunityController : Controller
{
    public IActionResult EnterCommunity()
    {
        return View();
    }

    //熟人策略
    [Authorize(Policy = "Acquaintance")]
    public IActionResult EnterUnitBuilding()
    {
        return View();
    }
}
```
## 策略授权
上述声明授权中,虽然说的是声明授权,但是定义却是用的AddPolicy,这不是添加策略的意思么?
其实,策略可以看作声明和角色的并集,所以使用策略授权定义声明也就没什么好奇怪的了.
> 基于角色的授权和基于声明的授权，只是一种语法上的便捷，最终都会生成授权策略

接着是上面所说的网吧最低年龄这种限制,我们可以添加一个自定义授权策略:
```csharp
//定义授权要求
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public MinimumAgeRequirement(int minimumAge) =>
        MinimumAge = minimumAge;

    public int MinimumAge { get; }
}
```
针对授权要求使用处理程序
```csharp
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst(c => c.Type == ClaimTypes.DateOfBirth);

        if (dateOfBirthClaim is null)
        {
            return Task.CompletedTask;
        }

        var dateOfBirth = Convert.ToDateTime(dateOfBirthClaim.Value);
        int calculatedAge = DateTime.Today.Year - dateOfBirth.Year;
        if (dateOfBirth > DateTime.Today.AddYears(-calculatedAge))
        {
            calculatedAge--;
        }

        if (calculatedAge >= requirement.MinimumAge)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```
- 处理程序通过调用 context.Succeed(IAuthorizationRequirement requirement) 并传递已成功验证的要求来指示成功。
- 处理程序通常不需要处理失败，因为针对相同要求的其他处理程序可能会成功。
- 为了保证失败，即使其他要求处理程序成功，也需调用 context.Fail。

添加自定义授权要求,并注入对应处理程序
```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AtLeast18", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
});

builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```
一个授权策略可以添加多个要求,可以按照需求添加多个要求甚至直接组合.

## 参考
- [ASP.NET Core 中的授权简介](https://docs.microsoft.com/zh-cn/aspnet/core/security/authorization/)
- [ASP.NET Core 认证与授权](https://www.cnblogs.com/rainingnight/p/introduce-basic-authentication-in-asp-net-core.html)