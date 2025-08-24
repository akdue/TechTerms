# **Tier**

## الترجمة الحرفية  
**Tier** — **طبقة نشر** (حدّ فصل على مستوى **العمليّة/الخادم/الشبكة**).

## الوصف العربي المختصر  
تقسيم التطبيق إلى **مكوّنات موزّعة** تفصلها **حدود نشر**:  
**Web Tier** (واجهة/بوابة)، **App Tier** (منطق/خدمات)، **Data Tier** (قواعد بيانات).  
الهدف: **عزل أمني**، **قابلية تحجيم** مستقل، و**إدارة** أسهل للبنية.

## الشرح المبسّط  
- **Layer** ينظّم الكود داخل المشروع. **Tier** يوزّع التشغيل عبر **عمليات/خوادم** مختلفة.  
- الانتقال بين Tiers يتم عبر **شبكة** (HTTP/gRPC/رسائل)، غالبًا مع **جدر نارية** و**هوية**.  
- أشكال شائعة: **2-Tier (Web↔DB)**، **3-Tier (Web↔App↔DB)**، أو أكثر (Cache/Queue/Reporting…).

## تشبيه  
مطعم كبير:  
الاستقبال (Web) يستقبل الطلب، المطبخ (App) يطبخ، المخزن (Data) يوفّر المكوّنات.  
كل قسم مستقل لكنهم يتواصلون عبر **ممرّات** مضبوطة.

---

## مثال عملي — **3-Tier** مبسّط (Web BFF ↔ App API ↔ DB)

> فكرتنا: مشروعان C# منفصلان + قاعدة بيانات. في الإنتاج تُنشر على **خوادم/حاويات** مختلفة.  
> الكود مختصر للتوضيح.

### (1) Web Tier — بوابة أمامية (BFF) تستدعي App API عبر HTTP
```csharp
// WebBff/Program.cs  (.NET 8/9)
// dotnet new web -n WebBff
using System.Net.Http.Json;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHttpClient("app", c => c.BaseAddress = new Uri("http://app-api")); // DNS داخلي
var app = builder.Build();

app.MapGet("/products", async (IHttpClientFactory f) =>
{
    var http = f.CreateClient("app");
    var items = await http.GetFromJsonAsync<object[]>("/api/products");
    return Results.Ok(new { from = "web-bff", items });
});

app.Run();
```

### (2) App Tier — منطق أعمال + الوصول للبيانات
```csharp
// AppApi/Program.cs  (.NET 8/9)
// dotnet new web -n AppApi
using Microsoft.AspNetCore.Mvc;
using System.Data;
using Microsoft.Data.SqlClient;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IDbConnection>(_ =>
    new SqlConnection(Environment.GetEnvironmentVariable("DB_CONN")!)); // حقن اتصال

var app = builder.Build();

app.MapGet("/api/products", async ([FromServices] IDbConnection db) =>
{
    await db.OpenAsync();
    using var cmd = db.CreateCommand();
    cmd.CommandText = "SELECT TOP 10 Id, Name, Price FROM Products ORDER BY Id";
    using var r = await cmd.ExecuteReaderAsync();
    var list = new List<object>();
    while (await r.ReadAsync())
        list.Add(new { Id = r.GetInt32(0), Name = r.GetString(1), Price = r.GetDecimal(2) });
    return Results.Ok(list);
});

app.Run();
```

### (3) Data Tier — SQL Server (حاوية/خادم منفصل)

```yaml
# docker-compose.yml  (تشغيل محلي لثلاث طبقات)
version: "3.9"
services:
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Your_strong_password1!
    ports: [ "1433:1433" ]
  app-api:
    build: ./AppApi
    environment:
      - DB_CONN=Server=db,1433;Database=Shop;User Id=sa;Password=Your_strong_password1!;TrustServerCertificate=True
    depends_on: [ db ]
  web-bff:
    build: ./WebBff
    ports: [ "8080:8080" ]           # تصدير الويب للخارج
    depends_on: [ app-api ]
```

> **الفكرة:**  
> - **WebBff** لا يكلّم DB مباشرة؛ يمرّ عبر **AppApi**.  
> - يمكن تحجيم **web-bff** أفقيًا دون لمس **db**.  
> - سياسات شبكة وجدر نارية بين الطبقات (فقط AppApi يرى DB).

---

## خطوات عملية لتصميم **Tiers** سليمة
1. **حدود واضحة**: ما الذي يراه كل Tier؟ امنع المسارات الجانبية (Web→DB ✗).  
2. **بروتوكولات رسمية**: HTTP/gRPC مع **توثيق** (OpenAPI)، و**هوية** (mTLS/JWT).  
3. **أمن الشبكة**: SG/NSG/Firewall، تقسيم شبكي (VNet/Subnet)، أسرار عبر **Key Vault**.  
4. **تحجيم مستقل**: Autoscale لكل Tier حسب الحمل (Web كثيف I/O، App CPU، DB IOPS).  
5. **المراقبة**: Telemetry مفرّق حسب Tier (Logs/Traces/Metrics + Correlation-Id).  
6. **استرجاع الأعطال**: صحّة/استعداد (Health/Readiness)، Circuit Breakers، و(Backoff/Retry).  
7. **التوزيع**: CI/CD لكل Tier، نشر تدريجي (Blue/Green، Canary).

---

## أخطاء شائعة
- الخلط بين **Layer** و**Tier**: تقسيم ملفات فقط بلا فصل نشر حقيقي.  
- قنوات غير آمنة بين الطبقات (بدون TLS أو مصادقة خدمة-لخدمة).  
- اعتماد **Web** على **DB** مباشرة → يختنق الأمن والتحجيم.  
- مشاركة **نموذج قاعدة البيانات** مع الويب (تسريب تفاصيل).  
- نقطة فشل واحدة (DB بلا نسخة/تكرار، أو App بلا أكثر من نسخة).  
- تجاهل الـ **Timeouts** و**Bulkhead** في مكالمات بينية.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Tier** | **فصل نشر عبر خوادم/شبكات** | حدود أمان/شبكة؛ تحجيم وعزل مستقل |
| [Layer](layer.md) | تنظيم الكود داخل المشروع | لا يفرض فصل نشر بحد ذاته |
| 2-Tier | Web ↔ Data مباشرة | بسيط؛ قيود أمان/تحجيم |
| **3-Tier** | Web ↔ App ↔ Data | النمط المؤسسي الشائع |
| [Microservices](microservices.md) | تفتيت App Tier لخدمات مستقلّة | مرونة أعلى + تعقيد شبكة/مراقبة |

---

## ملخص الفكرة  
**Tier** يعني أين **يعمل** كل جزء من نظامك وكيف **يتواصل** عبر الشبكة.  
ارسم حدودًا صلبة، أمّن القنوات، واسمح بتحجيم مستقل ومراقبة لكل طبقة—  
تحصل على معماريّة **أكثر أمانًا**، **مرونة** في التحجيم، وصيانة **أسهل** على المدى الطويل.
