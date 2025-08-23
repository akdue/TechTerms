# **TCP/IP**

## الترجمة الحرفية  
**Transmission Control Protocol / Internet Protocol (TCP/IP)** — **مجموعة بروتوكولات الإنترنت** (طبقات وشبكات واتصال موثوق/غير موثوق).

## الوصف العربي المختصر  
عائلة بروتوكولات تُشغِّل الإنترنت:  
- **IP** للتوجيه والعناوين.  
- **TCP** لاتصال موثوق موجّه اتصالًا.  
- **UDP** لإرسال خفيف بدون ضمانات.  
وتعلوها بروتوكولات التطبيقات مثل **HTTP/DNS/SMTP**، وتحتها طبقات الربط مثل **Ethernet/Wi-Fi**.

## الشرح المبسّط  
- **نموذج الطبقات** (مبسّط):  
  **Link** (Ethernet/Wi-Fi) → **Internet** (IP, ICMP) → **Transport** (TCP/UDP) → **Application** (HTTP/DNS/…).
- **IP** يقرّر “إلى أي عنوان تذهب الرزمة”.  
- **TCP** يضمن الوصول بالترتيب ومن دون فقد (تتابع/إقرار/إعادة إرسال).  
- **UDP** يرسل بسرعة وبدون حمل إضافي (قد تضيع رُزم).  
- المنافذ (**Ports**) تميّز التطبيقات على نفس المضيف (80/443/53…).

## تشبيه  
شحن بضائع:  
- **IP** = طريق الشحن وعناوين البيوت.  
- **TCP** = خدمة توصيل مع تتبع وتأكيد استلام.  
- **UDP** = درّاجة سريعة بلا إيصال؛ أسرع لكن قد تفوت طردًا.

## مثال كود C# بسيط (خادوم/عميل TCP Echo)

```csharp
// dotnet add package System.Net.Sockets
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;

class TcpEcho
{
    public static async Task RunServer()
    {
        var listener = new TcpListener(IPAddress.Any, 5000);
        listener.Start();
        Console.WriteLine("TCP Echo server on 0.0.0.0:5000");

        while (true)
        {
            using var client = await listener.AcceptTcpClientAsync();
            _ = Handle(client); // لا تنتظر — عالج متوازياً
        }
    }

    static async Task Handle(TcpClient client)
    {
        using var stream = client.GetStream();
        var buf = new byte[1024];
        int n = await stream.ReadAsync(buf, 0, buf.Length);
        await stream.WriteAsync(buf, 0, n); // Echo
    }

    public static async Task RunClient(string msg = "hello tcp")
    {
        using var client = new TcpClient();
        await client.ConnectAsync("127.0.0.1", 5000);
        var s = client.GetStream();
        var data = Encoding.UTF8.GetBytes(msg);
        await s.WriteAsync(data, 0, data.Length);
        var buf = new byte[1024];
        int n = await s.ReadAsync(buf, 0, buf.Length);
        Console.WriteLine(Encoding.UTF8.GetString(buf, 0, n)); // hello tcp
    }

    static async Task Main()
    {
        // شغّل الخادم في عملية، ثم العميل في أخرى (أو عدّل لتشغيل كليهما هنا بالتوازي).
        // await RunServer();
        // await RunClient();
    }
}
```
**الإخراج المتوقع:** عند إرسال `"hello tcp"` من العميل، يرد الخادم بنفس النص.

> ملاحظة سريعة: استبدل TCP بـ **UDP** (SocketType.Dgram, ProtocolType.Udp) لإرسال خفيف بدون اتصال وضمانات.

## خطوات عملية لفهم/تشخيص TCP/IP
- اعرف **عنوان IP** و**قناع الشبكة/CIDR** و**العبّارة الافتراضية**.  
- اختبر الاتصال:  
  - **Ping/ICMP** لوصول الشبكة.  
  - **tracert / traceroute** لمسار التوجيه.  
  - **netstat / ss** للمنافذ المفتوحة.  
- تأكد من **الجدار الناري/NAT** وفتح المنافذ المطلوبة.  
- استخدم **DNS** لتحويل الأسماء إلى IP (A/AAAA).  
- راقب **RTT وفقد الرزم**؛ اضبط **TCP Keep-Alive** و**Timeouts** للتطبيقات الحسّاسة.  
- للتوسّع: فكّر في **Load Balancer** أو **VIP** و**Anycast**.

## أخطاء شائعة
- الخلط بين **TCP** و**HTTP** (HTTP يعمل فوق TCP؛ ليس نفس الشيء).  
- توقّع ضمانات من **UDP** (لا ترتيب/لا إعادة إرسال).  
- نسيان إعداد **IPv6** أو قواعد جدار ناري منفصلة له.  
- الاعتماد على **DNS** دون التفكير في **TTL/Caching** أثناء التغييرات.  
- افتراض أن **NAT** = أمان كافٍ (يجب ضبط جدار ناري صريح).  

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [IP Address](ip-address.md) | عنوان للتوجيه بين الشبكات | طبقة الإنترنت؛ لا يضمن التسليم وحده |
| [TCP](tcp.md) | اتصال موثوق مرتّب مع تدفّق | بطيء نسبيًا لكنه يضمن الإيصال/الترتيب |
| [UDP](udp.md) | إرسال خفيف بلا اتصال ولا ضمانات | مثالي للبث/الزمن الحقيقي (صوت/فيديو/لُعب) |
| **TCP/IP** | **مجموعة بروتوكولات الإنترنت (IP + TCP/UDP + تطبيقات)** | **نموذج طبقي: Link → Internet → Transport → Application** |

## ملخص الفكرة  
**TCP/IP** هو الإطار الذي يجعل الإنترنت يعمل: **IP** يوجّه، و**TCP/UDP** ينقلان،  
وتعلوه **بروتوكولات التطبيقات**. اختر **TCP** للموثوقية و**UDP** للزمن الحقيقي، واضبط الشبكة (DNS/Firewall/NAT) بوعي.
