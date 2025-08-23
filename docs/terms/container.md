# **Container**

## الترجمة الحرفية  
**Container** — **حاوية** (عزل على مستوى نظام التشغيل لتشغيل تطبيقات بخفّة).

## الوصف العربي المختصر  
بيئة تشغيل **معزولة** تشارك **نواة النظام**، وتغلف التطبيق مع تبعياته داخل **صورة (Image)**.  
تعتمد على **Namespaces** و**cgroups** (لينكس) وتُدار عبر **Docker/Containerd** وتُوزَّع بصيغ **OCI**.

## الشرح المبسّط  
- **Image** = قالب للقراءة فقط.  
- **Container** = نسخة حيّة من الصورة مع طبقة كتابة صغيرة.  
- تشارك النواة لكن تفصل **الملفات/الشبكة/العمليات**.  
- تُقلّص “يعمل على جهازي فقط”؛ نفس البيئة أينما شغّلتها.  
- تُناسب النشر السحابي والتوسّع والأتمتة.

## تشبيه  
علبة طعام محكمة. تضع وجبتك (التطبيق + التبعيات) داخل العلبة،  
فتحصل على نفس الطعم أينما أخذتها (جهاز مطوّر، خادم، سحابة).

---

## مثال عملي — **ASP.NET Core** داخل حاوية (كود + Dockerfile)

### 1) تطبيق C# بسيط (Minimal API)
```csharp
// Program.cs  (.NET 8/9)
var builder = WebApplication.CreateBuilder(args);

// المنفذ تُمرّره منصّات PaaS عادةً عبر PORT
if (int.TryParse(Environment.GetEnvironmentVariable("PORT"), out var port))
    builder.WebHost.ConfigureKestrel(o => o.ListenAnyIP(port));

builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapGet("/", () => Results.Ok(new { msg = "Hello from container", at = DateTime.UtcNow }));
app.MapHealthChecks("/healthz");

app.Run();
```

### 2) Dockerfile (بناء متعدد المراحل — أصغر حجمًا)
```dockerfile
# مرحلة البناء
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app /p:PublishSingleFile=true /p:PublishTrimmed=true

# مرحلة التشغيل — صورة خفيفة
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app ./
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
# (صور ASP.NET الحديثة تعمل كمستخدم غير-جذر افتراضيًا، للتأكيد:)
USER app
ENTRYPOINT ["./Program"]
```

### 3) أوامر التشغيل
```bash
# بناء الصورة
docker build -t demo-api:latest .

# تشغيل الحاوية (منفذ، متغير بيئة، مجلد بيانات)
docker run --rm -p 8080:8080 -e PORT=8080 --name demo demo-api:latest

# اختبار
curl http://localhost:8080/
curl -f http://localhost:8080/healthz && echo "healthy"
```

> **الفكرة:** تضع تطبيقك داخل **Image** موحّدة.  
> تشغّل **Container** قابلًا للتكرار في أي مكان.

---

## خطوات عملية لاعتماد الحاويات
- صمّم صورة **صغيرة**: بناء متعدد المراحل، حذف الملفات المؤقتة.  
- اضبط **الصحة** (`/healthz`) لاستخدامها في الموازِن/الأوركستراتور.  
- عامِل التخزين داخل الحاوية كـ **عابر**؛ استخدم **Volumes/DB/Object Storage** للبيانات الدائمة.  
- حدِّد **موارد** (`--memory/--cpus`) وتابع المقاييس (CPU/RAM/GC).  
- أضِف **متغيّرات بيئة** للإعدادات (لا أسرار في الصورة).  
- استخدم **.dockerignore** لتقليل الحجم.  
- وقِّع الصور وافحص الثغرات (SLSA/Sign/Scanner).

## أخطاء شائعة
- الكتابة إلى نظام ملفات الحاوية كأنه دائم.  
- بناء صورة **ضخمة** (نسخ كل المستودع، عدم استخدام restore/publish بشكل ذكي).  
- تشغيل كـ **root** دون داعٍ.  
- وضع أسرار داخل الصورة بدل **Secret Manager** أو **Runtime Env**.  
- عدم تحديد **PORT/ASPNETCORE_URLS** أو نسيان **EXPOSE**.  
- تجاهل **صحة الخدمة** → إعادة تشغيل/توسّع خاطئ.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Container** | **تشغيل معزول وخفيف يشارك نواة النظام** | **سريع الإقلاع؛ يعتمد Images بصيغة OCI** |
| [Virtual Machine](virtual-machine.md) | عزل كامل مع نواة افتراضية | أثقل؛ مرونة أعلى في OS لكن أبطأ وأكبر |
| [Image (OCI)](image-oci.md) | قالب للملفات والتبعيات | للقراءة فقط؛ تؤسَّس منه الحاويات |

## ملخص الفكرة  
**الحاوية** تمنحك تشغيلًا **قابلًا للتكرار وخفيفًا** لتطبيقك.  
ابنِ صورًا صغيرة، أخفِ الأسرار، استخدم مسارات صحّة، واحفظ البيانات **خارج الحاوية**—واستعدّ للأتمتة عبر **Kubernetes/Compose** عند النمو.
