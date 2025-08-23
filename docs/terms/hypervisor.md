# **Hypervisor**

## الترجمة الحرفية  
**Hypervisor / Virtual Machine Monitor (VMM)** — **مراقِب/مُشرف الأجهزة الافتراضية**.

## الوصف العربي المختصر  
طبقة تشغّل **أنظمة تشغيل متعددة** كـ **ماكينات افتراضية (VMs)** فوق عتاد واحد.  
تدير **العزل**، **جدولة المعالجات**، **الذاكرة**، **الأجهزة الافتراضية**، وميزات مثل **Snapshots** و**Live Migration**.

## الشرح المبسّط  
- هدفه: **تقسيم جهاز واحد** إلى عدّة أجهزة منطقية معزولة.  
- نوعان رئيسيان:  
  - **Type-1 (Bare-Metal):** يعمل مباشرة على العتاد (Hyper-V/ESXi/KVM).  
  - **Type-2 (Hosted):** يعمل كبرنامج فوق نظام تشغيل (VirtualBox/VMware Workstation).  
- يستفيد من عتاد مثل **Intel VT-x/VT-d** و**AMD-V/IOMMU**.  
- يدعم **تعريفات شبه افتراضية (Paravirt/virtio)** لأداء أفضل، و**SR-IOV** لتمرير موارد الشبكة.

## تشبيه  
كفندق كبير. **الهايبرفايزر** هو الإدارة التي توزّع الغرف (CPU/RAM/IO) على النزلاء (الـ VMs)،  
وتضمن العزل، وتسمح بتجميد الحالة (Snapshot) أو نقل النزيل لغرفة أخرى حيًّا (Live Migration).

---

## مثال كود C# (قصير) — كشف هل التطبيق داخل VM واسم البائع
> ويندوز: عبر **WMI**. لينكس/ماك: عبر ملفات **DMI** إن وُجدت.

```csharp
// dotnet add package System.Management   // على ويندوز فقط
using System;
using System.IO;
using System.Runtime.InteropServices;
#if WINDOWS
using System.Management;
#endif

class VmDetect
{
    static void Main()
    {
        string vendor = "Unknown";
        bool inVm = false;

        if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
        {
#if WINDOWS
            using var cs = new ManagementObjectSearcher("SELECT Manufacturer, Model FROM Win32_ComputerSystem");
            foreach (ManagementObject mo in cs.Get())
            {
                var m = (mo["Manufacturer"]?.ToString() ?? "").ToLower();
                var model = (mo["Model"]?.ToString() ?? "").ToLower();
                vendor = $"{m} {model}";
                inVm = m.Contains("microsoft") && model.Contains("virtual")
                       || m.Contains("vmware") || m.Contains("xen")
                       || m.Contains("qemu")   || model.Contains("kvm");
            }
#endif
        }
        else
        {
            string Read(string p) => File.Exists(p) ? File.ReadAllText(p).Trim() : "";
            var sys = Read("/sys/class/dmi/id/sys_vendor").ToLower();
            var prod = Read("/sys/class/dmi/id/product_name").ToLower();
            vendor = $"{sys} {prod}".Trim();
            inVm = sys.Contains("microsoft") || sys.Contains("vmware") || sys.Contains("xen")
                   || sys.Contains("qemu") || prod.Contains("kvm") || prod.Contains("virtual");
        }

        Console.WriteLine($"In VM? {inVm} | Vendor: {vendor}");
    }
}
```

**الإخراج المتوقع:** يطبع إن كان التشغيل داخل VM واسم البائع التقريبي (VMware/Hyper-V/KVM…).

---

## مثال كود C# (اختياري) — أخذ Snapshot لـ VM على Hyper-V عبر PowerShell
> يتطلب ويندوز بصلاحيات مسؤول وتمكين Hyper-V.  
> يشغّل أمر PowerShell من C#.

```csharp
// dotnet add package Microsoft.PowerShell.SDK
using System;
using System.Management.Automation;

class HvSnapshot
{
    static void Main()
    {
        var vmName = "MyVM";
        using var ps = PowerShell.Create();
        ps.AddScript($"Checkpoint-VM -Name '{vmName}' -SnapshotName 'snap-{DateTime.UtcNow:yyyyMMdd-HHmmss}'");
        var results = ps.Invoke();
        if (ps.HadErrors) throw new Exception("Snapshot failed");
        Console.WriteLine($"Snapshot created for {vmName}");
    }
}
```

---

## خطوات عملية لاستخدام الهايبرفايزر بفعالية
- فعّل **VT-x/AMD-V** و**IOMMU (VT-d/AMD-Vi)** من الـ BIOS/UEFI.  
- خطّط **Sizing**: CPU/RAM/Storage و**Overcommit** مدروس (خاصة الذاكرة).  
- استخدم **Paravirtual/virtio** و**Synthetic Drivers** لتحسين I/O وNetwork.  
- فعّل **vTPM/Secure Boot** و**تشفير الأقراص** داخل الـ VM.  
- اعتمد **Templates/Cloud-Init** للتكرار السريع، و**Snapshots** بحذر (ليست بديلًا عن نسخ احتياطي).  
- للمؤسسات: **Live Migration**، مخازن مشتركة (SAN/NFS), و**Anti-Affinity** بين العُقد.  
- راقب: **CPU Ready/Steal**, ضغط الذاكرة/ballooning، IOPS/Latency، ودرجة الحرارة.

## أخطاء شائعة
- استخدام **Type-2** لحمولات إنتاجية ثقيلة بدل **Type-1**.  
- **Overcommit** مفرط للذاكرة → ضغط/Swapping شديد داخل المضيف والضيوف.  
- نسيان تعريفات **virtio** → I/O بطيء جدًا.  
- الاعتماد على **Snapshots** كنسخ احتياطي طويل الأمد.  
- تجاهل **NUMA/CPU Topology** عند أحمال قواعد البيانات الكثيفة.  
- تمرير أجهزة بلا عزل (PCI Passthrough) دون فهم تبعات الأمان/التوافر.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Hypervisor Type-1** | تشغيل VMs فوق العتاد مباشرة | أداء/ثبات أعلى؛ أمثلة: **Hyper-V, ESXi, KVM** |
| **Hypervisor Type-2** | تشغيل VMs فوق نظام مضيف | مناسب للتطوير/التجارب؛ أمثلة: **VirtualBox, Workstation** |
| [Virtual Machine](virtual-machine.md) | نظام تشغيل كامل معزول | مرن وقوي العزل؛ أثقل من الحاويات |
| [Container](container.md) | عزل على مستوى النظام | يشارك النواة؛ أخف وأسرع إقلاعًا |
| [Bare Metal](bare-metal.md) | تشغيل مباشر على العتاد | أعلى أداء؛ لا تعددية ضيوف |

## ملخص الفكرة  
**Hypervisor** يتيح لك تشغيل عدّة أنظمة تشغيل معزولة على نفس العتاد،  
مع إدارة موارد متقدّمة وميزات كـ **Snapshots** و**Live Migration**. اختر **Type-1** للإنتاج، واستخدم تعريفات **paravirt**، واضبط الموارد والمراقبة بعناية.
