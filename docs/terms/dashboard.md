# **Dashboard**

## الترجمة الحرفية  
**Dashboard** — **لوحة قيادة/لوحة مؤشرات**.

## الوصف العربي المختصر  
واجهة تُظهر **KPIs** و**رسومًا** و**تنبيهات** في شاشة واحدة.  
هدفها **رؤية لحظية**، واتخاذ قرار **سريع**، مع **تحديث مباشر** للبيانات.

## الشرح المبسّط  
- تجمع **مقاييس** من خدمات/قواعد مختلفة.  
- تلخّصها في **بطاقات** (Cards) و**رسوم** (Charts).  
- تحدّث الأرقام **تدريجيًا** (Polling/SSE/WebSocket).  
- تعرض **حالات** (OK/Warn/Crit) و**اتجاهات** (↑ ↓).

## تشبيه  
لوحة سيارة: سرعة، وقود، حرارة. نظرة واحدة تكفي لتعرف **الوضع الآن** وتقرّر.

---

## مثال C# — API للوحة مؤشرات مع بثّ مباشر **SSE** + لقطة حالية

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n DashboardApi && cd DashboardApi && dotnet run
using System.Threading.Channels;
using Microsoft.AspNetCore.Mvc;
using System.Text.Json;

var builder = WebApplication.CreateBuilder(args);

// قناة بثّ مقاييس + خدمة توليد تجريبية
builder.Services.AddSingleton<MetricsBus>();
builder.Services.AddHostedService<MetricsSampler>();

var app = builder.Build();
app.UseHttpsRedirection();

// 1) لقطة حالية: GET /dashboard/kpis  → JSON
app.MapGet("/dashboard/kpis", ([FromServices] MetricsBus bus)
    => Results.Ok(bus.Latest));

// 2) بثّ مباشر (SSE): GET /dashboard/stream  → text/event-stream
app.MapGet("/dashboard/stream", async (HttpContext ctx, MetricsBus bus) =>
{
    ctx.Response.Headers.CacheControl = "no-store";
    ctx.Response.Headers.Connection   = "keep-alive";
    ctx.Response.ContentType          = "text/event-stream";

    await foreach (var kpi in bus.Reader.ReadAllAsync(ctx.RequestAborted))
    {
        var json = JsonSerializer.Serialize(kpi);
        await ctx.Response.WriteAsync($"data: {json}\n\n", ctx.RequestAborted);
        await ctx.Response.Body.FlushAsync(ctx.RequestAborted);
    }
});

// (اختياري) سلسلة زمنية قصيرة للرسم الأولي: GET /dashboard/series
app.MapGet("/dashboard/series", ([FromServices] MetricsBus bus)
    => Results.Ok(bus.Recent()));

app.Run();


// ====== النماذج/الخدمات ======
public record KpisSnapshot(
    DateTime TsUtc,
    int ActiveUsers,
    decimal RevenueUsd,
    double ErrorRate,
    double P95LatencyMs);

public class MetricsBus
{
    private readonly Channel<KpisSnapshot> _ch = Channel.CreateUnbounded<KpisSnapshot>();
    private readonly LinkedList<KpisSnapshot> _buffer = new(); // آخر N نقاط
    private readonly object _gate = new();
    public KpisSnapshot Latest { get; private set; } = new(DateTime.UtcNow, 0, 0, 0, 0);

    public ChannelReader<KpisSnapshot> Reader => _ch.Reader;

    public void Publish(KpisSnapshot s)
    {
        Latest = s;
        lock (_gate)
        {
            _buffer.AddLast(s);
            if (_buffer.Count > 120) _buffer.RemoveFirst(); // ~ آخر دقيقتين (إذا كل ثانية)
        }
        _ch.Writer.TryWrite(s);
    }

    public IEnumerable<KpisSnapshot> Recent()
    {
        lock (_gate) return _buffer.ToArray();
    }
}

