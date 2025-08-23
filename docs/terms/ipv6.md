# **IPv6**

## الترجمة الحرفية  
**Internet Protocol version 6 (IPv6)** — **إصدار 6 من بروتوكول الإنترنت** (عناوين **128-بت** ستّ عشرية مفصولة بنقطتين).

## الوصف العربي المختصر  
نظام عنونة حديث بمساحة هائلة.  
لا يوجد **Broadcast**؛ يستخدم **Multicast** و**Anycast**.  
يدعم **SLAAC** و**NDP** لتكوين العناوين تلقائيًا.  
أمثلة: `2001:db8::1`, `fe80::1234` (رابط-محلي).

## الشرح المبسّط  
- الكتابة: ثمانية مقاطع ستّ عشرية مفصولة `:` مع ضغط الأصفار (`::`).  
- الشبكات عادةً **/64** للمضيفين.  
- أنواع مهمة:
  - **Global Unicast**: `2000::/3` (على الإنترنت).  
  - **Link-Local**: `fe80::/10` (داخل الوصلة فقط).  
  - **Unique Local (ULA)**: `fc00::/7`.  
  - **Loopback**: `::1`.  
- غالبًا **بدون NAT**: توصيل طرف-لطرف أسهل، لكن ضبط الجدار الناري ضروري.

## تشبيه  
انتقلت من شارع قديم مزدحم إلى **مدينة جديدة واسعة**.  
عناوين كثيرة، طرق واسعة، إشارات أوضح (Multicast/Anycast بدل Broadcast).

## مثال كود C# بسيط (تحقّق/تصنيف IPv6 + مستمع ثنائي النمط)
```csharp
using System;
using System.Net;
using System.Net.Sockets;

class Program
{
    static void Main()
    {
        // 1) تحليل عنوان IPv6 وتمييزه
        string s = "2001:db8::42";
        if (IPAddress.TryParse(s, out var ip) && ip.AddressFamily == AddressFamily.InterNetworkV6)
        {
            Console.WriteLine($"{ip} = IPv6");
            Console.WriteLine($"LinkLocal?   {ip.IsIPv6LinkLocal}");
            Console.WriteLine($"UniqueLocal? {ip.IsIPv6UniqueLocal}");
            Console.WriteLine($"Multicast?   {ip.IsIPv6Multicast}");
            Console.WriteLine($"Teredo?      {ip.IsIPv6Teredo}");
        }

        // 2) مستمع TCP IPv6 يستقبل IPv6 وIPv4 (Dual-Stack) على المنفذ 5000
        var listener = new TcpListener(IPAddress.IPv6Any, 5000);
        listener.Server.DualMode = true;    // يسمح بعناوين IPv4-mapped أيضًا (إن دعمها النظام)
        listener.Start();
        Console.WriteLine("Listening on [::]:5000 (dual-stack). Press Enter to quit…");
        Console.ReadLine();
        listener.Stop();
    }
}
```
الإخراج المتوقع (مختصر):  
- يطبع خصائص العنوان (`LinkLocal/UniqueLocal/…`).  
- يبدأ مستمعًا على `[::]:5000` يقبل IPv6 وIPv4-mapped.

## خطوات عملية للتعامل مع IPv6
- استخدم **/64** لشبكات المضيفين؛ لا تُجزّئ أصغر إلا لروابط نقطية/حالات خاصة.  
- فعّل **DNS AAAA** للأسماء العامة، ووفّر **PTR** في `ip6.arpa` إذا لزم.  
- اضبط **جدار ناري IPv6** صراحةً (لا تعتمد على NAT للحماية).  
- اختبر بالأدوات:
  - Windows: `ping -6`, `tracert -6`, `Get-NetIPAddress`.  
  - Linux/macOS: `ping6` أو `ping -6`, `ip -6 addr`, `ip -6 route`.  
- عند الخوادم: فعّل **Dual-Stack** فترة الانتقال (IPv4 + IPv6).  
- راقب **SLAAC/Privacy Extensions** على العملاء (عناوين مؤقتة).

## أخطاء شائعة
- استخدام **NAT** كما في IPv4 بدل الاعتماد على التوجيه + الجدار الناري.  
- نسيان فتح قواعد الجدار الناري لـ **IPv6**.  
- التعامل مع الشبكة كأنها **/24**؛ في IPv6 المضيفون عادة **/64**.  
- عدم إنشاء سجلات **AAAA** أو نسيان **PTR**.  
- افتراض وجود **Broadcast** (غير موجود؛ استخدم Multicast).  
- تجاهل **IPv6** في المراقبة والـ logging.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [IP Address](ip-address.md) | مُعرّف شبكة لمضيف/واجهة | إطار عام لنسختَي البروتوكول |
| [IPv4](ipv4.md) | عناوين 32-بت (نقاط) | مساحة محدودة؛ يعتمد على NAT بكثرة |
| **IPv6** | **عناوين 128-بت (ستّ عشرية مفصولة بنقطتين)** | **مساحة ضخمة؛ لا Broadcast؛ يفضّل /64 وSLAAC** |

## ملخص الفكرة  
**IPv6** يقدّم مساحة عناوين هائلة وتوجّهًا حداثيًا (Multicast، SLAAC، Anycast).  
اعمل بـ **/64**، أضف **AAAA/PTR**، واضبط **جدار IPv6** بوضوح—وغالبًا دون حاجة إلى NAT.
