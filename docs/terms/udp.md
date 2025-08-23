# **UDP**

## الترجمة الحرفية  
**User Datagram Protocol (UDP)** — **بروتوكول مخططات المستخدم** (إرسال خفيف بلا اتصال).

## الوصف العربي المختصر  
بروتوكول نقل **بدون اتصال**، **بدون ضمانات** للتسليم أو الترتيب أو التكرار.  
سريع وخفيف، يحافظ على **حدود الرسائل (Datagrams)**، ومناسب لـ **الزمن الحقيقي**: صوت/فيديو/ألعاب، **البث** و**متعدد الإرسال**.

## الشرح المبسّط  
- لا توجد مصافحة ولا جلسة: ترسل **رُزَمًا مستقلّة**؛ قد تضيع أو تتكرّر أو تصل خارج الترتيب.  
- التطبيق مسؤول عن **إعادة الإرسال/الترقيم/التحقق** إن احتاج الموثوقية.  
- يدعم **Broadcast** (`255.255.255.255`) و**Multicast** (`239.0.0.0/8`, `ff00::/8` في IPv6).  
- تجنّب تجزئة IP: **أبقِ الحمولة دون MTU** (≈ 1200 بايت عمليًا على الإنترنت).

## تشبيه  
**رسائل بريد عادي** بلا إيصال استلام. سريعة ورخيصة، لكن قد تضيع أو تصل متأخرة.  
إن أردت الضمانات، عليك إضافة **أرقام تتبّع** وقواعدك فوقها.

---

## مثال كود C# بسيط (خادم/عميل UDP Echo)

### خادم UDP (Echo)
```csharp
// dotnet add package System.Net.Sockets
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;

class UdpEchoServer
{
    public static void Main()
    {
        using var udp = new UdpClient(new IPEndPoint(IPAddress.Any, 6000)); // يستمع على :6000
        Console.WriteLine("UDP Echo on 0.0.0.0:6000");

        while (true)
        {
            IPEndPoint? remote = null;
            byte[] data = udp.Receive(ref remote); // قد تأتي رسائل متتابعة أو لا تأتي
            string text = Encoding.UTF8.GetString(data);
            Console.WriteLine($"[{remote}] {text}");

            // أعد نفس الحمولة (Echo)
            udp.Send(data, data.Length, remote);
        }
    }
}
```

### عميل UDP
```csharp
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;

class UdpEchoClient
{
    public static void Main()
    {
        using var udp = new UdpClient();
        udp.Client.ReceiveTimeout = 3000; // مهلة استقبال بالمللي ثانية

        var server = new IPEndPoint(IPAddress.Loopback, 6000);

        foreach (var msg in new[] { "hello", "udp", "world" })
        {
            byte[] payload = Encoding.UTF8.GetBytes(msg);
            udp.Send(payload, payload.Length, server);

            // استقبل الرد (قد يضيع؛ تعامل مع الاستثناءات/المحاولة لاحقًا)
            try
            {
                var from = new IPEndPoint(IPAddress.Any, 0);
                byte[] data = udp.Receive(ref from);
                Console.WriteLine(Encoding.UTF8.GetString(data)); // echo:...
            }
            catch (SocketException)
            {
                Console.WriteLine("timeout/no reply");
            }
        }
    }
}
```

> ملاحظات:
> - **UDP يحافظ على حدود الرسالة** (عكس TCP الذي هو **Stream**).  
> - الرسائل قد تضيع، تتكرّر، أو تصل خارج الترتيب—أضِف **Sequence/Ack** لو أردت الاعتمادية.  
> - للبث المتعدد: استخدم `udp.JoinMulticastGroup(IPAddress.Parse("239.1.1.1"));` واضبط `TTL`.

---

## خطوات عملية للعمل مع UDP
- صمّم **بروتوكول تطبيق** بسيطًا: `type | seq | payload` مع **Sequence/Ack** و**Retry/Backoff** عند الحاجة.  
- **حجّم الحمولة** < MTU (≈ 1200 بايت) لتجنّب تجزئة IP وفقدان الرسالة عند إسقاط جزء.  
- عالج **Out-of-Order/Duplicate** عبر أرقام تسلسلية ونافذة استقبال.  
- استخدم **Time-outs** و**إعادة المحاولة** و**إسقاط الحزم القديمة**.  
- للبث/المتعدد:  
  - **Broadcast** قد يُحظر على بعض الشبكات.  
  - **Multicast** يحتاج تمكينًا من الموجّهات وضبط `TTL` وعضوية المجموعة.  
- عبر NAT/الإنترنت: فكّر في **STUN/ICE** أو نفق آمن/وسيط.

## أخطاء شائعة
- افتراض أن UDP “سيوصل دائمًا” → تجاهل فقدان الحزم والتكرار.  
- إرسال رسائل ضخمة تتجزّأ على مستوى IP → فقدان مرتفع.  
- نسيان **التحكم في المعدل/Backpressure** في الطرف المرسل تحت الضغط.  
- استخدام Broadcast بلا حذر → ضجيج شبكي/حظر من الأجهزة.  
- مقارنة UDP بـ TCP من حيث الضمانات؛ هما لهدفين مختلفين.

## جدول مقارنة مختصر

| المفهوم         | الغرض الرئيسي                                        | ملاحظات مهمة                                             |
| --------------- | ---------------------------------------------------- | -------------------------------------------------------- |
| [TCP](tcp.md)   | اتصال **موثوق** مرتّب (Stream)                        | مصافحة/إقرار/إعادة إرسال؛ أبطأ نسبيًا                     |
| **UDP**         | **إرسال خفيف بلا اتصال (Datagrams)**                 | **لا ضمان تسليم/ترتيب/تفريد**؛ ممتاز للزمن الحقيقي والبث |
| [QUIC](quic.md) | نقل موثوق متعدد التدفقات فوق **UDP** مع **TLS** مدمج | يقلّل RTT ويحل مشاكل Head-of-line في TCP                  |

## ملخص الفكرة  
**UDP** = سرعة وخفّة وحدود رسائل، لكن **لا ضمانات**.  
إن احتجت الموثوقية أضِفها على طبقة التطبيق أو استخدم **QUIC/TCP**؛  
وللتطبيقات الحيّة (صوت/فيديو/ألعاب) غالبًا يكون **الاختيار الأمثل**.
