# **Methodology**

## الترجمة الحرفية  
**Methodology** — **منهجيّة** (مجموعة مبادئ + ممارسات + إجراءات منظّمة لإنجاز العمل).

## الوصف العربي المختصر  
طريقة عمل **مقنَّنة** تحدّد **الأدوار**، **الطقوس** (اجتماعات)، **المخرجات** (Artifacts)،  
و**القياسات** لتحقيق هدف التطوير/التشغيل بثبات. أمثلة: **Agile**، **Waterfall**، **Lean**، **XP**.

## الشرح المبسّط  
- المنهجيّة = **فلسفة + قواعد** لكيف نخطّط، ننفّذ، نختبر، وننشر.  
- تحدّد **متى** و**كيف** نتخذ قرارًا، وما هي **حواجز الجودة**.  
- تختلف عن **Framework** (أداة/هيكل تطبيقي)، وعن **Process** (خطوات تفصيلية)،  
  وعن **Approach** (مقاربة مختصرة لحل مشكلة)، و**SDLC** (مراحل عامّة للدورة).

## تشبيه  
**مطبخ إيطالي** = منهجيّة (فلسفة ومبادئ).  
**وصفة البيتزا** = إطار/قالب.  
**خطوات التحضير** = عملية.  
**اختيار مكوّنات خالية من الجلوتين** = مقاربة لحالة خاصة.

---

## مثال C# — محرك بسيط يفرض قواعد منهجيّة (Agile vs Waterfall)

> **الفكرة:** نمذجة انتقال بطاقة عمل بين حالات.  
> - **Agile**: `Todo → InProgress → Review → Done` (حدّ WIP).  
> - **Waterfall**: مراحل خطيّة لا تتجاوز بوابة (Gate).

```csharp
// .NET 8/9 — نموذج مبسّط
using System;
using System.Collections.Generic;
using System.Linq;

public enum State { Todo, InProgress, Review, Done, Requirements, Design, Build, Test, Deploy }

public interface IMethodology
{
    bool CanTransition(State from, State to, BoardContext ctx);
    string Name { get; }
}

public class BoardContext
{
    public int InProgressCount { get; set; }
    public int WipLimit { get; set; } = 2; // حدّ عمل جارٍ في Agile
}

// Agile: يسمح بتفرّغ الحالة وبحد WIP
public class AgileMethodology : IMethodology
{
    private static readonly (State From, State To)[] Allowed =
    {
        (State.Todo, State.InProgress),
        (State.InProgress, State.Review),
        (State.Review, State.Done)
    };

    public string Name => "Agile";
    public bool CanTransition(State from, State to, BoardContext ctx)
    {
        if (from == State.Todo && to == State.InProgress && ctx.InProgressCount >= ctx.WipLimit) return false;
        return Allowed.Any(r => r.From == from && r.To == to);
    }
}

// Waterfall: تسلسل صارم ببوابات
public class WaterfallMethodology : IMethodology
{
    private static readonly State[] Order = { State.Requirements, State.Design, State.Build, State.Test, State.Deploy };
    public string Name => "Waterfall";
    public bool CanTransition(State from, State to, BoardContext ctx)
    {
        int i = Array.IndexOf(Order, from), j = Array.IndexOf(Order, to);
        return i >= 0 && j == i + 1; // خطوة واحدة للأمام فقط
    }
}

public class Demo
{
    static void Main()
    {
        var ctx = new BoardContext();
        IMethodology agile = new AgileMethodology();
        IMethodology wf = new WaterfallMethodology();

        // Agile
        Console.WriteLine(agile.Name);
        Console.WriteLine(agile.CanTransition(State.Todo, State.InProgress, ctx)); // True
        ctx.InProgressCount = 2;
        Console.WriteLine(agile.CanTransition(State.Todo, State.InProgress, ctx)); // False (WIP)

        // Waterfall
        Console.WriteLine(wf.Name);
        Console.WriteLine(wf.CanTransition(State.Requirements, State.Design, ctx)); // True
        Console.WriteLine(wf.CanTransition(State.Requirements, State.Build, ctx));  // False
    }
}
```

**المغزى:** المنهجيّة تفرض **قواعد حركة** و**حواجز**.  
يمكنك تبديل الاستراتيجية دون إعادة كتابة منطق الأعمال.

---

## خطوات عملية لتحديد منهجيّة مناسبة
1. **حدّد السياق:** ثبات المتطلبات؟ قيود الامتثال؟ حجم الفريق؟ الحاجة للسرعة؟  
2. **اختر المنهجيّة/المزيج:** Agile/Scrum أو Kanban أو Waterfall أو هجين (Agile + حواجز امتثال).  
3. **عرّف الأدوار والمخرجات:** مالك منتج، قائد تقني، QA؛ وثائق لازمة (SRS، ADR، خطة اختبار).  
4. **طقوس وإيقاع:** Sprint/Retro/Review أو تدفّق Kanban وحدود WIP، أو Gates في Waterfall.  
5. **تعريف جاهزية/إنجاز (DoR/DoD):** معايير دخول وخروج لكل خطوة.  
6. **قياس وتحسين:** Lead Time، Throughput، عيوب/إصدار، التزام بالمواعيد؛ حسّن دوريًا.

---

## أخطاء شائعة
- **تقليد أعمى** (Cargo-Cult): تطبيق طقوس بلا فهم المشكلة.  
- عدم **تفصيل المنهجيّة** للسياق (فريق صغير ≠ مؤسّسة كبيرة).  
- طقوس كثيرة بلا **مخرجات** واضحة.  
- غياب **تعريف DoD/DoR** أو حواجز الجودة.  
- خلط أدوار ومسؤوليات، أو تجاهل **قياس** الأداء والتحسين.

---

## جدول مقارنة مختصر

| المفهوم                   | الغرض الرئيسي                | ملاحظات مهمة                    |
| ------------------------- | ---------------------------- | ------------------------------- |
| **Methodology**           | **فلسفة وقواعد تنظيم العمل** | تحدّد أدوار/طقوس/مخرجات وقياسات  |
| [Agile](agile.md)         | تكرار سريع موجّه بالقيمة      | Scrum/Kanban ضمن المظلّة         |
| [Waterfall](waterfall.md) | تسلسل مرحلي بحواجز اعتماد    | مناسب لثبات المتطلبات والامتثال |
| [SDLC](sdlc.md)           | مراحل عامة لتطوير البرمجيات  | يمكن تنفيذها بأي منهجيّة         |
| [Approach](approach.md)   | مقاربة حل قصيرة المدى        | تكتيك لتنفيذ جزء ضمن المنهجيّة   |
| [Framework](framework.md) | هيكل/أدوات لتطبيق المنهجيّة   | Scrum/Kanban = أُطر ضمن Agile    |

---

## ملخص الفكرة  
**Methodology** تنظّم **كيف** يعمل الفريق لتحقيق الهدف بجودة واستدامة.  
اختر ما يناسب سياقك، حدّد **الأدوار والمخرجات والإيقاع**، أضِف **حواجز جودة**،  
وقِس لتحسّن باستمرار—هكذا تتحول الطقوس إلى **نتائج** ملموسة. 
