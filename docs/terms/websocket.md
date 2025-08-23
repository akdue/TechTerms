# **WebSocket**

## الترجمة الحرفية  
**WebSocket** — **مقبس الويب** (اتصال ثنائي الاتجاه دائم فوق HTTP بعد ترقية).

## الوصف العربي المختصر  
قناة **ثنائية الاتجاه (Full-Duplex)** تبقى **مفتوحة** بين العميل والخادم بعد **ترقية** أولية عبر HTTP.  
مناسبة للتطبيقات الفورية: **دردشة، إشعارات لحظية، بث حي، لوحات实时**.

## الشرح المبسّط  
- يبدأ بطلب HTTP عادي ثم **Upgrade: websocket** → يتحول الاتصال إلى WebSocket.  
- بعدها يمكن للطرفين **إرسال/استقبال** رسائل **نصية** أو **ثنائية** في أي وقت.  
- يدعم **Ping/Pong** للحفاظ على الاتصال و**إطارات** تقسم الرسائل الطويلة.  
- المخططان: `ws://` و`wss://` (آمن فوق TLS — مُستحسن في الإنتاج).

## تشبيه  
بدل إرسال رسالة بريدية لكل كلمة، تفتح **مكالمة هاتفية** دائمة:  
الطرفان يتحدثان متى شاءا دون انتظار طلب جديد كل مرة.

## مثال كود C# بسيط (خادم Echo + عميل)

### خادم ASP.NET Core (Echo)
```csharp
// Program.cs
using System.Net.WebSockets;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseWebSockets(new WebSocketOptions
{
    KeepAliveInterval = TimeSpan.FromSeconds(30)
});

app.Map("/ws/echo", async (HttpContext ctx) =>
{
    if (!ctx.WebSockets.IsWebSocketRequest) { ctx.Response.StatusCode = 400; return; }

    using var socket = await ctx.WebSockets.AcceptWebSocketAsync();
    var buffer = new byte[4 * 1024];

    while (true)
    {
        var result = await socket.ReceiveAsync(buffer, CancellationToken.None);
        if (result.MessageType == WebSocketMessageType.Close)
        {
            await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "bye", CancellationToken.None);
            break;
        }

        var segment = new ArraySegment<byte>(buffer, 0, result.Count);
        await socket.SendAsync(segment, result.MessageType, result.EndOfMessage, CancellationToken.None);
    }
});

app.Run();
```

### عميل .NET (ClientWebSocket)
```csharp
using System.Net.WebSockets;
using System.Text;

using var ws = new ClientWebSocket();
await ws.ConnectAsync(new Uri("ws://localhost:5000/ws/echo"), CancellationToken.None);

var send = Encoding.UTF8.GetBytes("hello ws");
await ws.SendAsync(send, WebSocketMessageType.Text, endOfMessage: true, CancellationToken.None);

var buffer = new byte[1024];
var res = await ws.ReceiveAsync(buffer, CancellationToken.None);
Console.WriteLine(Encoding.UTF8.GetString(buffer, 0, res.Count)); // hello ws

await ws.CloseAsync(WebSocketCloseStatus.NormalClosure, "done", CancellationToken.None);
```
> في الإنتاج استخدم `wss://` (TLS) وخلف **Proxy** يدعم WebSocket (مثل Nginx/HTTP/2+3).

## خطوات عملية لاستخدام WebSocket بفعالية
- صمّم **بروتوكول رسائل** واضحًا (نوع الرسالة، الحقول، النسخة).  
- فعّل **Keep-Alive** و**Timeouts** وتعامل مع **إعادة الاتصال** على العميل.  
- ضع **حدودًا للحجم/المعدل** (Backpressure) وتحقق من المدخلات.  
- أضِف **مصادقة** في الترقيـة (كوكي/توكن) و**تحقق Origin** للمتصفح.  
- راقب المقاييس: جلسات مفتوحة، تأخّر، معدلات الإرسال، أخطاء إغلاق.  
- خطّط لتوسعة أفقيّة عبر **Sticky Sessions** أو **Pub/Sub** خلف الخوادم.

## أخطاء شائعة
- الاعتماد على **CORS** لمتصفح WebSocket (لا ينطبق مباشرة) بدل **فحص Origin**.  
- نسيان التعامل مع **انقطاع الشبكة** وإعادة المحاولة التدريجية.  
- إرسال رسائل ضخمة دون **تجزئة/ضغط** أو دون Backpressure.  
- الاكتفاء بـ `ws://` على الإنترنت العام بدل `wss://`.

## جدول مقارنة مختصر

| المفهوم                                     | الغرض الرئيسي                     | ملاحظات مهمة                               |
| ------------------------------------------- | --------------------------------- | ------------------------------------------ |
| [HTTP](http.md)                             | طلب/استجابة متقطّعة                | بسيط وواسع؛ ليس ثنائي الاتجاه بشكل دائم    |
| **WebSocket**                               | **قناة ثنائية الاتجاه دائمة**     | **ترقية من HTTP؛ نص/ثنائي؛ مناسب للفورية** |
| [Server-Sent Events](server-sent-events.md) | بث أحادي الاتجاه من الخادم للعميل | أبسط من WS؛ الخادم→العميل فقط؛ نصّي         |

## ملخص الفكرة  
**WebSocket** يوفّر اتصالًا دائمًا ثنائي الاتجاه بعد ترقية HTTP، مثاليًا للتطبيقات الفورية.  
استخدم `wss://`، صمّم بروتوكول رسائلك، واهتم بالاستقرار (Keep-Alive/Backpressure/إعادة الاتصال).
