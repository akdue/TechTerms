# **HTTPS**

## الترجمة الحرفية  
**Hypertext Transfer Protocol Secure (HTTPS)** — **بروتوكول نقل النص الفائق الآمن** (HTTP فوق **TLS**).

## الوصف العربي المختصر  
نسخة **مشفَّرة ومُوثَّقة** من HTTP تعمل فوق **TLS**:  
- **تشفير** يمنع التجسّس.  
- **سلامة** تمنع العبث بالمحتوى.  
- **توثيق** يثبت هوية الخادم عبر شهادة موثوقة.

## الشرح المبسّط  
- العميل يتصل بالخادم ويجريان **مصافحة TLS** لاختيار خوارزميات وإنشاء مفاتيح.  
- الخادم يقدّم **شهادة** موقّعة من مرجع ثقة (CA).  
- بعد نجاح المصافحة، تُنقل طلبات/استجابات HTTP داخـل قناة آمنة.  
- الإصدارات الحديثة: **TLS 1.2/1.3** (الأفضل اعتمادًا على دعمك).

## تشبيه  
ترسل رسالة في **ظرف مقفول ومختوم** باسم المرسل الحقيقي.  
حتى لو مرّت على وسيط، لا يستطيع فتحها أو تزوير مُرسلها.

## مثال كود C# (خادم ASP.NET Core + عميل آمن)

### خادم (تمكين HTTPS والتحويل التلقائي)
```csharp
// Program.cs (.NET 8/9)
var builder = WebApplication.CreateBuilder(args);

// يفعّل HTTPS Redirection وHSTS (في Production)
builder.Services.AddHsts(o => { o.MaxAge = TimeSpan.FromDays(180); o.IncludeSubDomains = true; });

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();                 // Strict-Transport-Security
    app.UseHttpsRedirection();     // 308 من HTTP إلى HTTPS
}

app.MapGet("/api/secure", () => Results.Ok(new { ok = true, tls = true }));

app.Run();
```

### عميل (التحقق من الشهادة + Pinning اختياري)
```csharp
using System.Net.Http;
using System.Security.Cryptography.X509Certificates;

var handler = new HttpClientHandler
{
    // افتراضيًا يتحقق من سلاسل الثقة واسم المضيف.
    // مثال (اختياري): Pinning عبر بصمة SHA-256 للمفتاح العام (SPKI)
    ServerCertificateCustomValidationCallback = (req, cert, chain, errors) =>
    {
        if (errors == System.Net.Security.SslPolicyErrors.None) return true;

        // إن أردت Pinning: قارن بصمة معروفة (مثال توضيحي فقط)
        // var spkiSha256 = Convert.ToHexString(cert!.GetPublicKey());
        // return spkiSha256.Equals("...");

        return false; // ارفض إن لم تطابق السياسة
    }
};

using var http = new HttpClient(handler) { BaseAddress = new Uri("https://api.example.com") };
var res = await http.GetAsync("/api/secure");
Console.WriteLine($"{(int)res.StatusCode} {res.IsSuccessStatusCode}");
```

> ملاحظات:
> - في الإنتاج استخدم شهادة موثوقة (مثال: **Let’s Encrypt**)، ولا تسمح بتجاوز التحقق.  
> - لا تعتمد pinning بلا خطة **دوران مفاتيح** (Key Rotation) لتفادي انقطاع الخدمة.

## خطوات عملية لتأمين HTTPS (Checklist)
- احصل على شهادة موثوقة (ACME/**Let’s Encrypt** أو مزوّدك)، وجدِّدها آليًا.  
- فعّل فقط **TLS 1.2/1.3** وأوقف البروتوكولات/الأجنحة الضعيفة.  
- طبّق **HSTS** (مع `includeSubDomains` و/أو **preload** بعد الاختبار).  
- حوّل HTTP ↦ HTTPS بـ **301/308**، وتحقق من الروابط الداخلية.  
- امنع **المحتوى المختلط** (Mixed Content)؛ حمّل كل الموارد عبر HTTPS.  
- استخدم **ALPN** (HTTP/2/3) لتحسين الأداء عند الإمكان.  
- راقب صلاحية الشهادة والتنبيهات قبل انتهائها بوقت كافٍ.

## أخطاء شائعة
- تشغيل **شهادة موقّعة ذاتيًا** في الإنتاج أو تعطيل التحقق في العميل.  
- ترك **TLS 1.0/1.1** أو أجنحة تشفير قديمة مفعّلة.  
- نسيان تفعيل **HSTS** أو السماح بمحتوى مختلط.  
- عدم تدوير المفاتيح أو خطط التجديد الآلي للشهادة.  
- افتراض أن HTTPS وحده يكفي دون **AuthZ/Rate Limit/CSRF/CORS**.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [HTTP](http.md) | بروتوكول طلب/استجابة بدون تشفير | مناسب فقط داخل قنوات موثوقة/اختبارات محلية |
| **HTTPS** | **HTTP عبر قناة مشفّرة (TLS)** | **سرية + سلامة + هوية الخادم؛ يوصى به دائمًا على الإنترنت** |
| [mTLS](mtls.md) | توثيق متبادل (عميل وخادم) فوق TLS | قوي للأنظمة الداخلية/الميكروسيرفِس؛ يحتاج إدارة شهادات للعملاء |

## ملخص الفكرة  
**HTTPS = HTTP + TLS**: يوفّر قناة آمنة وهوية مؤكدة.  
فعّل TLS الحديث، HSTS، والتحويل إلى HTTPS، وتحقق صارم من الشهادات لتأمين تطبيقاتك.
