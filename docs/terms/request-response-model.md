# **R&R Model — Request & Response Model**

## الترجمة الحرفية  
**Request & Response Model (R&R)** — **نموذج الطلب/الاستجابة**

## الوصف العربي المختصر  
نمط تواصل بين **عميل** و**خادم** حيث يرسل العميل **طلبًا** (Request) ويحصل على **استجابة** (Response).  
مناسب للعمليات المتزامنة القصيرة (قراءة/إنشاء/تحديث/حذف) مع **رموز حالة** واضحة و**هيئة بيانات** متفق عليها.

## الشرح المبسّط  
- العميل يقول: “أعطني /products/5” → الخادم يرد ببيانات المنتج أو بخطأ.  
- يتكوّن كل تفاعل من: **سطر طلب** + **رؤوس Headers** + (اختياريًا) **جسم Body**.  
- الاستجابة تحوي: **Status Code** (200/201/400/404/500…) + **Headers** + **Body**.  
- مناسب للويب (HTTP/HTTPS)، ولواجهات REST وGraphQL (عبر HTTP)، وحتى RPC فوق HTTPS.  

## تشبيه  
مكتب خدمة: تأخذ **نموذج طلب**، تملأه، وتسلّمه للموظّف.  
بعد دقائق تستلم **نتيجة** مختومة تبين النجاح أو الرفض وسبب القرار.

## مثال كود C# بسيط (خادم + عميل)

### خادم (ASP.NET Core Minimal API)
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/products/{id:int}", (int id) =>
{
    if (id <= 0) return Results.BadRequest(new { error = "Invalid id." });
    var product = new { id, name = "Laptop", price = 999 };
    return Results.Ok(product); // 200 + JSON
});

app.MapPost("/api/orders", (OrderDto dto) =>
{
    if (string.IsNullOrWhiteSpace(dto.ProductId)) return Results.BadRequest();
    // احفظ الطلب...
    return Results.Created($"/api/orders/123", new { orderId = 123, status = "created" }); // 201
});

app.Run();

record OrderDto(string ProductId, int Qty);
```

### عميل (استهلاك عبر HttpClient)
```csharp
using var http = new HttpClient { BaseAddress = new Uri("https://localhost:5001") };

// 1) طلب GET
var r1 = await http.GetAsync("/api/products/5");
Console.WriteLine((int)r1.StatusCode);                // 200 أو غيره
Console.WriteLine(await r1.Content.ReadAsStringAsync());

// 2) طلب POST مع JSON
var r2 = await http.PostAsJsonAsync("/api/orders", new { ProductId = "5", Qty = 2 });
Console.WriteLine((int)r2.StatusCode);                // 201 عند الإنشاء
Console.WriteLine(r2.Headers.Location);               // /api/orders/123
```

الإخراج المتوقع (مختصر):  
- `GET` يطبع `200` ثم JSON للمنتج.  
- `POST` يطبع `201` وموقع المورد الجديد في ترويسة `Location`.

## خطوات عملية لتصميم R&R فعّال
- حدّد **العقود** (نِماذج الطلب/الاستجابة) وهيئة البيانات (**JSON** غالبًا).  
- استخدم **رموز حالة HTTP** بدقّة، وأعد رسائل خطأ واضحة (مفتاح `error`/`code`).  
- وثّق الـ API عبر **OpenAPI/Swagger**.  
- راعِ **Idempotency**: اجعل `GET/PUT/DELETE` آمنة للتكرار؛ وادعم مفاتيح idempotency لعمليات الدفع.  
- أضف **Pagination/Sorting/Filtering** لنهايات القوائم.  
- نفّذ **Auth** (JWT/OAuth2) و**Rate Limiting** و**Caching** عند الحاجة.  
- اختبر عبر **Postman/curl** وأدخِلها في **CI**.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [Client-Server](client-server.md) | تقسيم الأدوار بين عميل يطلب وخادم يلبّي | نمط معماري عام؛ لا يحدّد شكل الرسائل بالتفصيل |
| **Request–Response Model** | **تبادل متزامن: طلب واحد → استجابة واحدة** | **واضح وبسيط؛ مناسب لعمليات CRUD القصيرة ورموز الحالة** |
| [Publish-Subscribe](publish-subscribe.md) | بث أحداث لمشتركين متعددين | غير متزامن غالبًا؛ جيد للبث والإشعارات والفاعليات |

## ملخص الفكرة  
**R&R (Request–Response)** = “اسأل… يصلك جواب”.  
صمّم العقود بوضوح، استخدم رموز الحالة بدقة، ووثّق كل شيء لضمان تواصل متين بين العميل والخادم.

## سؤال تثبيت (إجابة مخفيّة)
*متى تفضّل Pub/Sub على R&R؟*
<details>
  <summary><strong>إظهار الإجابة</strong></summary>
  عندما تحتاج **بث أحداث لعدة مستهلكين** أو **تفاعل غير متزامن** (إشعارات، Telemetry، تكاملات ضعيفة الارتباط).  
  أمّا **العمليات المتزامنة القصيرة** (قراءة/تعديل مورد) فـ R&R أبسط وأنسب.
</details>
