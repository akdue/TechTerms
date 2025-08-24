# **Razor Pages**

## الترجمة الحرفية  
**Razor Pages (ASP.NET Core)** — **نمط صفحات مبني على ريزر** لكتابة صفحات ويب **تركّز على الصفحة** بدل وحدات تحكّم كاملة.

## الوصف العربي المختصر  
كل صفحة = **ملف .cshtml** + **صفّ PageModel** ملاصق لها.  
تعريف المسار، العرض، وربط البيانات/التحقّق يتم في **مكان واحد**؛ أبسط من MVC التقليدي للصفحات القياسية (Forms/CRUD).

## الشرح المبسّط  
- الصفحة تملك **Handlers** مثل `OnGet/OnPost` (و`OnPostDelete`…)،  
- **ربط نموذج** عبر `[BindProperty]` و**تحقّق** بـ DataAnnotations،  
- **Tag Helpers** (`asp-for`, `asp-page`, `asp-page-handler`) لبناء HTML آمن ونظيف،  
- **توجيه** من توجيه الصفحة نفسها (`@page "{id:int?}"`).

## تشبيه  
بدل المرور على **موحّد اتصالات** (Controller) لكل مسار، لديك **مكتب واحد** لكل صفحة: الاستلام والمعالجة والعرض في مكان واحد.

---

## مثال عملي — مشروع Razor Pages بسيط (GET/POST + ربط + تحقّق)

```csharp
// Program.cs  (.NET 8/9)
// dotnet new webapp -n RazorPagesDemo && cd RazorPagesDemo && dotnet run
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorPages();
var app = builder.Build();

if (!app.Environment.IsDevelopment()) { app.UseExceptionHandler("/Error"); app.UseHsts(); }

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();

app.MapRazorPages();   // <-- مهم: تفعيل صفحات ريزر
app.Run();
```

```csharp
// Pages/Contact.cshtml.cs  (صفّ الصفحة: PageModel)
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

public class ContactModel : PageModel
{
    [BindProperty, Required, EmailAddress]
    public string Email { get; set; } = "";

    [BindProperty, Required, StringLength(200)]
    public string Message { get; set; } = "";

    public string? Status { get; set; }

    public void OnGet() { }  // عرض أولي

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid) return Page(); // إعادة نفس الصفحة بالأخطاء
        // TODO: إرسال بريد/حفظ قاعدة بيانات…
        Status = "تم الاستلام، شكرًا!";
        ModelState.Clear();
        return Page(); // عرض نفس الصفحة بنتيجة
    }

    // مثال Handler مسمّى: POST /Contact?handler=Preview
    public IActionResult OnPostPreview()
    {
        if (!ModelState.IsValid) return Page();
        Status = $"معاينة: {Message[..Math.Min(30, Message.Length)]}...";
        return Page();
    }
}
```

```html
@* Pages/Contact.cshtml *@
@page
@model ContactModel
@{
    ViewData["Title"] = "اتصل بنا";
}

<h2>@ViewData["Title"]</h2>

@if (!string.IsNullOrEmpty(Model.Status))
{
  <div class="alert alert-success">@Model.Status</div>
}

<form method="post">
  <div class="mb-3">
    <label asp-for="Email" class="form-label"></label>
    <input asp-for="Email" class="form-control" />
    <span asp-validation-for="Email" class="text-danger"></span>
  </div>

  <div class="mb-3">
    <label asp-for="Message" class="form-label"></label>
    <textarea asp-for="Message" class="form-control"></textarea>
    <span asp-validation-for="Message" class="text-danger"></span>
  </div>

  <button type="submit" class="btn btn-primary">إرسال</button>
  <button type="submit" asp-page-handler="Preview" class="btn btn-secondary">معاينة</button>
</form>

@section Scripts {
  <partial name="_ValidationScriptsPartial" />
}
```

> نقاط مهمّة:  
> - **Anti-forgery** مفعّل تلقائيًا مع `<form method="post">` (عبر Tag Helper).  
> - Handlers: `OnPostPreview` يُستدعى بزر `asp-page-handler="Preview"`.  
> - يمكن تخصيص المسار من أعلى الملف: `@page "{id:int?}"`.

---

## خطوات عملية للبدء بسرعة
1. أنشئ القالب: `dotnet new webapp -n MyApp`.  
2. في `Program.cs`: `builder.Services.AddRazorPages();` ثم `app.MapRazorPages();`.  
3. أضِف صفحة: ملفان تحت `Pages/`:  
   - `X.cshtml` (العلامات وTag Helpers)،  
   - `X.cshtml.cs` (PageModel مع `OnGet/OnPost`).  
4. استخدم `[BindProperty]` وDataAnnotations للتحقّق؛ أعد `Page()` عند عدم الصّحة.  
5. استعمل **Tag Helpers** بدلاً من HTML يدوي (`asp-for`, `asp-validation-for`).  
6. Handler مخصّص؟ أنشئ `OnPostName` واستخدم زر `asp-page-handler="Name"`.

---

## أخطاء شائعة
- نسيان `app.MapRazorPages()` → صفحات ترجع 404.  
- كتابة **منطق أعمال** ثقيل في PageModel بدل استدعاء **خدمة/Use Case**.  
- تجاهل التحقّق وإرجاع Redirect دائمًا → فقدان رسائل الأخطاء.  
- استخدام `Request.Form` يدويًا بدل **Model Binding** وDataAnnotations.  
- خلط Razor Pages مع Controllers لنفس المسارات بلا تخطيط واضح.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Razor Pages** | **صفحات تركز على الصفحة (Forms/CRUD سريع)** | أبسط من MVC؛ ملف صفحة + PageModel |
| MVC (Controllers/Views) | تحكّم مركزي لمجالات واسعة | مناسب لـ APIs معقدة ومسارات كثيرة |
| **Minimal APIs** | نقاط نهاية API فقط | بلا واجهات؛ مثالي لـ خدمات خفيفة |
| Blazor (Server/WASM) | مكوّنات تفاعلية C# | DOM افتراضي وتفاعلية عالية |
| Razor Class Library | مشاركة واجهات/جزءات Razor | تعليب مكوّنات/Layouts قابلة لإعادة الاستخدام |

---

## ملخص الفكرة  
**Razor Pages** تُبسّط بناء صفحات الويب في ASP.NET Core:  
صفحة واحدة تجمع **التوجيه + العرض + الربط/التحقّق**، مع Handlers واضحة وTag Helpers—  
مناسبة جدًا لصفحات **Forms/CRUD** النظيفة، مع إبقاء منطق الأعمال في خدمات منفصلة. 
