# **Inheritance**

## الترجمة الحرفية  
**Inheritance** — **الوراثة** (اشتقاق صنف جديد من صنف أساسي لإعادة الاستخدام والتوسعة).

## الوصف العربي المختصر  
آلية OOP تسمح للصنف **المشتق** أن يرث **خصائص** و**سلوك** الصنف **الأساسي**،  
ويضيف أو يبدّل التنفيذ عبر `override`. تدعم **التخصص** و**إعادة الاستخدام**.

## الشرح المبسّط  
- نفكّر بعلاقة **"is-a"**: المدير **موظّف**، الشاحنة **مركبة**.  
- الصنف الأساسي يعرّف **عقدًا** وسلوكًا افتراضيًا، والمشتق **يخصّص**.  
- استخدم **`abstract`** لفرض تنفيذ في الأصناف المشتقة، و**`sealed`** لمنع مزيد من الوراثة.

## تشبيه  
رخصة قيادة عامة (الأساس) + تصاريح إضافية للشاحنات أو الحافلات (المشتقات).

---

## مثال C# — وراثة مع `abstract/virtual/override/sealed` + استدعاء `base`

```csharp
// .NET 8/9
using System;
using System.Collections.Generic;
using System.Linq;

// الأساس: عقد مشترك لحساب الأجر
public abstract class Employee
{
    public string Name { get; }
    protected Employee(string name) => Name = name;

    // يجب على المشتقات تنفيذها
    public abstract decimal PayFor(int hours);

    // سلوك مشترك قابل للتجاوز
    public virtual string Describe() => $"Employee: {Name}";
}

// موظف بالساعة
public class HourlyEmployee : Employee
{
    private readonly decimal _rate;
    public HourlyEmployee(string name, decimal hourlyRate) : base(name) => _rate = hourlyRate;

    public override decimal PayFor(int hours)
    {
        // وقت إضافي بسيط بعد 8 ساعات/يوم (مثال)
        int overtime = Math.Max(0, hours - 8);
        return _rate * (hours + overtime * 0.5m);
    }

    public override string Describe() => base.Describe() + " (Hourly)";
}

// موظف براتب شهري
public class SalariedEmployee : Employee
{
    protected readonly decimal Monthly; // متاح للمشتقات فقط
    public SalariedEmployee(string name, decimal monthly) : base(name) => Monthly = monthly;

    public override decimal PayFor(int hours) => Monthly / 22m; // تقدير يومي (مثال)
    public override string Describe() => base.Describe() + " (Salaried)";
}

// مدير يضيف بدلات، ويمنع مزيدًا من التخصيص
public sealed class Manager : SalariedEmployee
{
    private readonly decimal _allowance;
    public Manager(string name, decimal monthly, decimal allowance) : base(name, monthly) => _allowance = allowance;

    public override decimal PayFor(int hours)
    {
        var basePay = base.PayFor(hours); // استخدام منطق الأساس
        return basePay + _allowance;
    }

    public override string Describe() => base.Describe() + " + Allowance";
}

class Program
{
    static void Main()
    {
        List<Employee> team = new()
        {
            new HourlyEmployee("Omar", 10m),
            new SalariedEmployee("Lina", 2200m),
            new Manager("Sara", 3000m, 150m)
        };

        int hoursToday = 9;
        var total = team.Sum(e => e.PayFor(hoursToday));

        foreach (var e in team)
            Console.WriteLine($"{e.Describe()} → Pay={e.PayFor(hoursToday):0.00}");

        Console.WriteLine($"Total payroll today = {total:0.00}");
    }
}
```

**الفكرة:** نتعامل مع جميع العناصر كـ `Employee`، ويُختار التنفيذ الصحيح **وقت التشغيل**.  
استخدمنا `base` لضم منطق الأساس، و`sealed` لمنع اشتقاق إضافي من `Manager`.

---

## متى أستخدمه؟ ومتى أتجنّبه؟
- **استخدم الوراثة** عندما توجد علاقة **is-a** واضحة وعقد مشترك قوي.  
- **تجنّبها** عندما يكون الهدف **مجرّد مشاركة كود** فقط—فضّل **التركيب (Composition)** أو **Interfaces**.

---

## خطوات عملية
1. حدّد **العقد** في الأساس (`abstract`/`virtual`) بدقة.  
2. حافظ على **LSP**: المشتق لا يكسر توقّعات الأساس.  
3. استدعِ `base` عند الحاجة لتفادي تكرار السلوك.  
4. اقفل التسلسل بـ **`sealed`** عند اكتمال التخصيص.  
5. ابقِ التسلسل **ضحلًا**؛ عمق كبير يزيد الهشاشة.

---

## أخطاء شائعة
- وراثة بلا علاقة **is-a** → تصميم هش.  
- **توريث عميق** بدل التركيب → صعوبة صيانة.  
- تغيير دلالات الأساس (كسر **LSP**).  
- نسيان `virtual/override` أو استخدام `new` لإخفاء الأعضاء (يلبّس السلوك).  
- كشف حقول الأساس للعموم بدل حمايتها (`private/protected`).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Inheritance** | **تخصص وإعادة استخدام عبر اشتقاق** | `abstract/virtual/override/sealed` |
| [Composition](composition.md) | بناء كائن من أجزاء | غالبًا أكثر مرونة من الوراثة |
| Interface | عقد سلوك بدون حالة | يسهّل التبديل والاختبار |
| [Polymorphism](polymorphism.md) | نفس الواجهة بسلوك مختلف | يتحقق عبر الوراثة أو الواجهات |
| [Encapsulation](encapsulation.md) | إخفاء الحالة خلف واجهة | الوراثة يجب ألا تكسر التغليف |

---

## ملخص الفكرة  
**الوراثة** تمكّنك من **تخصيص** سلوك مشترك عبر تسلسل منطقي.  
استخدمها بحذر لعلاقات **is-a**، وحافظ على **LSP**، وفضّل **التركيب** عندما يكون أنسب—  
تحصل على تصميم **قابل للتوسعة** دون تعقيد غير ضروري.
