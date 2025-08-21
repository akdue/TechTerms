# **OOP — Object-Oriented Programming**

## الترجمة الحرفية (EN → AR)
**Object-Oriented Programming** — **البرمجة كائنيّة التوجّه**.

## English description
A programming paradigm that models software as **objects** with **state** (data) and **behavior** (methods).  
It promotes **encapsulation**, **inheritance**, **polymorphism**, and **abstraction** to build modular, reusable, and maintainable code.

## الوصف العربي المختصر
أسلوب برمجة يمثّل البرنامج على شكل كائنات لها بيانات وأفعال.  
يهدف لتنظيم الكود وتقليل التكرار وتسهيل الصيانة والاختبار.

## الشرح المبسّط (خطوات ذكية)
- فكّر بالبرنامج كساحة لعب فيها **أشياء**: سيارة، لاعب، فاتورة.  
- كل شيء = **كائن**: يمتلك **صفات** (حالة) و**أفعال** (سلوك).  
- **التغليف**: نخفي التفاصيل الداخلية ونكشف واجهة بسيطة وآمنة.  
- **الوراثة**: نبني جديدًا فوق الموجود بدل تكرار الكود.  
- **تعدد الأشكال**: نفس الواجهة، سلوك مختلف حسب نوع الكائن.  
- **التجريد**: نركّز على “ما يفعل” قبل “كيف يفعل”.

## تشبيه بسيط
لعبة سيارات: كل سيارة لها سرعة ولون وتعرف “تزيح” و“تفرمل”.  
سيارة كهربائية وسيارة بنزين تشتركان في مفهوم “سيارة”، لكن صوت البوق مختلف.  
هذا يوضّح **الوراثة + تعدد الأشكال**، مع الحفاظ على السرعة كقيمة داخلية محمية (**التغليف**).

## مثال كود C# قصير (يلمس المفاهيم الأربعة)
```csharp
using System;

// التجريد: فئة أساسية تحدد "ما الذي يجب أن تفعله" المركبات
abstract class Vehicle
{
    public string Model { get; }                 // خاصية للقراءة فقط
    protected Vehicle(string model) => Model = model;

    // تعدد الأشكال: سلوك يمكن تغييره في الأبناء
    public virtual void Honk() => Console.WriteLine($"{Model}: Beep!");
}

// الوراثة: ElectricCar ترث من Vehicle
class ElectricCar : Vehicle
{
    private int _honkCount;                      // التغليف: إخفاء الحالة الداخلية
    public ElectricCar(string model) : base(model) { }

    // نفس الواجهة لكن سلوك مختلف (Polymorphism)
    public override void Honk()
    {
        _honkCount++;
        Console.WriteLine($"{Model}: Toot Toot!");
    }

    public int HonkCount => _honkCount;          // كشف آمن للقراءة
}

class Program
{
    static void Main()
    {
        Vehicle v = new ElectricCar("Tesla");    // مرجع عام → كائن محدد
        v.Honk();                                 // يطبع: Toot Toot!
        v.Honk();                                 // يطبع: Toot Toot!
        Console.WriteLine(((ElectricCar)v).HonkCount); // 2
    }
}
```

## مصطلحات مرتبطة/مقابلة (مع فرق بجملة)
- **Class vs Object**: الصنف هو القالب/Blueprint؛ الكائن نسخة حيّة منه في الذاكرة.  
- **Interface vs Abstract Class**: الواجهة “عقد” للسلوك فقط؛ المجرّدة قد تحمل سلوكًا افتراضيًا وحالة محمية.  
- **Composition vs Inheritance**: التركيب يضم كائنات داخل كائن آخر (أكثر مرونة)؛ الوراثة ترث واجهة وسلوك (قد تُقيِّد).  
- **OOP vs Procedural**: الكائنية تنظّم حول الكائنات؛ الإجرائية تنظّم حول الدوال والخطوات.

## متى أستخدمه؟ ومتى لا؟
- **استخدم OOP** عند نمذجة كيانات مجال واضحة (منتج، عميل، طلب) وتوسعة سلوكها بمرور الوقت.  
- **لا تعتمد عليه وحده** في سكربتات صغيرة/مهام خطية حسابية؛ قد تكفي البرمجة الإجرائية أو الوظيفية.

## أخطاء شائعة
- جعل كل شيء **public** ⇒ فقدان **التغليف** وصعوبة الضبط والاختبار.  
- الإفراط في **الوراثة** بدل **التركيب** ⇒ تشابك هرمي وتعقيد غير ضروري.

## ملخص الفكرة
**OOP** = نمذجة البرنامج ككائنات منظّمة.  
المفاتيح الأربعة: **Encapsulation, Inheritance, Polymorphism, Abstraction**.  
النتيجة: كود قابل لإعادة الاستخدام، أسهل صيانة، ونمو طبيعي مع تعقّد المشروع.

## سؤال سريع للتثبيت
لو عندك `Animal` وورثت منه `Cat` و`Dog` وكل واحد عنده `Speak()` مختلف… أي مفهوم يشرح هذا السلوك؟  
> جرّب تجاوب بكلمة واحدة: **Encapsulation** أو **Inheritance** أو **Polymorphism**.
<details>
  <summary><strong>عرض الإجابة</strong></summary>

**Polymorphism (تعدد الأشكال).**  
السبب: نفس الواجهة `Speak()` لكن السلوك يختلف حسب النوع الفعلي للكائن (Cat/Dog).

</details>