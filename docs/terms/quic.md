# **QUIC — Quick UDP Internet Connections**

## الترجمة الحرفية  
**Quick UDP Internet Connections (QUIC)** — **اتصالات إنترنت سريعة فوق UDP**.

## الوصف العربي المختصر  
**بروتوكول نقل حديث** يعمل فوق **UDP**، يدمج **TLS 1.3**، ويقدّم **تعدّد تدفّقات** داخل اتصال واحد دون مشكلة **Head-of-Line** على مستوى النقل.  
يدعم **اتصالًا أسرع** (1-RTT/‎0-RTT)، و**هجرة الاتصال** (Connection Migration) عند تغيّر عنوان العميل، وامتدادات **Datagrams غير موثوقة**.  
تعمل عليه **HTTP/3**.

## الشرح المبسّط  
- يبدأ بمصافحة **TLS 1.3** داخل QUIC (إطار CRYPTO) → قناة آمنة بسرعة.  
- **Streams** مستقلّة: لو تأخّر تدفّق لا يعلّق الباقي.  
- **Connection IDs** تسمح بتبديل شبكة/IP بدون إسقاط الجلسة.  
- تحكّم ازدحام وكشف فقدان شبيه بـ TCP، لكن على UDP.  
- 0-RTT لإعادة الاتصال (أسرع، لكن انتبه لقابلية إعادة التشغيل/Replay).

## تشبيه  
نفق سريع متعدد المسارات داخل طريق واحد: كل مسار **Stream** مستقل.  
لو تعطّل مسار، **لا يوقف** بقية المسارات.

---

## مثال كود C# — HTTP/3 (فوق QUIC) خادم + عميل

### خادم Kestrel يفعّل HTTP/3
```csharp
// Program.cs (.NET 8/9)
using Microsoft.AspNetCore.Server.Kestrel.Core;

var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(k =>
{
    k.ListenAnyIP(443, o =>
    {
        o.UseHttps("cert.pfx", "P@ssw0rd");                 // شهادة صالحة
        o.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;  // تمكين H3 فوق QUIC
    });
    // ملاحظة: افتح المنفذ UDP/443 على الجدار الناري. يحتاج النظام دعم MsQuic.
});

var app = builder.Build();

app.MapGet("/", () => Results.Ok(new { via = "HTTP/3 over QUIC" }));

app.Run();
```

### عميل HttpClient يطلب HTTP/3
```csharp
using System.Net;
using System.Net.Http;

// يطلب H3 ويقبل ترقية إلى أقل عند عدم الدعم
var req = new HttpRequestMessage(HttpMethod.Get, "https://localhost/")
{
    Version = HttpVersion.Version30,
    VersionPolicy = HttpVersionPolicy.RequestVersionOrHigher
};

using var http = new HttpClient(new SocketsHttpHandler
{
    // تأكد من الثقة بشهادتك محليًا أو استخدم شهادة موثوقة
    ConnectTimeout = TimeSpan.FromSeconds(5)
});

var res = await http.SendAsync(req);
Console.WriteLine($"HTTP Version: {res.Version}"); // 3.0 عند نجاح H3
Console.WriteLine(await res.Content.ReadAsStringAsync());
```

> إن أردت بروتوكولك الخاص فوق QUIC (بدون HTTP/3)، استخدم مساحة الأسماء **`System.Net.Quic`**  
> (‎`QuicListener/QuicConnection/QuicStream`)، مع **ALPN** مخصّص وشهادة TLS.

---

## خطوات عملية لاعتماد QUIC/HTTP/3
- **افتح UDP/443** (بالإضافة إلى TCP/443) على الجدار الناري وSecurity Groups.  
- استخدم **شهادة TLS** موثوقة، وفعّل **HTTP/3** في الخادم/الـ Proxy (Kestrel/NGINX/Envoy/CDN).  
- تحقق من **Alt-Svc** أو الإعلان المباشر عن H3، وفعّل **ALPN = h3**.  
- اختبر من العميل: اطبع **نسخة HTTP** وتحقق أنها **3.0**، واستخدم أدوات تتبّع (Wireshark/MsQuic logs).  
- راقب المقاييس: **RTT، فقدان، معدل إعادة الإرسال، عدد التدفقات**.  
- فعّل **Fallback** تلقائيًا إلى H2/H1 عندما لا يتوفر QUIC.

## أخطاء شائعة
- نسيان فتح **UDP/443** → يبقى كل شيء على H2/H1.  
- توقّع تحسّن دائم بلا قياس؛ بعض الشبكات/الأجهزة قد تحدّ من QUIC.  
- استخدام **0-RTT** لعمليات غير **Idempotent** (خطر إعادة التشغيل).  
- إهمال **ALPN** أو عدم إعداد الشهادة/السلسلة بشكل صحيح.  
- افتراض أن QUIC “لا يحتاج ضبط ازدحام” — ما زال يحتاج إعدادًا ومراقبة.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [TCP](tcp.md) | اتصال موثوق مرتّب (Stream) | HOL على النقل عند التعدد عبر التطبيق |
| [UDP](udp.md) | رسائل خفيفة بلا اتصال | لا ضمان تسليم/ترتيب |
| **QUIC** | **نقل موثوق متعدد التدفّقات فوق UDP + TLS 1.3** | **1-RTT/0-RTT، Migration، تقليل HOL، يتطلب UDP/443** |
| HTTP/3 | HTTP فوق QUIC | يستخدم تدفّقات QUIC وALPN=h3 |

## ملخص الفكرة  
**QUIC** يجمع موثوقية شبيهة بـ TCP مع سرعة ومرونة UDP، ويُشغّل **HTTP/3**.  
فعّل H3 على الخادم، افتح UDP/443، واختبر النسخة من العميل—ستحصل على اتصالات أسرع وأقل تأثّرًا بتأخير تدفّق واحد.
