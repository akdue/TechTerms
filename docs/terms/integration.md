# **Integration**

## الترجمة الحرفية  
**System Integration** — **تكامل الأنظمة** (ربط خدمات وتطبيقات لتبادل البيانات/الأوامر).

## الوصف العربي المختصر  
طريقة **منظَّمة** لربط نظامين أو أكثر عبر **واجهات**: REST/gRPC/رسائل/ملفات.  
تشمل **المخطط (Schema)**، **المصادقة**، **التحويل (Mapping)**، **الوثوقية**، و**المراقبة**.

## الشرح المبسّط  
- **سِكّة** بين الأنظمة: إمّا **متزامن** (Request/Response) أو **غير متزامن** (Events/Queue).  
- نتفق على **عقد** (Endpoints/Topics/Schema) و**أمان** (OAuth2, mTLS, HMAC).  
- نضيف **Idempotency**، **Retries**، و**Versioning** لتفادي التكرار والكسر.  
- نراقب: **Logs/Metrics/Tracing** لاكتشاف الأعطال مبكرًا.

## تشبيه  
شركة شحن بين متجرين: **طلب** فوري = اتصال متزامن. **تنبيهات** الشحن = أحداث تُدفَع لاحقًا (Webhooks/Queue).

---

## مثال كود C# مختصر — استهلاك REST + استقبال Webhook (تحقّق توقيع)

```csharp
// Program.cs (.NET 8/9)
// - استدعاء REST خارجي مع مهلة ومحاولات وIdempotency-Key.
// - نقطة Webhook مع تحقق HMAC للتوقيع وتصفية التكرار.

using System.Net.Http.Headers;
using System.Security.Cryptography;
using System.Text;
using Microsoft.Extensions.Caching.Memory;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddMemoryCache();
var app = builder.Build();

var http = new HttpClient { BaseAddress = new Uri("https://api.example.com"), Timeout = TimeSpan.FromSeconds(5) };

// 1) استدعاء واجهة خارجية (Pull)
app.MapPost("/pay", async (IMemoryCache cache) =>
{
    var body = """{"amount":120,"currency":"USD"}""";
    string idem = Guid.NewGuid().ToString("N");            // مفتاح Idempotency
    var req = new HttpRequestMessage(HttpMethod.Post, "/v1/payments");
    req.Content = new StringContent(body, Encoding.UTF8, "application/json");
    req.Headers.Add("Idempotency-Key", idem);
    req.Headers.Authorization = new AuthenticationHeaderValue("Bearer", Environment.GetEnvironmentVariable("API_TOKEN"));

    for (int attempt = 1; attempt <= 3; attempt++)
    {
        using var res = await http.SendAsync(req);
        if (res.IsSuccessStatusCode) return Results.Text(await res.Content.ReadAsStringAsync(), "application/json");
        if ((int)res.StatusCode is 429 or >= 500) await Task.Delay(TimeSpan.FromMilliseconds(200 * Math.Pow(2, attempt)));
        else return Results.StatusCode((int)res.StatusCode);
    }
    return Results.StatusCode(504); // Gateway Timeout
});

// 2) استقبال Webhook (Push) مع HMAC-SHA256
app.MapPost("/webhooks/provider", async (HttpRequest request, IMemoryCache cache) =>
{
    // اقرأ الجسم كاملًا
    using var reader = new StreamReader(request.Body, Encoding.UTF8);
    var payload = await reader.ReadToEndAsync();

    // تحقق التوقيع
    var sig = request.Headers["X-Signature"].ToString();    // مثال: sha256=HEX
    var secret = Environment.GetEnvironmentVariable("WEBHOOK_SECRET") ?? "dev-secret";
    using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
    var hash = Convert.ToHexString(hmac.ComputeHash(Encoding.UTF8.GetBytes(payload))).ToLowerInvariant();
    var expected = "sha256=" + hash;

    if (!CryptographicOperations.FixedTimeEquals(Encoding.UTF8.GetBytes(sig), Encoding.UTF8.GetBytes(expected)))
        return Results.Unauthorized();

    // مانع تكرار بسيط (Idempotency)
    var eventId = request.Headers["X-Event-Id"].ToString();
    if (string.IsNullOrEmpty(eventId) || !cache.Set(eventId, true, TimeSpan.FromMinutes(10)))
        return Results.Ok(); // مكرّر — تجاهل بهدوء

    // TODO: عالج الحدث (مثلاً تحديث حالة الدفع)
    Console.WriteLine($"Webhook OK: {eventId}");
    return Results.Ok();
});

app.Run();
```

**الفكرة:**  
- **Outbound**: مهلة + محاولات تدريجية + **Idempotency-Key**.  
- **Inbound**: **تحقّق توقيع** HMAC + **منع تكرار** + استجابة سريعة 2xx.

---

## خطوات عملية لتكامل ناجح
- حدّد **النطاق/السيناريوهات** و**مصدر الحقيقة** للبيانات.  
- اختر **النمط**:  
  - متزامن (REST/gRPC) للقراءة الفورية.  
  - غير متزامن (Events/Webhooks/Queue) للتدفق والمرونة.  
- صمّم **Schema** ثابتًا وقابلًا للتطوير: حقول اختيارية، **Version**، و**Compatibility**.  
- أمان: **OAuth2/OIDC** أو **mTLS**؛ لـ Webhooks استخدم **توقيع/HMAC** وسجّل IP/الوقت.  
- الموثوقية: **Retries + Backoff**، **Idempotency**، **DLQ** للرسائل الفاشلة.  
- المراقبة: **Tracing** عبر Correlation-Id، لوحات **SLIs/SLOs**، تنبيهات.  
- اختبر: **Contract/Integration Tests** وبيئة **Sandbox** و**Data Replay**.

---

## أخطاء شائعة
- عدم **التمييز** بين الأخطاء المؤقّتة (قابلة لإعادة المحاولة) والدائمة.  
- إهمال **Idempotency** في المدفوعات/الطلبات → عمليات مكرّرة.  
- قبول Webhook **بدون توقيع** أو وقت صلاحية.  
- كسر التوافق بإزالة/تغيير حقل بدون **Versioning**.  
- تجاهل **حدود المعدّل (Rate Limits)** والمهلات.  
- دمج منطق الأعمال في طبقة النقل بدل **خدمات** منفصلة قابلة للاختبار.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [API](api.md) | عقد للوصول إلى وظائف/بيانات | REST/gRPC/GraphQL؛ يُستهلك مباشرة |
| **Integration** | **ربط أنظمة لتبادل البيانات/الأوامر** | **Sync/Async، Mapping، Idempotency، مراقبة** |
| [Webhook](webhook.md) | دفع أحداث من مزوّد إلى مستهلك | تحقّق توقيع + إعادة محاولة |
| [ETL/ELT](etl-elt.md) | تكامل بيانات دفعي/تحويلي | أحجام كبيرة؛ زمن أعلى |
| [iPaaS](ipaas.md) | منصّة تكامل مُدارة | موصلات جاهزة، تدفّقات رسومية |

---

## ملخص الفكرة  
**Integration** = ربط أنظمة بعقد واضح وأمان وموثوقية.  
اجعل النقل **آمنًا**، التنفيذ **Idempotent**، الفشل **قابلًا للتعافي**، وتتبّع كل طلب—تحصل على تكامل مستدام وسهل الصيانة.
