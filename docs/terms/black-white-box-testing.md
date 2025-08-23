# **Black-Box & White-Box Testing**

## الترجمة الحرفية  
**Black-Box Testing** — **اختبار بصندوق أسود** (بدون معرفة الداخل).  
**White-Box Testing** — **اختبار بصندوق أبيض** (مع معرفة الداخل).  
> (يوجد وسط بينهما: **Gray-Box** — معرفة جزئية).

## الوصف العربي المختصر  
- **Black-Box**: نختبر **المدخلات ↔ المخرجات** فقط. لا نهتم بكيفية التنفيذ.  
- **White-Box**: نختبر **المسارات الداخلية/الفروع** والتعقيد والسطر/الفرع المغطّى.  
- **Gray-Box**: نعرف بعض التفاصيل (Schema/تصميم) ونستغلها في السيناريوهات.

## الشرح المبسّط  
- الأسود مناسب لقبول الميزة، التوافق، عقود API، وسلوك المستخدم النهائي.  
- الأبيض مناسب لضمان تغطية الفروع، التعامل مع الحواف، والأمان الداخلي.  
- غالبًا نستخدمهما معًا: **سلوك صحيح خارجيًا** + **فروع داخلية مغطّاة**.

## تشبيه  
تجربة شاي جاهز:  
- **Black-Box**: تتذوّقه فقط (النتيجة).  
- **White-Box**: تدخل المطبخ، تفحص الخطوات والمقادير (المسار).  
- **Gray-Box**: معك الوصفة العامة، لكنك لا تطبخ بنفسك.

---

## مثال عملي C# — نفس الوظيفة، اختبار أسود وأبيض

> متطلب: `ValidatePassword` ترجع سبب الفشل أو “OK”.  
> الشروط: طول ≥ 8، يحوي رقمًا، يحوي محرفًا خاصًا.

```csharp
// PasswordValidator.cs
using System;
using System.Linq;

public static class PasswordValidator
{
    // White-Box: لدينا فروع واضحة نريد تغطيتها
    public static string Validate(string pwd)
    {
        if (pwd is null) throw new ArgumentNullException(nameof(pwd));
        if (pwd.Length < 8) return "too_short";
        if (!pwd.Any(char.IsDigit)) return "no_digit";
        if (!pwd.Any(ch => "!@#$%^&*?_".Contains(ch))) return "no_special";
        return "OK";
    }
}
```

### Black-Box Tests — نركّز على العقد (Input/Output)
```csharp
// PasswordValidator_BlackBoxTests.cs  (xUnit)
// dotnet add package xunit && dotnet add package xunit.runner.visualstudio
using Xunit;

public class PasswordValidator_BlackBoxTests
{
    [Theory]
    [InlineData("Abcd123!", "OK")]
    [InlineData("short1!",  "too_short")]
    [InlineData("NoDigits!", "no_digit")]
    [InlineData("NoSpecial1", "no_special")]
    public void Validate_Returns_Expected_Result(string pwd, string expected)
    {
        Assert.Equal(expected, PasswordValidator.Validate(pwd));
    }

    [Fact]
    public void Validate_Null_Throws()
    {
        Assert.Throws<ArgumentNullException>(() => PasswordValidator.Validate(null!));
    }
}
```

### White-Box Tests — نغطي الفروع/الحواف حسب معرفتنا الداخلية
```csharp
// PasswordValidator_WhiteBoxTests.cs
using Xunit;

public class PasswordValidator_WhiteBoxTests
{
    [Fact] public void TooShort_Branch_Covered()    => Assert.Equal("too_short", PasswordValidator.Validate("A1!aaaa"));
    [Fact] public void NoDigit_Branch_Covered()     => Assert.Equal("no_digit",  PasswordValidator.Validate("Abcdefg!"));
    [Fact] public void NoSpecial_Branch_Covered()   => Assert.Equal("no_special",PasswordValidator.Validate("Abcdefg1"));
    [Fact] public void AllOk_Branch_Covered()       => Assert.Equal("OK",        PasswordValidator.Validate("Abcd123!"));
}
```

**الفكرة:**  
- الأسود يثبت العقد الظاهر للمستخدم/العميل.  
- الأبيض يضمن أن **كل فرع** منطق مغطّى (يساعد مع **تغطية الفروع/الأسطر**).

---

## خطوات عملية مختصرة
- حدّد **العقد** (ماذا تتعهد الدالة/الواجهة به؟) → اختبارات **Black-Box**.  
- استخرج **الفروع والحواف** من الكود/التصميم → اختبارات **White-Box**.  
- غطِّ حالات: طبيعي، حواف، أخطاء، مدخلات عدائية.  
- قِس **التغطية** (Line/Branch/Mutation) لكن لا تجعل النسبة هدفًا بحد ذاته.  
- للأنظمة: أضف **اختبارات تكامل** و**قبول** و**أداء**.

---

## أخطاء شائعة
- الاكتفاء بـ **Black-Box**: فروع غير مغطّاة تَخْتَبئ.  
- الاكتفاء بـ **White-Box**: يغطي الكود الحالي لكنه قد يفوّت شروط العقد الفعلية.  
- خلط البيانات الاختبارية مع معلومات إنتاجية حسّاسة.  
- اختبارات هشّة تعتمد على تفاصيل تنفيذية تتغيّر كثيرًا.  
- تجاهل **اختبارات سلبية** (مدخلات Null/فارغة/خارج النطاق).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Black-Box Testing** | **التحقق من السلوك الخارجي وفق العقد** | لا تعتمد على الداخل؛ مناسبة للقبول/التكامل |
| **White-Box Testing** | **تغطية الفروع والمسارات الداخلية** | يحتاج معرفة الكود؛ يقيس Line/Branch/Path |
| Gray-Box Testing | مزيج بمعرفة جزئية | يستغل سكيمة/تصميم لتحسين التغطية |
| Property-Based | يولّد بيانات عشوائية وفق خاصية عامة | يكمل الأسود/الأبيض ويكشف حواف غير متوقعة |

---

## ملخص الفكرة  
اجمع بين **Black-Box** و**White-Box**:  
الأول يضمن **العقد والسلوك**، والثاني يضمن **تغطية المسارات**.  
أضِف **Gray-Box/Property-Based** عند الحاجة، وركّز على **حالات الحواف** وجودة الاختبارات لا النسبة فقط.
