# **Authorization**

## الترجمة الحرفية  
**Authorization** — **تفويض الصلاحيات** (ماذا يُسمح لك أن تفعل بعد إثبات الهوية).

## الوصف العربي المختصر  
قواعد تحدّد **الوصول** إلى الموارد/العمليات بعد نجاح **المصادقة**.  
قد تكون **أدوارًا** (RBAC)، **مطالبات/خصائص** (Claims/ABAC)، أو **سياسات** مخصّصة (Policies).

## الشرح المبسّط  
- بعد تسجيل الدخول يتكوّن **Principal/Claims**.  
- **Authorization** تقارن هذه المطالبات بسياساتك: *Admin فقط*، *يملك المورد*، *لديه نطاق/تصريح محدد*.  
- يمكن تطبيقها **تصريحيًا** (`RequireAuthorization`) أو **برمجيًا** عبر `IAuthorizationService`.

## تشبيه  
أظهرت هويتك للبوّاب (**Authentication**).  
الآن يقرّر أي **أبواب** يمكنك فتحها (**Authorization**) وفق بطاقتك وقواعد المبنى.

---

## مثال C# — سياسات أدوار/مطالبات + تفويض قائم على المورد (ASP.NET Core)

```csharp
// Program.cs  (.NET 8/9)
// فكرة مبسّطة: مصادقة "رأسية" للشرح (Headers) + سياسات تفويض.
// جرّب بإرسال X-User وX-Roles وX-Perms في الرؤوس.

using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);

// ===== (1) مصادقة رؤوس مبسّطة لبناء Claims من الطلب =====
builder.Services.AddAuthentication("HeaderAuth")
    .AddScheme<AuthenticationSchemeOptions, HeaderAuthHandler>("HeaderAuth", null);

// ===== (2) تفويض: سياسات Roles/Claims + متطلب امتلاك المورد =====
builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("AdminOnly",   p => p.RequireRole("admin"));
    o.AddPolicy("OrdersRead",  p => p.RequireClaim("perm", "orders.read"));
    o.AddPolicy("OwnOrderOrAdmin", p => p.Requirements.Add(new OwnsOrderRequirement()));
});
builder.Services.AddSingleton<IAuthorizationHandler, OwnsOrderHandler>();

// مستودع طلبات بسيط مع مالك لكل طلب
builder.Services.AddSingleton<IOrdersRepo, InMemoryOrdersRepo>();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// (A) نقطة تتطلب دور "admin"
app.MapGet("/admin/metrics", () => new { ok = true, at = DateTime.UtcNow })
   .RequireAuthorization("AdminOnly");

// (B) نقطة تتطلب تصريح قراءة الطلبات
app.MapGet("/orders", (IOrdersRepo repo) => repo.All())
   .RequireAuthorization("OrdersRead");

// (C) تفويض قائم على المورد: المالك أو الأدمن فقط
app.MapGet("/orders/{id:int}", async (int id, ClaimsPrincipal user, IAuthorizationService auth, IOrdersRepo repo) =>
{
    var authz = await auth.AuthorizeAsync(user, id, "OwnOrderOrAdmin");
    if (!authz.Succeeded) return Results.Forbid();
    var order = repo.Find(id);
    return order is null ? Results.NotFound() : Results.Ok(order);
});

app.Run();


// ====== المصادقة بالرؤوس (للتجربة فقط) ======
public class HeaderAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public HeaderAuthHandler(IOptionsMonitor<AuthenticationSchemeOptions> o, ILoggerFactory l, UrlEncoder e, ISystemClock c)
        : base(o, l, e, c) {}

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var user = Request.Headers["X-User"].ToString();          // مثال: alice@example.com
        if (string.IsNullOrWhiteSpace(user))
            return Task.FromResult(AuthenticateResult.NoResult());

        var roles = (Request.Headers["X-Roles"].ToString() ?? "")
                        .Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);
        var perms = (Request.Headers["X-Perms"].ToString() ?? "")
                        .Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);

        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user),
            new(ClaimTypes.Name, user)
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));
        claims.AddRange(perms.Select(p => new Claim("perm", p)));

        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}

// ====== المتطلب/المعالج: OwnsOrderRequirement ======
public record OwnsOrderRequirement : IAuthorizationRequirement;

public class OwnsOrderHandler : AuthorizationHandler<OwnsOrderRequirement, int>
{
    private readonly IOrdersRepo _repo;
    public OwnsOrderHandler(IOrdersRepo repo) => _repo = repo;

    protected override Task HandleRequirementAsync(AuthorizationHandlerContext ctx, OwnsOrderRequirement req, int orderId)
    {
        var isAdmin = ctx.User.IsInRole("admin");
        var sub = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier);
        var owner = _repo.OwnerOf(orderId);

        if (isAdmin || (!string.IsNullOrEmpty(sub) && string.Equals(owner, sub, StringComparison.OrdinalIgnoreCase)))
            ctx.Succeed(req);

        return Task.CompletedTask;
    }
}

// ====== نموذج/مستودع أوامر ======
public record Order(int Id, string Owner, decimal Amount);

public interface IOrdersRepo
{
    IEnumerable<Order> All();
    Order? Find(int id);
    string? OwnerOf(int id);
}

public class InMemoryOrdersRepo : IOrdersRepo
{
    private readonly List<Order> _db = new() {
        new(1, "alice@example.com", 120),
        new(2, "bob@example.com",   75)
    };
    public IEnumerable<Order> All() => _db;
    public Order? Find(int id) => _db.FirstOrDefault(o => o.Id == id);
    public string? OwnerOf(int id) => Find(id)?.Owner;
}
```

