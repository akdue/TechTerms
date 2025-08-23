# **Waterfall**

## الترجمة الحرفية  
**Waterfall Model** — **نموذج الشلّال** (تسلسل مراحل تطوير ثابتة مع حواجز اعتماد).

## الوصف العربي المختصر  
منهجية **تتابعية خطّية** في SDLC:  
**المتطلبات → التصميم → التنفيذ → الاختبار → النشر → الصيانة**.  
لا ننتقل لمرحلة إلا بعد **اكتمال واعتماد** المرحلة السابقة (Phase Gate).

## الشرح المبسّط  
- كل مرحلة تسلّم **مخرجات ووثائق** للمرحلة التالية.  
- تغييرات المتطلبات **متأخرة = مكلفة**؛ لذا التركيز على التحليل المسبق.  
- مناسب للمشاريع **الثابتة المتطلبات** و/أو ذات **امتثال صارم** (عقود/لوائح).

## تشبيه  
بناء **جسر**: تصاريح ودراسات ثم خرائط، ثم صبّ خرسانة، ثم فحص تحمّل، ثم افتتاح.  
إن اكتشفت خطأً بعد الصبّ، التكلفة عالية جدًا للعودة.

---

## مثال كود C# — تمثيل **Phase Gates** مع قوائم تحقق واعتماد

```csharp
// .NET 8/9
using System;
using System.Collections.Generic;

enum Phase { Requirements, Design, Implementation, Verification, Deployment, Maintenance }

class PhaseGate
{
    private readonly Dictionary<Phase, bool> _approved = new();
    public Phase Current { get; private set; } = Phase.Requirements;

    public void Approve(Phase p, string approver, string note = "")
    {
        if (p != Current) throw new InvalidOperationException($"لا يمكنك اعتماد {p}؛ المرحلة الحالية {Current}.");
        Console.WriteLine($"✔ Approved {p} by {approver}. {note}");
        _approved[p] = true;
    }

    public void Next()
    {
        if (!_approved.TryGetValue(Current, out var ok) || !ok)
            throw new InvalidOperationException($"مرحلة {Current} غير معتمدة بعد.");
        Current = Current switch
        {
            Phase.Requirements   => Phase.Design,
            Phase.Design         => Phase.Implementation,
            Phase.Implementation => Phase.Verification,
            Phase.Verification   => Phase.Deployment,
            Phase.Deployment     => Phase.Maintenance,
            _ => Phase.Maintenance
        };
        Console.WriteLine($"→ moved to {Current}");
    }
}

class Program
{
    static void Main()
    {
        var gate = new PhaseGate();

        // 1) اعتماد المتطلبات قبل الانتقال
        gate.Approve(Phase.Requirements, "PO", "SRS v1.3 signed");
        gate.Next();

        // 2) اعتماد التصميم (معماريّة وواجهات)
        gate.Approve(Phase.Design, "Architect", "C4 + ADRs complete");
        gate.Next();

        // 3) التنفيذ… ثم الاختبار… ثم النشر
        gate.Approve(Phase.Implementation, "Tech Lead");
        gate.Next();

        gate.Approve(Phase.Verification, "QA", "All test cases passed");
        gate.Next();

        gate.Approve(Phase.Deployment, "Ops", "Runbook + Rollback plan");
        gate.Next();

        Console.WriteLine($"Current Phase: {gate.Current}"); // Maintenance
    }
}
```

**الفكرة:** لا يمكن التقدّم دون **اعتماد** المرحلة. هكذا يعمل Waterfall بحواجز واضحة.

---

## خطوات عملية لتطبيق Waterfall (عندما يكون مناسبًا)
1. **توثيق متطلبات قوي (SRS):** نطاق واضح، حالات استخدام، معايير قبول، قيود/امتثال.  
2. **تصميم مفصّل:** معماريّة C4، واجهات/عقود API، نماذج بيانات، قرارات ADR.  
3. **خطّة اختبار V&V:** مصفوفة تتبّع المتطلبات ↔ حالات الاختبار (RTM)، خطط قبول المستخدم (UAT).  
4. **إدارة تغييرات صارمة:** لجنة تغيير (CCB)، تقييم الأثر، نسخ وثائق.  
5. **حواجز جودة (Phase Gates):** قوائم تحقق لكل مرحلة + توقيعات مالك المنتج/المعماري/QA/العمليات.  
6. **نشر مضبوط:** Runbook، خطة رجوع، توثيق التشغيل/الدعم.  
7. **صيانة:** إدارة عيوب، رقع أمان، إصدارات ثانوية مخطَّطة.

---

## أخطاء شائعة
- **افتراض ثبات المتطلبات** في بيئات سريعة التغيّر.  
- اندماج متأخر (Big-Bang) → مفاجآت عند الاختبار.  
- وثائق كثيرة **بدون** مراجعة قيمة أو تحديث مستمر.  
- تجاهل أصحاب المصلحة حتى مرحلة متأخرة (UAT متأخر).  
- عدم وجود **قياسات** (جودة/أداء) قبل الاعتماد.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Waterfall** | **تسلسل خطّي مع حواجز اعتماد** | مناسب لعقود/امتثال واحتياجات ثابتة |
| [Agile](agile.md) | تكرارات قصيرة وتغذية راجعة مستمرة | يتقبّل التغيير؛ قيمة مبكرة |
| [DevOps](devops.md) | أتمتة بناء/نشر/تشغيل | يكمل أي منهجية؛ يقلّل زمن الدورة |
| [SDLC](sdlc.md) | إطار المراحل العام | Waterfall أحد طرق تنفيذه |

---

## ملخص الفكرة  
**Waterfall** يفرض **تسلسلًا صارمًا** مع وثائق واعتمادات قبل كل انتقال.  
اختره عندما تكون **المتطلبات ثابتة** والامتثال عالي، وادعم كل مرحلة بـ **حواجز جودة** واضحة—  
وإلا ففكّر في **Agile/DevOps** لتقليل مخاطر التغيير المتأخر.
