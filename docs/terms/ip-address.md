# **IP Address**

## الترجمة الحرفية  
**Internet Protocol Address** — **عنوان بروتوكول الإنترنت** (مُعرِّف رقمي لواجهة شبكة).

## الوصف العربي المختصر  
معرّف يُستخدم لتوجيه الرُّزم عبر الشبكات.  
يوجد **IPv4 (32-بت)** و**IPv6 (128-بت)**، وقد يكون العنوان **عامًا** أو **خاصًا**، **ثابتًا** أو **ديناميكيًا** (DHCP).

## الشرح المبسّط  
- يفيد لتحديد **من أين** وإلى **أين** تُرسل البيانات.  
- **IPv4** بصيغة نقاط: `192.168.1.10`.  
- **IPv6** بصيغة ست عشريّة: `2001:db8::1`.  
- عناوين **خاصّة** داخل الشبكات المحليّة (NAT)، و**عامّة** على الإنترنت.  
- **قناع الشبكة/CIDR** يحدّد جزء الشبكة وجزء المضيف (مثل: `192.168.1.0/24`).

## تشبيه  
مثل **عنوان بيتك** على الخريطة: يتيح لسيّارة التوصيل الوصول إليك.  
بعض العناوين داخل **مجمع سكني خاص** (خاصة)، وأخرى على **الشارع العام** (عامّة).

## مثال كود C# بسيط (قراءة IPv4/IPv6 + تمييز الخاص/العام)
```csharp
using System;
using System.Linq;
using System.Net;
using System.Net.Sockets;

class Program
{
    static void Main()
    {
        // 1) عناوين الجهاز الحالي
        var host = Dns.GetHostName();
        var entry = Dns.GetHostEntry(host);

        foreach (var ip in entry.AddressList)
        {
            string ver = ip.AddressFamily == AddressFamily.InterNetwork ? "IPv4"
                      : ip.AddressFamily == AddressFamily.InterNetworkV6 ? "IPv6"
                      : "Other";

            Console.WriteLine($"{ip} [{ver}] {(IsPrivateIPv4(ip) ? "(Private)" : "")}");
        }

        // 2) تحليل نصّ إلى IP ومعرفة النسخة
        if (IPAddress.TryParse("192.168.1.10", out var parsed))
        {
            Console.WriteLine($"Parsed: {parsed} -> {(parsed.AddressFamily == AddressFamily.InterNetwork ? "IPv4" : "IPv6")}");
        }
    }

    // تحقّق بسيط لمدى العناوين الخاصّة في IPv4 (RFC1918) + loopback/link-local
    static bool IsPrivateIPv4(IPAddress ip)
    {
        if (ip.AddressFamily != AddressFamily.InterNetwork) return false;
        var b = ip.GetAddressBytes();

        // 10.0.0.0/8
        if (b[0] == 10) return true;

        // 172.16.0.0/12  => 172.16.0.0 - 172.31.255.255
        if (b[0] == 172 && b[1] >= 16 && b[1] <= 31) return true;

        // 192.168.0.0/16
        if (b[0] == 192 && b[1] == 168) return true;

        // 127.0.0.0/8 loopback
        if (b[0] == 127) return true;

        // 169.254.0.0/16 link-local (APIPA)
        if (b[0] == 169 && b[1] == 254) return true;

        return false;
    }
}
```
**الإخراج المتوقع (مختصر):**  
- يسرد عناوين الجهاز (IPv4/IPv6)، ويعلّم الخاصّة بـ `(Private)`.  
- يطبع نتيجة تحليل نصّ عنوان.

## خطوات عملية للتعامل مع عناوين IP
- اعرف بيئتك: هل تحتاج **عامًا** (على الإنترنت) أم **خاصًا** (داخل LAN/NAT)؟  
- استخدم **DHCP** للتوزيع الآلي، و**حجوزات DHCP** للأجهزة الثابتة.  
- عند إعداد خادم: حدّد **المنفذ** وافتحه في الجدار الناري ووجّه **NAT/Port Forwarding** إن لزم.  
- افهم **CIDR** (`/24`, `/32`, …) لتقسيم الشبكات.  
- استخدم أدوات النظام:  
  - Windows: `ipconfig`, `Get-NetIPAddress`  
  - Linux/macOS: `ip addr`, `ifconfig`  
- للمجالات: اربط الاسم بـ IP عبر **DNS** (A/AAAA) واختر **TTL** مناسبًا.

## أخطاء شائعة
- الخلط بين **DNS** (اسم) و**IP** (عنوان) — تعيين الاسم يتم عبر سجلات DNS.  
- افتراض أن عنوانًا خاصًا يمكن الوصول إليه من الإنترنت دون **NAT**.  
- تعيين **Static IP** داخل مدى DHCP دون حجز → تعارض عناوين.  
- نسيان **IPv6**: بعض الشبكات تُفعّله تلقائيًا؛ اختبر الأمان/الجدار الناري له أيضًا.  
- استخدام قناع/‏CIDR خاطئ → عزل غير مقصود أو اصطدام شبكات.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [IPv4](ipv4.md) | عناوين 32-بت (نقاط) | مساحة محدودة، استخدام واسع لـ NAT |
| **IP Address** | **مُعرّف شبكة لمضيف/واجهة** | **IPv4/IPv6؛ عام/خاص؛ ثابت/ديناميكي** |
| [IPv6](ipv6.md) | عناوين 128-بت (ست عشري) | مساحة ضخمة، دعم ميزات مثل SLAAC |

## ملخص الفكرة  
**عنوان IP** هو الهوية الشبكية التي تسمح بتوجيه البيانات إلى المضيف الصحيح.  
اختر بين **IPv4/IPv6**، افهم **عام/خاص** و**CIDR**، واضبط DHCP/NAT وDNS وفق حاجتك.
