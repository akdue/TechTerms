# **SPA — Single-Page Application**

## الترجمة الحرفية  
**Single-Page Application (SPA)** — **تطبيق صفحة واحدة**.

## الوصف العربي المختصر  
واجهة ويب تُحمِّل **صفحة HTML واحدة** ثم تُحدِّث المحتوى **ديناميكيًا** عبر **JavaScript**.  
التنقّل **عميلـي** (Routing على المتصفّح). البيانات من **APIs** (عادة REST/gRPC/SSE).

## الشرح المبسّط  
- أول زيارة: تحميل **HTML + JS/CSS**.  
- بعد ذلك: تغيّر الشاشات **بدون** إعادة تحميل الصفحة.  
- البيانات تُجلب من الخادم عبر **AJAX/Fetch** أو **WebSocket/SSE**.  
- مشاكل شائعة: **SEO** للأولية، **حجم الباندل**، **إدارة الحالة**.

## تشبيه  
تطبيق هاتف داخل المتصفّح: نفس الشاشة الأساسية، ومكوّنات تتبدّل **محليًا**،  
وتجلب البيانات عند الحاجة من الخادم.

---

## مثال C# — استضافة **SPA** عبر ASP.NET Core + **Fallback** للمسارات + API

```csharp
// Program.cs (.NET 8/9)
// سيناريو: تبني الواجهة (React/Angular/Vite) لتخرج ملفات إلى wwwroot/
// ASP.NET Core يقدّم الملفات الثابتة + API، مع Fallback لأي مسار غير API إلى index.html
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// CORS أثناء التطوير (اسمح لتطبيق الواجهة إن كان يعمل على منفذ آخر)
builder.Services.AddCors(o => o.AddPolicy("dev", p => p
    .AllowAnyHeader().AllowAnyMethod().WithOrigins("http://localhost:5173","http://localhost:3000")));

var app = builder.Build();

app.UseHttpsRedirection();
app.UseCors("dev");

// 1) ملفات الواجهة (بعد build إلى wwwroot)
app.UseDefaultFiles();   // يبحث عن index.html
app.UseStaticFiles();    // يقدّم /assets/* مع Cache-Control

// 2) API صغيرة تُخدم تحت /api/*
app.MapGet("/api/products", () =>
{
    var items = new[] {
        new { id=1, name="Keyboard", price=120m },
        new { id=2, name="Mouse",    price=60m  }
    };
    return Results.Ok(items);
})
.WithName("GetProducts")
.Produces(StatusCodes.Status200OK);

// 3) **SPA Fallback** — أي مسار لا يبدأ بـ /api يُرجَع له index.html
// هذا يمكّن الـ Client-Side Routing مثل /dashboard أو /product/42
app.MapFallbackToFile("index.html");

app.Run();

/*
بنية المشروع (ملخّص):
- wwwroot/
   ├─ index.html          ← نقطة الدخول للـ SPA
   └─ assets/...          ← Bundles/CSS/Images (مع هاش للأسماء للتخزين المؤقت)
- Program.cs              ← هذا الملف
ملاحظة: أثناء التطوير يمكنك تشغيل واجهة Vite/React على 5173 وتفعيل CORS أعلاه.
في الإنتاج: نفّذ build، وانسخ ناتج الواجهة داخل wwwroot.
*/
```

> **الفكرة:**  
> - `/api/*` للباك-إند.  
> - باقي المسارات تعود إلى **index.html** ليتولّى **Router** العميلي العرض.  
> - استخدم أسماء ملفات بأختام (hash) لتفعيل **Cache** قوية للأصول.

---

## خطوات عملية لبناء SPA “صحيّة”
1. **تقسيم الكود** (Code-Splitting) + **تحميل كسول** للمسارات الثقيلة.  
2. **حالة واجهة** منظّمة (Store/Signals) وتبسيط التدفق.  
3. **تحسين الانطباع الأول**: Skeleton، Prefetch، و—عند الحاجة—**SSR/Hydration**.  
4. **الأمان**: حماية **XSS/CSRF**، تخزين الرموز بعناية (يفضَّل **HttpOnly** للكوكيز).  
5. **SEO**: إن كان محتوى عامًا، أضف **SSR/Prerender** أو **Meta tags** ديناميكية.  
6. **المراقبة**: تتبّع أخطاء المتصفّح، أداء **LCP/CLS/TTI**، وطلبات API.  
7. **Routing عميلـي + Fallback** على الخادم لمنع 404 عند تحديث الصفحة.  
8. **خدمة PWA** (اختياري): Service Worker للتخزين دون اتصال وإشعارات.

---

## أخطاء شائعة
- باندل ضخم (كل شيء في أول زيارة) → زمن أولي بطيء.  
- عدم إعداد **Fallback** → مسارات عميقة تُظهر 404 عند التحديث.  
- الاعتماد على **Polling** فقط بدل SSE/WebSocket للحالات المباشرة.  
- خلط منطق أعمال ثقيل داخل المكوّنات.  
- تجاهل **Accessibility**/Keyboard/Focus وإدارة العناوين (document.title/aria).  
- تخزين **Tokens** في `localStorage` بلا حذر (قابل للـ XSS).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **SPA** | **واجهة عميل واحدة + Routing عميل** | سريع بعد التحميل؛ يحتاج API/حل SEO |
| MPA | صفحات تُولَّد من الخادم | انتقال كامل الصفحة؛ SEO سهل |
| **SSR** | توليد HTML على الخادم | يحسّن الـ SEO ووقت الظهور الأول |
| SSG/ISR | بناء صفحات مسبقًا | أداء ممتاز لمحتوى شبه ثابت |
| **Blazor WASM** | SPA بـ C# على WebAssembly | حجم أولي أكبر غالبًا؛ بدون JS ثقيل |

---

## ملخص الفكرة  
**SPA** = صفحة واحدة تتحكّم بكل التنقّل وتتعامل مع API للبيانات.  
استخدم **Fallback** للمسارات، قسّم الباندل، واحمِ الواجهة—تحصل على تجربة **سلسة**،  
سريعة الاستجابة بعد التحميل، وقابلة للتوسّع مع نمو التطبيق. 
