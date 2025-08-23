# **IIS — Internet Information Services**

## الترجمة الحرفية  
**Internet Information Services (IIS)** — **خدمات معلومات إنترنت** (خادم ويب من مايكروسوفت على ويندوز).

## الوصف العربي المختصر  
خادم ويب/عكس وكيل يُدير **مواقع وتطبيقات** على ويندوز: إعداد **المواقع/Bindings**، **App Pools**، **Modules** (مثل URL Rewrite)،  
ويدعم **HTTP/2/3**، **TLS/SSL**، **المصادقة** (Windows/Basic)، والسجلات/التتبّع.

## الشرح المبسّط  
- **واجهة استقبال**: يستقبل الطلبات على المنافذ والشهادات، ويُوجّهها للتطبيقات.  
- مع **ASP.NET Core** يعمل كـ **Reverse Proxy** عبر **ASP.NET Core Module (ANCM)**،  
  إما **In-Process** داخل عامل IIS (`w3wp`) أو **Out-of-Process** خلف **Kestrel**.  
- إدارة **App Pools** تمنح **عزلًا**، إعادة تدوير، وحدود موارد.  
- **Modules** تضيف ميزات: إعادة كتابة عناوين، ضغط، تصفية طلبات، تخويل.

## تشبيه  
فكّره كـ **لوبي المبنى**: البوّاب (IIS) يستقبل الزوّار، يفحص البطاقات (TLS/Auth)،  
ثم يوجّههم للمكتب المناسب (تطبيقك) ويطبّق سياسات المبنى (Modules/Rules).

---

## مثال كود C# (ASP.NET Core خلف IIS + رؤوس Forwarded)
```csharp
// Program.cs (.NET 8/9)
using Microsoft.AspNetCore.HttpOverrides;

var builder = WebApplication.CreateBuilder(args);

// إذا كنت خلف IIS/Proxy، فعّل تحليل الرؤوس لاستعادة Scheme/IP الحقيقيين
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
});

var app = builder.Build();

app.UseForwardedHeaders();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();                 // مع IIS: فعل HTTPS وHSTS
    app.UseHttpsRedirection();
}

app.MapGet("/", (HttpContext ctx) => new
{
    scheme = ctx.Request.Scheme,   // سيصبح "https" عند تمرير X-Forwarded-Proto
    ip = ctx.Connection.RemoteIpAddress?.ToString(),
    server = "IIS + ASP.NET Core"
});

app.Run();
```

### ملف `web.config` نموذجي (جذر النشر)
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <!-- ASP.NET Core Module V2 -->
    <handlers>
      <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified"/>
    </handlers>
    <aspNetCore processPath="dotnet"
                arguments="MyApp.dll"
                hostingModel="inprocess"
                stdoutLogEnabled="true"
                stdoutLogFile=".\logs\stdout"
                disableStartUpErrorPage="true">
      <environmentVariables>
        <add name="ASPNETCORE_ENVIRONMENT" value="Production" />
      </environmentVariables>
    </aspNetCore>
    <!-- مثال قاعدة لإجبار HTTPS إن لم تكن تستخدم إعادة التوجيه على التطبيق -->
    <rewrite>
      <rules>
        <rule name="HTTPS redirect" enabled="true" stopProcessing="true">
          <match url="(.*)" />
          <conditions>
            <add input="{HTTPS}" pattern="off" ignoreCase="true" />
          </conditions>
          <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

> ملاحظة: مع **In-Process** يكفي عادة `hostingModel="inprocess"` و`processPath="dotnet"` + `arguments="MyApp.dll"`.  
> تأكد من أذونات مجلد النشر (`IIS_IUSRS`) ومجلد `logs`.

---

## خطوات عملية لنشر ASP.NET/ويب على IIS
1. **تثبيت IIS**: من *Turn Windows features on/off* → **Web Server (IIS)** (+ Management Tools).  
2. **(لـ ASP.NET Core)** ثبّت **.NET Hosting Bundle** على الخادم.  
3. **إنشاء موقع**:  
   - حدّد **Physical Path** لمجلد النشر (`dotnet publish -c Release`).  
   - أنشئ **App Pool** مخصّصًا واستخدم **No Managed Code** لتطبيقات Core.  
4. **Bindings**: أضِف **HTTPS** واختر شهادة صحيحة (SNI عند وجود أسماء متعددة).  
5. **Modules/Rewrite**: ثبّت **URL Rewrite** عند الحاجة لسياسات (www/HTTPS/SPA).  
6. **Logging/Tracing**: فعّل **W3C Logs** و**Failed Request Tracing (FREB)** لتشخيص 4xx/5xx.  
7. **أمان**: فعّل المصادقة المطلوبة (Windows/Basic/Anonymous)، **Request Filtering**، وحدود الأحجام.  
8. **Reverse Proxy**: إن كنت خلف جهاز توجيه/Proxy، اضبط **X-Forwarded-\*** في الـ Proxy،  
   وفعّل `UseForwardedHeaders` في التطبيق كما في المثال.  
9. **صيانة**: استخدم **إعادة تدوير App Pool** المجدولة، والمراقبة عبر Event Viewer/PerfMon.

---

## أخطاء شائعة
- نسيان تثبيت **.NET Hosting Bundle** → خطأ **500.31/500.30 (ANCM)**.  
- إعداد **App Pool** على **CLR v4** بدل **No Managed Code** لتطبيقات Core.  
- أذونات مجلد النشر غير كافية لمستخدم **IIS AppPool\YourPool**.  
- تفعيل إعادة التوجيه مرتين (في IIS وفي التطبيق) → حلقات Redirect.  
- عدم تمرير رؤوس `X-Forwarded-*` أو عدم الثقة بالـ Proxy الموثوق.  
- ترك **HTTP فقط** بلا TLS/HSTS في الإنتاج.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **IIS** | **خادم ويب ويندوز + إدارة App Pools/Modules** | **تكامل مع Windows/TLS/Auth/URL Rewrite؛ يدعم ASP.NET/Core** |
| IIS Express | نسخة خفيفة للتطوير | مع Visual Studio؛ غير مناسبة للإنتاج |
| Kestrel | خادم HTTP الخاص بـ ASP.NET Core | يستخدم خلف IIS/NGINX عادة؛ خفيف وسريع |
| Nginx| خادم ويب/وكيل عكسي على لينُكس | بديل شائع خارج ويندوز؛ مرن في الـ Rewrite/Proxy |

## ملخص الفكرة  
**IIS** هو خادم الويب الرسمي على ويندوز: يستقبل الاتصالات، يطبق سياسات الأمان/إعادة الكتابة،  
ويشغّل تطبيقاتك (خاصة ASP.NET Core) بكفاءة عبر **ANCM**—مع سجلات وأدوات تشخيص متقدمة.
