# **Middleware Pipeline**
**خطّ أنابيب الوسائط الوسطية (ASP.NET Core)**

## الترجمة الحرفية  
**Middleware Pipeline** — **تسلسل وسائط وسطية** تُعالِج طلب HTTP وردّه.

## الوصف العربي المختصر  
سلسلة مرتَّبة من مكوّنات تمرّ عليها **كلّ** الطلبات.  
كل Middleware يمكنه: تنفيذ منطق **قبل** الطلب، ثم استدعاء **التالي** `next()`، ثم منطق **بعد** الرد.  
**الترتيب مهم**، ويمكن **الاختصار** (Short-circuit) بإرجاع ردّ دون متابعة السلسلة.

## الشرح المبسّط  
- فلاتر على الطريق: مصادقة، تسجيل، CORS، ضغط، مسارات… حسب **الترتيب**.  
- **Terminal Middleware**: يُنهي السلسلة (لا يستدعي `next`).  
- **Branching**: تفرّعات بشرط/مسار (`Map`, `MapWhen`, `UseWhen`).  
- في Minimal APIs/Controllers، **التوجيه** يحدّد المعالج النهائي ضمن السلسلة.

## تشبيه  
نقاط تفتيش في المطار: تفتيش آلي، ثم جوازات، ثم بوّابة الصعود.  
لو فشلت في التفتيش—تتوقّف رحلتك (Short-circuit).

---

## مثال C# عملي — Pipeline مع ترتيب، فرع، وTerminal

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n PipelineDemo && cd PipelineDemo && dotnet run
using System.Diagnostics;

var builder = WebApplication.CreateBuilder(args);

// تسجيل Middlewares تعتمد DI (اختياري)
builder.Services.AddSingleton<RequestTimingMiddleware>();
builder.Services.AddSingleton<CorrelationIdMiddleware>();

var app = builder.Build();

// 1) معالجة الأخطاء في أعلى السلسلة
app.Use(async (ctx, next) =>
{
    try { await next(); }
    catch (Exception ex)
    {
        Console.Error.WriteLine($"[ERR] {ex.Message}");
        ctx.Response.StatusCode = 500;
        await ctx.Response.WriteAsJsonAsync(new { error = "server_error" });
    }
});

// 2) معرّف ترابط للطلب (قبل أي تسجيل)
app.UseMiddleware<CorrelationIdMiddleware>();

// 3) فرع داخلي لمسارات /internal فقط (UseWhen لا يغيّر المسار)
app.UseWhen(ctx => ctx.Request.Path.StartsWithSegments("/internal"), branch =>
{
    branch.Use(async (ctx, next) =>
    {
        // حماية بسيطة: مفتاح داخلي
        if (ctx.Request.Headers["X-Internal-Key"] != "secret")
        {
            ctx.Response.StatusCode = 401;      // Short-circuit
            await ctx.Response.WriteAsync("unauthorized");
            return;
        }
        await next();
    });
});

// 4) قياس زمن الطلب (قبل/بعد next)
app.UseMiddleware<RequestTimingMiddleware>();

// 5) نقطة صحّة Terminal سريعة (Map = Endpoint نهائي)
app.MapGet("/healthz", () => Results.Ok(new { status = "ok" }));

// 6) API عادي
var api = app.MapGroup("/api");
api.MapGet("/hello", (HttpContext ctx) =>
{
    // رمي خطأ تجريبي لترى المعالج بالأعلى
    if (ctx.Request.Query.ContainsKey("boom")) throw new Exception("demo");
    return Results.Ok(new { msg = "hello", cid = ctx.Response.Headers["X-Correlation-Id"].ToString() });
});

// 7) فرع لمسار ثابت (MapWhen يوجّه بقية السلسلة)
app.MapWhen(ctx => ctx.Request.Path.StartsWithSegments("/internal"), branch =>
{
    branch.MapGet("/internal/info", (HttpContext ctx) =>
        Results.Ok(new { env = app.Environment.EnvironmentName }));
});

app.Run();

