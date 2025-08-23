# **mTLS — Mutual TLS**

## الترجمة الحرفية  
**Mutual Transport Layer Security (mTLS)** — **توكيد متبادل عبر طبقة النقل** (خادم **و**عميل يثبتان هويتهما بشهادات).

## الوصف العربي المختصر  
امتداد لـ **HTTPS** حيث لا يكتفي الخادم بشهادة، بل يطلب أيضًا **شهادة عميل**.  
النتيجة: قناة **مشفّرة** مع **توثيق ثنائي** (Server & Client) مناسبة للأنظمة الداخلية والـ Microservices والـ B2B.

## الشرح المبسّط  
- يبدأ اتصال TLS عادي.  
- الخادم يعرض **شهادته**، والعميل يتحقق منها.  
- الخادم يطلب **شهادة العميل**، والعميل يرسلها.  
- يتحقق الخادم من سلسلة ثقة العميل (CA/CRL/OCSP) ثم يُكمَل الاتصال.  
- بعدها تُمرَّر طلبات HTTP بأمان وهوية مؤكدة للطرفين.

## تشبيه  
بوابة شركة آمنة:  
تُظهر **بطاقة عمل** عند الدخول (شهادة الخادم)، ويُطلب منك **بادج موظف** أيضًا (شهادة العميل).  
لا دخول إلا إذا كانت **البطاقتان** موثوقتين.

## مثال كود C# مختصر

### خادم ASP.NET Core يطلب شهادة عميل
```csharp
// Program.cs (.NET 8/9)
// NuGet: Microsoft.AspNetCore.Authentication.Certificate
using Microsoft.AspNetCore.Authentication.Certificate;
using Microsoft.AspNetCore.Server.Kestrel.Https;

var builder = WebApplication.CreateBuilder(args);

// 1) Kestrel يطلب شهادة عميل أثناء TLS
builder.WebHost.ConfigureKestrel(k =>
{
    k.ConfigureHttpsDefaults(h =>
    {
        h.ClientCertificateMode = ClientCertificateMode.RequireCertificate; // mTLS
    });
});

// 2) تحقق واعتماد شهادة العميل
builder.Services.AddAuthentication(CertificateAuthenticationDefaults.AuthenticationScheme)
    .AddCertificate(options =>
    {
        options.AllowedCertificateTypes = CertificateTypes.All; // Dev: اضبطها كما يلزمك
        options.Events = new CertificateAuthenticationEvents
        {
            OnCertificateValidated = ctx =>
            {
                // TODO: تحقّق من المُصدِر/الموضوع/السياسات/القائمة السوداء …
                // مثال: ربط الشهادة بهوية
                ctx.Principal = new System.Security.Claims.ClaimsPrincipal(
                    new System.Security.Claims.ClaimsIdentity(
                        new[] { new System.Security.Claims.Claim("sub", ctx.ClientCertificate.Subject) },
                        CertificateAuthenticationDefaults.AuthenticationScheme));
                ctx.Success();
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/secure", () => Results.Ok(new { ok = true }))
   .RequireAuthorization(); // لن تُخدم دون شهادة عميل صالحة

app.Run();
```

### عميل .NET يرسل شهادة (PFX)
```csharp
using System.Net.Http;
using System.Security.Cryptography.X509Certificates;

var handler = new HttpClientHandler();

// أضِف شهادة العميل (تتضمن المفتاح الخاص) – امتداد PFX
handler.ClientCertificates.Add(
    new X509Certificate2("client.pfx", "P@ssw0rd", X509KeyStorageFlags.MachineKeySet));

// التحقق من شهادة الخادم يتم افتراضيًا (لا تعطلّه في الإنتاج)
using var http = new HttpClient(handler) { BaseAddress = new Uri("https://api.example.com") };

var res = await http.GetAsync("/secure");
Console.WriteLine($"{(int)res.StatusCode} {res.IsSuccessStatusCode}");
```

> تذكير مهم:  
> - **خادم**: EKU = *Server Authentication*، و**عميل**: EKU = *Client Authentication*.  
> - استخدم **SAN** بالأسماء/المضيفين الصحيحة.  
> - وفّر **سلسلة ثقة** كاملة وراجع **CRL/OCSP**.

## خطوات عملية لتطبيق mTLS (Checklist)
1. أنشئ/وفّر **CA** موثوقًا (خاص داخلي أو موفّر سحابي).  
2. أصدِر **شهادة خادم** لكل خدمة (SAN صحيحة) و**شهادة عميل** لكل مستهلك.  
3. نزّل **سلسلة الثقة** (Root/Intermediates) إلى الخوادم والعملاء بأمان.  
4. فعِّل الطلب الإلزامي لشهادة العميل في **Kestrel/NGINX/Ingress**.  
5. تحقق من الشهادة (المُصدِر، الصلاحية، الإبطال، السياسة، القيود).  
6. اربط الشهادة بهوية/حساب (Mapping) أو ادعم **AuthZ** إضافي (Scopes/Claims).  
7. خطّط لـ **دوران المفاتيح** وتجديد الشهادات تلقائيًا (ACME/Cert-Manager).  
8. راقب الانتهاءات والرفض والإبطال وسجّل البصمات (Thumbprints).

## أخطاء شائعة
- استخدام شهادات **ذاتية التوقيع** عشوائيًا أو تعطيل التحقق في العميل.  
- نسيان إضافة **الوسيطات (Intermediate CAs)** → فشل السلسلة.  
- غياب **CRL/OCSP** أو عدم التعامل مع حالات الإبطال.  
- عدم تقييد **EKU/SAN** أو الثقة بمرجع CA واسع جدًا.  
- الاعتماد على mTLS وحده دون **سياسات تفويض** على مستوى التطبيق.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [HTTP](http.md) | بروتوكول طلب/استجابة بدون تشفير | مناسب فقط داخل قنوات موثوقة أو محلية |
| [HTTPS](https.md) | تشفير + توثيق **الخادم** فقط (TLS) | حماية القناة وهوية الخادم؛ العميل لا يقدّم شهادة |
| **mTLS** | **تشفير + توثيق متبادل (خادم وعميل)** | **قوي للأنظمة الداخلية/B2B؛ يتطلب إدارة شهادات للعملاء** |

## ملخص الفكرة  
**mTLS = HTTPS مع شهادة عميل**.  
يعطيك قناة مشفّرة وهوية مؤكدة للطرفين—ممتاز لواجهات داخلية وتكاملات حسّاسة—بشرط إدارة شهادات محكمة وتجديدات تلقائية.
