# **Microservices**

## الترجمة الحرفية  
**Microservices Architecture** — **عمارة الخدمات المصغّرة**.

## الوصف العربي المختصر  
نظام مكوَّن من **خدمات صغيرة مستقلة**.  
كل خدمة تملك **حدّ مجال واضح** و**بياناتها الخاصة**، وتُنشَر وتتحجّم **بشكل مستقل**.  
التواصل عبر **HTTP/gRPC/رسائل** مع **مراقبة** و**مرونة** ضد الأعطال.

## الشرح المبسّط  
- كل خدمة = **وظيفة عمل محدّدة** (Orders, Inventory, Billing…).  
- تملك **قاعدة بيانات منفصلة** (لا مشاركة جداول بين الخدمات).  
- التواصل **غير متزامن** عندما يمكن (رسائل/أحداث)، ومتزامن عند الحاجة (HTTP/gRPC).  
- تحتاج **ملاحَظية** (Logs/Traces/Metrics)، **صمود** (Retries/Timeouts/Circuit Breaker)، و**إصدار عقود**.

## تشبيه  
شركة كبيرة: أقسام مستقلّة (مبيعات/مخزون/شحن).  
كل قسم له **سجلّاته** و**قوانينه**، ويتعاونون عبر **نماذج/مستندات** رسمية (عقود API/Events).

---

## مثال كود C# — خدمتان مستقلّتان (Orders ↔ Inventory) مع مرونة أساسية

```csharp
// ===== Service A: Inventory =====
// dotnet new web -n InventorySvc
// يوفر تحقق مخزون بسيط: GET /api/stock/{sku}?qty=5  → { ok: true/false }
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// "قاعدة بيانات" في الذاكرة (لكل خدمة مخزنها الخاص)
var stock = new Dictionary<string,int>(StringComparer.OrdinalIgnoreCase) {
    ["KB-01"] = 10, ["MS-77"] = 2
};

app.MapGet("/api/stock/{sku}", ([FromRoute] string sku, [FromQuery] int qty) =>
{
    if (qty <= 0) return Results.BadRequest(new { error = "qty_must_be_positive" });
    var ok = stock.TryGetValue(sku, out var have) && have >= qty;
    return Results.Ok(new { ok });
});

app.MapGet("/healthz", () => Results.Ok(new { status = "healthy" }));
app.Run();
```

```csharp
// ===== Service B: Orders =====
// dotnet new web -n OrdersSvc
// ينادي Inventory عبر HttpClient مع سياسات زمنية (Timeout/Retry).
using System.Net.Http.Json;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// عنوان Inventory من البيئة/التهيئة (خدمتان منفصلتان)
var inventoryBase = Environment.GetEnvironmentVariable("INVENTORY_URL") ?? "http://localhost:5080";

// .NET 8: معالج مرونة جاهز (Timeout/Retry/Jitter) عبر Polly
builder.Services.AddHttpClient("inventory", c => c.BaseAddress = new Uri(inventoryBase))
                .AddStandardResilienceHandler();

var app = builder.Build();

record OrderItem(string Sku, int Qty, decimal Price);
record PlaceOrder(int OrderId, string CustomerId, List<OrderItem> Items);

app.MapPost("/api/orders", async ([FromBody] PlaceOrder dto, IHttpClientFactory f, CancellationToken ct) =>
{
    if (dto.Items is null || dto.Items.Count == 0) 
        return Results.ValidationProblem(new() { ["items"] = ["at_least_one_item"] });

    // تحقق مخزون متزامن (مثال مبسّط). يفضَّل أحداث/حجز لا متزامن في الإنتاج.
    var http = f.CreateClient("inventory");
    foreach (var it in dto.Items)
    {
        var resp = await http.GetAsync($"/api/stock/{it.Sku}?qty={it.Qty}", ct);
        if (!resp.IsSuccessStatusCode) return Results.Problem("inventory_unavailable", statusCode: 503);

        var payload = await resp.Content.ReadFromJsonAsync<dynamic>(cancellationToken: ct);
        if (payload?.ok != true) return Results.BadRequest(new { error = "out_of_stock", it.Sku });
    }

    var total = dto.Items.Sum(i => i.Price * i.Qty);
    // حفظ الطلب في قاعدة Orders (متروك لك؛ قاعدة بيانات منفصلة عن Inventory)
    return Results.Created($"/api/orders/{dto.OrderId}", new { dto.OrderId, total, status = "confirmed" });
});

app.MapGet("/healthz", () => Results.Ok(new { status = "healthy" }));
app.Run();
```

