# **N-Tier**

## الترجمة الحرفية  
**N-Tier Architecture** — **بنية نشر متعددة الطبقات** (أكثر من 3 طبقات شبكية منفصلة).

## الوصف العربي المختصر  
تقسيم النظام إلى **عدة طبقات نشر** مستقلة (Web/BFF، API Gateway، خدمات أعمال، تكاملات، بيانات، Cache، Queue…).  
الهدف: **عزل أمني**، **تحجيم مستقل**، و**ملكية واضحة** لكل طبقة، مع تواصل عبر بروتوكول شبكي (HTTP/gRPC/رسائل).

## الشرح المبسّط  
- هي تعميم لـ **3-Tier**: بدل (Web↔App↔DB) فقط، نضيف طبقات مثل **API Gateway**، **BFF**، **Cache**، **Message Broker**، **Analytics**.  
- كل طبقة تعمل في **عملية/خادم/شبكة** مختلفة، وتتواصل عبر واجهات رسمية موثّقة.  
- يسمح بتطبيق سياسات **أمان**، **مراقبة**، و**تحجيم** لكل طبقة على حدة.

## تشبيه  
مطار كبير متعدد المراحل: بوابات دخول (Gateway)، نقاط تفتيش (Security)، صالات ترانزيت (BFF)،  
خدمات أرضية (خدمات الأعمال)، مخازن (DB/Cache)، وخطوط شحن (Queue). كل مرحلة منفصلة ومسؤولة عن جزء محدّد.

---

## مثال مختصر (C#) — N-Tier بسيط: **Gateway (YARP) → Orders API → Redis Cache → DB**

> الفكرة:  
> 1) **API Gateway** يوجّه الطلبات لطبقات داخلية.  
> 2) **Orders API** يخدم الموارد ويقرأ من **Redis** (طبقة Cache) قبل **DB** (طبقة بيانات).

### (1) API Gateway (YARP)
```csharp
// Gateway/Program.cs  (.NET 8/9)
// dotnet add package Yarp.ReverseProxy
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy().LoadFromMemory(
    new() { // Routes
        new("orders-route",      "/orders/{**catch-all}", "orders-cluster"),
        new("catalog-route",     "/catalog/{**catch-all}", "catalog-cluster")
    },
    new() { // Clusters (خدمات داخل الشبكة الخاصة)
        new("orders-cluster",  new[] { "http://orders-api" }),
        new("catalog-cluster", new[] { "http://catalog-api" })
    });
var app = builder.Build();
app.MapReverseProxy();
app.Run();
```

### (2) Orders API (طبقة خدمات الأعمال + Cache طبقي)
```csharp
// OrdersApi/Program.cs  (.NET 8/9)
// dotnet add package StackExchange.Redis
// dotnet add package Dapper  (اختياري)
using StackExchange.Redis;
using Microsoft.AspNetCore.Mvc;
using System.Text.Json;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IConnectionMultiplexer>(
    _ => ConnectionMultiplexer.Connect(Environment.GetEnvironmentVariable("REDIS") ?? "redis:6379"));

builder.Services.AddScoped<IDbConnection>(_ =>
    new Microsoft.Data.SqlClient.SqlConnection(Environment.GetEnvironmentVariable("DB_CONN")!));

var app = builder.Build();

app.MapGet("/orders/{id:int}", async (int id, IConnectionMultiplexer mux, IDbConnection db) =>
{
    var cache = mux.GetDatabase();
    var key = $"order:{id}";
    var cached = await cache.StringGetAsync(key);
    if (cached.HasValue) return Results.Ok(JsonSerializer.Deserialize<object>(cached!));

    // تبسيط: نموذج قراءة (استبدل بـ Dapper/EF حسب الحاجة)
    await db.OpenAsync();
    using var cmd = db.CreateCommand();
    cmd.CommandText = "SELECT Id, Customer, Total FROM Orders WHERE Id=@id";
    var p = cmd.CreateParameter(); p.ParameterName="@id"; p.Value=id; cmd.Parameters.Add(p);
    using var r = await cmd.ExecuteReaderAsync();
    if (!await r.ReadAsync()) return Results.NotFound();

    var dto = new { Id = r.GetInt32(0), Customer = r.GetString(1), Total = r.GetDecimal(2) };

    await cache.StringSetAsync(key, JsonSerializer.Serialize(dto), TimeSpan.FromMinutes(5));
    return Results.Ok(dto);
});

app.Run();
```

