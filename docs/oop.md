# **OOP — Object-Oriented Programming**

## الترجمة الحرفية
**Object-Oriented Programming** — البرمجة كائنية التوجّه.

## English description
A programming paradigm that models software as interacting objects with state and behavior. It enables encapsulation, inheritance, and polymorphism to build modular, reusable code.

## الوصف العربي المختصر
أسلوب برمجة يمثّل البرنامج كـ «كائنات» لها بيانات (حالة) ودوال (سلوك). يسهّل تنظيم الكود وإعادة استخدامه وصيانته.

## الشرح المبسّط
- فكّر بكل جزء من برنامجك كأنه «شيء» (سيارة، طالب…).
- كل «شيء» له صفات (سرعة) وأفعال (يمشي/يُصدر صوت).
- بنستخدم مبادئ: **التغليف** (إخفاء التفاصيل)، **الوراثة** (إعادة الاستخدام)، **تعدد الأشكال** (نفس الواجهة بس سلوك مختلف).
- الهدف: كود قابل للتوسعة، أسهل للاختبار والصيانة.

## مثال كود C# قصير
```csharp
using System;

class Car
{
    // فعل افتراضي يمكن تغييره في الفئات المشتقة
    public virtual void Honk() => Console.WriteLine("Beep");
}

class ElectricCar : Car
{
    // نفس الواجهة، سلوك مختلف (Polymorphism)
    public override void Honk() => Console.WriteLine("Beep (EV)");
}

class Program
{
    static void Main()
    {
        Car c1 = new Car();
        Car c2 = new ElectricCar(); // نوع مرجعي Car لكن السلوك من ElectricCar
        c1.Honk(); // Beep
        c2.Honk(); // Beep (EV)
    }
}
```
##  مصطلحات مرتبطة/مقابلة

Encapsulation (التغليف): إخفاء التفاصيل الداخلية خلف واجهة واضحة.

Inheritance (الوراثة): فئة جديدة ترث صفات/سلوك فئة موجودة لإعادة الاستخدام.

Polymorphism (تعدد الأشكال): نفس التوقيع، سلوك مختلف حسب النوع الفعلي.

متى أستخدمه؟ ومتى لا؟

✅ استخدمه لمشاريع متوسطة/كبيرة تحتاج تنظيمًا وتوسعة على المدى الطويل.

❌ لا تفرط فيه في سكربتات صغيرة جدًا أو مهام مؤقتة.

أخطاء شائعة

حصر OOP في «الوراثة» فقط؛ الصحيح أنه يشمل مبادئ متعددة.

المبالغة بإنشاء أصناف كثيرة بلا حاجة (Over-engineering).