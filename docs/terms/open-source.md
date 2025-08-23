# **Open Source**

## الترجمة الحرفية  
**Open Source Software (OSS)** — **برمجيات مفتوحة المصدر**

## الوصف العربي المختصر  
برمجيات يُتاح **شيفرتها المصدرية** للجميع مع **رخصة** تسمح بالاستخدام، الدراسة، التعديل، وإعادة التوزيع—وفق شروط محدّدة (مثل **MIT**, **Apache-2.0**, **GPLv3**).

## الشرح المبسّط  
- **الوصول للمصدر**: ترى كيف صُنِع البرنامج، وتعدّله.  
- **رخصة** تحدّد الحقوق والالتزامات (نَسْب/تصاريح/إشعارات/مفتاح المصدر).  
- **حَوْكمة**: Maintainers، إصدارات، Issues/PRs، قواعد مساهمة (CONTRIBUTING).  
- **أنماط رخص**:
  - **Permissive (سماحية)**: مثل **MIT/Apache-2.0**—حرية واسعة مع شرط نَسْب/إشعار.  
  - **Copyleft (متبادلة)**: مثل **GPL**—إن دمجت الكود، يجب فتح المصدر المشتق بنفس الرخصة.  
- **نماذج عمل**: **دعم مدفوع**، **Open-Core**، **Dual Licensing**.

## تشبيه  
كتاب تعليم مفتوح: النصّ متاح، يمكنك نسخه وتعديله ونشر نسخة محسّنة—لكن مع **ذكر المصدر** واحترام **شروط الرخصة**.

---

## مثال كود C# — خدمة صغيرة تستخدم مكتبتين OSS + صفحة تراخيص (الامتثال)

> نستخدم **Serilog** للتسجيل و**Newtonsoft.Json** للتحويل، ونعرِض نقطة `/licenses` لعرض الإشعارات (THIRD-PARTY-NOTICES).

```csharp
// Program.cs  (.NET 8/9)
// dotnet add package Serilog.AspNetCore
// dotnet add package Serilog.Sinks.Console
// dotnet add package Newtonsoft.Json
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Serilog;
using Newtonsoft.Json;

var builder = WebApplication.CreateBuilder(args);

// تهيئة Serilog (OSS)
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateLogger();
builder.Host.UseSerilog();

var app = builder.Build();

// مثال استخدام Newtonsoft.Json (OSS)
app.MapGet("/echo", (object payload) =>
{
    var json = JsonConvert.SerializeObject(payload); // تحويل سريع
    Log.Information("Echo payload: {Json}", json);
    return Results.Text(json, "application/json");
});

// صفحة تراخيص/إشعارات الطرف الثالث (امتثال OSS)
app.MapGet("/licenses", async () =>
{
    var path = Path.Combine(AppContext.BaseDirectory, "THIRD-PARTY-NOTICES.txt");
    if (File.Exists(path))
        return Results.File(path, "text/plain; charset=utf-8");
    return Results.Text("No third-party notices found.", "text/plain");
});

app.Run();
```

**ضع ملفًا باسم `THIRD-PARTY-NOTICES.txt`** بجانب التنفيذ، يحوي نصوص الرخص/الإشعارات للمكتبات المستخدمة.  
**ملاحظة:** بعض الرخص (مثل **Apache-2.0**) تطلب إدراج NOTICE.

---

## خطوات عملية لاعتماد OSS بأمان
1. **حدّد سياسة رخص**: اسمح بـ MIT/Apache-2.0/BSD، وراجع **Copyleft** (GPL/LGPL) حسب ملاءمة منتجك.  
2. **تحقّق قبل الدمج**: راجع الرخصة، الشهرة، النشاط، الأمن، الحَوْكمة.  
3. **إدارة تبعيات**: ثبّت الإصدارات (pin/lock)، وفعّل فحص الثغرات (Dependabot/OSS-Index).  
4. **الامتثال**:  
   - أدرج **نَسْب/NOTICE** وملف **THIRD-PARTY-NOTICES**.  
   - التزم بشروط العلامات التجارية/براءات الاختراع (Apache-2.0 يمنح ترخيص براءات).  
5. **أمن السلسلة**: SBOM (CycloneDX/Syft)، توقيع صور (cosign)، مراجعة تغييرات الحرجة.  
6. **المساهمة upstream**: أصلِح الأخطاء في المصدر بدل رقع محلية طويلة الأمد.  
7. **حوكمة داخلية**: دوّن قرارات التبنّي (ADR)، وعرّف مسؤول التحديثات والتصحيحات.

---

## أخطاء شائعة
- افتراض أن **OSS = مجاني بلا شروط** → تجاهل الرخص/الإشعارات.  
- خلط **Source-Available** مع **Open Source** الحقيقي (ليست كل المتاحة للمشاهدة مفتوحة بحق).  
- ربط كود مغلق مع **GPL** دون فهم التزامات **Copyleft**.  
- إهمال تحديثات أمنية/تصحيحات CVE.  
- الاعتماد على Maintainer واحد (**Bus Factor**) دون خطة بديلة.  
- عدم توثيق التبعيات/الرخص في المنتج (غياب SBOM/NOTICES).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Open Source** | **كود متاح مع رخص تسمح بالاستخدام/التعديل/التوزيع** | **Permissive vs Copyleft**، مجتمع وحَوْكمة |
| Source-Available | إتاحة قراءة الكود دون حريات OSS الكاملة | قد تمنع الاستخدام التجاري/التعديل |
| Proprietary | مغلق المصدر، ترخيص مقيّد | لا وصول للمصدر؛ دعم رسمي عادةً |
| Permissive Licenses | حرية واسعة مع النَّسْب | MIT/Apache-2.0/BSD |
| Copyleft Licenses | مشاركة مماثلة للأعمال المشتقة | GPL/LGPL/AGPL (التزامات أقوى) |

---

## ملخص الفكرة  
**Open Source** يسرّع التطوير ويعزّز الجودة عبر المشاركة،  
لكن نجاحه يتطلّب **فهم الرخص والامتثال**، **إدارة تبعيات وأمن**، و**مساهمة مسؤولة** في المجتمع—عندها تحصل على سرعة ومرونة مع التزام قانوني وأمني سليم.
