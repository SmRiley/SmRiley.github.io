---
title: ASP.NET Core中的认证与授权
date: 2022-08-17 16:08:41
tags:
    - asp.net core
    - 认证
    - 鉴权
---
无论做什么应用,除非是完全公开的静态官网,总是会接触到认证与授权这两个概念.也许在小项目中这二者经常被混淆甚至误用,但是在现代化应用中,二者职责已相当分明,前者判断你是否合法,比如你登录爱奇艺时,认证中间件只判断账号密码是否正确,而当你点进一部新剧时,授权中间件则会对你账号的进行鉴权,比如是否有会员,是否有试看卷,通俗点来说,认证是鉴别用户是谁,而授权则是判断用户能做什么.

在ASP.NET core中认证是Authentication,授权是Authorization.也就是在项目入口中经常添加的`app.UseAuthentication()`和`app.UseAuthorization();`  

不同于以前在PHP中随便写写Cookie login,现在对安全的提高和权限的细分,已经使传统的基于Cookie和Session的认证方式无法完成部分环境要求(比如前后端分离后,部分接口认证转向JWT,或者Oauth2的授权码模式),传统ASP.NET MVC应用中,通常使用IdentityServier或者直接在Controller校验后HttpClient.Sigin(),现在正在方式仍然可以用,而在ASP.NET Core中对这块权限跨分更未精细,能做的更广也更为精确.  
## Cliam(标识)和ClaimsIdentity(证件)
首先,ASP.NET Core中定义的最小身份信息是`Cliam`,原意是声明或者主张,不必在意翻译是否准确,可以将它理解为证件上的某一项标识,比如身份证上的姓名:张三、性别:男或者身份证号:101XXXXXXXXXXXX等等,这些都可以看成一个个键值对,Key是类型(CliamType/string),Value是具体值(string),并且这种键值对是可以重用的,比如你身份证和你驾照上的身份证上就有相同的标识,我们就不必去重复定义标识,不同标识可以组合成不同的证件,而这个"证件"在ASP.NET Core中是`ClaimsIdentity`,如果把前者理解为键值对,后者可以理解为`SortedList`(可重复键值对).有时候一个证件不足以证明一个人的身份,一个人也可以拥有多张证件.而这个证件持有人就是`ClaimsPrincipal `.  

想象一下,写字楼大门保安只看你的工牌是否属于写字楼里面某个楼层的公司,那他就没有必要在乎你上面的名字,而等你进入公司后,公司又需要判断你的级别,不至于把普通员工安排到总经理办公室.等你下班后,你小区的保安又需要你小区的门禁卡,即便你们公司的工牌上和门禁卡上的姓名标识是一样的,保安也不会看你的工牌就放你进去,但是如果你恰巧没带门禁卡,却带着房产证的话,看房产证的信息判断你是小区业主,依然可以放行.  

这几个场景下,要求的验证信息和方式各不相同不同,写字楼保安只在乎你的工牌上是否标识了某个公司,不管你是男是女;而公司需要你的工牌属于公司且需要查看工牌的身份信息;小区保安则要求你出示的是小区的门禁卡,至于你进入小区后怎么刷卡坐电梯那就是另一回事.应用到上面的`Cliam`和`ClaimsIdentity`,写字楼保安需要你的`Cliam`的Type是公司,Key是写字楼的某一项公司,并不关心你的ClaimsIdentity.而公司则需要你的`ClaimsIdentity`是XXX公司的工牌,并会查看你的`ClaimsIdentity`中`Cliam Type`为部门的值.而小区保安则判断你的`ClaimsPrincipal`是否为业主,而不在乎你拿什么证件来证明.

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