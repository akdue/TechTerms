# **Platform**

## الترجمة الحرفية  
**Software/Computing Platform** — **منصّة برمجية/حوسبية**.

## الوصف العربي المختصر  
بيئة تنفيذ متكاملة تُوفّر **نظام تشغيل/عتاد** + **Runtime** + **واجهات ونُظُم** (SDK/APIs) وأحيانًا **خدمات سحابية**.  
تسمح لتطبيقاتك أن **تعمل** وتتكامل دون أن تعيد اختراع الأساسيات.

## الشرح المبسّط  
- المنصّة = **الأرضية** التي يقف عليها تطبيقك: OS + مكتبات + أدوات + أحيانًا متجر/توزيع.  
- أنواع شائعة: **منصّة نظام** (Windows/Linux/iOS/Android)، **منصّة Runtime** (.NET/JVM)، **منصّة سحابية** (Azure/AWS)، **منصّة أجهزة** (x64/ARM).  
- الفرق عن *Framework*: الإطار يحدّد **هيكل العمل** داخل التطبيق؛ المنصّة توفّر **بيئة التشغيل** حوله.

## تشبيه  
ابنِ مطعَمك داخل **مجمّع متكامل**: الكهرباء، المياه، الأمن، المساحات، اللوحات، الدفع.  
هذا المجمّع هو **المنصّة**. طريقة ترتيب المطعم من الداخل هي **الإطار**.

---

## مثال كود C# بسيط — التعرف على المنصّة والتفرّع الآمن
```csharp
using System;
using System.IO;
using System.Runtime.InteropServices;

class Program
{
    static void Main()
    {
        // معلومات المنصّة
        Console.WriteLine($"OS: {RuntimeInformation.OSDescription}");
        Console.WriteLine($"Arch: {RuntimeInformation.ProcessArchitecture}");
        Console.WriteLine($".NET: {RuntimeInformation.FrameworkDescription}");

        // تفرّع سلوكي حسب المنصّة (Cross-Platform مع فروق صغيرة)
        string home =
            OperatingSystem.IsWindows() ? Environment.GetFolderPath(Environment.SpecialFolder.UserProfile) :
            OperatingSystem.IsLinux()   ? Environment.GetEnvironmentVariable("HOME") ?? "/home" :
            OperatingSystem.IsMacOS()   ? Environment.GetEnvironmentVariable("HOME") ?? "/Users" :
            Environment.CurrentDirectory;

        // مسار متوافق عبر Path.Combine
        string appDir = Path.Combine(home ?? ".", ".myapp");
        Console.WriteLine($"App Dir: {appDir}");

        // مثال: أمر قصير خاص بويندوز (اختياري وبحراسة)
        if (OperatingSystem.IsWindows())
        {
            Console.WriteLine("Running on Windows — enable Windows-specific integration here.");
        }
    }
}
```
**الفكرة:** اكتب كودًا **عابرًا للمنصّات**، ثم أضف **تكييفات صغيرة** عند الحاجة (OS checks) مع استخدام واجهات قياسية (`Path`, `Environment`).

---

## خطوات عملية لاختيار/تبنّي منصّة
1. **حدد الجمهور وبيئة التشغيل**: أجهزة المستخدمين، OS، اتصال الشبكة، قيود الأمان.  
2. **قرّر نموذج النشر**: سطح مكتب/محمول/ويب/سحابة/حاويات.  
3. **اختر Runtime/SDK**: مثل **.NET** أو **JVM** أو **Node.js** وفق فريقك ومتطلباتك.  
4. **خدمات أساسية**: قاعدة بيانات، هوية، تخزين، مراسلة — وفّرها من المنصّة إن أمكن (PaaS).  
5. **قابلية النقل**: استهدف **Cross-Platform**، واجعل التفرّعات **محصورة وواضحة**.  
6. **الأمان والامتثال**: TLS/HSTS وCSP على الويب، أسرار في Vault، وسياسات وصول.  
7. **المراقبة والانتشار**: سجلات، قياسات، تتبّع؛ CI/CD؛ قنوات تحديث (Store/Packages/Containers).

---

## أخطاء شائعة
- خلط **Framework** بـ **Platform** → قرارات أدوات غير مناسبة.  
- كتابة مسارات/ملفات بأسلوب **منصّة واحدة** (فواصل المسارات، ترميزات) دون استخدام واجهات .NET الموحّدة.  
- تجاهل **المعماريات** (x64/ARM) أو **إعدادات الأمان** الخاصة بالمنصّة.  
- ربط التطبيق بخدمة سحابية بأسلوب **محكم** دون طبقة تجريد → صعوبة نقل المنصّة لاحقًا.  
- الاعتماد على **features** حصرية دون بدائل → يقلّ **Portability**.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [Framework](framework.md) | هيكل عمل داخل التطبيق (توجيه/DI/…)| الإطار **ينادي كودك** (IoC)؛ مثال: ASP.NET Core |
| **Platform** | **بيئة التشغيل والخدمات حول التطبيق** | **OS/Runtime/SDK/خدمات؛ نظام توزيع وتكامل** |
| Runtime | تنفيذ الشيفرة المُدارة/VM | مثل .NET CLR أو JVM؛ جزء من المنصّة |
| [Operating System](operating-system.md) | إدارة العتاد والعمليات والملفات | طبقة أساسية ضمن المنصّة |
| [PaaS](paas.md) | منصّة سحابية مُدارة للنشر والتشغيل | قواعد بيانات/هوية/مراسلة جاهزة |

---

## ملخص الفكرة  
**المنصّة** = البيئة التي تُشغّل تطبيقك وتزوّد خدماته الأساسية.  
ابنِ تطبيقك فوق منصّة **متوافقة مع جمهورك**، واجعل الكود **عابرًا للمنصّات** مع تكييفات موضعية واضحة.
