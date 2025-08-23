# **Virtual Machine (VM)**

## الترجمة الحرفية  
**Virtual Machine (VM)** — **آلة افتراضيّة** تشغّل نظام تشغيل كامل فوق **Hypervisor**.

## الوصف العربي المختصر  
بيئة حوسبة **معزولة** تقلّد جهازًا كاملًا: **CPU/RAM/قرص/شبكة**.  
تشغَّل على **Hypervisor** (Type-1/Type-2) وتستخدم **صورًا (Images)**، **لقطات (Snapshots)**، و**انتقالًا حيًّا (Live Migration)**.

## الشرح المبسّط  
- الهايبرفايزر يقسّم العتاد إلى **عدّة أجهزة منطقية**.  
- كل VM يملك **نظام تشغيل** خاص به وسواقات افتراضية/شبه افتراضية (virtio).  
- مناسب عندما تحتاج **عزلًا قويًا** أو **نظامًا كاملاً** (AD/DB/خدمات قديمة).  
- أثقل من الحاويات؛ لكن يمنحك **تحكّمًا تامًّا بالنظام**.

## تشبيه  
كشقق داخل برج واحد: لكل شقّة **أبواب وكهرباء ومطبخ** مستقل.  
الإدارة (Hypervisor) توزّع الموارد وتسمح بتبديل الشقة دون إخراج الأثاث (Live Migration).

---

## مثال كود C# 1 — إيقاف لطيف داخل VM (التعامل مع إشارة الإنهاء)
> مهم عند **إيقاف** VM من السحابة ليُنهي التطبيق أعماله ويغلق الاتصالات.

```csharp
// .NET 8/9
using System;
using System.Threading;
using System.Threading.Tasks;

class GracefulVm
{
    static async Task Main()
    {
        var cts = new CancellationTokenSource();
        Console.CancelKeyPress += (s, e) => { e.Cancel = true; cts.Cancel(); }; // Ctrl+C أو SIGINT
        AppDomain.CurrentDomain.ProcessExit += (s, e) => cts.Cancel();          // إيقاف/إعادة تشغيل VM

        Console.WriteLine("Running... press Ctrl+C or stop the VM.");
        try
        {
            while (!cts.IsCancellationRequested)
            {
                // منطق العمل
                await Task.Delay(1000, cts.Token);
                Console.Write(".");
            }
        }
        catch (OperationCanceledException) { /* متوقّع */ }

        // إنهاء لطيف: إغلاق اتصالات/Flush/حفظ حالة
        Console.WriteLine("\nShutting down gracefully...");
        await Task.Delay(500);
    }
}
```

## مثال كود C# 2 — التعرف على بيئة VM السحابيّة عبر **Metadata Service**
> آمن بمهلات قصيرة. يحاول **AWS** ثم **Azure** ثم **GCP**.

```csharp
// dotnet add package System.Net.Http.Json
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text.Json;
using System.Threading.Tasks;

class CloudMetadata
{
    static async Task Main()
    {
        var http = new HttpClient { Timeout = TimeSpan.FromMilliseconds(500) };

        // AWS EC2 IMDSv2
        try
        {
            var tokenReq = new HttpRequestMessage(HttpMethod.Put, "http://169.254.169.254/latest/api/token");
            tokenReq.Headers.Add("X-aws-ec2-metadata-token-ttl-seconds", "60");
            var token = await (await http.SendAsync(tokenReq)).Content.ReadAsStringAsync();

            var idReq = new HttpRequestMessage(HttpMethod.Get, "http://169.254.169.254/latest/meta-data/instance-id");
            idReq.Headers.Add("X-aws-ec2-metadata-token", token);
            var instanceId = await (await http.SendAsync(idReq)).Content.ReadAsStringAsync();
            Console.WriteLine($"AWS EC2 instance-id: {instanceId}");
            return;
        } catch { /* ليس AWS أو مغلق */ }

        // Azure IMDS
        try
        {
            var req = new HttpRequestMessage(HttpMethod.Get, "http://169.254.169.254/metadata/instance?api-version=2021-02-01");
            req.Headers.Add("Metadata", "true");
            var json = await (await http.SendAsync(req)).Content.ReadAsStringAsync();
            using var doc = JsonDocument.Parse(json);
            var vmId = doc.RootElement.GetProperty("compute").GetProperty("vmId").GetString();
            Console.WriteLine($"Azure VM id: {vmId}");
            return;
        } catch { /* ليس Azure */ }

        // GCP
        try
        {
            var req = new HttpRequestMessage(HttpMethod.Get, "http://metadata.google.internal/computeMetadata/v1/instance/id");
            req.Headers.Add("Metadata-Flavor", "Google");
            var id = await (await http.SendAsync(req)).Content.ReadAsStringAsync();
            Console.WriteLine($"GCE instance id: {id}");
            return;
        } catch { /* ليس GCP */ }

        Console.WriteLine("No cloud metadata detected (قد تكون VM محلية أو ميتاداتا معطّلة).");
    }
}
```
**ملاحظات:**  
- عطّل **IMDSv1** وفرض **IMDSv2** على AWS. اضبط **Hop Limit**.  
- لا تعتبر الميتاداتا **هوية مستخدم**؛ هي معلومات بنيويّة عن المثيل.

