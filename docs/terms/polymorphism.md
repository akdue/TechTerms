# **Polymorphism**

## الترجمة الحرفية  
**Polymorphism** — **تعدّد الأشكال** (نفس الواجهة، سلوك مختلف حسب النوع الفعلي).

## الوصف العربي المختصر  
آلية في OOP تسمح بإستدعاء **نفس الدالة/الواجهة** على كائنات متعددة  
لتُنفّذ **بطريقة مختلفة** وفق النوع الحقيقي للكائن وقت التشغيل (**Runtime Dispatch**).

## الشرح المبسّط  
- نكتب كودًا يعتمد على **نوع أساسي** (أو **Interface**).  
- نمرّر له **أنواع مشتقّة**؛ تُنفَّذ الدوال **المُعاد تعريفها** `override` تلقائيًا.  
- يوجد أيضًا **تعدّد أشكال تجميعي/وقت ترجمة** مثل **Overloading** و**Generics**.

## تشبيه  
زر “تشغيل” واحد: يشغّل **فيديو** على التلفاز، **أغنية** على المشغّل، و**عرض** على البروجكتر.  
الزر موحّد، لكن **التنفيذ** يتبدّل حسب الجهاز.

---

## مثال C# — تعدّد أشكال وقت التشغيل (توريث + override)

```csharp
// .NET 8/9
using System;
using System.Collections.Generic;
using System.Linq;

abstract class Shape
{
    public abstract double Area(); // واجهة موحّدة لحساب المساحة
}

class Circle : Shape
{
    public double Radius { get; }
    public Circle(double r) => Radius = r;
    public override double Area() => Math.PI * Radius * Radius;
}

class Rectangle : Shape
{
    public double W { get; }  // العرض
    public double H { get; }  // الارتفاع
    public Rectangle(double w, double h) { W = w; H = h; }
    public override double Area() => W * H;
}

class Program
{
    static void Main()
    {
        // مرجع من النوع الأساسي لعدة أنواع فعلية
        List<Shape> shapes = new() { new Circle(2), new Rectangle(3, 4) };

        // Runtime Dispatch: كل عنصر ينفّذ Area الخاصة به
        double total = shapes.Sum(s => s.Area());
        Console.WriteLine($"Total area = {total:F2}");

        // ✗ مضاد نمط: فحص النوع بدل الاعتماد على polymorphism
        // foreach (var s in shapes)
        //   if (s is Circle c) ... else if (s is Rectangle r) ...
    }
}
```

**الفكرة:** استدعاء `Area()` على قائمة `Shape` يوجّه تلقائيًا إلى التنفيذ المطابق لنوع الكائن.

---

## مثال C# — تعدّد أشكال عبر **Interface** (تبديل سلوك بدون وراثة)

```csharp
public interface INotifier { void Send(string to, string msg); }

public class EmailNotifier : INotifier
{
    public void Send(string to, string msg) => Console.WriteLine($"Email → {to}: {msg}");
}

public class SmsNotifier : INotifier
{
    public void Send(string to, string msg) => Console.WriteLine($"SMS → {to}: {msg}");
}

public class AlertsService
{
    private readonly INotifier _notifier;               // اعتماد على واجهة
    public AlertsService(INotifier notifier) => _notifier = notifier;

    public void Critical(string user) => _notifier.Send(user, "CRITICAL ALERT!");
}
```

> **الاستفادة:** استبدل `EmailNotifier` بـ `SmsNotifier` دون تعديل `AlertsService`.

---

## مثال قصير — **Overloading/Generics** (تعدّد أشكال وقت ترجمة)

```csharp
// Overloading: نفس الاسم، توقيعات مختلفة (Compile-time)
void Print(int x)    => Console.WriteLine($"int:{x}");
void Print(string s) => Console.WriteLine($"string:{s}");

// Generics: سلوك عام لأنواع متعددة بقيد واجهة
T Max<T>(T a, T b) where T : IComparable<T> => a.CompareTo(b) >= 0 ? a : b;
```

> **ملاحظة:** هذا **يختلف** عن الـ Runtime Polymorphism (لا يحتاج `virtual/override`).

---

## خطوات عملية لاستخدام تعدّد الأشكال بذكاء
1. صمّم حول **واجهات/أنواع أساسية**؛ لا تربط الكود بـ **أنواع ملموسة**.  
2. استخدم `virtual/abstract` و`override` للسلوك القابل للتبديل.  
3. فضّل **Interfaces** عندما تريد **تركيبًا** لا **توريثًا** عميقًا.  
4. حقّق **DI** لتمرير التنفيذ المناسب (Email/SMS/…)، وسهّل الاختبار.  
5. تجنّب فحوص النوع المتكرّرة (`is/switch`)—اشرح السلوك داخل الكائن نفسه.  
6. إن احتجت **تخصيصًا موضعيًا**، استعمل **استراتيجية** (Strategy Pattern).

---

## أخطاء شائعة
- سلاسل `if (obj is TypeA) … else if …` بدل الاستفادة من `override`.  
- **توريث عميق** يقيّد التصميم—فضّل **Composition**.  
- جعل الدوال `virtual` دون حاجة (سطح تغيير كبير).  
- كسر **LSP** (استبدال ليسليسكوف): صنف مشتق لا يحترم عقد الأساسي.  
- خلط **Overloading** مع **Overriding** (الأول ترجمي، الثاني وقت تشغيل).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Runtime Polymorphism** | **نفس الواجهة → تنفيذ مختلف وقت التشغيل** | `virtual/abstract` + `override` أو **Interface** |
| Overriding | استبدال تنفيذ الأساس في المشتق | يحتاج `virtual/abstract` في الأساس |
| Overloading | نفس الاسم بتوقيعات مختلفة | قرار المترجم عند **الترجمة** |
| Interface Polymorphism | تبديل سلوك عبر عقد موحّد | يقلّل اقتران، يسهّل الاختبار |
| Generics | خوارزميات عامة لأنواع مختلفة | تعدّد أشكال **ترجمي** مع قيود |

---

## ملخص الفكرة  
**Polymorphism** يمكّنك من كتابة كود **عام** يعمل مع **أنواع متعددة** دون فروع شرطية.  
اعتمد على **واجهات** و**override**، ومرّر التنفيذ بـ **DI**—تحصل على تصميم **مرن**، **قابل للاختبار**، وسهل التطوير. 