> الفكرة: خدمتان منفصلتان، نشرٌ مستقل، تواصل عبر HTTP مع **مرونة** أساسية.  
> في الأنظمة الكبيرة: فضّل **الأحداث (Event-Driven)** لحجز/تحديث المخزون وتجنّب الاقتران.

---

## خطوات عملية لبناء **Microservices** بذكاء
1. **حدود المجال** أولًا (Domain-Driven): خدمة لكل قدرة أعمال واضحة.  
2. **بيانات لكل خدمة**: قاعدة خاصة بكل خدمة. لا مشاركة جداول. التبادل عبر **عقود** فقط.  
3. **التواصل**:  
   - متزامن قليل وقصير (HTTP/gRPC) مع **Timeout/Retry/Circuit Breaker**.  
   - غير متزامن للعمليات الموزّعة (Kafka/RabbitMQ + **Sagas/Outbox**).  
4. **العقود والإصدار**: OpenAPI/Protobuf/Events + **Backward Compatibility**.  
5. **الملاحَظية**: Logs مرتبطة بـ **Correlation-Id**، Tracing (OpenTelemetry)، Metrics.  
6. **الأمن**: mTLS أو Token (JWT/OAuth2)، **Least Privilege**، أسرار في Vault.  
7. **النشر**: حاويات + CI/CD لكل خدمة، Blue/Green/Canary، **Autoscaling** مستقل.  
8. **المرونة**: Idempotency، Retries مع Backoff، Circuit Breaker، Bulkhead، Timeouts.  
9. **البيانات الموزّعة**: تجنّب المعاملات عبر الخدمات؛ استخدم **Saga/Compensation** واتساقًا لاحقًا.

---

## أخطاء شائعة
- تقسيم مبكّر بدون حدود مجال واضحة → شبكات معقّدة بلا قيمة.  
- مشاركة قاعدة بيانات بين خدمات → تزاوج قوي وكسر الاستقلالية.  
- اتصال متزامن كسلسلة طويلة (N+1 عبر الشبكة) بدون **Timeout/Retry/Circuit**.  
- غياب **Observability** → صعوبة تشخيص الأعطال.  
- تجاهل **Backward Compatibility** للعقود → كسور عند النشر.  
- استخدام الميكروسيرفس كـ “طبقات HTTP” بدل **قدرات أعمال**.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Microservices** | **قدرات أعمال مستقلّة + نشر مستقل** | بيانات لكل خدمة، تواصل بعقود، صمود ومراقبة |
| Monolith | تطبيق واحد بكل الطبقات | أبسط نشر/تصحيح؛ تحجيم أقل مرونة |
| **Modular Monolith** | وحدة واحدة لكن بحدود نمطية صارمة | خطوة وسيطة ممتازة قبل التقسيم |
| SOA | خدمات أكبر + ESB | تاريخيًّا أثقل (حافلة مؤسسية) |
| Serverless Functions | دوال صغيرة مُدارة | رائعة للوظائف الحدثية؛ إدارة حالة أصعب |

---

## ملخص الفكرة  
**Microservices** = تقسيم النظام لقدرات صغيرة **مستقلة** ببياناتها وعقودها.  
أحسن حدود المجال، افصل البيانات، وأضِف **مرونة وملاحَظية** قوية—  
تحصل على نظام **قابل للتوسّع**، **متحمّل للأعطال**، وسهل التطوير المستقل للفِرَق. 
