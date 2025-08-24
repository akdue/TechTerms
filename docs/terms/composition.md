# **Composition**

## الترجمة الحرفية  
**Composition** — **تركيب** (بناء كائن كبير من كائنات/سلوكيّات أصغر “has-a”).

## الوصف العربي المختصر  
نصمّم كائنًا بضمّ **مكوّنات مستقلة** عبر **واجهات** بدل وراثة عميقة.  
النتيجة: **مرونة** أعلى، **اختبار أسهل**، وتبديل السلوك **دون** كسر الباقي.

## الشرح المبسّط  
- العلاقة: **has-a** (السيارة **تملك** محرّكًا)، لا **is-a**.  
- نحقن سلوكيات (استراتيجيات/سياسات) كاعتمادات عبر **DI**.  
- يمكن **تجميع** السلوك (Decorator/Strategy) بدل توارثه.

## تشبيه  
كمبيوتر مكتبي: تبدّل بطاقة الرسوميات أو الذاكرة **بدون** تغيير اللوحة الأم.  
هذا هو **التركيب**: أجزاء قابلة للفكّ والاستبدال تعمل معًا عبر منافذ قياسية.

---

## مثال C# — Strategy + Decorator (تركيب سلوك بدل وراثة)

```csharp
// س1) "استراتيجية" التسعير — سلوك قابل للاستبدال
public interface IPricingStrategy { decimal Final(decimal subtotal); }

public class StandardPricing : IPricingStrategy
{
    public decimal Final(decimal s) => s; // بدون خصم
}

public class HappyHourPricing : IPricingStrategy
{
    public decimal Final(decimal s) => decimal.Round(s * 0.80m, 2); // خصم 20%
}

// س2) "مُخطِر" قابل للتمديد عبر مُزيّن (Decorator)
public interface INotifier { void Send(string to, string msg); }

public class EmailNotifier : INotifier
{
    public void Send(string to, string msg) => Console.WriteLine($"Email → {to}: {msg}");
}

public class SlackNotifierDecorator : INotifier
{
    private readonly INotifier _inner;
    public SlackNotifierDecorator(INotifier inner) => _inner = inner;

    public void Send(string to, string msg)
    {
        _inner.Send(to, msg);                             // أعِد استخدام السلوك المركّب
        Console.WriteLine($"Slack → {to}: {msg}");        // أضِف سلوكًا جديدًا
    }
}

// س3) خدمة أعمال "تُرَكِّب" الاستراتيجية + المُخطِر عبر DI
public class OrderService
{
    private readonly IPricingStrategy _pricing;           // تركيب: has-a
    private readonly INotifier _notifier;                 // تركيب: has-a
    public OrderService(IPricingStrategy pricing, INotifier notifier)
    { _pricing = pricing; _notifier = notifier; }

    public decimal Checkout(string user, decimal subtotal)
    {
        var total = _pricing.Final(subtotal);             // سلوك مركّب
        _notifier.Send(user, $"Your total = {total:C}");
        return total;
    }
}

class Program
{
    static void Main()
    {
        IPricingStrategy pricing = new HappyHourPricing();                      // بدّلها في أي وقت
        INotifier notifier = new SlackNotifierDecorator(new EmailNotifier());   // زيّن بدون وراثة
        var svc = new OrderService(pricing, notifier);

        var total = svc.Checkout("user@example.com", 100m);
        Console.WriteLine(total); // 80.00
    }
}
```

> **لماذا تركيب؟**  
> لو استخدمنا وراثة مثل `HappyHourOrderService : OrderService` لتغيير التسعير، سنُنشىء **تفريعات كثيرة** لكل حالة.  
> بالتركيب نبدّل **جزءًا واحدًا** (`IPricingStrategy`) ونحصل على مرونة أعلى.

---

## خطوات عملية لتطبيق التركيب
1. حدّد **السلوكيات المتغيّرة** (تسعير، إشعار، تخزين…) وضع لها **Interfaces**.  
2. حقّنها عبر **Constructor DI** بدل `new` داخل الصنف.  
3. لا تجعل الصنف يعرف **نوعًا ملموسًا**؛ تعامل مع **عقد** فقط.  
4. استخدم **Decorator** لإضافة ميزة بدون تعديل الأصل، و**Strategy** لتبديل خوارزمية.  
5. وفّر **اختبارات عقد** (Contract Tests) لضمان توافق البدائل.  
6. اجعل الحدود واضحة: كل مكوّن **مسؤولية واحدة** (قابلية استبدال عالية).

---

## أخطاء شائعة
- اختيار الوراثة لمجرّد **إعادة استخدام كود** → تسلسل هشّ؛ ركّب بدلًا من ذلك.  
- كائن تجميعي **منتفخ** (God Object) يجمع كل شيء.  
- تعرية تفاصيل المكوّنات بدل إخفائها خلف واجهات.  
- إنشاء الاعتمادات يدويًا داخل الصنف (`new`) → تقييد الاختبار والتبديل.  
- حلقات اعتماد بين المكوّنات (A يعتمد على B وB على A).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Composition** | **بناء سلوك من مكوّنات مستقلّة (has-a)** | يفضَّل على الوراثة للتبديل والاختبار |
| [Inheritance](inheritance.md) | تخصص عبر اشتقاق (is-a) | قوي لكن يُستخدم بحذر؛ قد يقيّد |
| [Interface](interface.md) | عقد للسلوك | يمكّن التركيب ودي/اختبار |
| Strategy Pattern | تبديل خوارزمية وقت التشغيل | تطبيق عملي للتركيب |
| Decorator Pattern | إضافة ميزة بدون تعديل الأصل | تجميع سلوك بشكل طبقات |

---

## ملخص الفكرة  
**التركيب** = “اجمع قطعًا صغيرة بواجهات واضحة لتبني سلوكًا كبيرًا”.  
افصل **ما يتغيّر** في مكوّنات قابلة للاستبدال، ومرّرها عبر **DI**،  
وستحصل على نظام **مرن**، **قابل للاختبار**، وسهل التطوير بلا فروع وراثة متشابكة.