public class MetricsSampler : BackgroundService
{
    private readonly MetricsBus _bus;
    private readonly Random _rnd = new();
    public MetricsSampler(MetricsBus bus) => _bus = bus;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        double err = 0.01, p95 = 120;
        int users = 1200; decimal rev = 0;

        while (!ct.IsCancellationRequested)
        {
            // محاكاة تغيّر المقاييس
            users = Math.Max(0, users + _rnd.Next(-15, 25));
            rev += (decimal)_rnd.NextDouble() * 30m;
            err = Math.Clamp(err + (_rnd.NextDouble() - 0.5) * 0.002, 0, 0.2);
            p95 = Math.Clamp(p95 + (_rnd.NextDouble() - 0.5) * 5, 40, 400);

            _bus.Publish(new KpisSnapshot(
                TsUtc: DateTime.UtcNow,
                ActiveUsers: users,
                RevenueUsd: Math.Round(rev, 2),
                ErrorRate: Math.Round(err, 4),
                P95LatencyMs: Math.Round(p95, 1)));

            await Task.Delay(1000, ct); // تحديث كل ثانية
        }
    }
}
```

> الواجهة (SPA/Blazor/أي إطار) تتصل بـ:
> - **GET `/dashboard/kpis`** لملء البطاقات فورًا.  
> - **SSE `/dashboard/stream`** لتحديث الرسوم والبطاقات لحظيًا.

---

## خطوات عملية لبناء **Dashboard** مفيد
1. **اختر KPIs قليلة وواضحة**: أمثلة (DAU، الإيراد/اليوم، ErrorRate، P95).  
2. **مصادر موثوقة**: اجمع البيانات عبر **خدمة تجميع** (Aggregator) لا من الواجهة مباشرة.  
3. **تحديث مباشر**: **SSE/WebSocket** للحيّة، و**Cache** للقراءات الباردة.  
4. **حالات/ألوان**: قواعد عتبات (OK/Warn/Crit) واضحة وسهلة القراءة.  
5. **وقت/منطقة**: اعرض المنطقة الزمنية، واسمح بتغيير النطاق الزمني.  
6. **أداء الواجهة**: افصل الرسم إلى **canvas/virtualization**، وحدّث فرقّيًا (diff).  
7. **Observability**: سجّل كل تحديث مع **Correlation-Id** ومصدر القياس.  
8. **الأمان**: AuthZ بحسب الفريق/المشروع، وإخفاء البيانات الحسّاسة (PII).

---

## أخطاء شائعة
- **لوحة مزدحمة**: كثير من الرسوم بلا أولوية → لا قرارات.  
- **Polling كثيف** بدل SSE/WebSocket → حمل زائد وزمن أعلى.  
- خلط **التحويل/المنطق** في الواجهة؛ يجب أن يتم في API التجميعية.  
- عدم توحيد **التوقيت/الفواصل** بين الرسوم → مقارنة مضلِّلة.  
- تجاهل **العتبات/التنبيهات** → لوحة “جميلة” بلا فائدة عملية.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Dashboard** | **رؤية لحظية لاتخاذ قرار سريع** | بثّ مباشر، بطاقات + رسوم، عتبات |
| Report | توثيق دوري/تاريخي | PDF/Excel؛ تفصيلي لا لحظي |
| Monitoring Board | صحة الأنظمة والبنية | Latency/Errors/Uptime؛ Alerts |
| Analytics | تحليلات متقدمة/استكشاف | Segments/Attribution؛ أبطأ وأعمق |
| KPI Card | مقياس واحد مركّز | جزء من لوحة أو صفحة صغيرة |

---

## ملخص الفكرة  
**Dashboard** = شاشة **واحدة** تجمع أهم مؤشراتك **الآن**.  
اجمع البيانات في API، اعرض **KPIs مختصرة**، واستخدم **SSE/WebSocket** لتحديث حي—  
تحصل على لوحة **عملية** تقود القرار بدل أن تكون مجرد رسم جميل. 
