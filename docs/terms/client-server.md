# **Client–Server**

## الترجمة الحرفية  
**Client–Server** — **نمط العميل/الخادم** (تعاون عبر الشبكة بين طالب خدمة ومقدّمها).

## الوصف العربي المختصر  
أسلوب معماري يُقسّم المنظومة إلى **عميل** يرسل طلبات و**خادم** يوفّر خدمات/بيانات.  
يوفّر **فصل المسؤوليات**، وتوسعة الخادم، وتوحيد الوصول للبيانات عبر بروتوكولات مثل **HTTP/TCP**.

## الشرح المبسّط  
- العميل: واجهة/تطبيق يطلب عمليات (قراءة/إنشاء/تعديل/حذف).  
- الخادم: ينفّذ المنطق ويخزّن البيانات ويردّ بنتيجة.  
- القناة: شبكة موثوقة (TLS)، صيغة تبادل (JSON/Protobuf).  
- مزايا: أمن وتمركز البيانات، سهولة الصيانة، قابلية التوسعة أفقياً.  
- قيود: زمن انتقال الشبكة، الحاجة لنسخ/توازن أحمال ومراقبة.

## تشبيه  
**مطعم**: الزبون (Client) يطلب طبقًا؛ المطبخ (Server) يجهّزه ويرسله.  
نموذج موحّد يضمن الجودة، والقائمة (API) توضّح ما هو متاح وكيف يُطلب.

## مثال كود C# بسيط (خادم Minimal API + عميل HttpClient)

### خادم (ASP.NET Core Minimal API)
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/time", () => Results.Ok(new { utc = DateTime.UtcNow }));

app.MapPost("/api/echo", (Message m) => Results.Ok(new { youSent = m.Text }));

app.Run();

public record Message(string Text);
```

### عميل (استهلاك عبر HttpClient)
```csharp
using System;
using System.Net.Http;
using System.Net.Http.Json;
using System.Threading.Tasks;

class ClientApp
{
    static async Task Main()
    {
        using var http = new HttpClient { BaseAddress = new Uri("https://localhost:5001") };

        var timeRes = await http.GetFromJsonAsync<dynamic>("/api/time");
        Console.WriteLine(timeRes);

        var echoRes = await http.PostAsJsonAsync("/api/echo", new { Text = "Hello Server" });
        Console.WriteLine(await echoRes.Content.ReadAsStringAsync());
    }
}
```
الإخراج المتوقع (مختصر):  
- طلب `GET /api/time` يعيد JSON يحوي وقت UTC.  
- طلب `POST /api/echo` يعيد النص المُرسل داخل كائن JSON.

## خطوات عملية لتصميم/تشغيل Client–Server
- حدّد **العقود**: نقاط النهاية، النماذج، رموز الحالة (OpenAPI/Swagger).  
- اختر البروتوكول/التغليف: **HTTP/1.1 أو HTTP/2 + TLS**، JSON/Protobuf.  
- نفّذ **AuthN/AuthZ** (JWT/OAuth2)، و**CORS** للعملاء المتصفّحين.  
- أضف **الترقيم/الفرز/الترشيح** لنهايات القوائم، و**Cache-Control** حيث يناسب.  
- صمّم **إعادة المحاولة مع Backoff** و**Timeouts** في العميل.  
- وسّع الخادم: **Load Balancer**، **Health Checks**، **Observability** (Logs/Metrics/Tracing).  
- اختبر عبر **Postman/curl** وأدرج الاختبارات في **CI/CD**.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Client–Server** | **تقسيم الأدوار: عميل يطلب وخادم يلبّي** | **وضوح المسؤوليات؛ قابلية توسعة الخادم؛ يحتاج شبكة موثوقة** |
| [Request–Response Model](request-response-model.md) | تبادل متزامن: طلب → استجابة | يعمل غالبًا فوق HTTP؛ مناسب لعمليات CRUD |
| [Publish–Subscribe](publish-subscribe.md) | بث أحداث لمشتركين متعددين | غير متزامن؛ فصل ضعيف الارتباط؛ مناسب للتنبيهات والتدفّق |

## ملخص الفكرة  
**Client–Server** = واجهة تطلب وخادم يلبّي عبر عقد واضحة.  
صمّم العقود والأمان بعناية، وأضف مراقبة وتوسعة لتأمين الأداء والاستقرار.
