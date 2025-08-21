# **API**

## التسمية المعتمدة (EN ↔ AR)
**Application Programming Interface (API)** — **واجهة برمجة التطبيقات**

## تعريف موجز
A well-defined **contract** that lets software **talk** to other software via **requests** and **responses**.  
It standardizes how to **access data** and **trigger actions** without exposing internal details.

واجهة متّفق عليها ليتواصل **برنامج مع برنامج** عبر **طلب/استجابة**.  
تُحدّد شكل البيانات والقواعد للوصول إلى الوظائف دون كشف التفاصيل الداخلية.

---

## لماذا API؟ (مختصر مفيد)
- **فصل المسؤوليات:** فريق الواجهة الأمامية يستهلك، وفريق الخادم يقدّم.  
- **التكامل:** ربط أنظمتك بخدمات دفع/خرائط/رسائل.  
- **إعادة الاستخدام:** نفس الـAPI يخدم تطبيق ويب وموبايل وميكروسيرفِس.  
- **الأمان:** مشاركة ما تحتاج فقط عبر توثيق وصلاحيات محسوبة.

---

## الأنواع الشائعة (بنظرة سريعة)
- **REST:** موارد وعناوين (URLs) + أفعال HTTP (GET/POST/PUT/DELETE).  
- **GraphQL:** تستعلم *بالضبط* الحقول التي تريدها في طلب واحد.  
- **gRPC:** سريع ثنائي (Protocol Buffers)، ممتاز للميكروسيرفِس.  
- **SOAP:** قديم/ثقيل نسبيًا، XML ورسائل معيارية.

---

## مفاهيم أساسية
- **Endpoint:** عنوان قابل للنداء مثل `/api/products/5`.  
- **Resource:** الشيء الذي تتعامل معه (Product, User…).  
- **HTTP Methods:**  
  - GET (قراءة)، POST (إنشاء)، PUT (استبدال)، PATCH (تعديل جزئي)، DELETE (حذف).  
- **Status Codes:** 200 نجاح، 201 تم الإنشاء، 400 طلب غير صالح، 401 غير موثّق، 403 ممنوع، 404 غير موجود، 500 خطأ خادم.  
- **Auth:** API Key / OAuth2 / JWT.  
- **Versioning:** مثل `/v1/...` لتحديثات غير متوافقة مستقبلًا.

---

## مثال من الكود

### 1) إنشاء API صغير جدًا (ASP.NET Core Minimal API)
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Endpoint بسيط يعيد وقت الخادم
app.MapGet("/api/time", () => Results.Ok(new { serverTime = DateTime.UtcNow }));

app.Run();
```

### 2) استهلاك API من C# (عميل)
```csharp
using var http = new HttpClient();
var res = await http.GetAsync("https://api.example.com/products/1");
if (!res.IsSuccessStatusCode)
    Console.WriteLine($"خطأ: {(int)res.StatusCode}");
else
    Console.WriteLine(await res.Content.ReadAsStringAsync()); // JSON كنص
```

> تشبيه: تخيّل مطعمًا. **القائمة** = التوثيق (Docs)، **الطلب** = Request، **الطبق** = Response، **الشيف** خلف الكواليس = الخادم (Server/API).

---

## مصطلحات مرتبطة وتمييزات سريعة
- **API vs SDK:** الـAPI عقد النداء؛ الـSDK أدوات/مكتبات تسهّل استهلاكه.  
- **REST vs GraphQL:** REST موارد ثابتة؛ GraphQL مرن بالحقل/الشكل ويقلّل طلبات متعددة.  
- **PUT vs PATCH:** PUT يستبدل المورد كاملًا؛ PATCH يحدّث جزءًا.  
- **AuthN vs AuthZ:** التوثيق (من أنت؟) مقابل التفويض (ماذا يُسمح لك؟).  
- **Endpoint vs Route:** غالبًا يُستخدمان تبادليًا؛ الـRoute تعريف مسار داخل التطبيق، والـEndpoint نقطة النداء الفعلية المعرّفة به.

---

## متى أستخدمه؟ ومتى أتجنّبه؟
- **استخدم API** عند الحاجة لتبادل البيانات بين تطبيقات/أجهزة أو فصل الواجهة عن الخادم.  
- **تجنّب التعقيد** عند وجود منطق داخلي بسيط داخل نفس التطبيق ولا حاجة لمستوى شبكة.

---

## أخطاء شائعة
- خلط الأفعال: استخدام **GET** للتعديل بدلاً من **POST/PUT**.  
- تجاهل الأكواد: دائمًا أعد **status code** مناسب ورسالة خطأ مفهومة.  
- كشف نماذج قاعدة البيانات كما هي (Coupling) بدل **DTOs** واضحة.  
- لا توثيق/لا إصدار (Versioning) → كسر للعملاء عند تغييرات لاحقة.  
- تجاهل **الترقيم (Pagination)** و**الترشيح (Filtering)** في القوائم الكبيرة.

---

## خطوات عملية للبدء (Check-list)
1. حدّد **الموارد** و**الأفعال** المطلوبة.  
2. صمّم **العقود** (نماذج الطلب/الاستجابة) وعرّف رموز الحالة.  
3. أضف **Auth** مناسب (JWT مثلًا) و**CORS** للمتصفّح.  
4. فعّل **Swagger/OpenAPI** للتوثيق والتجربة السريعة.  
5. اختبر بـ **curl/Postman** وغيّر وفق الملاحظات.  
6. راقب الأداء والأخطاء (Logging/Tracing/Rate Limits).

---

## مثال أوضح للتوثيق الذاتي (Swagger في ASP.NET Core)
```csharp
// Program.cs (مقتطف)
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
app.UseSwagger();
app.UseSwaggerUI(); // واجهة تفاعلية لتجربة الـAPI

app.MapGet("/api/products/{id:int}", (int id) =>
{
    if (id <= 0) return Results.BadRequest(new { error = "Invalid id." });
    return Results.Ok(new { id, name = "Laptop", price = 999 });
});

app.Run();
```

---

## سؤال تثبيت (إجابة مخفيّة)
*"ما الفعل المناسب لإنشاء مورد جديد على `/api/orders`؟ وماذا تتوقع كود الحالة عند النجاح؟"*
<details>
  <summary><strong>إظهار الإجابة</strong></summary>
  **POST** هو الفعل المناسب، ونتوقع **201 Created** مع رابط المورد الجديد في العنوان `Location`.
</details>