---

## خطوات عملية لإدارة VM بفعالية
- اختر **Type-1 Hypervisor** للإنتاج (KVM/Hyper-V/ESXi).  
- فعّل **VT-x/AMD-V** و **IOMMU** من BIOS/UEFI.  
- استخدم **صورًا قياسية** + **Cloud-Init/Customization** لتهيئة سريعة.  
- فعّل **vTPM/Secure Boot** وتشفير الأقراص داخل VM.  
- خصّص **Security Groups/NSG + جدار النظام**. افتح المنافذ الضرورية فقط.  
- راقب **CPU/RAM/IOPS/Latency**، واستخدم تنبيهات.  
- Backups حقيقية (لقطات ≠ نسخ احتياطي). استخدم **Snapshots** قصير الأجل فقط.  
- خطّط **للتوافر**: مناطق توافُر/مجاميع توافُر + **Live Migration** إن توفّرت.  
- أتمت بـ **IaC** (Terraform/ARM/CloudFormation) واستخدم **Templates/Golden Images**.  

## أخطاء شائعة
- الاعتماد على **Snapshots** بديلًا عن النسخ الاحتياطي.  
- Overcommit للذاكرة/CPU بلا قياس → **Steal/Swap** مرتفع.  
- نسيان تعريفات **paravirt/virtio** → أداء تخزين/شبكة متدنٍ.  
- إهمال إيقاف لطيف للتطبيق عند **Stop/Deallocate**.  
- تشغيل كل شيء بحجم كبير بلا سبب (تكلفة عالية وعدم استغلال).  
- تجاهل **Patch Management** لنظام الضيف (مسؤوليتك أنت).

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Virtual Machine (VM)** | **نظام تشغيل كامل معزول** | **أقوى عزل من الحاويات؛ أثقل وأبطأ إقلاعًا** |
| [Container](container.md) | تشغيل معزول يشارك نواة المضيف | خفيف وسريع؛ يحتاج نواة متوافقة؛ عزل أضعف من VM |
| [Hypervisor](hypervisor.md) | طبقة تشغّل وتدير الـ VMs | Type-1 للأداء/الإنتاج؛ يدعم Live Migration |
| [Bare Metal](bare-metal.md) | تشغيل مباشر على العتاد | أعلى أداء؛ لا تعددية ضيوف |
| [PaaS](paas.md) | تشغيل الكود على منصّة مُدارة | لا تدير OS الضيف؛ أسرع نشرًا |

## ملخص الفكرة  
**الآلة الافتراضيّة** تمنحك **نظامًا كاملًا** بعزل قوي وتحكّم كامل.  
استخدمها عندما تحتاج OS كاملًا أو توافقًا مع برمجيات قديمة، واضبط **الأمن/الأداء/النسخ الاحتياطي** بعناية.  
للتطبيقات السريعة الخفيفة، فكّر في **الحاويات/PaaS**؛ وللعزل الأعلى على نفس العتاد، اعتمد **Hypervisor Type-1**.
