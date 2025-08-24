# **Workflow**

## الترجمة الحرفية  
**Workflow** — **سير/تدفق العمل** (سلسلة خطوات منظَّمة لإنجاز مهمة مع قواعد انتقال).

## الوصف العربي المختصر  
تصميم يُحدِّد **خطوات** (Tasks/Activities) و**ترتيبها** و**شروط الانتقال** و**المخرجات**،  
مع **حالات** (States) و**أحداث** (Events) وإجراءات **تعويض/تراجع** عند الفشل.

## الشرح المبسّط  
- يتكون من **خطوات** تنفَّذ **تسلسليًا** أو **بالتوازي**، مع **شروط/موافقات**.  
- يدير **الحالة** بين الخطوات (مكتملة/فاشلة/معلّقة).  
- قد يكون **قصيرًا** (Pipeline) أو **طويل الأمد** مع انتظار أحداث خارجية (Orchestration).  
- أمثلة: إنشاء طلب شراء، Onboarding موظف، معالجة مدفوعات.

## تشبيه  
رحلة معاملة في دائرة حكومية: تقديم الطلب → تدقيق → موافقة → دفع → استلام الوثيقة.  
كل محطة تتحقق وتوقّع قبل الانتقال، ولو فشلت محطة—تُعاد المعاملة أو تُلغى مع **إلغاء الإجراءات**.

---

## مثال C# — محرّك Workflow مبسّط مع **تعويض (Compensation)** و**سجل حالة**

```csharp
// .NET 8/9 — Workflow بسيط قابل للتوسعة
using System.Collections.Concurrent;

public record StepResult(bool Success, string? Error = null);

public class WorkflowContext
{
    // مخزن بيانات مشترك بين الخطوات
    private readonly ConcurrentDictionary<string, object> _bag = new();
    public T? Get<T>(string key) => _bag.TryGetValue(key, out var v) ? (T)v : default;
    public void Set<T>(string key, T value) => _bag[key] = value!;
    public string CorrelationId { get; } = Guid.NewGuid().ToString("N");
    public List<string> Log { get; } = new();
}

public interface IWorkflowStep
{
    string Name { get; }
    Task<StepResult> ExecuteAsync(WorkflowContext ctx, CancellationToken ct);
    Task CompensateAsync(WorkflowContext ctx, CancellationToken ct); // للتراجع عند الفشل
}

public class WorkflowEngine
{
    private readonly IReadOnlyList<IWorkflowStep> _steps;
    public WorkflowEngine(IEnumerable<IWorkflowStep> steps) => _steps = steps.ToList();

    public async Task<bool> RunAsync(WorkflowContext ctx, CancellationToken ct = default)
    {
        var executed = new Stack<IWorkflowStep>();
        foreach (var step in _steps)
        {
            ctx.Log.Add($"→ {step.Name}");
            var res = await step.ExecuteAsync(ctx, ct);
            if (res.Success) { executed.Push(step); continue; }

            ctx.Log.Add($"✗ {step.Name}: {res.Error}");
            // تعويض عكسي
            while (executed.Count > 0)
            {
                var prev = executed.Pop();
                try { await prev.CompensateAsync(ctx, ct); ctx.Log.Add($"↺ compensating {prev.Name}"); }
                catch (Exception ex) { ctx.Log.Add($"! compensation failed {prev.Name}: {ex.Message}"); }
            }
            return false;
        }
        return true;
    }
}

// ====== خطوات عيّنية لطلب شراء ======
public class ReserveInventoryStep : IWorkflowStep
{
    public string Name => "ReserveInventory";
    public async Task<StepResult> ExecuteAsync(WorkflowContext ctx, CancellationToken ct)
    {
        await Task.Delay(200, ct); // محاكاة I/O
        ctx.Set("reservationId", Guid.NewGuid().ToString("N"));
        return new StepResult(true);
    }
    public Task CompensateAsync(WorkflowContext ctx, CancellationToken ct)
    {
        // إلغاء الحجز إن وُجد
        ctx.Set("reservationId", null);
        return Task.CompletedTask;
    }
}

public class ChargePaymentStep : IWorkflowStep
{
    public string Name => "ChargePayment";
    public async Task<StepResult> ExecuteAsync(WorkflowContext ctx, CancellationToken ct)
    {
        await Task.Delay(200, ct);
        var amount = ctx.Get<decimal>("amount");
        if (amount <= 0) return new StepResult(false, "invalid_amount");
        // فشل مُفتعل للتجربة:
        if (amount > 500) return new StepResult(false, "limit_exceeded");
        ctx.Set("paymentId", Guid.NewGuid().ToString("N"));
        return new StepResult(true);
    }
    public Task CompensateAsync(WorkflowContext ctx, CancellationToken ct)
    {
        // رد المبلغ إن تم التحصيل
        ctx.Set("paymentId", null);
        return Task.CompletedTask;
    }
}

public class SendEmailStep : IWorkflowStep
{
    public string Name => "SendEmail";
    public async Task<StepResult> ExecuteAsync(WorkflowContext ctx, CancellationToken ct)
    {
        await Task.Delay(100, ct);
        var email = ctx.Get<string>("email");
        if (string.IsNullOrWhiteSpace(email)) return new StepResult(false, "email_required");
        ctx.Log.Add($"[email→{email}] order confirmed");
        return new StepResult(true);
    }
    public Task CompensateAsync(WorkflowContext ctx, CancellationToken ct)
    {
        // إرسال اعتذار/إلغاء (اختياري)
        return Task.CompletedTask;
    }
}

// ====== تشغيل المثال ======
public class Program
{
    public static async Task Main()
    {
        var steps = new IWorkflowStep[]
        {
            new ReserveInventoryStep(),
            new ChargePaymentStep(),
            new SendEmailStep()
        };
        var engine = new WorkflowEngine(steps);

        var ctx = new WorkflowContext();
        ctx.Set("amount", 750m);          // جرّب 150m للنجاح
        ctx.Set("email", "user@example.com");

        bool ok = await engine.RunAsync(ctx);
        Console.WriteLine($"Workflow({ctx.CorrelationId}) success? {ok}");
        Console.WriteLine(string.Join("\n", ctx.Log));
    }
}
```

