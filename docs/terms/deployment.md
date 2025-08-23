# **Deployment**

## الترجمة الحرفية  
**Deployment** — **نشر التطبيق** إلى بيئة تشغيل (Server/VM/Container/Kubernetes/PAAS).

## الوصف العربي المختصر  
عملية **وضع المخرجات الجاهزة** (حزمة/صورة) في البيئة،  
مع خطوات آمنة: **تهيئة، فحص صحيّة، توجيه حركة، مراقبة، وخطّة رجوع**.

## الشرح المبسّط  
- نبني **Artifact** (مثلاً صورة OCI).  
- نرفعها إلى المستودع.  
- ننشرها على الهدف (VM/K8s/PAAS).  
- نتحقق من **الجاهزية** قبل استقبال الحركة.  
- نراقب ونملك **Rollback** فوري.

## تشبيه  
افتتاح مطعم جديد: نجَهّز المطبخ، نختبر الغاز/الماء،  
نفتح الأبواب تدريجيًا، نراقب رضا الزبائن، ونبقي باب الرجوع مفتوحًا إن ظهر عطل.

---

## مثال كود C# — خدمة جاهزة للنشر الآمن (Health/Readiness/Warm-up/Shutdown)

```csharp
// Program.cs  (.NET 8/9)
// - /healthz: فحص حيّة (للموازِن/الـ Orchestrator).
// - /ready  : جاهزية بعد إتمام Warm-up.
// - /       : تقرير إصدار/مثيل.
// - إشارة إيقاف رشيق: يجعل /ready يفشل لكي يخرج من التجميع قبل الإنهاء.

using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<ReadinessState>();
builder.Services.AddHostedService<WarmupService>(); // تنفيذ مهام تهيئة قبل قبول الحركة

var app = builder.Build();

var state = app.Services.GetRequiredService<ReadinessState>();
var lifetime = app.Lifetime;

// عند بدء الإيقاف: أعلن عدم الجاهزية لكي يوقف الـ LB التوجيه لهذا المثيل
lifetime.ApplicationStopping.Register(() =>
{
    state.Ready = false;
    Console.WriteLine("Graceful shutdown: mark not-ready.");
});

app.MapGet("/", () => new
{
    service = "orders-api",
    version = typeof(Program).Assembly.GetName().Version?.ToString() ?? "0.0.0",
    instance = Environment.GetEnvironmentVariable("HOSTNAME") ?? Environment.MachineName,
    startedAtUtc = state.StartedAtUtc
});

app.MapGet("/healthz", () => Results.Ok(new { status = "healthy" })); // حيّة أساسية

app.MapGet("/ready", () =>
{
    return state.Ready
        ? Results.Ok(new { ready = true })
        : Results.StatusCode(StatusCodes.Status503ServiceUnavailable);
});

app.Run();

// حالة الجاهزية المشتركة
public class ReadinessState
{
    public bool Ready { get; set; } = false;
    public DateTime StartedAtUtc { get; } = DateTime.UtcNow;
}

// خدمة Warm-up: تحميل قوالب/كاش/اتصال DB إلخ قبل قبول الحركة
public class WarmupService : IHostedService
{
    private readonly ReadinessState _state;
    public WarmupService(ReadinessState state) => _state = state;

    public async Task StartAsync(CancellationToken ct)
    {
        Console.WriteLine("Warm-up: connecting to dependencies...");
        await Task.Delay(1500, ct); // محاكاة تهيئة
        _state.Ready = true;
        Console.WriteLine("Warm-up: READY.");
    }
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

**الفكرة:**  
- **/ready** لا يرجع 200 إلا بعد التهيئة.  
- عند الإيقاف، **/ready** يصبح 503 → الموازِن يخرج المثيل من التجميع قبل قتل العملية.  
- هذا يمكّن **Rolling/Blue-Green/Canary** بدون انقطاع.

---

## خطوات عملية لنشر موثوق (Pipeline مختصر)
1. **Build**: توليد Artifact (حزمة/صورة) + أرقام إصدار.  
2. **Test**: وحدات/تكامل/أمن/تحليل ساكن في CI.  
3. **Package**: صورة OCI موقّعة (SBOM وفحص ثغرات).  
4. **Deploy**: إلى بيئة **staging** أولاً.  
5. **Verify**: فحوص **/healthz** و**/ready** + اختبارات دخوليّة آلية.  
6. **Traffic**: حول جزءًا من الحركة (Canary) أو بدّل اللون (Blue/Green).  
7. **Observe**: سجلات/قياسات/P95 أخطاء.  
8. **Rollback**: زر رجوع واضح (إصدار سابق/ReplicaSet سابق).  

**أوامر تحقق سريعة بعد النشر:**
```bash
curl -f http://app/healthz   # يجب 200
curl -f http://app/ready     # يجب 200 قبل توجيه الحركة
```

---

## أخطاء شائعة
- عدم التفريق بين **حيّة** و**جاهزية** → انقطاع أثناء التحديثات.  
- نشر يدوي دون **Rollback** معلوم.  
- سرّ/مفاتيح داخل الصورة/المستودع.  
- تجاهل المراقبة → اكتشاف متأخر للأعطال.  
- إطلاق Big-Bang بدل **تدرّج الحركة** (Canary) أو **Blue/Green**.  
- الاعتماد على `latest` بدل وسوم/إصدارات ثابتة.

---

## جدول مقارنة مختصر

| المفهوم               | الغرض الرئيسي                     | ملاحظات مهمة                                   |
| --------------------- | --------------------------------- | ---------------------------------------------- |
| **Deployment**        | **وضع الإصدار في البيئة وتشغيله** | يشمل تهيئة، فحوص، توجيه حركة، مراقبة، Rollback |
| Release               | إتاحة الميزة للمستخدمين           | قد تُفعل عبر **Feature Flags** بعد نشر الكود    |
| Continuous Delivery   | جاهزية دائمة للنشر                | النشر يتطلب إشارة بشرية صغيرة                  |
| Continuous Deployment | نشر تلقائي لكل تغيير ناجح         | يعتمد بوابات جودة قوية                         |
| Blue/Green            | بيئتان، تبديل حركة بينهما         | رجوع فوري بتبديل الـ LB                        |
| Canary                | نسبة صغيرة من الحركة أولًا         | يقلّل المخاطر بالمراقبة التدريجية               |
| Rolling Update        | استبدال تدريجي للـ Pods/Instances | يحتاج **Readiness/Graceful shutdown** سليمة    |

---

## ملخص الفكرة  
**Deployment** الناجح = **Artifact جيد + صحّة/جاهزية + توجيه حركة مدروس + مراقبة + رجوع سريع**.  
جهّز خدمتك بنقاط فحص وتهيئة وإيقاف رشيق، ثم اختر استراتيجية (Canary/Blue-Green/Rolling)  
لتنشر **بدون توقف** ومع ثقة أعلى.
