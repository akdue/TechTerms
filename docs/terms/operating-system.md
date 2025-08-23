# **Operating System**

## الترجمة الحرفية  
**Operating System (OS)** — **نظام التشغيل**

## الوصف العربي المختصر  
برنامج أساسي يدير **العتاد** ويقدّم **خدمات** للتطبيقات: **عمليات/خيوط**، **ذاكرة**، **ملفات**، **شبكة**، **سائقو أجهزة**، و**تصاريح**.

## الشرح المبسّط  
- طبقة بين **البرنامج** و**العتاد**.  
- يختار أي مهمة تعمل الآن (**مُجدول**).  
- يعزل البرامج ويمنحها ذاكرة وملفات آمنة (**نمط مستخدم/نواة**).  
- يوفّر **استدعاءات نظام** (Syscalls) للوصول المنضبط للموارد.

## تشبيه  
نظام التشغيل هو **مدير المبنى**: ينظّم الكهرباء والمصاعد والأمن.  
المتاجر (تطبيقاتك) تطلب خدماته بدل أن تتعامل مع الأسلاك مباشرة.

---

## مثال كود C# عملي (استكشاف OS + استدعاءات خاصّة بنظام)

```csharp
// .NET 8/9
// مثال يُظهر: معلومات المنصّة، PID، زمن عمل النظام (uptime)، أمر نظامي حسب OS.
// مع استدعاء API خاص بويندوز (GetTickCount64) وبديل لينكس/ماك.

using System;
using System.Diagnostics;
using System.IO;
using System.Runtime.InteropServices;
using System.Threading;

class OSPlayground
{
    // Windows: زمن منذ الإقلاع (ms)
    [DllImport("kernel32.dll")]
    static extern ulong GetTickCount64();

    static void Main()
    {
        // 1) معلومات عامة
        Console.WriteLine($"OS: {RuntimeInformation.OSDescription}");
        Console.WriteLine($"Arch: {RuntimeInformation.ProcessArchitecture}");
        Console.WriteLine($"PID: {Environment.ProcessId}");

        // 2) زمن تشغيل النظام (Uptime)
        if (OperatingSystem.IsWindows())
        {
            var ms = GetTickCount64();
            Console.WriteLine($"Uptime: {TimeSpan.FromMilliseconds(ms)} (Win API)");
        }
        else
        {
            // لينكس: /proc/uptime (ثوانٍ) — ماك: fallback بـ Environment.TickCount64
            try
            {
                var first = File.ReadAllText("/proc/uptime").Split(' ', StringSplitOptions.RemoveEmptyEntries)[0];
                var sec = double.Parse(first, System.Globalization.CultureInfo.InvariantCulture);
                Console.WriteLine($"Uptime: {TimeSpan.FromSeconds(sec)} (/proc/uptime)");
            }
            catch
            {
                Console.WriteLine($"Uptime ~ {TimeSpan.FromMilliseconds(Environment.TickCount64)} (approx)");
            }
        }

        // 3) تشغيل أمر نظامي آمن (Platform-aware)
        var psi = OperatingSystem.IsWindows()
            ? new ProcessStartInfo("cmd", "/c dir") { RedirectStandardOutput = true }
            : new ProcessStartInfo("/bin/ls", "-la") { RedirectStandardOutput = true };

        using var p = Process.Start(psi)!;
        Console.WriteLine("--- Shell listing (first 3 lines) ---");
        for (int i = 0; i < 3 && !p.StandardOutput.EndOfStream; i++)
            Console.WriteLine(p.StandardOutput.ReadLine());

        // 4) أولوية خيط التنفيذ (جدولة مبسّطة داخل العملية)
        Thread.CurrentThread.Priority = ThreadPriority.AboveNormal;
        Console.WriteLine($"Thread Priority: {Thread.CurrentThread.Priority}");

        // 5) مسارات متوافقة عبر Path.Combine
        var home = Environment.GetFolderPath(Environment.SpecialFolder.UserProfile);
        var appData = Path.Combine(home ?? ".", ".myapp");
        Console.WriteLine($"AppData: {appData}");
    }
}
```

**الإخراج المتوقع (مختصر):**  
- معلومات OS/المعمارية وPID.  
- Uptime من API مناسب.  
- 3 أسطر من مخرجات `dir`/`ls`.  
- أولوية الخيط ومسار بيانات التطبيق.

---

## خطوات عملية للتعامل مع نظام التشغيل
- حدّد **الهدف**: Windows/Linux/macOS و**المعمارية** (x64/ARM64).  
- استخدم **واجهات .NET القياسية** أولًا (`File`, `Path`, `Sockets`).  
- عند الحاجة لميزة خاصة بالنظام: أضِف **فروعًا محروسة** (OS checks) أو **P/Invoke** آمنًا.  
- لخدمات الخلفية:  
  - Windows: **Windows Service** + Event Viewer.  
  - Linux: **systemd** (`service`, `journalctl`).  
- للجدولة: **Task Scheduler** مقابل **cron**.  
- للأمان: إدارة **Users/Groups/ACLs**، جدار ناري، تحديثات.  
- للمراقبة: سجلات، مقاييس، تتبّع؛ أدوات (`top/htop`, `perfmon`, مراقبة حاويات).

---

## أخطاء شائعة
- افتراض **حساسية حالة الأحرف**/المسارات نفسها في كل OS (Windows غير حساس افتراضيًا).  
- ترميز ولغة دون تثبيت **Culture** مناسب → أرقام/تواريخ مضطربة.  
- إهمال **إشارات الإنهاء** (SIGTERM) في لينكس أو أحداث الإنهاء في ويندوز.  
- استخدام مسارات ثابتة أو فواصل `\` بدل `Path.Combine`.  
- الاعتماد على **NAT** كأمن كافٍ دون جدار ناري/تصاريح.  
- تشغيل أوامر Shell مباشرة بلا تعقيم مدخلات.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Operating System (OS)** | **إدارة العتاد والعمليات والذاكرة والملفات والشبكة** | **نواة + خدمات +ドرايفرات + واجهة Syscalls** |
| [Kernel](kernel.md) | قلب النظام يتعامل مع العتاد والذاكرة والجدولة | يعمل في **نمط النواة**؛ الطبقة الأدنى |
| Runtime (.NET/JVM) | تنفيذ الشيفرة المُدارة، GC، JIT | فوق OS؛ يعتمد على Syscalls |
| [Platform](platform.md) | بيئة تشغيل أشمل (OS + SDK/APIs/خدمات) | قد تشمل متجر وتوزيع |
| [Container](container.md) | عزل خفيف يشارك نواة OS | يعتمد على نواة المضيف (cgroups/namespaces) |

---

## ملخص الفكرة  
**نظام التشغيل** يوفّر البنية التحتية: جدولة، ذاكرة، ملفات، شبكة، أمان.  
استخدم واجهات .NET الموحدة، وافصل التكييفات الخاصة بالنظام بوضوح، واضبط الخدمة/السجلات/التصاريح لضمان تطبيق مستقر وآمن.
