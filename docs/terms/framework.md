# **Framework**  
*(مثال عملي: ASP.NET Core)*

## الترجمة الحرفية  
**Software Framework** — **إطار عمل برمجي**

## الوصف العربي المختصر  
مكتبة + هيكل جاهز يحدّد **طريقة العمل** ويقدّم **مكوّنات** جاهزة (توجيه، DI، سجلات…).  
أنت تكتب **المنطق الخاص بك** داخل *فتحات* يوفّرها الإطار، وهو يتولّى **التشغيل** و**التركيب**.

## الشرح المبسّط  
- **المكتبة**: أنت تستدعيها متى شئت.  
- **الإطار**: هو من **يستدعيك** (Inversion of Control).  
- يمنحك **قواعد** + **خدمات** مشتركة: مصادقة، توجيه، حقن تبعيات، Middleware…  
- النتيجة: سرعة بناء، اتّساق، أقل **كود مكرّر**.

## تشبيه  
مثل **مركز تجاري مجهّز**: الكهرباء، الأمن، اللوحات، المخارج موجودة.  
أنت تفتح **متجرك** داخل هذا الإطار وتتّبع قواعده، فتنجز أسرع وبأمان.

---

## مثال كود C# (ASP.NET Core Minimal API)  
> يبيّن: **توجيه**، **DI**، **Middleware**، **تهيئة**.

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n ShopApi
// dotnet run

var builder = WebApplication.CreateBuilder(args);

// 1) Services (Dependency Injection) — تسجّل خدمات لتُحقن تلقائيًا
builder.Services.AddSingleton<IClock, SystemClock>();     // خدمة وقت
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();                         // توثيق تلقائي

var app = builder.Build();

// 2) Middleware Pipeline — خطوات تمرّ بها كلّ طلب
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();                                // تحويل إلى HTTPS
app.Use(async (ctx, next) =>                              // Middleware بسيط (Log)
{
    Console.WriteLine($"[{DateTime.UtcNow:O}] {ctx.Request.Method} {ctx.Request.Path}");
    await next();
});

// 3) Routing/Handlers — نقاط النهاية (Endpoints)
app.MapGet("/time", (IClock clock) => new                // حقن IClock تلقائياً
{
    utc = clock.UtcNow,
    hello = "ASP.NET Core Framework!"
});

app.MapPost("/echo", (Message m) => Results.Ok(new { youSent = m.Text }));

app.Run();

// نماذج/خدمات
record Message(string Text);

public interface IClock { DateTime UtcNow { get; } }
public class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}
```

**الإخراج المتوقع (مختصر):**  
- GET `/time` → JSON يحوي وقت UTC ورسالة.  
- POST `/echo` مع `{ "text": "hi" }` → يعيد `{ "youSent": "hi" }`.  
- Swagger UI متاح في التطوير.

---

## خطوات عملية (ASP.NET Core مثال)
1. **أنشئ مشروعًا**:  
   ```bash
   dotnet new web -n ShopApi
   cd ShopApi
   dotnet run
   ```
2. افتح `https://localhost:*****` وجرب `/time` و`/echo`.  
3. أضف خدمة جديدة (مثلاً `IEmailSender`) وسجّلها في `builder.Services`.  
4. أنشئ Middleware مخصصًا (مصادقة/سجلات/تعريب).  
5. فعّل **Swagger** و**HSTS/HTTPS** للإنتاج.  
6. انشر على **IIS/NGINX/Container** حسب بيئتك.

---

## أخطاء شائعة
- خلط **المكتبة** بالإطار: عدم احترام دورة حياة الإطار أو تجنّب DI.  
- وضع منطق ثقيل داخل **Middleware** دون حدود/مهلات.  
- تجاهل البيئة (`Development/Production`) وتفعيل أدوات التطوير في الإنتاج.  
- عدم تعريف **عقود واضحة** (نماذج الطلب/الاستجابة) وترك الـ JSON عشوائيًا.  
- تجاهل **التعامل مع الأخطاء** (Exception Handling) ونتائج HTTP المناسبة.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Framework (ASP.NET Core)** | **هيكل + خدمات جاهزة**؛ الإطار يستدعي كودك | **DI/Middleware/Routing/Config/Hosting** |
| [Library](library.md) | دوال/كلاسات تستدعيها أنت | لا يفرض أسلوب مشروعك |
| [SDK](sdk.md) | أدوات + حزم + قوالب | يساعد على البناء/النشر (CLI/قوالب) |
| [Platform](platform.md) | بيئة تشغيل/استضافة | نظام + Runtime + خدمات (مثلاً Azure/Windows) |
| Runtime (.NET) | تنفيذ التعليمات المُدارة و GC | يشغّل تطبيقاتك داخل المنصّة |

---

## ملخص الفكرة  
**Framework** يقدّم لك **الطريق السريع** لبناء تطبيقات متّسقة وسريعة التطوير.  
مع **ASP.NET Core** ستجد: **توجيه، Middleware، DI، توثيق، أمان** — اكتب منطقك واترك الإطار يدير الباقي.
