# **SDLC — Software Development Life Cycle**

## الترجمة الحرفية  
**Software Development Life Cycle (SDLC)** — **دورة حياة تطوير البرمجيات**

## الوصف العربي المختصر  
إطار عمل يصف **مراحل** تطوير النظام من الفكرة حتى الصيانة:  
**تخطيط → متطلبات → تصميم → بناء → اختبار → نشر → تشغيل/تحسين**،  
مع **حواجز جودة**، وثائق، وتتبع تغييرات.

## الشرح المبسّط  
- هدف SDLC: **تقليل المخاطر** و**رفع الجودة** عبر مراحل واضحة ومسؤوليات محدّدة.  
- يُستخدم مع منهجيات مختلفة: **Waterfall** (تسلسل خطّي)، **Agile** (تكرارات Sprint)، **DevOps** (أتمتة ونشر مستمر).  
- مخرجات أساسية: **وثائق متطلبات (SRS)**، **تصميم معماري**، **كود واختبارات**، **حِزم/إصدارات**، **مقاييس تشغيل**.

## تشبيه  
بناء منزل: دراسة جدوى (تخطيط)، مخططات (تصميم)، بناء (تنفيذ)، تفتيش (اختبار)، تسليم وسكن (نشر)، صيانة دورية (تشغيل).

---

## مثال C# عملي — ربط مراحل SDLC بكود صغير

> سنُظهر:  
> 1) **متطلبات**: “احسب السعر بعد خصم، ولا يقل الناتج عن صفر”.  
> 2) **اختبار (TDD)**: نكتب اختبارًا قبل التنفيذ.  
> 3) **بناء**: نُنفّذ المنطق مع توثيق.  
> 4) **تشغيل**: Log بسيط و“رقم إصدار” لتتبّع النشر.

### 1) اختبار وحدات (xUnit)
```csharp
// dotnet add package xunit
// dotnet add package xunit.runner.visualstudio
// dotnet test
using Xunit;

public class PriceCalculatorTests
{
    [Theory]
    [InlineData(100, 0.10, 90)]
    [InlineData(50,  0.00, 50)]
    [InlineData(10,  0.50, 5)]
    [InlineData(10,  2.00, 0)] // لا يقل عن صفر
    public void FinalPrice_Is_Capped_At_Zero(decimal price, double discount, decimal expected)
    {
        var calc = new PriceCalculator();
        Assert.Equal(expected, calc.Final(price, discount));
    }
}
```

### 2) التنفيذ
```csharp
using System;
using System.Diagnostics.CodeAnalysis;

/// <summary>
/// Req#PRC-001: احسب السعر بعد خصم نسبي (0..1) ولا يقل الناتج عن صفر.
/// </summary>
public class PriceCalculator
{
    /// <param name="price">السعر قبل الخصم (>=0)</param>
    /// <param name="discount">نسبة الخصم بين 0 و1 (قد تتجاوز 1 خطأ من المصدر)</param>
    public decimal Final(decimal price, double discount)
    {
        if (price < 0) throw new ArgumentOutOfRangeException(nameof(price));
        var d = Math.Clamp(discount, 0d, 1d);
        var result = price - (price * (decimal)d);
        return result < 0 ? 0 : decimal.Round(result, 2);
    }
}
```

### 3) نقطة تشغيل صغيرة (Logging + Version)
```csharp
using System;

class Program
{
    static void Main()
    {
        var version = typeof(Program).Assembly.GetName().Version?.ToString() ?? "0.0.0";
        Console.WriteLine($"App Version: {version}  (Deployed: {DateTime.UtcNow:O})");

        var calc = new PriceCalculator();
        var final = calc.Final(100m, 0.15);
        Console.WriteLine($"Final price: {final}"); // مراقبة سريعة للوظيفة الأساسية
    }
}
```

**الفكرة:**  
- **المتطلبات** موثّقة داخل التعليق واسم الاختبار.  
- **الاختبار** يثبت السلوك ويمنع الانحراف.  
- **التنفيذ** يحقّق الشرط ويعالج الحواف.  
- **التشغيل** يطبع الإصدار/الزمن لتتبّع النشر (Release Traceability).

---

## خطوات عملية لتطبيق SDLC
1. **التخطيط**: هدف العمل، أصحاب المصلحة، المخاطر، الميزانية، KPI.  
2. **المتطلبات (SRS)**: وظيفية/لا-وظيفية (أداء، أمان، توفّر)، قبول (Acceptance).  
3. **التصميم**: معماريّة (C4)، قرارات ADR، نماذج بيانات، واجهات.  
4. **التنفيذ**: معايير كود، مراجعات (PRs)، أتمتة بناء، تحليل ثابت (Analyzers).  
5. **الاختبار**: وحدات/تكامل/قبول/أمن/أداء، تغطية و“بوابات جودة”.  
6. **النشر**: تعليب (Packages/Containers)، بيئات `dev/stage/prod`، هويات سرّية آمنة.  
7. **التشغيل والمراقبة**: Logs/Metrics/Tracing، تنبيهات، إدارة حوادث، خطة رجوع.  
8. **التحسين**: تعلّم من المقاييس والأخطاء، إدارة الإصدارات و**الديون التقنية**.

---

## أخطاء شائعة
- بدء الكود قبل تعريف **المتطلبات ومعايير القبول**.  
- تجاهل **الاختبارات** أو جعلها هشة غير حتمية.  
- نشر يدوي بلا **أتمتة/بوابات** (يؤدي لأخطاء متكرّرة).  
- خلط بيئات وأسرار الإنتاج مع التطوير.  
- غياب **قياس** بعد النشر (لا Telemetry → لا تحسين فعلي).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **SDLC** | **إطار مراحل تطوير حتى الصيانة** | قابل للتطبيق مع Waterfall/Agile/DevOps |
| [Waterfall](waterfall.md) | تسلسل خطّي بمراحل ثابتة | واضح العقود؛ صعب التغيير المتأخر |
| [Agile](agile.md) | تكرارات قصيرة وتغذية راجعة | متطلبات متطوّرة؛ أولوية للقيمة |
| [DevOps](devops.md) | أتمتة بناء/اختبار/نشر ومراقبة | CI/CD، قابلية نشر متكرّر |
| [QA](qa.md) | ضمان الجودة عبر اختبارات وجوْدة | بوابات جودة، تغطية، فحوص أمان/أداء |

---

## ملخص الفكرة  
**SDLC** ينظّم رحلة البرمجية من الفكرة إلى التشغيل.  
وثّق المتطلبات، صمّم بحدود واضحة، ابنِ باختبارات، انشر بأتمتة، وراقب—  
ستحصل على **جودة أعلى** و**مخاطر أقل** و**سرعة تحسين** مستمرة.
