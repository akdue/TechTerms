# **Agile**

## الترجمة الحرفية  
**Agile Software Development** — **التطوير الرشيق**

## الوصف العربي المختصر  
طريقة عمل **تكرارية** و**تدريجية** تركّز على **القيمة** و**التغذية الراجعة السريعة**،  
بفرق **متعدّدة التخصّصات**، وقوائم عمل **قابلة للأولوية**، وتسليمات صغيرة متكرّرة.

## الشرح المبسّط  
- نعمل على **دُفعات قصيرة** (Sprints/Iterations).  
- نحدّد **قيمة واضحة** لكل عنصر (Story) ومعايير قبول.  
- نُراجع دائمًا: **وقوف يومي**، **استعراض**، **تحسين مستمر (Retro)**.  
- نقيس بالنتائج (قيمة/أثر) لا بعدد المهام فقط.

## تشبيه  
بدل بناء ناطحة سحاب دفعة واحدة، نبني **طابقًا صغيرًا يعمل** كل أسبوعين،  
ونُعدّل التصميم بسرعة حسب رأي الساكنين.

---

## مثال كود C# — ترتيب الـ Backlog بطريقة **WSJF** (قيمة/حجم)

```csharp
// dotnet new console -n AgileDemo && cd AgileDemo
using System;
using System.Linq;
using System.Collections.Generic;

// نموذج عنصر Backlog بعوامل WSJF
public record BacklogItem(string Title, int BusinessValue, int TimeCriticality, int RiskReduction, int JobSize)
{
    // WSJF = (BV + TC + RR) / JS  — كلما زادت، قفزت الأولوية
    public double Wsjf => (BusinessValue + TimeCriticality + RiskReduction) / Math.Max(1.0, JobSize);
}

class Program
{
    static void Main()
    {
        var backlog = new List<BacklogItem>
        {
            new("Checkout: Add Apple Pay", 8, 7, 3, 5),
            new("Fix intermittent login bug", 6, 9, 5, 3),
            new("Product page A/B test", 5, 5, 2, 2),
            new("Refactor cart module", 4, 3, 6, 8),
        };

        var ordered = backlog.OrderByDescending(i => i.Wsjf).ToList();

        Console.WriteLine("Sprint Candidates (by WSJF):");
        foreach (var i in ordered)
            Console.WriteLine($"- {i.Title} | WSJF={i.Wsjf:F2} (BV={i.BusinessValue}, JS={i.JobSize})");
    }
}
```

**الإخراج المتوقع (مختصر):**  
قائمة مرتّبة بعناصر أعلى **WSJF** أولًا → **أولوية** للتسليم القصير ذي الأثر الأكبر.

---

## خطوات عملية لتبنّي Agile
1. **Backlog موحّد**: قصص مستخدم **User Stories** بمعايير قبول واضحة.  
2. **تخطيط تكراري**: Sprint قصير (1–2 أسبوع) بأهداف قابلة للقياس.  
3. **شفافية**: لوحة عمل (To Do / In Progress / Done)، وتعريف واضح لـ **Done**.  
4. **التسليم المستمر**: CI/CD واختبارات آلية لتقليل زمن الدورة.  
5. **قياس**: Lead Time/Throughput، ومراجعات **Sprint Review** و **Retro** لتحسين العملية.  
6. **إدارة السعة**: حدّ العمل الجاري (**WIP**) وموازنة الفريق لتجنّب الاختناقات.

---

## أخطاء شائعة
- تحويل Agile إلى **اجتماعات بلا قيمة** دون تسليمات صغيرة قابلة للشحن.  
- قصص غامضة بلا **معايير قبول** أو بلا قيمة أعمال واضحة.  
- **WIP** مفتوح للجميع → عمل متشتّت وتأخير.  
- التركيز على **السرعة (Velocity)** كهدف بحد ذاته بدل **القيمة** والنتائج.  
- تجاهل **التقنية/الجودة** (ديون تقنية تتراكم) لزيادة “النقاط”.

---

## جدول مقارنة مختصر

| المفهوم             | الغرض الرئيسي                                  | ملاحظات مهمة                                 |
| ------------------- | ---------------------------------------------- | -------------------------------------------- |
| **Agile**           | **تسليم تكراري موجّه بالقيمة والتغذية الراجعة** | مظلة تضم Scrum/Kanban وأساليب أخرى           |
| Scrum               | إطار أدوار وطقوس وتتبّع Sprint                  | Product Owner / Scrum Master / Dev Team      |
| Kanban              | تدفّق مستمر وحدود WIP                           | يقلّل زمن الانتظار؛ لا Sprints ثابتة          |
| [DevOps](devops.md) | أتمتة بناء/نشر/تشغيل                           | يكمّل Agile لتقصير زمن الدورة                 |
| [Waterfall](waterfall.md)       | تسلسل خطّي لمراحل ثابتة                         | مناسب لعقود/امتثال صارم؛ صعب التغيير المتأخر |

---

## ملخص الفكرة  
**Agile** يعني **تسليم قيمة صغيرة بسرعة**، مع **تعلم مستمر** وتكيّف.  
احفظ قصصًا واضحة، رَتّب بالأثر (مثال **WSJF**)، قلّل WIP، وادعمها بـ **CI/CD**—تحصل على إصدارات أسرع وجودة أعلى.