**الفكرة:**  
- كل خطوة مستقلّة بواجهة موحّدة.  
- عند الفشل: **تعويض عكسي** للخطوات المنفَّذة.  
- **Context** يحمل بيانات مشتركة و**CorrelationId** للتتبّع.

---

## خطوات عملية لبناء **Workflow** جيّد
1. **حدِّد الحالات** والانتقالات (State Diagram/BPMN مبسّط).  
2. قسّم إلى **خطوات صغيرة** ذات **مسؤولية واحدة** وقابلة للتجربة منفردة.  
3. اجعل الخطوات **Idempotent** (لا تتأثر بإعادة المحاولة) واستخدم **مفاتيح مرجعية**.  
4. قرّر **سياسة الفشل**: تعويض (Compensation)، انتظار حدث، أو إيقاف مع **DLQ**.  
5. ضع **مهلات/إلغاء** (Timeout/CancellationToken) و**Backoff** لإعادة المحاولة.  
6. فعّل **التتبّع** (Logs/Tracing) مع **CorrelationId** من أوّل الطلب.  
7. للمهام الطويلة: استخدم **Orchestrator** (Durable/Temporal/Step Functions) بدل حلقات انتظار.  
8. وثّق **عقود الأحداث** (Schemas) ومتى تُرسل/تُستهلك.

---

## أخطاء شائعة
- منطق خطوات ضخم وغير قابل لإعادة الاستخدام.  
- غياب **Idempotency** → تكرارات/خصومات مالية مضاعفة.  
- نسيان **التعويض** أو تركه جزئيًا → حالة بيانات غير متسقة.  
- اعتماد على **ذاكرة العملية** لحالة طويلة الأمد (تفقدها عند إعادة التشغيل).  
- عدم قياس الزمن والـ **SLA** لكل خطوة → اختناقات خفيّة.  
- تشابك قوي بين الخطوات بدل عقود واضحة (Interfaces/DTOs/Events).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Workflow** | **تنسيق خطوات مع حالة وانتقالات** | قد يكون قصيرًا أو طويل الأمد |
| Process | عملية أعمال عالية المستوى | قد تحتوي عدة Workflows |
| Pipeline | خطوات خطيّة للمعالجة | عادةً بلا حالات انتظار طويلة |
| Orchestration | منسّق مركزي يدير الخطوات/الأحداث | قرارات مركزية، تعويض، تتبّع |
| Choreography | تفاعل لامركزي بالأحداث | كل خدمة تتفاعل وفق قواعد محليّة |
| BPMN | ترميز رسومي للعمليات | توحيد تصميم/تنفيذ مع محركات BPM |
| State Machine | حالات + انتقالات بقواعد | مناسب لسير واضح المعالم |

---

## ملخص الفكرة  
**Workflow** = ترتيب **خطوات** مع **حالة** و**قواعد انتقال** و**تعويض** عند الحاجة.  
قسِّم السلوك، اجعل الخطوات **Idempotent**، أضِف **مهلات/تتبّع**، واختر **Orchestration/Choreography** وفق السياق—  
ستحصل على سير عمل **موثوق**، **قابل للتوسّع**، وسهل الصيانة. 
