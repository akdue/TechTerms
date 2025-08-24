# **Interface**

## الترجمة الحرفية  
**Interface** — **واجهة** (عقد سلوك بدون تنفيذ/حالة).

## الوصف العربي المختصر  
تعريف **ما يجب أن يفعله** النوع (دوال/خصائص/أحداث/فهارس) دون كيف.  
تمكّن **التجريد**، **تعدّد الأشكال**، و**التركيب** عبر استبدال التنفيذ بحرية.

## الشرح المبسّط  
- الواجهة = **عقد**: أسماء دوال، تواقيع، وخصائص متوقَّعة.  
- الأصناف/الهياكل **تنفّذ** الواجهة (`: IInterface`) وتوفّر الكود.  
- العميل يتعامل مع **الواجهة**؛ التنفيذ يمكن تغييره (اختبار أسهل، اقتران أقل).  
- في C# يمكن أن تحوي الواجهة **خصائص، أحداث، فهارس**، ومنذ C# 8 قد تحتوي **تنفيذًا افتراضيًا** (بحذر).

## تشبيه  
مقبس USB: شكل/دبابيس موحّدة (**عقد**). يمكنك وصل **ماوس** أو **كيبورد** أو **ذاكرة** (تنفيذ مختلف) دون تغيير اللابتوب.

---

## مثال C# (1) — واجهة + حقن تبعيات + تبديل التنفيذ

```csharp
// العقد: بوابة دفع عامة
public interface IPaymentGateway
{
    bool Charge(decimal amount, string currency, string customerId);
}

// تنفيذ 1: وهمي للاختبار/التطوير
public class FakeGateway : IPaymentGateway
{
    public bool Charge(decimal amount, string currency, string customerId) => amount > 0;
}

// تنفيذ 2: Stripe (مبسّط لشرح الفكرة)
public class StripeGateway : IPaymentGateway
{
    public bool Charge(decimal amount, string currency, string customerId)
    {
        Console.WriteLine($"[Stripe] amount={amount} {currency} customer={customerId}");
        return true; // استدعاء API حقيقي في الواقع
    }
}

// خدمة أعمال تعتمد على "الواجهة" فقط
public class CheckoutService
{
    private readonly IPaymentGateway _gateway;
    public CheckoutService(IPaymentGateway gateway) => _gateway = gateway;

    public string Pay(string customerId, decimal total)
        => _gateway.Charge(total, "USD", customerId) ? "paid" : "failed";
}
```

```csharp
// Program.cs  (.NET 8/9) — تبديل التنفيذ دون تعديل CheckoutService
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IPaymentGateway, FakeGateway>();   // dev
// builder.Services.AddSingleton<IPaymentGateway, StripeGateway>(); // prod
builder.Services.AddSingleton<CheckoutService>();

var app = builder.Build();
app.MapPost("/checkout", (string customerId, decimal total, CheckoutService svc)
    => Results.Ok(new { status = svc.Pay(customerId, total) }));
app.Run();
```

> **الفكرة:** العميل يتعامل مع `IPaymentGateway`. يمكن استبدال التنفيذ بسهولة (اختبار/بيئات).

---

## مثال C# (2) — **Interface Segregation** + تنفيذ صريح (Explicit)

```csharp
// فصل القراءة عن الكتابة — واجهات صغيرة محددة الغرض
public interface IReadRepo<T> { T? Find(int id); IEnumerable<T> All(); }
public interface IWriteRepo<T> { T Add(T item); void Delete(int id); }

public class InMemoryRepo<T> : IReadRepo<T>, IWriteRepo<T> where T : class
{
    private readonly Dictionary<int, T> _db = new();
    private int _next = 1;

    public T? Find(int id) => _db.TryGetValue(id, out var v) ? v : null;
    public IEnumerable<T> All() => _db.Values;

    // تنفيذ صريح: لا يظهر على النوع ما لم نعامله كـ IWriteRepo<T>
    T IWriteRepo<T>.Add(T item) { _db[_next] = item; return _db[_next++]; }
    void IWriteRepo<T>.Delete(int id) { _db.Remove(id); }
}

public class CatalogService
{
    private readonly IReadRepo<string> _readOnly;
    public CatalogService(IReadRepo<string> readOnly) => _readOnly = readOnly;

    public IEnumerable<string> List() => _readOnly.All(); // لا يحتاج صلاحية كتابة
}

class Demo
{
    static void Main()
    {
        var repo = new InMemoryRepo<string>();
        // نحتاج كاست إلى واجهة الكتابة لاستدعاء Add (تنفيذ صريح)
        var writer = (IWriteRepo<string>)repo;
        writer.Add("Keyboard"); writer.Add("Mouse");

        var svc = new CatalogService(repo); // يستهلك الواجهة القارئة فقط
        foreach (var name in svc.List()) Console.WriteLine(name);
    }
}
```

**الفكرة:** واجهات **صغيرة ومركّزة** تقلّل الاقتران. التنفيذ الصريح يمنع كشف عمليات الكتابة بطريق الخطأ.

---

## خطوات عملية لاستخدام الواجهات بذكاء
1. ابدأ من **العقد**: سمِّ الدوال/الخصائص بما يصف **الهدف** لا التنفيذ.  
2. **قسّم** الواجهات الكبيرة (ISP)؛ تجنّب “واجهة كل شيء”.  
3. اكتب **اختبارات عقد** (Contract Tests) تُشغَّل على كل تنفيذ للواجهة.  
4. مرّر الواجهة عبر **DI**؛ لا تستخدم **Service Locator**.  
5. وثّق **الضمانات**: الاستثناءات، الحدود، التعامل مع `null`، القيود الزمنية.  
6. انتبه لـ **الإصدارات**: تغييرات الواجهة **تكسر العملاء**؛ أضف أعضاء جدد بتروٍّ أو قدّم v2.

---

## أخطاء شائعة
- واجهة منتفخة (God Interface) → أصناف مضطرة لتنفيذ ما لا تحتاجه.  
- استخدام واجهة **لبيانات فقط** (DTO) — اجعلها **record/class** بدلًا من ذلك.  
- كشف تفاصيل تنفيذية في العقد (مثل أنواع بنية قاعدة البيانات).  
- خلط **Abstraction** مع **Encapsulation**: الواجهة تصف *ما*؛ لا تكشف *كيف*.  
- الاعتماد على نوع ملموس بدل الواجهة في التواقيع → فقدان المرونة.  
- الإفراط في **Default Interface Methods**؛ قد يعقّد التوافق.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Interface** | **عقد سلوك دون حالة** | تمكّن **التجريد** و**التبديل/الاختبار** |
| [Abstract Class](abstraction.md) | عقد + سلوك/حالة مشتركة | وراثة مفردة؛ مناسب للقوالب المشتركة |
| [Composition](composition.md) | تركيب سلوك عبر واجهات | يفضَّل على الوراثة للتبديل |
| [Polymorphism](polymorphism.md) | تنفيذات مختلفة لنفس العقد | يتحقق عبر الواجهات/الوراثة |
| [Encapsulation](encapsulation.md) | إخفاء التفاصيل خلف واجهة | الواجهة تساعد على تطبيقه |

---

## ملخص الفكرة  
**Interface** = عقد واضح لما يمكن فعله.  
تعامل مع **عقود صغيرة مركّزة**، ومرّرها عبر **DI**، واكتب **اختبارات عقد**—  
تحصل على نظام **مرن**، **قابل للاستبدال**، وسهل الصيانة دون اقتران بالتنفيذ. 
