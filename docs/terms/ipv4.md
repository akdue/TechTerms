# **IPv4**

## الترجمة الحرفية  
**Internet Protocol version 4 (IPv4)** — **إصدار 4 من بروتوكول الإنترنت** (عناوين 32-بت بصيغة نقاط).

## الوصف العربي المختصر  
نظام عنونة شبكي بطول **32 بت**. يعرّف **عناوين مضيفين**، و**شبكات فرعية (CIDR)**، ويدعم **التوجيه** و**الإذاعة (Broadcast)**.  
صيغة الكتابة: `a.b.c.d` مثل `192.168.1.10`. نطاق العناوين محدود، لذا يُستخدم **NAT** بكثرة.

## الشرح المبسّط  
- العنوان يقسم إلى **شبكة** + **مضيف** حسب القناع/السابقة (`/24` مثلًا).  
- **خاص (Private)** للاستخدام الداخلي: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.  
- **عام (Public)** على الإنترنت.  
- **Broadcast** = آخر عنوان بالشبكة، و**Network** = أول عنوان.  
- **NAT** يترجم الخاص ↔ العام لتوفير العناوين.

## تشبيه  
مجمّع سكني: أرقام **العمارة** (جزء الشبكة) ثم **الشقة** (جزء المضيف).  
آخر شقة مخصّصة للإذاعة العامة في العمارة (Broadcast).

## مثال كود C# بسيط (تحليل IPv4 + حساب شبكة/Broadcast)
```csharp
using System;
using System.Linq;
using System.Net;
using System.Net.Sockets;

static class IPv4Util
{
    public static bool TryParseIPv4(string s, out IPAddress ip)
        => IPAddress.TryParse(s, out ip!) && ip.AddressFamily == AddressFamily.InterNetwork;

    public static bool IsPrivate(IPAddress ip)
    {
        var b = ip.GetAddressBytes();
        return (b[0] == 10)
            || (b[0] == 172 && b[1] >= 16 && b[1] <= 31)
            || (b[0] == 192 && b[1] == 168)
            || (b[0] == 127)                            // Loopback
            || (b[0] == 169 && b[1] == 254);            // Link-local
    }

    public static (IPAddress network, IPAddress broadcast, IPAddress? firstHost, IPAddress? lastHost)
        Calc(string ipStr, int prefix)
    {
        if (!TryParseIPv4(ipStr, out var ip) || prefix < 0 || prefix > 32)
            throw new ArgumentException("Bad IPv4 or prefix");

        uint ipU = ToUInt32(ip);
        uint mask = prefix == 0 ? 0u : 0xFFFFFFFFu << (32 - prefix);
        uint net  = ipU & mask;
        uint bcast = net | ~mask;

        IPAddress? first = null, last = null;
        if (prefix <= 30) { first = FromUInt32(net + 1); last = FromUInt32(bcast - 1); }

        return (FromUInt32(net), FromUInt32(bcast), first, last);
    }

    static uint ToUInt32(IPAddress ip)
    {
        var bytes = ip.GetAddressBytes();               // network order
        if (BitConverter.IsLittleEndian) Array.Reverse(bytes);
        return BitConverter.ToUInt32(bytes, 0);
    }
    static IPAddress FromUInt32(uint v)
    {
        var bytes = BitConverter.GetBytes(v);
        if (BitConverter.IsLittleEndian) Array.Reverse(bytes);
        return new IPAddress(bytes);
    }
}

class Program
{
    static void Main()
    {
        var ip = "192.168.1.42";
        var (net, bcast, first, last) = IPv4Util.Calc(ip, 24);

        Console.WriteLine($"{ip}/24");
        Console.WriteLine($"Network   = {net}");        // 192.168.1.0
        Console.WriteLine($"Broadcast = {bcast}");      // 192.168.1.255
        Console.WriteLine($"FirstHost = {first}");      // 192.168.1.1
        Console.WriteLine($"LastHost  = {last}");       // 192.168.1.254

        IPv4Util.TryParseIPv4(ip, out var ipObj);
        Console.WriteLine(IPv4Util.IsPrivate(ipObj));   // True
    }
}
```
**الإخراج المتوقع (مختصر):**  
يعرض عنوان الشبكة، البث، أوّل/آخر مضيف، ويُظهر أنّ العنوان خاص.

## خطوات عملية للتعامل مع IPv4
- خطّط **CIDR** واضحًا (مثل `/24` لكل شبكة قسم) وتجنّب **تداخل الشبكات**.  
- احجز مدى **DHCP** منفصلًا عن العناوين الثابتة.  
- للإنترنت: اضبط **NAT/Port Forwarding** وافتـح المنافذ المطلوبة في الجدار الناري.  
- راقب الاستهلاك؛ عند ضغط العناوين فكّر في **IPv6** أو تقسيم شبكي أدق.  
- استخدم أدوات التحقق:  
  - Windows: `ipconfig`, `route print`  
  - Linux/macOS: `ip addr`, `ip route`  

## أخطاء شائعة
- خلط أقنعة (`/24` مقابل قناع 255.255.0.0).  
- توزيع عناوين ثابتة داخل مدى **DHCP** → تعارض.  
- افتراض وصول عناوين **خاصة** من الإنترنت بدون NAT.  
- نسيان **قواعد الجدار الناري** أو إعداد **PTR** للبريد الصادر.  
- استخدام `/31` أو `/32` دون فهم خصائصها (لا مضيفين تقليديين).

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [IP Address](ip-address.md) | مُعرّف شبكة لمضيف/واجهة | إطار عام لنسختَي البروتوكول |
| **IPv4** | **عناوين 32-بت بصيغة نقاط** | **مساحة محدودة؛ يعتمد على NAT؛ شائع جدًا** |
| [IPv6](ipv6.md) | عناوين 128-بت بصيغة ستّ عشرية | مساحة ضخمة؛ ميزات حديثة (SLAAC, بدون NAT غالبًا) |

## ملخص الفكرة  
**IPv4** يزوّدك بعناوين 32-بت لإدارة الشبكات الفرعية وتوجيه الرزم.  
خطّط CIDR بوضوح، افصل DHCP عن الثوابت، واضبط NAT/Firewall جيدًا—وانتقل تدريجيًا لـ **IPv6** عند الحاجة.
