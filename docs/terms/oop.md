# **OOP**

## التسمية المعتمدة (EN ↔ AR)
**Object-Oriented Programming (OOP)** — **البرمجة كائنيّة التوجّه**

## تعريف موجز
A paradigm that builds software from **objects** combining **state** (data) and **behavior** (methods).  
It emphasizes **encapsulation**, **inheritance**, **polymorphism**, and **abstraction** to improve modularity, reuse, and maintainability.

أسلوب برمجة يكوّن برنامجك من **كائنات** لها بيانات ودوال.  
الركائز الأربع: **التغليف**، **الوراثة**، **تعدد الأشكال**، **التجريد**.  
النتيجة: كود **منظّم**، **قابل لإعادة الاستخدام**، وسهل **الصيانة**.

---

## لماذا OOP؟ (بسيطة ولذيذة)
- تمثّل العالم الحقيقي: *عميل، طلب، فاتورة* يصبح لكلٍّ كائن.  
- تقسيم المشكلة لقطع صغيرة مفهومة (Classes).  
- تبديل القطع لاحقًا بلا فوضى (Interfaces & Polymorphism).  
- اختبار أسهل لأن كل جزء مسؤول عن شيء محدّد.

---

## الركائز الأربع… بسرعة وبدون فذلكة
- **Encapsulation | التغليف:** نخفي التفاصيل الداخلية ونكشف واجهة آمنة.  
- **Inheritance | الوراثة:** نعيد استخدام سلوك الصنف الأساسي ونوسّعه.  
- **Polymorphism | تعدد الأشكال:** نفس الدالة تُنفَّذ بطرق مختلفة حسب النوع الفعلي.  
- **Abstraction | التجريد:** نصف *ما* يجب فعله قبل الدخول في *كيف* يُفعل.

> تشبيه: كل السيارات عندها “دوسة بوري” (واجهة موحّدة)، لكن صوتها يختلف من سيارة لأخرى (تعدد الأشكال). عدّاد “كم مرّة زمّرنا” داخل السيارة، وليس للشارع أن يعبث به (تغليف).

---

## مثال من الكود
```csharp
using System;

// Abstraction: تعريف "ما ينبغي" على الحيوان فعله دون تفاصيل التنفيذ
abstract class Animal
{
    public string Name { get; }                 // خاصية للقراءة فقط
    protected Animal(string name) => Name = name;

    // Polymorphism: قابل للاستبدال في الأصناف المشتقة
    public virtual void Speak() => Console.WriteLine($"{Name}: ???");
}

// Inheritance: Cat ترث من Animal
class Cat : Animal
{
    // Encapsulation: حالة داخلية خاصة لا تُعدّل مباشرة من الخارج
    private int _meowCount;

    public Cat(string name) : base(name) { }

    // Polymorphism: تنفيذ مختلف لنفس الواجهة
    public override void Speak()
    {
        _meowCount++;
        Console.WriteLine($"{Name}: Meow!");
    }

    // كشف آمن للقراءة فقط
    public int MeowCount => _meowCount;
}

class Dog : Animal
{
    public Dog(string name) : base(name) { }
    public override void Speak() => Console.WriteLine($"{Name}: Woof!");
}

class Program
{
    static void Main()
    {
        Animal a1 = new Cat("Milo");   // مرجع من النوع الأساسي لنوع فعلي مشتق
        Animal a2 = new Dog("Rex");

        a1.Speak(); // يحدّد التنفيذ وقت التشغيل (runtime dispatch)
        a2.Speak();

        Console.WriteLine(((Cat)a1).MeowCount); // 1 ← مثال على تغليف حالة داخلية
    }
}
```

---

## مصطلحات مرتبطة وتمييزات سريعة
- **Class vs Object:** الصنف قالب؛ الكائن نسخة حيّة منه في الذاكرة.  
- **Interface vs Abstract Class:** الواجهة *عقد للسلوك فقط*؛ المجرّدة قد تحوي سلوكًا افتراضيًا وحالة محمية.  
- **Composition vs Inheritance:** التركيب يضم كائنات داخل كائن آخر (أكثر مرونة)؛ الوراثة تورّث الواجهة والسلوك (قد تُقيّد).  
- **OOP vs Procedural:** الكائنية تبني حول الكائنات؛ الإجرائية تبني حول الدوال والخطوات.

---

## متى أستخدمه؟ ومتى أتجنّبه؟
- **استخدم OOP** عند نمذجة كيانات مجال واضحة وتحتاج للتوسع بمرور الوقت.  
- **تجنّب الإفراط** به في سكربتات صغيرة/مهام حسابية خطيّة؛ قد تكفي الإجرائية أو الوظيفية.

---

## أخطاء شائعة
- جعل معظم الأعضاء **public** → كسر التغليف وصعوبة الاختبار.  
- الاعتماد المفرط على **الوراثة العميقة** بدل **التركيب** → تعقيد وتشابك هرمي.

---

## خلاصة
OOP تنظّم برنامجك حول كائنات واضحة الحدود، وتمنحك مرونة التطوير والتبديل وإعادة الاستخدام.  
المفاتيح: **Encapsulation، Inheritance، Polymorphism، Abstraction**.

---

## سؤال تثبيت (إجابة مخفية)
عندك `Animal` وتَرِث منه `Cat` و`Dog` ويملكان `Speak()` بسلوك مختلف. ما المفهوم المعبّر؟
<details>
  <summary><strong>إظهار الإجابة</strong></summary>
  **Polymorphism — تعدد الأشكال.**  
  نفس الواجهة `Speak()` لكن التنفيذ يختلف حسب النوع الفعلي (`Cat`/`Dog`).
</details>
