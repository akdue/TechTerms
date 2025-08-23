# **TCP**

## الترجمة الحرفية  
**Transmission Control Protocol (TCP)** — **بروتوكول التحكّم بالنقل** (اتصال موثوق موجّه اتصالًا).

## الوصف العربي المختصر  
يوفّر قناة **موثوقة ومرتّبة** بين طرفين: **تجزيء/تتابع/إقرار/إعادة إرسال**،  
مع **ضبط تدفّق** (Windows) و**تحكّم ازدحام**. يُنشئ اتصالًا بـ **مصافحة ثلاثية** ويُنهيه بـ **FIN/ACK**.

## الشرح المبسّط  
- **اتصال**: عميل يفتح منفذًا مؤقّتًا → `SYN` → خادم يرد `SYN-ACK` → عميل `ACK`.  
- **الدفق**: ترى **Stream** بلا حدود رسائل؛ **التأطير مسؤولية التطبيق**.  
- **الموثوقية**: كل بايت له رقم تسلسلي، تُعاد الإرسال عند الفقد.  
- **الترتيب**: يصل بالترتيب (أو يُعاد ترتيبه داخليًا).  
- **الإغلاق**: تبادل **FIN/ACK** (وقد ترى حالات مثل `TIME_WAIT`).  
- تحسينات: **Nagle**, **Delayed ACK**, **Keep-Alive**, **MSS/PMTUD**.

## تشبيه  
شركة شحن مع **تتبّع لكل طرد** وإيصالات استلام.  
قد تتأخر قليلًا عن الدراجة السريعة (UDP)، لكنها **تضمن وصول المحتوى كاملًا وبالترتيب**.

---

## مثال كود C# عملي (خادم/عميل TCP + تأطير طول-أولًا)

> **ملحوظة مهمّة:** TCP **دفق** Bytes؛ لذلك نستخدم **طولًا أولًا** (length-prefix) لتمييز الرسائل.

### خادم Echo بسيط
```csharp
// dotnet add package System.Net.Sockets
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.IO;
using System.Threading.Tasks;

class TcpEchoServer
{
    public static async Task Main()
    {
        var listener = new TcpListener(IPAddress.Any, 5000);
        listener.Start(backlog: 100);
        Console.WriteLine("TCP Echo on 0.0.0.0:5000");

        while (true)
        {
            var client = await listener.AcceptTcpClientAsync();
            _ = Handle(client); // معالجة متوازية
        }
    }

    static async Task Handle(TcpClient client)
    {
        using (client)
        {
            client.NoDelay = true; // تعطيل Nagle لرسائل صغيرة قليلة الكمون
            client.Client.SetSocketOption(SocketOptionLevel.Socket, SocketOptionName.KeepAlive, true);

            using var ns = client.GetStream();
            using var br = new BinaryReader(ns, Encoding.UTF8, leaveOpen: true);
            using var bw = new BinaryWriter(ns, Encoding.UTF8, leaveOpen: true);

            try
            {
                while (true)
                {
                    int len = br.ReadInt32();                 // طول الرسالة (LE هنا)
                    if (len <= 0 || len > 1_000_000) break;   // حراسة
                    var data = br.ReadBytes(len);             // قد تأتي على دفعات
                    var msg = Encoding.UTF8.GetString(data);
                    Console.WriteLine($"[{client.Client.RemoteEndPoint}] {msg}");

                    var reply = Encoding.UTF8.GetBytes("echo:" + msg);
                    bw.Write(reply.Length);
                    bw.Write(reply);
                    bw.Flush();
                }
            }
            catch (EndOfStreamException) { /* أغلق الطرف الآخر */ }
            catch (IOException) { /* مهلة/انقطاع */ }
        }
    }
}
```

