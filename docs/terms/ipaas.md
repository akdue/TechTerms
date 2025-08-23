# **iPaaS**

## الترجمة الحرفية  
**Integration Platform as a Service (iPaaS)** — **منصّة تكامل كخدمة**.

## الوصف العربي المختصر  
خدمة سحابية لتوصيل الأنظمة والخدمات عبر **موصلات جاهزة** (Connectors)،  
وتدفّقات رسومية/قابلة للبرمجة: **Triggers**، **Mapping**، **تحويل**، **تنبيهات**، **مراقبة**.

## الشرح المبسّط  
- تبني تكاملات **بدون/بقليل كود**: سحب بيانات، تحويلها، وإرسالها لنظام آخر.  
- تدعم أنماط **Sync** (REST/gRPC) و**Async** (رسائل/أحداث/Webhooks).  
- توفر **Connectors** لعشرات SaaS/قواعد بيانات وبروتوكولات.  
- إدارة مركزية: أسرار، جداول فشل (DLQ)، إعادة محاولة، سجلات وتتبع.

## تشبيه  
مثل **مطار تحويل** للشحن: يستقبل شحنات مختلفة، يعيد **تغليفها**،  
ثم يرسلها لوجهات متعدّدة وفق **قواعد** وتوقيت.

---

## مثال كود C# مختصر — تشغيل تدفّق iPaaS (Trigger) + استقبال ردّ (Callback)

> الفكرة:  
> 1) **Outbound**: نستدعي **Webhook Trigger** للـ iPaaS مع **Bearer** و**Idempotency-Key**.  
> 2) **Inbound**: نستقبل **Callback** موقّع بـ HMAC ونتأكد من التوقيع.

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n IpaasDemo && cd IpaasDemo && dotnet run
using System.Net;
using System.Net.Http.Headers;
using System.Security.Cryptography;
using System.Text;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// ===== 1) استدعاء تدفّق iPaaS (Trigger) =====
app.MapPost("/trigger-order", async () =>
{
    using var http = new HttpClient { Timeout = TimeSpan.FromSeconds(8) };

    var url   = Environment.GetEnvironmentVariable("IPAAS_TRIGGER_URL") ?? "https://ipaas.example/flows/order/trigger";
    var token = Environment.GetEnvironmentVariable("IPAAS_TOKEN") ?? "demo-token";
    var idem  = Guid.NewGuid().ToString("N");

    var payload = """{"orderId":"A-1001","amount":120.50,"currency":"USD"}""";
    using var req = new HttpRequestMessage(HttpMethod.Post, url);
    req.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
    req.Headers.Add("Idempotency-Key", idem);
    req.Content = new StringContent(payload, Encoding.UTF8, "application/json");

    for (int attempt = 1; attempt <= 3; attempt++)
    {
        using var res = await http.SendAsync(req);
        if (res.IsSuccessStatusCode)
            return Results.Text(await res.Content.ReadAsStringAsync(), "application/json");

        if (res.StatusCode is HttpStatusCode.TooManyRequests or >= HttpStatusCode.InternalServerError)
            await Task.Delay(TimeSpan.FromMilliseconds(200 * Math.Pow(2, attempt)));
        else
            return Results.StatusCode((int)res.StatusCode);
    }
    return Results.StatusCode(504);
});

