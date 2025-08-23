# **IaaS**

## الترجمة الحرفية  
**Infrastructure as a Service (IaaS)** — **بنية تحتية كخدمة**.

## الوصف العربي المختصر  
تستأجر **بنية تحتية افتراضية** (خوادم، شبكات، تخزين، موازنات حمل) من مزوّد سحابي.  
أنت تدير **النظام التشغيلي والتطبيقات**؛ المزوّد يدير **العتاد، الطاقة، الشبكات الفيزيائية، الافتراضية**.

## الشرح المبسّط  
- تبني **آلات افتراضية (VMs)**، **شبكات (VPC/VNet)**، **أقراصًا**، **IP عام/خاص**.  
- تدفع **بالاستخدام** (ساعة/ثانية، تخزين/نقل بيانات).  
- تتحكّم في **النظام** (Windows/Linux)، التحديثات، الجدار الناري على مستوى النظام، وعمليات النشر.  
- الأتمتة عبر **SDKs** و**IaC** (مثل Terraform/ARM/CloudFormation).

## تشبيه  
بدل شراء مزرعة خوادم، **تستأجر رفًا افتراضيًا** في مركز بيانات: تختار عدد الخوادم، الشبكات، والأقراص كما تشاء، وتدير نظامها وبرامجها.

---

## مثال كود C# بسيط — إنشاء/إيقاف آلة افتراضية (AWS EC2)
> يبيّن كيف تتعامل برمجيًا مع IaaS. (تحتاج حساب AWS ومفاتيح إعداد عبر `~/.aws/credentials`؛ قد تُحتسب تكلفة!)

**التثبيت:**
```bash
dotnet add package AWSSDK.EC2
```

**الكود:**
```csharp
// Program.cs (.NET 8/9)
// مثال مختصر لتشغيل ثم إيقاف EC2 Instance
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Amazon;
using Amazon.EC2;
using Amazon.EC2.Model;

class IaasDemo
{
    static async Task Main()
    {
        // اختر الإقليم المناسب
        var ec2 = new AmazonEC2Client(RegionEndpoint.EUWest1);

        // 1) تشغيل آلة افتراضية صغيرة (Ubuntu مثال) داخل شبكة فرعية محددة
        var run = new RunInstancesRequest
        {
            ImageId = "ami-xxxxxxxx",            // ضع AMI صالحًا للإقليم
            InstanceType = InstanceType.T3Micro, // طبقة مجانية غالبًا
            MinCount = 1,
            MaxCount = 1,
            KeyName = "my-keypair",              // لمفاتيح SSH
            SecurityGroupIds = new List<string> { "sg-xxxxxxxx" }, // يفتح المنفذ 22/80… حسب حاجتك
            SubnetId = "subnet-xxxxxxxx",
            TagSpecifications = new List<TagSpecification> {
                new TagSpecification {
                    ResourceType = "instance",
                    Tags = new List<Tag> { new Tag("Name","demo-iaas") }
                }
            }
        };

        var resp = await ec2.RunInstancesAsync(run);
        var id = resp.Reservation.Instances[0].InstanceId;
        Console.WriteLine($"Launched: {id}");

        // … هنا تنفّذ تهيئة الخادم (SSH/Cloud-Init)، ثم عند الانتهاء:
        var stopReq = new StopInstancesRequest { InstanceIds = new() { id } };
        await ec2.StopInstancesAsync(stopReq);
        Console.WriteLine($"Stopped: {id}");
    }
}
```
**الإخراج المتوقع (مختصر):**  
- طباعة معرّف الآلة عند الإطلاق ثم رسالة الإيقاف.  
> بدّل `ImageId/SubnetId/SecurityGroupIds/KeyName` بقيمك. افتح المنافذ المطلوبة في Security Group.

---

## خطوات عملية لاعتماد IaaS (Checklist)
- اختَر **الإقليم** القريب من المستخدمين.  
- صمّم **الشبكة**: VPC/VNet، الشبكات الفرعية (عام/خاص)، NAT/Internet Gateway.  
- حدّد **أحجام الأجهزة** (CPU/RAM)، **نوع الأقراص** (SSD/HDD)، و**التخزين الاحتياطي/اللقطات**.  
- أعدّ **الأمان**: Security Groups/NSG، جدار النظام، مفاتيح SSH، إدارة هويّة وصلاحيات (IAM).  
- أتمت النشر بـ **IaC** (Terraform/CloudFormation/ARM/Bicep) و/**CI/CD**.  
- فعّل **المراقبة** والتنبيهات (Logs/Metrics/Tracing).  
- خطّط **للتوسّع** (Scale Out/In) و**التوافر** (Multi-AZ/Availability Set).  
- راجع **التكلفة** دوريًا (Right-sizing، إطفاء غير المستخدم، حجوزات).

## أخطاء شائعة
- تشغيل خوادم **بدون** Security Group/NSG مضبوط → منافذ مفتوحة للعالم.  
- نسيان **النسخ الاحتياطي/اللقطات** للأقراص.  
- الاعتماد على IP عام ثابت دون **DNS/TTL** مناسب؛ صعوبات عند التبديل.  
- إعداد يدوي متفرّق بدون **IaC** → بيئات متباينة وصعوبة تكرار.  
- إهمال **التحديثات الأمنية** للنظام؛ في IaaS مسؤوليتك أنت.  
- سوء تقدير **الحجم/التكلفة** (Instances كبيرة بلا حاجة، أقراص مفرطة).

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **IaaS** | **بنية تحتية افتراضية تدير نظامها بنفسك** | **VM/شبكات/أقراص/IP**؛ مرونة عالية مقابل مسؤولية أعلى |
| [PaaS](paas.md) | تشغيل الكود على منصّة مُدارة | نشر سريع، Autoscale، تحكّم أقل بالنظام |
| [SaaS](saas.md) | تطبيق جاهز للمستخدم النهائي | لا تنشر كودًا؛ أقل تخصيص على المستوى الداخلي |

## ملخص الفكرة  
**IaaS** يمنحك قطع LEGO منخفضة المستوى (VM/شبكات/تخزين) لتبني ما تشاء بمرونة.  
استخدم **SDK/IaC** للأتمتة، شدّد الأمان (IAM/SG/تحديثات)، وراقب التكلفة والأداء باستمرار.
