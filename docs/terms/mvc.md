# **MVC — Model–View–Controller**

## الترجمة الحرفية  
**Model–View–Controller (MVC)** — **النموذج–العرض–المتحكِّم**

## الوصف العربي المختصر  
نمط معماري يفصل تطبيق الويب إلى ثلاث طبقات:  
**Model** (البيانات والقواعد)، **View** (واجهة العرض)، **Controller** (تنسيق الطلب/الاستجابة).

## الشرح المبسّط  
- **Controller** يستقبل طلب HTTP، ينسّق المنطق، ويختار **View** مناسبًا.  
- **Model** يمثّل البيانات وقواعد المجال (Validation/Business Rules).  
- **View** يقدّم HTML (Razor) باستعمال **Model** أو **ViewModel**.  
- الفائدة: **فصل المسؤوليات** → اختبار أسهل، تبديل واجهات دون لمس القواعد.

## تشبيه  
مطعم:  
**النادل (Controller)** يأخذ طلبك،  
**المطبخ (Model)** يطبّق الوصفة،  
**التقديم (View)** يخرج الطبق بشكل جميل للزبون.

---

## مثال كود C# (ASP.NET Core MVC مختصر)

### 1) Model (بيانات + تحقّق)
```csharp
// Models/Product.cs
using System.ComponentModel.DataAnnotations;

public record Product(
    int Id,
    [property: Required, StringLength(60)] string Name,
    [property: Range(0.01, 99999)] decimal Price
);
```

### 2) Controller (تنسيق الطلب/الاستجابة)
```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;
using System.Collections.Generic;
using System.Linq;

public class ProductsController : Controller
{
    // للتبسيط: قائمة في الذاكرة
    private static readonly List<Product> _data =
    [
        new Product(1, "Keyboard", 120),
        new Product(2, "Mouse", 60)
    ];

    // GET /Products
    public IActionResult Index() => View(_data);

    // GET /Products/Create
    public IActionResult Create() => View();

    // POST /Products/Create
    [HttpPost]
    public IActionResult Create(Product input)
    {
        if (!ModelState.IsValid) return View(input);      // Validation من Model
        var nextId = _data.Any() ? _data.Max(p => p.Id) + 1 : 1;
        _data.Add(input with { Id = nextId });
        return RedirectToAction(nameof(Index));
    }
}
```

### 3) View (Razor) — قائمة المنتجات
```cshtml
@* Views/Products/Index.cshtml *@
@model IEnumerable<Product>
<h1>Products</h1>
<a asp-action="Create">Add</a>
<table>
  <thead><tr><th>Id</th><th>Name</th><th>Price</th></tr></thead>
  <tbody>
  @foreach (var p in Model)
  {
    <tr><td>@p.Id</td><td>@p.Name</td><td>@p.Price</td></tr>
  }
  </tbody>
</table>
```

### 4) View (Razor) — إضافة منتج
```cshtml
@* Views/Products/Create.cshtml *@
@model Product
<h1>Add Product</h1>
<form asp-action="Create" method="post">
  <label>Name</label>
  <input asp-for="Name" />
  <span asp-validation-for="Name"></span>

  <label>Price</label>
  <input asp-for="Price" />
  <span asp-validation-for="Price"></span>

  <button type="submit">Save</button>
</form>

@section Scripts { <partial name="_ValidationScriptsPartial" /> }
```

**الفكرة:** المتحكّم ينسّق، النموذج يطبّق القواعد، العرض يقدّم HTML.  
يمكن استبدال **View** بتصميم آخر دون المساس بالمنطق.

---

## خطوات عملية (ASP.NET Core MVC سريعًا)
1. أنشئ مشروعًا:  
   ```bash
   dotnet new mvc -n ShopMvc
   cd ShopMvc
   ```
2. أضِف الملفّات أعلاه (Model/Controller/Views).  
3. شغّل: `dotnet run` ثم افتح `https://localhost:****/Products`.  
4. إن أردت قواعد أقوى: أنشئ **ViewModel** منفصلًا عن **Domain Model**.  
5. افصل منطق المجال في **خدمات** تُحقن عبر **DI** بدل وضعه داخل Controller.

---

## أخطاء شائعة
- **Fat Controller**: منطق أعمال داخل المتحكّم بدل خدمات/نطاق.  
- تمرير **Entity من قاعدة البيانات مباشرةً** إلى العرض دون **ViewModel** مناسب.  
- منطق في **View** (Razor) أكثر من اللازم.  
- خلط مسؤوليات **Model** (نطاق) و**DAL/ORM** (وصول بيانات).  
- إهمال **Validation** على الخادم والاكتفاء بواجهة المستخدم.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **MVC** | **فصل: Controller ينسّق، Model قواعد، View عرض** | **شائع في ASP.NET Core MVC/Rails**؛ Views بـ Razor |
| [MVVM](mvvm.md) | ربط بيانات (Binding) بين View وViewModel | شائع بواجهات سطح المكتب/SPA؛ يقلّل كود الـ Controller |
| [MVP](mvp.md) | Presenter ينسّق بدل Controller | مناسب حين يكون الـ View “غبيًا” بالكامل |
| [Minimal APIs](minimal-apis.md) | نقاط نهاية خفيفة مباشرة | مناسب لخدمات صغيرة/واجهات داخلية |

---

## ملخص الفكرة  
**MVC** يضمن فصلًا واضحًا: **التحكّم** خارج **العرض**، وقواعد البيانات/المجال داخل **النموذج/الخدمات**.  
النتيجة: تطبيق أسهل **للاختبار**، **للتطوير**، و**للتبديل** في الواجهة دون كسر المنطق.