// ===== 2) استقبال ردّ iPaaS (Callback/Webhook) مع توقيع HMAC =====
app.MapPost("/callbacks/ipaas", async (HttpRequest req) =>
{
    string sig = req.Headers["X-Signature"];         // مثال: sha256=HEX
    string ts  = req.Headers["X-Timestamp"];         // Unix seconds

    using var reader = new StreamReader(req.Body, Encoding.UTF8);
    var body = await reader.ReadToEndAsync();

    if (!long.TryParse(ts, out var t) ||
        DateTimeOffset.UtcNow - DateTimeOffset.FromUnixTimeSeconds(t) > TimeSpan.FromMinutes(5))
        return Results.Unauthorized();

    var secret = Environment.GetEnvironmentVariable("IPAAS_WEBHOOK_SECRET") ?? "dev-secret";
    using var h = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
    var expect = "sha256=" + Convert.ToHexString(h.ComputeHash(Encoding.UTF8.GetBytes($"{ts}.{body}"))).ToLowerInvariant();

    if (!CryptographicOperations.FixedTimeEquals(Encoding.UTF8.GetBytes(sig), Encoding.UTF8.GetBytes(expect)))
        return Results.Unauthorized();

    Console.WriteLine($"Callback OK: {body}");
    // TODO: ضع المعالجة في طابور ثم أعد 200 سريعًا
    return Results.Ok();
});

app.Run();
```

**الإخراج المتوقع (مختصر):**  
- `POST /trigger-order` → يرجع نتيجة تشغيل التدفّق (RunId/Status).  
- `POST /callbacks/ipaas` → `200 OK` بعد التحقق من التوقيع.

---

## خطوات عملية لاعتماد iPaaS
1. **نطاق التكاِمل**: مصادر/وجهات، حجْم البيانات، تأخير مقبول (SLA).  
2. **اختيار الموصلات**: SaaS/DB/بروتوكولات مطلوبة. تأكد من **القيود** (حصة/حدود معدّل).  
3. **التصميم**: Triggers، قواعد **Routing**، **Mapping/Transforms**، وإثراء البيانات (Enrichment).  
4. **الأمان**: أسرار في Vault، **OAuth2/mTLS/HMAC** للاتصالات، وإخفاء بيانات حساسة.  
5. **الموثوقية**: **Retries + Backoff**، **Idempotency**, **DLQ**، ومعالجة أخطاء مركزية.  
6. **المراقبة**: سجلات، قياسات، **Tracing/Correlation-Id**، تنبيهات وإعادة تشغيل آلي.  
7. **الإصدار**: بيئات `dev/staging/prod`، **Versioning** للتدفّقات والـ Schemas، وعمليات نشر آمنة.  
8. **التكلفة/الإقامة**: راقب الاستهلاك/عدد التشغيلات، و**موقع البيانات** (Residency).

---

## أخطاء شائعة
- استخدام iPaaS لكل شيء بدل **حالاته المناسبة** → تكلفة/تعقيد زائد.  
- تجاهل **Idempotency** عند المدفوعات/الطلبات.  
- ترك أسرار في خطوات التدفّق بدل **Secret Store**.  
- عدم توثيق **Schema** ونسخه → تكاملات هشة عند التغيير.  
- الاعتماد على Polling عندما يتوفر **Webhook** أو **Events**.  
- إهمال **DLQ/Alerts** → ضياع أحداث بصمت.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [Integration](integration.md) | ربط أنظمة بعقود وأمن | قد تبنيه يدويًا أو عبر iPaaS |
| **iPaaS** | **تكامل مُدار بموصلات وتدفّقات جاهزة** | **قليل/بدون كود، مراقبة/أتمتة مدمجة** |
| [ETL/ELT](etl-elt.md) | تحريك/تحويل بيانات دفعية | تركيز بيانات أكثر من التكامل التشغيلي |
| API Gateway| توحيد ونشر واجهات API | تحكّم بالوصول/المعدل؛ ليس بديلاً عن التدفقات |
| ESB | ناقل مؤسسي داخل الداتا سنتر | تاريخيًا on-prem؛ iPaaS = معادل سحابي مرن |

---

## ملخص الفكرة  
**iPaaS** يسرّع بناء التكاملات عبر **موصلات جاهزة** وتدفّقات قابلة للضبط،  
مع أمان ومراقبة وإعادة محاولة خارج الصندوق.  
استخدمه عندما تريد **سرعة وموثوقية** بدون بناء منصّة تكامل من الصفر—واحفظ عقودك ونسخك وجودة بياناتك تحت السيطرة.
