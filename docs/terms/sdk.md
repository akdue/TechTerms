# **SDK**

## الترجمة الحرفية  
**Software Development Kit (SDK)** — **حِزمة تطوير البرمجيات** (عدة التطوير لمنصّة/منتج معيّن).

## الوصف العربي المختصر  
مجموعة أدوات متكاملة للتطوير: **مترجمات/Runtime**، **مكتبات وواجهات**، **أدوات سطر أوامر**، **وثائق**، وأحيانًا **محاكيات**.  
يتيح لك **البناء والتشغيل والنشر** لتطبيقات تستهدف منصّة محدّدة (مثل .NET، Android، iOS، Windows).

## الشرح المبسّط  
- فكّر بالـ **SDK** كـ *عدة نجّار كاملة*: مطرقة (Compiler)، مسامير (Libraries)، تعليمات (Docs)، وجداول قياس (CLI).  
- ليس محرّرًا بواجهة رسومية؛ يمكن استعماله من **IDE** أو **Code Editor** أو **الطرفيّة**.  
- يوفّر **واجهات قياسية** لبناء وتشغيل واختبار ونشر تطبيقات لمنصّة معيّنة.  
- أمثلة: **.NET SDK**, **Android SDK**, **iOS/macOS SDK (Xcode)**, **Windows SDK**.

## تشبيه  
شراء **طقم أدوات** لبناء أثاث: لديك الأدوات، القطع القياسية، وكتيّب التعليمات.  
يمكنك العمل بأي ورشة (IDE/Editor)، لكن **العدة نفسها** هي الـ SDK.

## مثال عملي مختصر (.NET SDK: CLI + كود)
### أوامر CLI باستخدام .NET SDK
```bash
# التحقق من التنصيب والإصدارات
dotnet --info

# إنشاء مشروع كونسول واستهداف net9.0
dotnet new console -n SdkDemo -f net9.0
cd SdkDemo

# إضافة حزمة (مثال: Newtonsoft.Json)
dotnet add package Newtonsoft.Json

# البناء والتشغيل
dotnet build
dotnet run

# نشر كتطبيق مستقل لنظام محدّد (Self-contained)
dotnet publish -c Release -r win-x64 --self-contained true
```

### ملف مشروع بأسلوب “SDK-Style”
```xml
<!-- SdkDemo.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

### برنامج C# بسيط
```csharp
// Program.cs
using System;
class Program
{
    static void Main() => Console.WriteLine("Hello from the .NET SDK!");
}
```
الإخراج المتوقع:  
- تعرض `dotnet --info` تفاصيل المنصّات والأطر.  
- يطبع التطبيق: `Hello from the .NET SDK!`  
- يولّد `publish` ملفات تشغيل بحسب الهدف (مثل win-x64).

## خطوات عملية لاستخدام SDK بكفاءة
- **نزّل وثبّت** الـ SDK المناسب لمنصّتك (تأكّد من توافق الإصدار مع مشروعك).  
- **اضبط المتغيّرات البيئية** إن لزم (مثال Android: `JAVA_HOME`, `ANDROID_HOME`).  
- **تحقّق** عبر أمر معلومات/نسخة (`dotnet --info`, `java -version`, `xcodebuild -version`).  
- **ابدأ بقوالب جاهزة** (مشروع Console/Web/Library) ثم أضف التبعيات.  
- استخدم **CLI/Tooling** مع **IDE** لتوحيد البناء في فرق العمل وCI.  
- **وثّق الأوامر** وخيارات النشر لكل منصّة، وأضف سكربتات `build`/`publish`.

## أخطاء شائعة
- الخلط بين **SDK** و**IDE**: الأول حزمة أدوات، الثاني بيئة تحرير متكاملة.  
- اختلاف إصدارات **Runtime/SDK** بين أجهزة الفريق → نتائج بناء مختلفة.  
- نسيان إعداد **المحاكيات/المنصّات** (Android/iOS) قبل البناء أو الاختبار.  
- استهداف إطار خاطئ (مثل `net8.0` بينما أدواتك لا تدعمه) أو معماريّة غير متوفّرة.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [Code Editor](code-editor.md) | تحرير نص الكود فقط | خفيف؛ يتوسّع عبر إضافات |
| [IDE](ide.md) | تحرير + بناء + تصحيح + إدارة مشروع | يتكامل عادةً مع SDKs وVCS واختبارات |
| **SDK** | **مترجمات/مكتبات/أدوات بناء ونشر** | **يُستخدم من IDE/Editor أو CLI؛ ليس محرّرًا بحدّ ذاته** |
| [Debugger](debugger.md) | تنفيذ خطوة بخطوة وفحص الحالة | قد يأتي ضمن IDE أو كأداة منفصلة |

## ملخص الفكرة  
**SDK** = العِدّة الفعلية للتطوير (Compiler/Tools/Libraries/Docs).  
استخدمه مع محرّرك أو IDE لتبني وتُشغّل وتَنشر بثبات عبر بيئات التطوير والتكامل المستمر.
