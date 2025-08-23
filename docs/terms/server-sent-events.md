# **(SSE) Server-Sent Events**

## الترجمة الحرفية  
**Server-Sent Events (SSE)** — **أحداث يُرسلها الخادم للعميل عبر اتصال HTTP مستمرّ**.

## الوصف العربي المختصر  
قناة **أحاديّة الاتجاه** من **الخادم → العميل** فوق **HTTP/HTTPS**.  
يبقي الاتصال **مفتوحًا** ويُرسل **أحداثًا نصيّة** (stream) يدعم **إعادة الاتصال تلقائيًا** و**Last-Event-ID**.

## الشرح المبسّط  
- يبدأ العميل طلب HTTP إلى مسار SSE.  
- الخادم يردّ بـ `Content-Type: text/event-stream` ويبقي الاتصال مفتوحًا.  
- يرسل سطورًا على شكل:  
  - `data: ...` لمحتوى الحدث  
  - `event: name` لاسم مخصص (اختياري)  
  - `id: 123` لمعرّف الحدث (اختياري)  
  - سطر فارغ لإنهاء الرسالة  
- المتصفح يستقبل عبر **EventSource** ويعيد الاتصال تلقائيًا عند الانقطاع.

## تشبيه  
بدل إرسال رسالة جديدة كل مرة، تفتح **حنفيّة ماء** من الخادم للعميل.  
الماء (الأحداث) يتدفّق بانتظام أو عند حدوث شيء جديد.

## مثال خادم C# (ASP.NET Core Minimal API — بث SSE)
```csharp
// Program.cs
using System.Text.Json;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/sse", async context =>
{
    context.Response.Headers.Append("Cache-Control", "no-cache");
    context.Response.Headers.Append("Content-Type", "text/event-stream");

    var i = 0;
    while (!context.RequestAborted.IsCancellationRequested)
    {
        var payload = JsonSerializer.Serialize(new { tick = i++, at = DateTime.UtcNow });
        await context.Response.WriteAsync($"id: {i}\n");
        await context.Response.WriteAsync("event: tick\n");
        await context.Response.WriteAsync($"data: {payload}\n\n"); // سطر فارغ ينهي الحدث
        await context.Response.Body.FlushAsync();
        await Task.Delay(1000, context.RequestAborted);
    }
});

app.Run();
```

## مثال عميل متصفح (JavaScript — EventSource)
```html
<script>
  const es = new EventSource("/sse", { withCredentials: true }); // يسمح بالكوكيز إن لزم
  es.addEventListener("tick", (e) => {
    const data = JSON.parse(e.data);
    console.log("tick:", data.tick, "at:", data.at);
  });
  es.onerror = () => {
    console.log("SSE error — سيعيد الاتصال تلقائيًا");
    // المتصفح يعيد الاتصال تلقائيًا؛ يمكن ضبط backoff على الخادم عبر retry:
    // مثال: إرسال "retry: 5000\n" من الخادم لضبط 5 ثوانٍ
  };
</script>
```

الإخراج المتوقع (مختصر):  
- الخادم يبث حدثًا كل ثانية باسم `tick` ويحمل JSON.  
- الكونسول في المتصفح يطبع القيم المستلمة بشكل لحظي.  

## خطوات عملية لاستخدام SSE بفعالية
- اضبط الاستجابة: `text/event-stream`, `Cache-Control: no-cache`, و**Flush** بعد كل حدث.  
- أرسل **تعليقات keep-alive** دورية (`: ping\n\n`) لمنع إغلاق الوسطاء.  
- استخدم **`id:`** مع **Last-Event-ID** لدعم الاستئناف بعد الانقطاع.  
- للأمان: **HTTPS** دائمًا، تحقّق من **Auth** (كوكي/Token) واسم الأصل (Origin) عند اللزوم.  
- وسّع أفقيًا عبر **Pub/Sub** خلف الخوادم و**Sticky Sessions** إذا احتجت حالة اتصال.  
- راقب: عدد الاتصالات، معدل الأحداث، زمن التأخير، وإخفاقات إعادة الاتصال.

## أخطاء شائعة
- إرسال JSON خام بدون صيغة SSE الصحيحة (نسيان `data:` و**السطر الفارغ**).  
- الاعتماد على **CORS مُعطّل** أو نسيان تفعيل **Credentials** إذا كانت Auth بالكوكيز.  
- بث رسائل ضخمة دون **تقسيم منطقي** أو دون keep-alive → الاتصالات تُغلق عبر Proxies.  
- استخدام SSE حيث يتطلب التطبيق **اتجاهين** (خادم↔عميل) — حينها **WebSocket** أنسب.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [HTTP](http.md) | طلب/استجابة متقطّعة | بسيط ومتزامن؛ لا بث مستمر |
| [WebSocket](websocket.md) | قناة ثنائية الاتجاه دائمة | فورية واتجاهان؛ أعقد قليلًا من SSE |
| **Server-Sent Events** | **بث أحادي الاتجاه من الخادم للعميل** | **نصي فقط؛ اتصال HTTP مستمر؛ إعادة اتصال تلقائية** |

## ملخص الفكرة  
**SSE** يمنحك بثًا لحظيًا **من الخادم إلى العميل** فوق HTTP بسهولة وبدون تعقيد WebSocket.  
إن أردت **اتجاهين** استخدم WebSocket، وإن أردت **بثًا بسيطًا** فـ **SSE** خيار عملي وفعّال.
