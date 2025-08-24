# **Encapsulation**

## الترجمة الحرفية  
**Encapsulation** — **تغليف/احتواء المعلومات** (إخفاء التفاصيل الداخلية وكشف واجهة آمنة).

## الوصف العربي المختصر  
مبدأ من مبادئ OOP: **نخفي الحالة** وراء واجهة مضبوطة (**Properties/Methods**)،  
ونمنع التلاعب المباشر ببيانات الكائن. نحافظ على **ثوابت** (Invariants) ونقدّم **عمليات صالحة فقط**.

## الشرح المبسّط  
- لا نكشف **الحقول** مباشرة؛ نوفّر **واجهات** تضبط القراءة/الكتابة والتحقق.  
- نسمح بما يلزم فقط (**مبدأ أقل امتياز**).  
- يفصل “*ما* يمكن فعله” عن “*كيف* يُنفّذ داخليًا”.

## تشبيه  
بطاقة بنك: لك زر “سحب/إيداع” فقط. لا تصل لقاعدة البيانات أو تعدّل الرصيد يدويًا.  
التطبيق يفرض الحدود: لا سحب أكثر من الرصيد، لا مبالغ سالبة.

---

## مثال كود C# — حساب بنك مع تغليف صارم (تحقّق + ثوابت)

```csharp
// .NET 8/9
using System;

public class BankAccount
{
    // الحالة الداخلية مخفيّة
    private decimal _balance;
    private readonly string _iban;

    // واجهة قراءة فقط
    public decimal Balance => _balance;
    public string Iban => _iban;

    // التغليف يبدأ من المُنشئ: تحقق أولي
    public BankAccount(string iban, decimal openingBalance = 0m)
    {
        if (string.IsNullOrWhiteSpace(iban)) throw new ArgumentException("IBAN required");
        if (openingBalance < 0) throw new ArgumentOutOfRangeException(nameof(openingBalance));
        _iban = iban;
        _balance = openingBalance;
    }

    // عمليات آمنة فقط (لا Setter للرصيد)
    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        _balance += amount;
    }

    public bool TryWithdraw(decimal amount, out string? error)
    {
        if (amount <= 0) { error = "amount_must_be_positive"; return false; }
        if (amount > _balance) { error = "insufficient_funds"; return false; }
        _balance -= amount;
        error = null;
        return true;
    }
}

class Demo
{
    static void Main()
    {
        var acc = new BankAccount("JO71CBJO0000000000000131000302", 1000m);
        acc.Deposit(250m);
        Console.WriteLine(acc.Balance); // 1250

        if (!acc.TryWithdraw(2000m, out var err))
            Console.WriteLine(err);     // insufficient_funds

        // acc._balance = 1;          // ✗ غير مسموح (مخفي)
        // acc.Balance = 0;           // ✗ لا Setter
    }
}
```

**الفكرة:** لا طريقة لكسر القواعد. كل تعديل يمر عبر **واجهات موثوقة** تتحقق من الصلاحية.

---

## خطوات عملية لتطبيق التغليف
- ابدأ بـ **حقول خاصة** (`private`) ووفّر **Properties**/طرق للتحكم.  
- استخدم **واجهات** لإخفاء التفاصيل وتمكين الاستبدال.  
- اجعل **المنشئ** وطرق التعديل تتحقق من الثوابت (Validation/Invariants).  
- فضّل **`private set`** و**`init`** للخصائص التي تُكتب مرة.  
- قلّل مساحة السطح العامة: **لا تكشف** ما لا تحتاجه (`internal`/`private`).  
- عند الحاجة للتهيئة المعقّدة، استخدم **Factory/Builder** بدل Setters مفتوحة.

---

## أخطاء شائعة
- جعل الحقول/الخصائص **public** ثم محاولة ضبطها يدويًا في كل مكان.  
- Getter/Setter يمرانر دون **تحقق** → كسر الثوابت بسهولة.  
- كشف **نموذج داخلي** بدل **DTO/واجهة** → اقتران قوي.  
- استخدام **Service Locator** داخل الكائن بدل تمرير التبعيات (يتعارض مع التغليف/DI).  
- خلط مسؤوليات كثيرة في صنف واحد (كسر **التماسك**).

---

## جدول مقارنة مختصر

| المفهوم                                 | الغرض الرئيسي                             | ملاحظات مهمة                                     |
| --------------------------------------- | ----------------------------------------- | ------------------------------------------------ |
| **Encapsulation**                       | **إخفاء الحالة خلف واجهة آمنة**           | يحمي الثوابت ويبسّط الاستبدال/الاختبار            |
| [Abstraction](abstraction.md)           | وصف *ما* دون *كيف*                        | التغليف يطبّق التجريد عمليًا عبر الواجهات          |
| [Inheritance](inheritance.md)           | إعادة استخدام عبر توريث                   | قد يكشف تفاصيل؛ استخدمه بحذر مع التغليف          |
| Runtime [Polymorphism](polymorphism.md) | **نفس الواجهة → تنفيذ مختلف وقت التشغيل** | `virtual/abstract` + `override` أو **Interface** |

---

## ملخص الفكرة  
**Encapsulation** = حماية الحالة خلف واجهة تُفرض فيها القواعد.  
استخدم **حقول خاصة + واجهات + تحقق**، وقلّل ما تكشفه—تحصل على كود **أكثر أمانًا**، **أسهل اختبارًا**، و**أقل اقترانًا**. 