**تجربة سريعة (curl):**
```bash
# مستخدم عادي يملك الطلب 1
curl -s http://localhost:5000/orders/1 \
  -H "X-User: alice@example.com" -H "X-Perms: orders.read"

# مستخدم عادي يحاول قراءة طلب غير مملوك
curl -i http://localhost:5000/orders/2 \
  -H "X-User: alice@example.com" -H "X-Perms: orders.read"   # 403

# أدمن يتجاوز الملكية
curl -s http://localhost:5000/orders/2 \
  -H "X-User: admin@example.com" -H "X-Roles: admin"

# الوصول لقائمة الطلبات يتطلب تصريح orders.read
curl -i http://localhost:5000/orders \
  -H "X-User: bob@example.com"                    # 403
curl -s http://localhost:5000/orders \
  -H "X-User: bob@example.com" -H "X-Perms: orders.read"
```

> الفكرة:  
> - **Policy Roles** لأدمن.  
> - **Policy Claims** لتصريح معيّن.  
> - **Resource-Based**: مالك المورد أو أدمن.

---

## خطوات عملية لتفويض سليم
1. عرّف **مصادر الحقيقة** للصلاحيات: أدوار، مطالبات، نطاقات (Scopes).  
2. استخدم **Policies** بأسماء معبّرة: `"AdminOnly"`, `"OrdersRead"`, `"OwnOrderOrAdmin"`.  
3. للفحص الدقيق على الموارد، استخدم **Authorization Handlers** ومرّر **المورد** (`AuthorizeAsync(user, resource, policy)`).  
4. ضع قواعدك في **طبقة واضحة** (Policies/Handlers) وليس مبعثرة في المعالجات.  
5. سجّل وراقب **محاولات الرفض/القبول**، واربطها بـ **Correlation-Id**.  
6. راعِ **Least Privilege** و**Separation of Duties**، وجدّد الصلاحيات دوريًا.

---

## أخطاء شائعة
- الخلط بين **Authentication** و**Authorization** أو الاكتفاء بأحدهما.  
- اعتماد **أدوار عامة جدًا** بدل **سياسات دقيقة** أو مطالبات/نطاقات.  
- كتابة منطق التفويض داخل كل Endpoint (تكرار/أخطاء) بدل **Policies/Handlers**.  
- تجاهل **تفويض الموارد** (الملكية/النطاق) والاعتماد على دور فقط.  
- عدم تحديث الـ Claims بعد تغيّر صلاحيات المستخدم (جلسات طويلة بلا تحديث).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Authorization** | **تحديد ما يُسمح بعد المصادقة** | Roles/Claims/Policies، يمكن أن تكون Resource-Based |
| [Authentication](authentication.md) | إثبات الهوية | ينتج Principal + Claims |
| Role-Based (RBAC) | منح صلاحيات بحسب الدور | سهل؛ خشن الحبيبات أحيانًا |
| Claim/Attribute-Based (ABAC) | قواعد اعتمادًا على خصائص المستخدم/المورد | أدقّ وأقوى من الأدوار فقط |
| Policy-Based | تركيب قواعد RBAC/ABAC باسم قابل لإعادة الاستخدام | تطبّق تصريحيًا أو برمجيًا |
| Resource-Based | قرار لكل مورد (ملكية/نطاق) | عبر `IAuthorizationService` وتمرير المورد |

---

## ملخص الفكرة  
**Authorization** = تحويل هويتك إلى **حقوق وصول** مضبوطة.  
ابنِ **سياسات واضحة**، وامزج **الأدوار والمطالبات**، واستخدم **Handlers** للموارد—  
فتحصل على تفويض **آمن** و**قابل للصيانة** يواكب تغيّر متطلبات منتجك. 
