# **Webhook**

## الترجمة الحرفية  
**Webhook** — **استدعاء عكسي عبر الويب** (مزود الخدمة يدفع حدثًا إلى عنوانك).

## الوصف العربي المختصر  
آلية **Push**: خدمة خارجيّة ترسل **طلب HTTP** إلى **نقطة نهاية** لديك عند وقوع حدث  
(مثال: *دُفع الطلب، اكتمل الشحن*). أنت تتحقق من **التوقيع** وتتعامل مع الحدث ثم تردّ **2xx** سريعًا.

## الشرح المبسّط  
- تعرِّف عنوانًا عامًا `POST /webhooks/provider`.  
- المزوّد يرسل JSON مع **رؤوس**: معرّف الحدث، توقيع HMAC، طابع زمني.  
- أنت تتحقق من **التوقيع/الوقت**، تمنع **التكرار**، وتضع العمل في **طابور** ثم تعيد **200 OK**.  
- عند فشل الاستلام قد يُعاد الإرسال تلقائيًا (Retries).

## تشبيه  
مثل **رسالة نصية** من شركة التوصيل: هم يبادرون بإخبارك بوصول الطلب،  
بدل أن تسأل أنت كل دقيقة (*Polling*).

---

## مثال كود C# — Minimal API: استقبال Webhook + تحقق توقيع + Idempotency

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n WebhookDemo && cd WebhookDemo && dotnet run
using System.Security.Cryptography;
using System.Text;
using Microsoft.Extensions.Caching.Memory;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddMemoryCache();
var app = builder.Build();

app.UseHttpsRedirection();

// نقطة الاستقبال (غيّر المسار/الاسم كما يطلب مزوّدك)
app.MapPost("/webhooks/provider", async (HttpRequest req, IMemoryCache cache) =>
{
    // 1) اقرأ الرؤوس الشائعة (أسماء الرؤوس تختلف بين المزودين)
    string eventId = req.Headers["X-Event-Id"].ToString();        // معرّف فريد للحدث
    string sig     = req.Headers["X-Signature"].ToString();        // مثال: "sha256=..."
    string ts      = req.Headers["X-Timestamp"].ToString();        // Unix seconds

    // 2) اقرأ الجسم كنص (كما حُسب توقيعه عند المزود)
    using var reader = new StreamReader(req.Body, Encoding.UTF8);
    string payload = await reader.ReadToEndAsync();

    // 3) تحقّق مبدئي: طابع زمني حديث لمنع إعادة التشغيل (Replay)
    if (!long.TryParse(ts, out var unix) ||
        DateTimeOffset.UtcNow - DateTimeOffset.FromUnixTimeSeconds(unix) > TimeSpan.FromMinutes(5))
        return Results.Unauthorized();

    // 4) تحقّق التوقيع HMAC-SHA256 مع سرّ مشترك
    string secret = Environment.GetEnvironmentVariable("WEBHOOK_SECRET") ?? "dev-secret";
    using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
    // التالي بروتوكول شائع: hash( $"{ts}.{payload}" )
    var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes($"{ts}.{payload}"));
    string expected = "sha256=" + Convert.ToHexString(hash).ToLowerInvariant();

    // مقارنة ثابتة الوقت (تجنّب هجمات التوقيت)
    if (!CryptographicOperations.FixedTimeEquals(
            Encoding.UTF8.GetBytes(sig), Encoding.UTF8.GetBytes(expected)))
        return Results.Unauthorized();

    // 5) Idempotency: امنع معالجة الحدث نفسه أكثر من مرة
    if (!string.IsNullOrWhiteSpace(eventId))
    {
        if (cache.TryGetValue(eventId, out _))
            return Results.Ok();                      // تكرار — تجاهله بهدوء
        cache.Set(eventId, true, TimeSpan.FromMinutes(30));
    }

    // 6) TODO: ضع المعالجة الثقيلة في طابور/خلفية ثم ردّ بسرعة
    Console.WriteLine($"Webhook OK: {eventId} | {payload}");

    return Results.Ok(); // 200 خلال ثوانٍ قليلة
});

app.Run();
```

**اختبار سريع بـ curl (توليد توقيع يدويًا مثال):**
```bash
BODY='{"ping":1}'
TS=$(date +%s)
SECRET='dev-secret'
SIG="sha256=$(printf "%s.%s" "$TS" "$BODY" | openssl dgst -sha256 -hmac "$SECRET" -binary | xxd -p -c 256)"
curl -X POST https://localhost:5001/webhooks/provider \
  -H "Content-Type: application/json" \
  -H "X-Event-Id: ev_123" \
  -H "X-Timestamp: $TS" \
  -H "X-Signature: $SIG" \
  -d "$BODY" -k
```

> **ملاحظات أمنية سريعة:** استخدم **HTTPS** دائمًا، سرّ قوي بطول كافٍ، واحتفِظ بمفتاح السرّ في **Secret Manager**.

---

## خطوات عملية لتصميم Webhook متين
- اختر **مسارًا مميّزًا** وصعب التخمين، وأبقِه خلف **HTTPS** فقط.  
- تحقّق من **التوقيع** و**الطابع الزمني** (نافذة 5–10 دقائق).  
- فعّل **Idempotency** باستخدام `X-Event-Id` أو Digest للجسم.  
- أعد **2xx** سريعًا ووُجِّه العمل إلى **خلفية/طابور** (Channel, Queue).  
- سجّل **Correlation-Id**، وراقب معدلات النجاح/الفشل.  
- اضبط **Rate Limit** واستقبل **Retries** دون آثار جانبية.  
- دوّن **مخطط الرسالة (Schema)** و**نسخ الإصدار** (Versioning) للحفاظ على التوافق.

---

## أخطاء شائعة
- قبول Webhook **بدون توقيع** أو بدون فحص **الوقت** → إعادة تشغيل/تلاعب.  
- معالجة العمل الثقيل داخل الطلب → **مهلات** وإعادة إرسال متكرّر.  
- تجاهل **Idempotency** → عمليات مزدوجة (خاصة المدفوعات).  
- ردّ بأكواد غير دقيقة (مثل 500 على تكرار معروف) بدل 200/204.  
- الاعتماد على **Whitelist IP** فقط؛ توقيع مشفّر أو **mTLS** أكثر قوة.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Webhook** | **Push HTTP عند حدوث حدث** | **تحقّق توقيع + طابع زمني + Idempotency؛ ردّ سريع 2xx** |
| Polling | سحب دوري من العميل | أبسط، لكن استهلاك زائد وتأخير أعلى |
| [WebSocket](websocket.md) | اتصال ثنائي حيّ | مناسب للتفاعلات الفورية؛ يتطلب جلسة طويلة |
| Server-Sent Events | بث أحادي من الخادم للعميل | خفيف للتيارات؛ ليس ثنائي الاتجاه |

---

## ملخص الفكرة  
**Webhook** = إشعار فوري من مزوّدك إلى خدمتك عبر **HTTP POST**.  
أمِّن التبادلات بـ **توقيع/زمن**، اجعل المعالجة **Idempotent**، وردّ **سريعًا**—تحصل على تكامل موثوق وقابل للتوسّع.
