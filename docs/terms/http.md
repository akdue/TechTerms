**# **HTTP**

## الترجمة الحرفية  
**Hypertext Transfer Protocol (HTTP)** — **بروتوكول نقل النص الفائق**

## الوصف العربي المختصر  
بروتوكول **طلب/استجابة** على طبقة التطبيق لربط **عميل** (متصفح/تطبيق) بـ **خادم** عبر الشبكة.  
يدعم طرقًا (GET/POST/PUT/DELETE…) ورؤوسًا (Headers) وهيئات بيانات (JSON/HTML/ملفات)، مع رموز حالة واضحة.

## الشرح المبسّط  
- العميل يرسل **Request** إلى مسار (URL) مع **Method** ورؤوس/جسم اختياري.  
- الخادم يردّ بـ **Status Code** (200/201/404/500…) + رؤوس + **Body**.  
- الإصدارات: **HTTP/1.1** (سلسلي)، **HTTP/2** (تعدد قنوات على اتصال واحد)، **HTTP/3** (أسرع فوق QUIC).  
- مفاهيم مهمّة: **Idempotency** (GET/PUT/DELETE آمنة للتكرار)، **Caching** (ETag, Cache-Control)، **Content-Type/Accept**.

## تشبيه  
مكتب بريد: تضع رسالتك داخل **ظرف** بعناوين وطوابع (Headers/Method/URL)،  
ثم تصلك **رسالة جواب** مختومة بنتيجة الطلب (Status + محتوى).

## مثال كود C# بسيط (خادم + عميل)

### خادم (ASP.NET Core Minimal API)
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/products/{id:int}", (int id) =>
{
    if (id <= 0) return Results.BadRequest(new { error = "Invalid id" });

    // أمثلة رؤوس مفيدة
    return Results.Ok(new { id, name = "Laptop", price = 999 })
                 .WithHeader("Cache-Control", "public, max-age=60")
                 .WithHeader("ETag", "\"p-999\"");
});

app.MapPost("/api/orders", (OrderDto dto) =>
{
    if (string.IsNullOrWhiteSpace(dto.ProductId) || dto.Qty <= 0)
        return Results.BadRequest(new { error = "Invalid payload" });

    var id = 123;
    return Results.Created($"/api/orders/{id}", new { orderId = id, status = "created" });
});

app.Run();
record OrderDto(string ProductId, int Qty);
```

### عميل (استهلاك عبر HttpClient)
```csharp
using var http = new HttpClient { BaseAddress = new Uri("https://localhost:5001") };

// GET مع ترويسة Accept
var req = new HttpRequestMessage(HttpMethod.Get, "/api/products/5");
req.Headers.Accept.ParseAdd("application/json");
using var res = await http.SendAsync(req, HttpCompletionOption.ResponseHeadersRead);
Console.WriteLine((int)res.StatusCode);
Console.WriteLine(await res.Content.ReadAsStringAsync());

// POST JSON
var create = await http.PostAsJsonAsync("/api/orders", new { ProductId = "5", Qty = 2 });
Console.WriteLine((int)create.StatusCode);         // 201
Console.WriteLine(create.Headers.Location);        // /api/orders/123

// مهلات وإلغاء
http.Timeout = TimeSpan.FromSeconds(10);
// استخدم CancellationToken عند الطلبات الطويلة
```

الإخراج المتوقع (مختصر):  
- `GET` يعيد `200` مع JSON للمنتج ورؤوس Caching.  
- `POST` يعيد `201 Created` وترويسة `Location` برابط المورد الجديد.

## خطوات عملية لاستخدام HTTP بفعالية
- صمّم **العقود** بوضوح (نماذج JSON، أنواع الأخطاء، الحقول المطلوبة).  
- اختر **الطريقة المناسبة**: GET للقراءة، POST للإنشاء، PUT للاستبدال، PATCH للتعديل الجزئي، DELETE للحذف.  
- استخدم **Status Codes** بدقة ورسائل خطأ مفهومة.  
- فعّل **Caching** (ETag/Last-Modified/Cache-Control) للقوائم والموارد الثابتة.  
- احترم **Idempotency** وادعم **مفاتيح Idempotency** للعمليات الحسّاسة (دفع/إنشاء).  
- استخدم **HTTPS** دائمًا (TLS)، وحدّد **Content-Type/Accept** و**CORS** للمتصفح.  
- أضف **Rate Limiting** و**Retry + Backoff** و**Timeouts** في العميل.  
- استغل مزايا **HTTP/2/3** (Multiplexing/تقليل الـ RTT) عند الإمكان.  
- وثّق عبر **OpenAPI/Swagger** واختبر بـ **curl/Postman** وCI.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **HTTP** | **بروتوكول طلب/استجابة لنقل النص والوسائط** | **إصدارات 1.1/2/3؛ يعمل فوق TCP (1.1/2) أو QUIC (3)** |
| [HTTPS](https.md) | HTTP عبر قناة مشفّرة (TLS) | يحمي السرية والسلامة وهوية الخادم |
| [WebSocket](websocket.md) | اتصال ثنائي مستمر | يبدأ عبر HTTP ثم يترقّى؛ للأحداث الفورية |

## ملخص الفكرة  
**HTTP** هو العمود الفقري للويب: طلب واضح → استجابة واضحة.  
اختَر الطرق الصحيحة، التزم بالرموز والرؤوس، واستخدم HTTPS وميزات caching وHTTP/2/3 لتحصل على واجهات سريعة وآمنة.
**