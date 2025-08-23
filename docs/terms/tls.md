# **TLS — Transport Layer Security**

## الترجمة الحرفية  
**Transport Layer Security (TLS)** — **أمن طبقة النقل** (بروتوكول لتأمين القناة بين طرفين).

## الوصف العربي المختصر  
TLS يوفّر ثلاث ركائز للقناة: **تشفير** البيانات، **سلامتها** (منع العبث)، و**توثيق الهوية** عبر **شهادات رقمية**.  
يُستخدم تحت بروتوكولات مثل **HTTPS/SMTP/IMAP/MQTT** لتأمين المرور عبر الشبكات.

## الشرح المبسّط  
- يبدأ الطرفان **مصافحة (Handshake)** لاختيار الخوارزميات وإنشاء **مفاتيح جلسة**.  
- الخادم يقدّم **شهادة** موقّعة من مرجع ثقة (CA) ليثبت هويته.  
- بعد نجاح المصافحة، تُشفَّر كل الرسائل وتُحمى من التلاعب.  
- إصدارات مخصّصة للأمان والسرعة: **TLS 1.2** و**TLS 1.3** (الأحدث والأفضل).

## تشبيه  
ترسل رسالة داخل **ظرف مُقفَل ومختوم** مع بطاقة تعريف موثوقة للمرسِل.  
أي وسيط بالمنتصف لا يستطيع **قراءة** الرسالة ولا **تزوير** المُرسِل.

## مثال كود C# بسيط

### 1) تفعيل سياسات TLS في خادم Kestrel (ASP.NET Core)
```csharp
// Program.cs (.NET 8/9)
using System.Security.Authentication;

var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(k =>
{
    k.ConfigureHttpsDefaults(h =>
    {
        // حصر البروتوكولات على TLS 1.2 و 1.3 فقط
        h.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
        // ملاحظات:
        // - استخدم شهادة خادم موثوقة (Let's Encrypt/ACME) عبر الإعدادات أو reverse proxy.
        // - عطّل الأجنحة الضعيفة إن كنت تتحكم بها (على مستوى النظام/الخادم).
    });
});

var app = builder.Build();
app.MapGet("/secure", () => Results.Ok(new { ok = true }));
app.Run();
```

### 2) عميل HttpClient يفرض TLS حديثًا ويتحقق من الشهادة
```csharp
using System.Net.Http;
using System.Security.Authentication;

var handler = new HttpClientHandler
{
    SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13
    // التحقّق من شهادة الخادم مُفعّل افتراضيًا — لا تعطِّله في الإنتاج
};

using var http = new HttpClient(handler) { BaseAddress = new Uri("https://example.com") };
var res = await http.GetAsync("/secure");
Console.WriteLine($"{(int)res.StatusCode} {res.IsSuccessStatusCode}");
```

### 3) (اختياري) استخدام SslStream مباشرةً فوق TCP
```csharp
using System.Net.Security;
using System.Net.Sockets;
using System.Security.Authentication;
using System.Text;

using var tcp = new TcpClient();
await tcp.ConnectAsync("example.com", 443);

using var ssl = new SslStream(tcp.GetStream(), leaveInnerStreamOpen: false);
await ssl.AuthenticateAsClientAsync(new SslClientAuthenticationOptions
{
    TargetHost = "example.com",
    EnabledSslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13
});

var req = "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n";
var bytes = Encoding.ASCII.GetBytes(req);
await ssl.WriteAsync(bytes);
await ssl.FlushAsync();

var buf = new byte[8192];
int n = await ssl.ReadAsync(buf);
Console.WriteLine(Encoding.ASCII.GetString(buf, 0, n));
```
> الفكرة: TLS يؤمّن القناة نفسها، سواء استعملتها عبر **HTTP** (HTTPS) أو مباشرةً عبر **SslStream**.

## خطوات عملية لتأمين TLS (Checklist)
- اعتمد **TLS 1.2/1.3** فقط؛ عطّل البروتوكولات والأجنحة القديمة.  
- استخدم **شهادة موثوقة** بسلسلة ثقة كاملة و**SAN** صحيح لاسم المضيف.  
- فعّل **HSTS** عند الويب، وحوِّل HTTP ↦ HTTPS.  
- راقب **انتهاء الشهادات** وجدّدها تلقائيًا (ACME/Cert-Manager).  
- دعم **HTTP/2/3 (ALPN)** لتحسين الأداء عند الإمكان.  
- لا تُعطِّل التحقّق من الشهادة في العميل؛ إن احتجت pinning فخطّط لـ **دوران المفاتيح**.  

## أخطاء شائعة
- استخدام شهادات **موقّعة ذاتيًا** في الإنتاج أو تجاوز التحقّق.  
- ترك **TLS 1.0/1.1** مفعّلًا أو أجنحة تشفير ضعيفة.  
- نسيان سلسلة الثقة الوسيطة → فشل التحقق لدى العملاء.  
- الاعتماد على TLS وحده دون **AuthZ/Rate-Limit/CSRF/CORS** على طبقة التطبيق.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [HTTPS](https.md) | HTTP عبر TLS مع توثيق الخادم | قناة ويب مشفّرة وهوية الخادم |
| **TLS** | **تشفير/سلامة/توثيق على طبقة النقل** | **يُستخدم تحت بروتوكولات كثيرة (HTTPS/SMTP/IMAP/…)** |
| [mTLS](mtls.md) | توثيق متبادل (خادم وعميل) فوق TLS | يحتاج شهادات للعملاء وإدارة مفاتيح |

## ملخص الفكرة  
**TLS** هو الغلاف الأمني للقنوات الشبكية: يقدّم **تشفيرًا + سلامة + هوية**.  
استعمله عبر الإصدارات الحديثة، بشهادات موثوقة، وسياسات صارمة لتحصل على قناة آمنة ومستقرة.

## سؤال تثبيت (إجابة مخفيّة)
*أنت تبني API عامة. ما الإعدادات الأساسية المتعلقة بـ TLS التي ينبغي تفعيلها؟*
<details>
  <summary><strong>إظهار الإجابة</strong></summary>
  **TLS 1.2/1.3 فقط**، شهادة موثوقة بسلسلة كاملة و**SAN** صحيح، **HSTS** + تحويل **HTTP → HTTPS**،  
  تعطيل الأجنحة الضعيفة، وتجديد تلقائي للشهادة مع مراقبة الانتهاء.
</details>
