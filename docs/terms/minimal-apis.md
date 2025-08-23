# **Minimal APIs (ASP.NET Core)**

## الترجمة الحرفية  
**Minimal APIs** — **واجهات برمجية بأقل هيكلية**

## الوصف العربي المختصر  
أسلوب في **ASP.NET Core** لكتابة REST **بأقل تعقيد**: ملف/ملفّان،  
نقاط نهاية **MapGet/MapPost…** مباشرة، مع **DI** و**Binding** و**Filters** و**Swagger**.

## الشرح المبسّط  
- لا **Controllers** ولا **Attributes** كثيرة.  
- تكتب **Handlers** كدوال، وتربطها بمسارات.  
- تحصل على **حقن تبعيات**، **Model Binding**، **Auth**، و**OpenAPI** جاهزة.  
- مناسب لخدمات صغيرة/داخلية و**Prototypes**، ويمكن التوسّع لاحقًا.

## تشبيه  
عربة قهوة متنقّلة بدل مطعم كبير: نفس القهوة لكن تجهيز أسرع وأخفّ.

---

## مثال كود C# كامل — Minimal API مع DI + Validation + Filters + Swagger

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n MiniShop && cd MiniShop && dotnet run

using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;
using Microsoft.OpenApi.Models;

// 1) البنية والخدمات
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<IProductsRepo, InMemoryProductsRepo>();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(o =>
{
    o.SwaggerDoc("v1", new OpenApiInfo { Title = "MiniShop API", Version = "v1" });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// 2) مجموعة مسارات (Versioning بسيط عبر prefix)
var api = app.MapGroup("/api/v1/products").WithTags("Products");

// فلتر بسيط لرفض النماذج غير الصحيحة
api.AddEndpointFilter(new ValidateProductFilter());

// GET: جميع المنتجات
api.MapGet("/", (IProductsRepo repo) =>
    Results.Ok(repo.All()))
   .WithName("GetProducts")
   .WithOpenApi();

// GET: منتج بالمعرّف
api.MapGet("/{id:int}", (int id, IProductsRepo repo) =>
{
    var p = repo.Find(id);
    return p is null ? Results.NotFound() : Results.Ok(p);
}).WithOpenApi();

// POST: إنشاء منتج
api.MapPost("/", (ProductCreate dto, IProductsRepo repo) =>
{
    var created = repo.Add(dto);
    return Results.Created($"/api/v1/products/{created.Id}", created);
}).WithOpenApi();

// PUT: تحديث كامل
api.MapPut("/{id:int}", (int id, ProductUpdate dto, IProductsRepo repo) =>
{
    var ok = repo.Update(id, dto);
    return ok ? Results.NoContent() : Results.NotFound();
}).WithOpenApi();

// DELETE: حذف
api.MapDelete("/{id:int}", (int id, IProductsRepo repo) =>
    repo.Delete(id) ? Results.NoContent() : Results.NotFound()).WithOpenApi();

app.Run();

// ======= النماذج (DTOs) =======
public record Product(int Id, string Name, decimal Price);
public record ProductCreate([FromBody][property: BindRequired] string Name,
                            [FromBody] decimal Price);
public record ProductUpdate([FromBody] string Name, [FromBody] decimal Price);

// ======= المستودع (Repo) =======
public interface IProductsRepo
{
    IEnumerable<Product> All();
    Product? Find(int id);
    Product Add(ProductCreate dto);
    bool Update(int id, ProductUpdate dto);
    bool Delete(int id);
}

public class InMemoryProductsRepo : IProductsRepo
{
    private readonly List<Product> _db = new() { new(1, "Keyboard", 120), new(2, "Mouse", 60) };
    private int _nextId = 3;

    public IEnumerable<Product> All() => _db;
    public Product? Find(int id) => _db.FirstOrDefault(p => p.Id == id);
    public Product Add(ProductCreate dto)
    {
        var p = new Product(_nextId++, dto.Name, dto.Price);
        _db.Add(p); return p;
    }
    public bool Update(int id, ProductUpdate dto)
    {
        var i = _db.FindIndex(p => p.Id == id);
        if (i < 0) return false;
        _db[i] = _db[i] with { Name = dto.Name, Price = dto.Price };
        return true;
    }
    public bool Delete(int id) => _db.RemoveAll(p => p.Id == id) > 0;
}

// ======= فلتر تحقق بسيط =======
public class ValidateProductFilter : IEndpointFilter
{
    public ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        foreach (var arg in ctx.Arguments)
        {
            switch (arg)
            {
                case ProductCreate c when string.IsNullOrWhiteSpace(c.Name) || c.Price <= 0:
                case ProductUpdate u when string.IsNullOrWhiteSpace(u.Name) || u.Price <= 0:
                    return ValueTask.FromResult<object?>(Results.ValidationProblem(new()
                    {
                        ["Name/Price"] = ["الاسم مطلوب والسعر > 0"]
                    }));
            }
        }
        return next(ctx);
    }
}
```

**الإخراج المتوقع (مختصر):**  
- `GET /api/v1/products` → قائمة JSON.  
- `POST /api/v1/products` بجسم `{ "name":"SSD","price":250 }` → `201 Created` مع الكيان.  
- Swagger UI متاح في التطوير.

---

## خطوات عملية سريعة
1. **إنشاء المشروع**  
   ```bash
   dotnet new web -n MiniShop
   dotnet run
   ```
2. افتح Swagger على رابط التطبيق للتجربة.  
3. أضف **DI** و**MapGroup** و**Filters** عند الحاجة.  
4. فعّل **AuthN/AuthZ** لاحقًا:  
   ```csharp
   app.UseAuthentication();
   app.UseAuthorization();
   api.RequireAuthorization(); // حماية المجموعة
   ```

---

## أخطاء شائعة
- تضخيم ملف `Program.cs` بمنطق أعمال → انقله إلى **خدمات/Repo/Handlers**.  
- تجاهل **التحقق** وترك البيانات تدخل عشوائيًا (أضف **Filters** أو FluentValidation).  
- خلط **Binding** داخل الدالة مع وصول مباشر إلى الـ DB دون طبقة.  
- نسيان **Swagger/Versioning** → صعوبة الاستهلاك.  
- استعمال Minimal APIs لمشروع ضخم جدًا دون تنظيم → فكّر في **MVC/Razor Pages** أو تقسيم إلى **Modules**.

---

## جدول مقارنة مختصر

| المفهوم                       | الغرض الرئيسي                   | ملاحظات مهمة                                    |
| ----------------------------- | ------------------------------- | ----------------------------------------------- |
| **Minimal APIs**              | **REST خفيف وسريع البدء**       | **Handlers مباشرة، DI/Binding/Filters/Swagger** |
| [MVC](mvc.md)                 | فصل Controller/Views/Filters    | هيكل منطّم، مناسب للمشاريع الكبيرة               |
| [Razor Pages](razor-pages.md) | CRUD صفحات بسرعة                | صفحة = Model + Handlers؛ أقل ضوضاء              |
| sgRPC                          | تواصل عالي الأداء ثنائي الاتجاه | بروتوكول/Schema (Protobuf) بدل REST             |

---

## ملخص الفكرة  
**Minimal APIs** تمنحك REST **سريع الإنشاء** و**واضحًا** بنقاط نهاية مباشرة.  
حافظ على **المنطق خارج Program.cs**، أضِف **تحقق/توثيق/أمان**، وستحصل على خدمة صغيرة نظيفة قابلة للتوسّع لاحقًا.
