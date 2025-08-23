# **Razor**

## الترجمة الحرفية  
**Razor View Engine / Syntax** — **محرّك/صياغة Razor** (دمج **C#** داخل **HTML**).

## الوصف العربي المختصر  
Razor هو **صياغة قوالب** تكتب بها صفحات HTML مع **تضمين C#** بسلاسة.  
يُستخدم في **MVC Views (.cshtml)** و**Razor Pages** و**Blazor (.razor)**،  
ويحوَّل القالب إلى **كلاس C#** يُنفَّذ (مع **ترميز HTML تلقائي** للحماية من XSS).

## الشرح المبسّط  
- إشارة **`@`** = التحويل إلى C# داخل HTML.  
- توجيهات أساسية: `@model`, `@page` (لـ Razor Pages), `@inject`, `@layout`.  
- **Tag Helpers**: سمات مثل `asp-for/asp-action/asp-page` تبني HTML صحيحًا وآمنًا.  
- بنية عامة: **Layout** + **Sections** + **Partials/View Components** لإعادة الاستخدام.

## تشبيه  
فكّره كمُترجم **حيّ** داخل HTML: تعطيه **C#** قصيرة، فيولّد لك **صفحة HTML** ديناميكية آمنة.

---

## مثال 1 — View MVC مختصر (`.cshtml`) مع Tag Helpers
```cshtml
@* Views/Products/Index.cshtml *@
@model IEnumerable<Product>
@{
    ViewData["Title"] = "Products";
    Layout = "_Layout";
}
<h1>@ViewData["Title"]</h1>

<a asp-action="Create" class="btn btn-primary">Add</a>

<table class="table">
  <thead><tr><th>Id</th><th>Name</th><th>Price</th></tr></thead>
  <tbody>
  @foreach (var p in Model)
  {
      <tr>
        <td>@p.Id</td>
        <td>@p.Name</td>
        <td>@p.Price.ToString("C")</td>
      </tr>
  }
  </tbody>
</table>

@* ترميز HTML افتراضيًا؛ استخدم Html.Raw بحذر شديد عند الحاجة *@
```

---

## مثال 2 — Razor Pages (صفحة + PageModel)
```cshtml
@* Pages/Index.cshtml *@
@page
@model IndexModel
<h2>Quick Note</h2>

<form method="post" asp-page-handler="Add">
  <input asp-for="Note" class="form-control" />
  <span asp-validation-for="Note" class="text-danger"></span>
  <button class="btn btn-success" type="submit">Save</button>
</form>

<ul>
  @foreach (var n in Model.Notes) { <li>@n</li> }
</ul>

@section Scripts { <partial name="_ValidationScriptsPartial" /> }
```

```csharp
// Pages/Index.cshtml.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using System.ComponentModel.DataAnnotations;
public class IndexModel : PageModel
{
    [BindProperty, Required, StringLength(100)]
    public string? Note { get; set; }
    public List<string> Notes { get; } = new();

    public void OnGet() { }                          // GET
    public IActionResult OnPostAdd()                 // POST /?handler=Add
    {
        if (!ModelState.IsValid) return Page();
        Notes.Add(Note!);
        ModelState.Clear(); Note = "";
        return Page();
    }
}
```
> **مهم:** **Form Tag Helper** يضيف **Anti-forgery Token** تلقائيًا في POST. لا تلغِه.

---

## مثال 3 — Partial View لإعادة الاستخدام
```cshtml
@* Views/Shared/_ProductRow.cshtml *@
@model Product
<tr><td>@Model.Id</td><td>@Model.Name</td><td>@Model.Price</td></tr>
```

```cshtml
@* الاستدعاء داخل أي View *@
<tbody>
  @foreach (var p in Model)
  { <partial name="_ProductRow" model="p" /> }
</tbody>
```

---

## خطوات عملية سريعة
1. **البدء**  
   - MVC: `dotnet new mvc` → استخدم Views/Layouts/Partials.  
   - Razor Pages: `dotnet new webapp` → صفحات بـ `@page`.
2. **النماذج والتحقّق**: أضف **Data Annotations** + `asp-validation-*`.  
3. **الأمان**: اترك الترميز التلقائي، وتجنّب `@Html.Raw` إلا لمحتوى موثوق.  
4. **التنظيم**: Layout موحّد، Partials/Components لإعادة الاستخدام، و**ViewModels** واضحة.  
5. **الأداء**: استخدم **Cache Tag Helper** عند الحاجة، وقلّل منطق العرض في القالب.

---

## أخطاء شائعة
- منطق أعمال ثقيل داخل `.cshtml` بدل خدمات/Handlers.  
- استعمال `Html.Raw` لمحتوى غير موثوق → **XSS**.  
- إهمال **ModelState** والتحقّق عند POST.  
- بناء روابط/نماذج يدويًا بدل **Tag Helpers** → مسارات/أسماء غير متناسقة.  
- خلط Razor Pages مع MVC بلا تنظيم واضح للمجلدات والمسارات.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Razor** | **صياغة قوالب C# داخل HTML** | **ترميز تلقائي، Layout/Sections/Partials، Tag Helpers** |
| [Razor Pages](razor-pages.md) | صفحة = واجهة + PageModel | أبسط لـ CRUD؛ توجيه مبني على الملفات |
| [MVC](mvc.md) | Views + Controllers | فصل Controller/Model/View؛ مرن للمشاريع الكبيرة |
| [Blazor](blazor.md) | مكونات تفاعلية بـ `.razor` | يعمل Server/WebAssembly؛ نفس صياغة Razor لكن نموذج مكوّنات |

---

## ملخص الفكرة  
**Razor** يمنحك قوالب **HTML + C#** نظيفة وآمنة.  
استعمل **Tag Helpers**، افصل المنطق عن القوالب، وحافظ على التحقّق/مكافحة التزوير—تحصل على صفحات قابلة للصيانة وسريعة الإنشاء.
