# **Dependency Injection**

## الترجمة الحرفية  
**Dependency Injection (DI)** — **حقن التبعيات**.

## الوصف العربي المختصر  
أسلوب يفصل **إنشاء الكائنات** عن **استخدامها**.  
الاعتماد (Dependency) يُمرَّر من الخارج (Container/Caller) بدل `new` داخل الكود.  
النتيجة: **تفكيك ارتباط**، **اختبار أسهل**، و**قابلية تبديل**.

## الشرح المبسّط  
- أي صنف يحتاج خدمة؟ **لا يُنشئها** بنفسه. يستقبلها في **البنّاء** (Constructor).  
- يوجد **Container** يعرِف *كيف* ينشئ كل خدمة (Lifetimes/تهيئة).  
- في ASP.NET Core، الحاوية مدمجة (`Microsoft.Extensions.DependencyInjection`).  
- أنماط الحقن: **Constructor** (المفضل)، ثم **Method/Property** للحالات الخاصة.

## تشبيه  
الأجهزة في المنزل لا تُولّد كهرباء. **تتّصل بمقبس** (DI).  
غيّرت شركة الكهرباء؟ لا تغيّر الأجهزة—فقط المقبس يمدّك بالمصدر الجديد.

---

## مثال C# مختصر — قبل/بعد DI + تسجيل الخدمات + اختبار

### (1) قبل DI — اقتران قوي
```csharp
public class OrderService
{
    public string PlaceOrder(decimal amount)
    {
        // اقتران قوي: صعب الاختبار/التبديل
        var gateway = new PaymentGateway();      // new داخل الكود
        if (!gateway.Charge(amount)) return "fail";
        return "ok";
    }
}

public class PaymentGateway
{
    public bool Charge(decimal amount) => amount > 0; // تبسيط
}
```

### (2) بعد DI — واجهة + حقن عبر البنّاء
```csharp
public interface IPaymentGateway { bool Charge(decimal amount); }

public class PaymentGateway : IPaymentGateway
{
    public bool Charge(decimal amount) => amount > 0; // تبسيط
}

public class OrderService
{
    private readonly IPaymentGateway _gateway;
    public OrderService(IPaymentGateway gateway) => _gateway = gateway;

    public string PlaceOrder(decimal amount)
        => _gateway.Charge(amount) ? "ok" : "fail";
}
```

### (3) تسجيل واستخدام في ASP.NET Core
```csharp
// Program.cs  (.NET 8/9)
var builder = WebApplication.CreateBuilder(args);

// Lifetimes: Transient (كل مرّة) / Scoped (لكل طلب) / Singleton (طيلة عمر التطبيق)
builder.Services.AddScoped<IPaymentGateway, PaymentGateway>();
builder.Services.AddScoped<OrderService>();

var app = builder.Build();

// Minimal API: الحقن مباشرة في المعالج
app.MapPost("/orders", (decimal amount, OrderService svc)
    => Results.Ok(new { status = svc.PlaceOrder(amount) }));

app.Run();
```

### (4) اختبار وحدات — تبديل الاعتماد بـ Fake
```csharp
// xUnit
public class OrderServiceTests
{
    private class FakeGateway : IPaymentGateway
    {
        private readonly bool _ok;
        public FakeGateway(bool ok) => _ok = ok;
        public bool Charge(decimal amount) => _ok;
    }

    [Fact] public void Succeeds_When_Gateway_Ok()
        => Assert.Equal("ok", new OrderService(new FakeGateway(true)).PlaceOrder(100));

    [Fact] public void Fails_When_Gateway_Fails()
        => Assert.Equal("fail", new OrderService(new FakeGateway(false)).PlaceOrder(100));
}
```

**الفكرة:** استبدال `PaymentGateway` الحقيقي في الاختبار **بدون** تعديل `OrderService`.  
هذا هو **التفكيك** الحقيقي الذي يقدمه DI.

---

## خطوات عملية لاعتماد DI
1. **عرّف واجهات** للخدمات القابلة للتبديل (Email, Payment, Clock, Repo…).  
2. **حقن عبر البنّاء** فقط ما يلزم. تجنّب “God Constructor” (قائمة كبيرة).  
3. قدّم **Composition Root** واحدًا (مثل `Program.cs`) لتسجيل كل الخدمات.  
4. اختر **الـLifetime** الصحيح:  
   - **Singleton**: ثابت وخفيف وخيوط آمن (مثلاً `HttpClientFactory` نفسها).  
   - **Scoped**: لكل طلب ويب (شائع مع `DbContext`).  
   - **Transient**: خفيف قصير العمر.  
5. لا تحقن **Scoped** داخل **Singleton** (تسريب عمر). إن احتجت، استخدم **Factory**/`IServiceScopeFactory`.  
6. أضِف **واجهات اختبار** للأشياء الخارجية (الوقت، I/O، HTTP) لتجنّب العشوائية.

---

## أخطاء شائعة
- استدعاء الحاوية داخل الكود (`IServiceProvider.GetService`) = **Service Locator** (مضاد نمط).  
- Lifetimes خاطئة → تسريبات/تضارب حالة (خصوصًا مع `DbContext`).  
- تسجيلات مكررة أو دوائر إعتماد (Circular Dependency).  
- منطق إعداد ثقيل داخل البنّاء → اجعله في **Factory** أو **Hosted Service**.  
- حقن **الكثير** في صنف واحد → فكّه إلى خدمات أصغر.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Dependency Injection (DI)** | **فصل الإنشاء عن الاستخدام** | **Constructor Injection** + **Lifetimes** + **Composition Root** |
| IoC Container | إدارة الإنشاء والحقن تلقائيًا | مدمج في ASP.NET Core؛ يدعم Scopes |
| Service Locator | استرجاع الخدمة من الحاوية داخل الكود | يُضعف الاختبار والوضوح؛ يُنصح بتجنّبه |
| Factory | إنشاء كائنات بمنطق مخصّص | جيد عندما يعتمد الإنشاء على مدخلات وقت التشغيل |
| Manual Wiring | `new` يدوي بدون حاوية | بسيط لمشاريع صغيرة؛ ينهار مع الحجم/الاختبار |

---

## ملخص الفكرة  
**DI** يجعل الكود **قابلًا للاختبار والتبديل** عبر تمرير التبعيات بدل إنشائها.  
سجّل الخدمات بــ **Lifetime** مناسب، **احقن عبر البنّاء**، وابقِ التوصيل في **Composition Root**—  
تحصل على تصميم أنظف، قابلية صيانة أعلى، وتغيير أسهل دون كسر النظام.