### عميل يرسل عدّة رسائل وينتظر الرد
```csharp
using System;
using System.Net.Sockets;
using System.Text;
using System.IO;
using System.Threading.Tasks;

class TcpEchoClient
{
    public static async Task Main()
    {
        using var client = new TcpClient();
        client.ReceiveTimeout = 5000; // ملّي ثانية
        client.SendTimeout = 5000;

        await client.ConnectAsync("127.0.0.1", 5000);

        using var ns = client.GetStream();
        using var br = new BinaryReader(ns, Encoding.UTF8, leaveOpen: true);
        using var bw = new BinaryWriter(ns, Encoding.UTF8, leaveOpen: true);

        foreach (var text in new[] { "hello", "tcp", "world" })
        {
            var payload = Encoding.UTF8.GetBytes(text);
            bw.Write(payload.Length);
            bw.Write(payload);
            bw.Flush();

            int len = br.ReadInt32();
            var data = br.ReadBytes(len);
            Console.WriteLine(Encoding.UTF8.GetString(data)); // echo:...
        }

        // إغلاق أنيق (اختياري): أبلغ الخادم أنني انتهيت من الإرسال
        client.Client.Shutdown(SocketShutdown.Send);
    }
}
```

> **التأطير**: اخترت Little-Endian هنا اختصارًا؛ الأفضل التوافق على **Network Order (Big-Endian)** بين الأنظمة.

---

## خطوات عملية لاستخدام/تشخيص TCP
- **صمّم التأطير** (Length-Prefix / Delimiter / بروتوكول ثنائي)؛ TCP لا يوفّر “رسائل”.  
- فعّل/عطّل **`NoDelay`** حسب نمطك:  
  - رسائل صغيرة فورية → `NoDelay = true`.  
  - دفعات كبيرة → اترك Nagle مفيدًا.  
- استخدم **مهلات** (Timeouts) و**إلغاء** (CancellationToken) و**Backoff** لإعادة المحاولة.  
- راقب الحالة عبر أدوات النظام:  
  - Windows: `netstat -ano`, `Get-NetTCPConnection`  
  - Linux/macOS: `ss -tna`, `tcpdump`/`wireshark`  
- انتبه للحالات: `SYN-SENT/RECEIVED`, `ESTABLISHED`, `FIN_WAIT`, `CLOSE_WAIT`, `TIME_WAIT`.  
- اضبط **backlog** و**Keep-Alive** و**حدود المخزن المؤقت** عند الضغط العالي.  
- مع الحاويات/السُحُب: افتح المنافذ، واضبط **Security Groups/NACL**، وتحقق من **NAT/Health Checks**.  

## أخطاء شائعة
- الاعتماد على TCP كأنه **يقدّم رسائل** → ظهور **Sticky/Coalesced** و**Partial Reads**.  
- تجاهل **TIME_WAIT** واعتباره “تسريب مقبض”؛ هو سلوك طبيعي لحماية الاتصالات السابقة.  
- إرسال ضخم دون **Backpressure** أو حدود؛ يملأ ذاكرة المتلقي.  
- توقّع زمن ثابت؛ **تحكّم الازدحام** قد يبطّئ النقل تحت الضغط.  
- نسيان قواعد **IPv6** أو الجدار الناري للمنافذ المطلوبة.  

## جدول مقارنة مختصر

| المفهوم         | الغرض الرئيسي                         | ملاحظات مهمة                                                 |
| --------------- | ------------------------------------- | ------------------------------------------------------------ |
| **TCP**         | **اتصال موثوق مرتّب (Stream)**         | **Handshake/ACK/إعادة إرسال/تحكّم ازدحام؛ يحتاج تأطير تطبيق** |
| [UDP](udp.md)   | إرسال خفيف بدون اتصال                 | لا ترتيب ولا إعادة إرسال؛ ممتاز للزمن الحقيقي                |
| [QUIC](quic.md) | نقل موثوق فوق **UDP** مع **TLS** مدمج | تسليم متعدّد التدفقات وتقليل الـ RTT                          |

## ملخص الفكرة  
**TCP** يمنحك **قناة موثوقة ومرتّبة**—لكن عليك **تحديد شكل الرسائل** وإدارة المهلات والازدحام.  
استعمل `NoDelay` بحكمة، ضع تأطيرًا واضحًا، وراقب حالات الاتصال لضمان أداء مستقر وقابلية صيانة.