// ======= Middlewares مخصّصة =======
public class CorrelationIdMiddleware
{
    private const string Header = "X-Correlation-Id";
    private readonly RequestDelegate _next;
    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext ctx)
    {
        var cid = ctx.Request.Headers.TryGetValue(Header, out var v) && !string.IsNullOrWhiteSpace(v)
            ? v.ToString()
            : Guid.NewGuid().ToString("N");

        ctx.Response.Headers[Header] = cid;         // قبل next: تجهيز السياق
        await _next(ctx);                            // مرّر للذي بعده
        // بعد next: يمكن إضافة Headers/Logging اعتمادًا على النتيجة
    }
}

public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    public RequestTimingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext ctx)
    {
        var sw = Stopwatch.StartNew();
        await _next(ctx);
        sw.Stop();
        ctx.Response.Headers["X-Elapsed-ms"] = sw.ElapsedMilliseconds.ToString();
        Console.WriteLine($"{ctx.Request.Method} {ctx.Request.Path} -> {ctx.Response.StatusCode} in {sw.ElapsedMilliseconds}ms");
    }
}
```

**جرّب سريعًا:**
```bash
curl -i http://localhost:5000/healthz
curl -i http://localhost:5000/api/hello
curl -i "http://localhost:5000/api/hello?boom=1"      # سيُلتقط الاستثناء في الأعلى
curl -i http://localhost:5000/internal/info           # 401
curl -i -H "X-Internal-Key: secret" http://localhost:5000/internal/info
```

**الملاحظة:** تَظهر Headers مثل `X-Correlation-Id` و`X-Elapsed-ms`؛  
الطلبات الداخلية تتوقّف مبكّرًا إن لم تحمل المفتاح (Short-circuit).

---

## خطوات عملية لبناء Pipeline سليم
- ضع **Exception Handling** و**HTTP Logging** مبكرًا.  
- طبّق **HTTPS/HSTS/CORS** قبل المصادقة.  
- لـ MVC: `UseRouting()` → **Auth/Authorization** → `MapControllers()`.  
- لـ Minimal APIs: طبّق `Use…` قبل `Map…` حيث يلزم (الترتيب مؤثّر).  
- استخدم **Map/MapWhen/UseWhen** للتفرّعات، و**Terminal** لنهايات سريعة (Health/Static).  
- اجعل الوسائط **صغيرة مسؤوليّات** وقابلة لإعادة الاستخدام والاختبار.

---

## أخطاء شائعة
- نسيان `await next()` → تعليق الطلبات.  
- ترتيب خاطئ (مثل Authorization قبل Routing) → نتائج غير متوقّعة.  
- الكتابة للجسم **قبل** مناداة `next()` ثم محاولة التعديل **بعده** بشكل يتعارض مع الإرسال.  
- توسعة Middleware واحدة بمهام كثيرة (God Middleware).  
- عدم التعامل مع الإلغاء/المهلات للعمليات البعيدة.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Middleware Pipeline** | **معالجة كل الطلبات بسلسلة مرتّبة** | **Before/Next/After**، ترتيب حسّاس، Short-circuit |
| Endpoint/Handler | نقطة النهاية الفعلية للطلب | يُختار عبر Routing داخل السلسلة |
| Filters (ASP.NET) | اهتمامات مقطعية على مستوى Endpoint | تعمل حول المعالج؛ تكمّل الـ Middleware |
| DelegatingHandler (HttpClient) | Pipeline للطلبات **الصادرة** | مماثل للفكرة ولكن Outbound |
| Reverse Proxy (NGINX/YARP) | طبقة خارجية أمام التطبيق | يمكن إضافة فلاتر قبل دخول الـ Pipeline الداخلي |

---

## ملخص الفكرة  
**Middleware Pipeline** هو **طريق الطلب** داخل تطبيقك:  
رتّبه بعناية، اجعل مبكّرات مثل **الأخطاء/الأمن** في الأعلى، استعمل **تفرّعات** عند الحاجة،  
وادمج **Terminal** لنقاط سريعة—تحصل على سلوك متوقّع، أداء جيّد، وصيانة أسهل.