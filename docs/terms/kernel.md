# **Kernel**

## الترجمة الحرفية  
**Operating System Kernel** — **نواة نظام التشغيل**

## الوصف العربي المختصر  
المكوّن **الأدنى امتيازًا** داخل نظام التشغيل.  
يدير **العمليات والخيوط**، **الذاكرة الافتراضية**، **السواقة/الأجهزة**، **الملفات والشبكة**، ويتعامل مع **المقاطعات** و**استدعاءات النظام (Syscalls)**.

## الشرح المبسّط  
- **نمط نواة (Kernel Mode)**: صلاحيات كاملة للوصول للعتاد.  
- **نمط مستخدم (User Mode)**: التطبيقات تعمل بصلاحيات محدودة؛ تطلب خدمات عبر **Syscalls**.  
- **المُجدول (Scheduler)** يقرّر من يعمل الآن.  
- **مدير الذاكرة** يخصص صفحات، يحمي العناوين، وينقلها (Paging).  
- **سواقات** تجعل الأجهزة (قرص/شبكة/USB) قابلة للاستخدام عبر واجهات موحّدة.

## تشبيه  
**برومانا** المبنى: ينسّق المصاعد، الكهرباء، الأمن.  
المكاتب (التطبيقات) لا تلمس الأسلاك مباشرة؛ يطلبون الخدمة عبر مكتب الاستقبال (Syscall).

---

## مثال كود C# بسيط — رؤية “أثر” النواة من وضع المستخدم
> نستدعي وظائف نظامية مختلفة:  
> - على ويندوز: `GetTickCount64` (زمن منذ الإقلاع).  
> - على لينكس/ماك: `uname` من **libc** (معلومات النواة).  
> نعرض أيضًا **الأولوية/المعالج** (قرارات المُجدول) وقراءة ملف (Syscalls فتح/قراءة).

```csharp
// .NET 8/9
using System;
using System.Diagnostics;
using System.IO;
using System.Runtime.InteropServices;
using System.Threading;

class KernelPeek
{
    // Windows API: زمن منذ الإقلاع (ms) — يمر عبر طبقة النظام إلى النواة
    [DllImport("kernel32.dll")] static extern ulong GetTickCount64();

    // Unix libc: uname — معلومات عن النواة
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Ansi)]
    struct Utsname
    {
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 65)] public string sysname;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 65)] public string nodename;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 65)] public string release;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 65)] public string version;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 65)] public string machine;
    }
    [DllImport("libc")] static extern int uname(ref Utsname uts);

    static void Main()
    {
        Console.WriteLine($"OS: {RuntimeInformation.OSDescription}");
        Console.WriteLine($"Arch: {RuntimeInformation.ProcessArchitecture}");
        Console.WriteLine($"PID: {Environment.ProcessId}");

        if (OperatingSystem.IsWindows())
        {
            var up = TimeSpan.FromMilliseconds(GetTickCount64());
            Console.WriteLine($"Uptime (kernel): {up}");
        }
        else
        {
            var u = new Utsname();
            if (uname(ref u) == 0)
                Console.WriteLine($"Kernel: {u.sysname} {u.release} ({u.machine})");
        }

        // جدولة: أولوية الخيط وتقارب المعالجات
        Thread.CurrentThread.Priority = ThreadPriority.AboveNormal;
        var proc = Process.GetCurrentProcess();
        Console.WriteLine($"Thread Priority: {Thread.CurrentThread.Priority}");
        Console.WriteLine($"Processor Affinity (mask): 0x{proc.ProcessorAffinity.ToInt64():X}");

        // I/O ملف — تحتها Syscalls: open/read/close
        var tmp = Path.GetTempFileName();
        File.WriteAllText(tmp, "hello kernel");
        var txt = File.ReadAllText(tmp);
        Console.WriteLine($"Read: {txt}");
        File.Delete(tmp);

        Console.WriteLine("Tip: راقب الـ Syscalls عبر strace/ltrace (Linux) أو Process Monitor (Windows).");
    }
}
```

**الإخراج المتوقع (مختصر):**  
- وصف النظام والمعمارية وPID.  
- على ويندوز: زمن الإقلاع. على لينكس/ماك: إصدار النواة.  
- أولوية الخيط، قناع المعالجات، وقراءة ملفٍ تمت بنجاح.

---

## خطوات عملية لفهم/التعامل مع النواة بأمان
- **صمّم تطبيقك لوضع المستخدم**؛ لا حاجة لامتيازات عالية إلا لضرورة واضحة.  
- استخدم واجهات **عالية المستوى** أولًا (`File`, `Sockets`, `Process`)، فهي تغلف Syscalls بشكل آمن.  
- عند الحاجة لميزة خاصة بالنظام:  
  - استخدم **P/Invoke** بحذر، وحدّد المنصّة بـ `OperatingSystem.IsWindows()/IsLinux()`.  
  - وثّق البُنى (Structs) بدقة لتفادي أخطاء **Marshalling**.  
- راقب تفاعلاتك مع النواة:  
  - **Linux**: `strace -f -p <pid>`, `perf`, `dmesg`.  
  - **Windows**: Process Monitor/Explorer, ETW, Event Viewer.  
- للأداء: قلّل **Syscalls** الصغيرة المتكررة (Batching/Buffering)، ووازن بين **I/O Sync/Async**.  
- للأمان: أقلّ صلاحيات ممكنة (Least Privilege)، وإغلاق الموارد سريعًا، ومعالجة الأخطاء دون كشف تفاصيل النظام.

---

## أخطاء شائعة
- افتراض أن **TCP** يرسل “رسائل” ثابتة (هو **Stream**؛ النواة قد تقسّم/تجمّع).  
- خلط **وضع المستخدم/النواة** ومحاولة الوصول المباشر للأجهزة من التطبيق.  
- P/Invoke ببُنى خاطئة أو استدعاءات بلا تحقق من القيم المرجعة/`errno` → أعطال.  
- تجاهل اختلافات المنصّات (مسارات، صلاحيات، حدود الملفات) في الشيفرة المشتركة.  
- الاعتماد على **NAT** كأمن بدل إعداد **ACL/Firewall** على مستوى النواة.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [Operating System](operating-system.md) | مجموعة النواة + أدوات/خدمات المستخدم | واجهات، مدير حزم، أدوات، Shell |
| **Kernel** | **قلب النظام: جدولة/ذاكرة/أجهزة/Syscalls** | **أعلى امتيازًا؛ يتعامل مع المقاطعات والسواقات** |
| [Hypervisor](hypervisor.md) | تشغيل عدّة أنظمة فوق عتاد واحد (افتراضية) | طبقة تحت/بجانب النواة، تعزل أنظمة كاملة |
| [Firmware](firmware.md) | برمجيات منخفضة المستوى على اللوحة/الأجهزة | تقود الإقلاع (BIOS/UEFI) وتجهّز العتاد |

---

## ملخص الفكرة  
**النواة** هي القلب الذي يدير العتاد والموارد ويوفّر **Syscalls** للتطبيقات.  
اكتب كودك في **وضع المستخدم** باستخدام واجهات عالية المستوى، واستعمل الاستدعاءات المنخفضة بحذر،  
راقب الأداء/الأمان، وتذكّر اختلافات المنصّات عند الاقتراب من الحدود مع النواة.