> في التشغيل الحقيقي تُنشر هذه الخدمات في **شبكات فرعية** مختلفة مع سياسات جدار ناري،  
> ويوفَّر **DNS داخلي** للأسماء (`orders-api`, `catalog-api`, `redis`, `db`).

---

## خطوات عملية لتصميم **N-Tier** سليم
1. **حدود واضحة**: عرّف مسؤولية كل طبقة (Gateway، BFF، Services، Data، Cache، Queue…).  
2. **بروتوكولات وعقود**: وثّق واجهات الطبقات (OpenAPI/gRPC/Events) مع **نسخ** وإدارة تغيّر.  
3. **الأمان الشبكي**: تقسيم شبكي (VNet/Subnets)، جدر نارية، mTLS/JWT بين الطبقات، إدارة أسرار (Vault).  
4. **التحجيم والاستيعاب**: اضبط Autoscale لكل طبقة حسب خصائصها (Web I/O، Services CPU، DB IOPS).  
5. **المراقبة والتتبّع**: Logs/Metrics/Tracing عبر الطبقات مع **Correlation-Id** من البوابة.  
6. **المرونة**: Circuit Breaker، Retry/Backoff، Timeout لكل استدعاء بيني.  
7. **التوزيع**: CI/CD لكل طبقة، نشر تدريجي (Blue/Green/Canary)، واختبارات عقد (Contract Tests).  
8. **البيانات**: اختَر أنماط الوصول (CQRS/Cache-Aside)، وقيود الحوكمة والنسخ الاحتياطي للطبقة البيانات.

---

## أخطاء شائعة
- التفتيت المفرط (طبقات كثيرة بلا قيمة) → **تشابك شبكي** و**زمن استجابة** أعلى.  
- قنوات غير آمنة بين الطبقات (بدون TLS أو هوية).  
- **تشابك عقود**: تغييرات طبقة تكسر أخرى لغياب النسخ/التوافق.  
- تجاهل **المراقبة الطرفية-إلى-الطرفية** → صعوبة تتبّع الأعطال عبر الطبقات.  
- بوابة بلا حدود معدل/حماية (DoS/Abuse).  
- استدعاءات متسلسلة بلا تجميع/توازي → N+1 عبر الشبكة.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **N-Tier** | **فصل نشر عبر أكثر من ثلاث طبقات** | Gateway/BFF/Services/Data/Cache/Queue… بتحجيم وأمن مستقل |
| 3-Tier | Web ↔ App ↔ Data | أبسط تنظيمًا؛ نقطة انطلاق شائعة |
| [Tier](tier.md) | فصل فيزيائي/شبكي عام | قد يكون 2/3/N حسب الحاجة |
| [Layer](layer.md) | تنظيم الكود داخليًا | لا يفرض فصل نشر |
| [Microservices](microservices.md) | تفتيت App-Tier لخدمات مستقلّة | قد تُنشر ضمن N-Tier تحت Gateway |

---

## ملخص الفكرة  
**N-Tier** يوزّع نظامك على **عدّة طبقات نشر** بحدود أمان وعقود واضحة.  
ضع Gateway/BFF أمامي، خدمات أعمال وسطية، وطبقات بيانات/Cache/Queue خلفية،  
وأمّن القنوات وراقب التدفّق—تحصل على بنية **مرنة**، **قابلة للتوسّع**، وسهلة الحوكمة. 
