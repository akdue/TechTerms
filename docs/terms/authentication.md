# **Authentication**

## الترجمة الحرفية  
**Authentication** — **إثبات الهوية** (من أنت؟).

## الوصف العربي المختصر  
آلية تتحقق أن المستخدم/العميل **هو فعلاً** من يدّعي.  
قد تكون عبر **كلمة مرور، رمز لمرة واحدة (OTP/MFA)، شهادة، مفتاح FIDO2، أو مزوّد هوية (IdP)**.

## الشرح المبسّط  
- الهدف: **توليد هوية موثوقة** (Claims/Principal) بعد التحقق.  
- النتائج تُخزَّن كـ **جلسة (Cookie)** لتطبيقات الويب، أو تُحمل كـ **Token (JWT/OAuth2)** لواجهات API.  
- تُكمِّلها **Authorization** (التفويض): *ماذا* يُسمح لك بعد المصادقة.

## تشبيه  
بوابة مبنى: تُظهر بطاقة هويتك للحارس (**Authentication**).  
ثم يحدد لك الطوابق المسموح دخولها (**Authorization**).

---

## مثال كود C# — Minimal API مع **JWT Bearer** (مصادقة) + سياسة أدوار

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n AuthDemo && cd AuthDemo
// dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
// dotnet add package Microsoft.IdentityModel.Tokens
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

// 1) إعداد سرّ التوقيع ومعاملات الـ JWT
string issuer   = "authdemo";
string audience = "authdemo.api";
string secret   = Environment.GetEnvironmentVariable("JWT_SECRET") ?? "dev-secret-please-change-to-32bytes+";
var key         = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret));

// 2) تسجيل Authentication + Authorization
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o =>
    {
        o.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true, ValidIssuer = issuer,
            ValidateAudience = true, ValidAudience = audience,
            ValidateIssuerSigningKey = true, IssuerSigningKey = key,
            ValidateLifetime = true, ClockSkew = TimeSpan.FromSeconds(30)
        };
    });

builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("admin", p => p.RequireRole("admin"));
});

var app = builder.Build();

app.UseHttpsRedirection();           // مهم للأمان
app.UseAuthentication();             // يجب قبل UseAuthorization
app.UseAuthorization();

// مخزن مستخدمين مبسّط (للتجربة فقط)
var users = new Dictionary<string,(string Pwd,string Role)>(StringComparer.OrdinalIgnoreCase)
{
    ["user@example.com"]  = ("N3wP@ss","user"),
    ["admin@example.com"] = ("Adm1nP@ss","admin")
};

// 3) إصدار توكن بعد التحقق (بديلًا: استخدم مزوّد هوية/Identity)
app.MapPost("/login", (string email, string password) =>
{
    if (!users.TryGetValue(email, out var u) || u.Pwd != password)
        return Results.Unauthorized();

    var claims = new[]
    {
        new Claim(JwtRegisteredClaimNames.Sub, email),
        new Claim(ClaimTypes.Name, email),
        new Claim(ClaimTypes.Role, u.Role) // role = user/admin
    };
    var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
    var jwt = new JwtSecurityToken(
        issuer: issuer, audience: audience,
        claims: claims, expires: DateTime.UtcNow.AddMinutes(30),
        signingCredentials: creds);

    string token = new JwtSecurityTokenHandler().WriteToken(jwt);
    return Results.Ok(new { access_token = token, token_type = "Bearer", expires_in = 1800 });
})
.WithSummary("Authenticate user and return JWT");

// 4) نهاية تتطلب مصادقة
app.MapGet("/me", (ClaimsPrincipal user) => new {
    name = user.Identity?.Name,
    roles = user.Claims.Where(c => c.Type == ClaimTypes.Role).Select(c => c.Value)
}).RequireAuthorization();

// 5) نهاية لأدمن فقط (Authorization فوق Authentication)
app.MapGet("/admin/metrics", () => new { ok = true, at = DateTime.UtcNow })
   .RequireAuthorization("admin");

app.Run();
```

**جرّب سريعًا:**
```bash
# دخول
TOKEN=$(curl -s -X POST "https://localhost:5001/login?email=admin@example.com&password=Adm1nP@ss" -k | jq -r .access_token)

# وصول محمي
curl -H "Authorization: Bearer $TOKEN" https://localhost:5001/me -k
curl -H "Authorization: Bearer $TOKEN" https://localhost:5001/admin/metrics -k
```

> **ملاحظات أمنية سريعة:**  
> استخدم **HTTPS**، سرّ ≥ 32 بايت، مهلات قصيرة، و**Refresh Tokens** عند الحاجة.  
> لا تحفظ كلمات المرور نصيًا—استعمل **هاش بطيء** (PBKDF2/BCrypt/Argon2) و**MFA**.

---

## خطوات عملية لتطبيق مصادقة سليمة
1. **اختر النمط**:  
   - **Cookie** لتطبيقات الويب (جلسات على المتصفح).  
   - **JWT Bearer** أو **OAuth2/OIDC** لواجهات API والهواتف.  
2. **مصادر الهوية**:  
   - **Identity Provider** (Azure AD/Keycloak/Auth0) أو **ASP.NET Core Identity** محلي.  
3. **تعزيز الأمان**: MFA، قفل محاولات، سياسات كلمة مرور، **CSRF** (مع Cookies)، **CORS** للـ APIs.  
4. **إدارة الأعمار**: **Access Token** قصير و**Refresh Token** لتجديد آمن.  
5. **Claims/Scopes** واضحة، ومطابقة مع **Authorization** (سياسات/أدوار).  
6. **مراقبة**: سجّل محاولات الدخول، مصدر IP، إنذارات على السلوك الشاذ.

---

## أخطاء شائعة
- خلط **Authentication** مع **Authorization** أو الاعتماد على أحدهما فقط.  
- سرّ JWT ضعيف/قصير، أو **توقيع غير مُتحقّق** على الخادم.  
- تعطيل **ValidateLifetime** أو مهلات طويلة جدًا.  
- إرسال **Cookies** حساسة بدون `Secure`/`HttpOnly`/`SameSite`.  
- عدم حماية نقاط تجديد التوكن أو عدم تدوير **Refresh Tokens**.  
- نسيان **CSRF** في POST/PUT عند استخدام Cookies.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Authentication** | **إثبات الهوية** | ينتج **Claims/Principal** (من أنت) |
| [Authorization](authorization.md) | تحديد الصلاحيات بعد إثبات الهوية | **أدوار/سياسات/Scopes** (ماذا يُسمح لك) |
| Identity Provider (IdP) | مصدر هوية مركزي | يدعم **OIDC/OAuth2**، تسجيل دخول موحّد |
| OAuth2 / OIDC | بروتوكولات تفويض/هوية | Tokens، تدفّقات (Auth Code + PKCE) |
| JWT vs Cookie | API عديم الحالة vs جلسة متصفح | JWT: مناسب للـ APIs؛ Cookie: للويب التقليدي |

---

## ملخص الفكرة  
**Authentication** يبني **هوية موثوقة**، غالبًا عبر **JWT** أو **Cookies**.  
أمّن السلسلة: **سر قوي، مهلات، MFA، تحقق صارم**، ثم اربطها بـ **Authorization** (سياسات/أدوار)  
لتحصل على وصول آمن ودقيق يلبّي متطلبات منتجك.
