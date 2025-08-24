# **REST — Representational State Transfer**

## الترجمة الحرفية  
**Representational State Transfer (REST)** — **نقل الحالة التمثيلية**.

## الوصف العربي المختصر  
أسلوب تصميم واجهات ويب يعرّف **موارد** (Resources) بعناوين **URI**،  
ويتعامل معها عبر **HTTP** بشكل **عديم الحالة** باستخدام دلالات الطرق (GET/POST/PUT/PATCH/DELETE)  
وتمثيلات قياسية (JSON/XML)، مع **أكواد حالة** واضحة و**تفاوض محتوى** و**تخزين مؤقت**.

## الشرح المبسّط  
- **كل شيء مورد**: منتج، طلب، مستخدم. لكل مورد **URI** ثابت.  
- **HTTP هو البروتوكول**:  
  - `GET` لجلب. `POST` لخلق. `PUT` لاستبدال. `PATCH` لتعديل جزئي. `DELETE` للحذف.  
  - **أكواد حالة**: 200/201/204/304/400/401/403/404/409/412/422/429/5xx.  
- **Stateless**: كل طلب مكتفٍ بذاته (Auth، Locale… ضمن الرؤوس/التوكن).  
- **تمثيلات**: JSON الأكثر شيوعًا. **Content-Type/Accept** للتفاوض.  
- **الروابط/الاكتشاف**: يمكن إرجاع روابط (HATEOAS) لتوضيح التنقل.  
- **التخزين المؤقت**: `ETag`/`Last-Modified` + `If-None-Match` لخفض النقل.  
- **التوافق**: الإصدار (Versioning) في المسار أو الرؤوس.

## تشبيه  
مكتبة عامة: لكل كتاب **رقم رفّ** (URI).  
تستخدم أفعالًا موحّدة: اقرأ (GET)، أضِف كتابًا (POST)، عدّل بيانات كتاب (PUT/PATCH)، احذفه (DELETE).  
اللوائح واضحة (أكواد حالة) والموظّف لا يتذكّر زيارتك السابقة (Stateless).

---

## مثال كود C# — **Minimal REST API** (قائمة منتجات + ETag + Concurrency)

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n RestDemo && cd RestDemo && dotnet run
using Microsoft.AspNetCore.Mvc;
using System.Security.Cryptography;
using System.Text;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IProductsRepo, InMemoryProductsRepo>();
var app = builder.Build();

app.UseHttpsRedirection();

var api = app.MapGroup("/v1/products").WithTags("Products");

// GET /v1/products?skip=0&take=20&q=ssd
api.MapGet("/", ([FromQuery] int skip, [FromQuery] int take, [FromQuery] string? q, IProductsRepo repo) =>
{
    skip = Math.Max(0, skip);
    take = take is <= 0 or > 100 ? 20 : take;
    var data = repo.All(q).Skip(skip).Take(take);
    return Results.Ok(new { items = data, count = data.Count() });
});

// GET /v1/products/{id}  — يدعم ETag/If-None-Match
api.MapGet("/{id:int}", (int id, HttpRequest req, IProductsRepo repo) =>
{
    var p = repo.Find(id);
    if (p is null) return Results.NotFound();

    var etag = ETag.Of(p.UpdatedAtUtc, p.Id); // توليد ETag بسيط
    if (req.Headers.IfNoneMatch == etag)      // Cache hit
        return Results.StatusCode(StatusCodes.Status304NotModified);

    return Results.Ok(p).WithETag(etag);
});

// POST /v1/products  — 201 Created + Location
api.MapPost("/", ([FromBody] ProductCreate dto, IProductsRepo repo, HttpContext ctx) =>
{
    if (string.IsNullOrWhiteSpace(dto.Name) || dto.Price <= 0)
        return Results.ValidationProblem(new() { ["name/price"] = ["اسم مطلوب وسعر > 0"] });

    var created = repo.Add(dto);
    var url = $"{ctx.Request.Scheme}://{ctx.Request.Host}/v1/products/{created.Id}";
    return Results.Created(url, created).WithETag(ETag.Of(created.UpdatedAtUtc, created.Id));
});

// PUT /v1/products/{id}  — استبدال كامل + تفادي تعارض باستخدام If-Match
api.MapPut("/{id:int}", (int id, [FromBody] ProductUpdate dto, HttpRequest req, IProductsRepo repo) =>
{
    var existing = repo.Find(id);
    if (existing is null) return Results.NotFound();

    var currentEtag = ETag.Of(existing.UpdatedAtUtc, existing.Id);
    var ifMatch = req.Headers.IfMatch.ToString();
    if (!string.IsNullOrEmpty(ifMatch) && ifMatch != currentEtag)
        return Results.StatusCode(StatusCodes.Status412PreconditionFailed); // تعارض نسخة

    if (string.IsNullOrWhiteSpace(dto.Name) || dto.Price <= 0)
        return Results.ValidationProblem(new() { ["name/price"] = ["اسم مطلوب وسعر > 0"] });

    var updated = repo.Replace(id, dto);
    return Results.Ok(updated).WithETag(ETag.Of(updated.UpdatedAtUtc, updated.Id));
});

// DELETE /v1/products/{id}  — 204 No Content
api.MapDelete("/{id:int}", (int id, IProductsRepo repo) =>
{
    return repo.Delete(id) ? Results.NoContent() : Results.NotFound();
});

app.Run();


// ====== النماذج/المستودع والـ ETag ======
public record Product(int Id, string Name, decimal Price, DateTime UpdatedAtUtc);
public record ProductCreate(string Name, decimal Price);
public record ProductUpdate(string Name, decimal Price);

public interface IProductsRepo
{
    IEnumerable<Product> All(string? q = null);
    Product? Find(int id);
    Product Add(ProductCreate dto);
    Product Replace(int id, ProductUpdate dto);
    bool Delete(int id);
}

public class InMemoryProductsRepo : IProductsRepo
{
    private readonly List<Product> _db = new() {
        new(1,"Keyboard",120, DateTime.UtcNow),
        new(2,"Mouse",60,    DateTime.UtcNow)
    };
    private int _next = 3;

    public IEnumerable<Product> All(string? q = null)
        => string.IsNullOrWhiteSpace(q) ? _db : _db.Where(p => p.Name.Contains(q, StringComparison.OrdinalIgnoreCase));

    public Product? Find(int id) => _db.FirstOrDefault(p => p.Id == id);

    public Product Add(ProductCreate dto)
    {
        var p = new Product(_next++, dto.Name, dto.Price, DateTime.UtcNow);
        _db.Add(p); return p;
    }

    public Product Replace(int id, ProductUpdate dto)
    {
        var i = _db.FindIndex(p => p.Id == id);
        var updated = new Product(id, dto.Name, dto.Price, DateTime.UtcNow);
        _db[i] = updated; return updated;
    }

    public bool Delete(int id) => _db.RemoveAll(p => p.Id == id) > 0;
}

public static class ETag
{
    public static string Of(DateTime updatedUtc, int id)
    {
        // ETag ضعيف مبسّط يعتمد على (id + ticks)
        var payload = $"{id}-{updatedUtc.Ticks}";
        var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(payload));
        return $"W/\"{Convert.ToBase64String(bytes)[..16]}\"";
    }

    public static IResult WithETag(this IResult res, string etag)
        => Results.Extensions(res, ctx => ctx.Response.Headers.ETag = etag);
}
```

**جرّب سريعًا (curl):**
```bash
# قائمة
curl -s http://localhost:5000/v1/products

# عنصر واحد + ETag
curl -i http://localhost:5000/v1/products/1
# استخدم قيمة ETag من الاستجابة:
curl -i http://localhost:5000/v1/products/1 -H 'If-None-Match: W/"abc..."'   # 304

# إنشاء
curl -i -X POST http://localhost:5000/v1/products \
  -H 'Content-Type: application/json' \
  -d '{ "name":"SSD", "price":250 }'

# تحديث مع If-Match (تفادي الكتابة المفقودة)
curl -i http://localhost:5000/v1/products/2
# ضع ETag الذي استلمته:
curl -i -X PUT http://localhost:5000/v1/products/2 \
  -H 'Content-Type: application/json' -H 'If-Match: W/"abc..."' \
  -d '{ "name":"Mouse Pro", "price":75 }'
```

---

## خطوات عملية لتصميم REST سليم
1. **نمذجة الموارد**: أسماء جمع واضحة (`/users`, `/orders/{id}/items`).  
2. **دلالات HTTP** صحيحة: لا تستخدم `POST` للتعديل إذا كان `PUT/PATCH` مناسبًا.  
3. **أكواد حالة** دقيقة + رسائل خطأ مفهومة (حقول `code`, `message`, `details`).  
4. **التصفية/الترقيم/الفرز** عبر استعلامات (`?q=&sort=&skip=&take=`).  
5. **التوافق المتدرّج**: أضف خصائص جديدة دون كسر العملاء؛ للإضافات الكسّارة استخدم **الإصدار** (`/v1`).  
6. **التخزين المؤقت**: `ETag/If-None-Match`, `Cache-Control`, و`Last-Modified`.  
7. **الوثائق**: OpenAPI/Swagger + أمثلة وقيود.  
8. **الأمان**: HTTPS، Auth (JWT/OAuth2)، CORS، حدود معدل (429)، إدخال آمن.  
9. **المراقبة**: Trace/Log/Correlation-Id، مقاييس (`2xx/4xx/5xx`, زمن الاستجابة).

---

## أخطاء شائعة
- استخدام مسارات **أفعال** بدل **أسماء موارد** (`/createUser` بدل `/users`).  
- تجاهل **أكواد الحالة** أو إرجاع 200 دائمًا.  
- خلط دلالات الطرق (تعديل بـ `GET`، أو حذف بـ `POST`).  
- عدم دعم **الترقيم/التصفية** → استجابات ضخمة.  
- كسر التوافق بإزالة حقول دون إصدار جديد.  
- نسيان **التخزين المؤقت/ETag** أو **Concurrency** (`If-Match`) → سباقات تحديث.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **REST** | **موارد على HTTP بدلالات قياسية** | بسيط، واسع الانتشار، يدعم التخزين المؤقت |
| RPC/JSON-RPC | استدعاء إجراءات مباشرة | أفعال على طرق؛ ليس موجّه موارد |
| **gRPC** | ثنائي عالي الأداء مع Streaming | يتطلب HTTP/2 وملفات `.proto` |
| GraphQL | جلب بيانات مرن بطلب واحد | نقطة واحدة؛ تعقيد في التخزين المؤقت/الأمن |
| **SOAP** | رسائل XML بعقد WSDL وWS-* | ثقيل لكنه غنيّ بالمعايير المؤسسية |

---

## ملخص الفكرة  
**REST** يوفّر واجهات **واضحة وقابلة للتوسع** عبر موارد وطرق HTTP وأكواد حالة.  
نمذج مواردك جيّدًا، استخدم **دلالات صحيحة**، وثّق بـ **OpenAPI**، وفعّل **ETag/Versioning**—  
تحصل على API **متين**، **قابل للصيانة**، وسهل الاستهلاك. 
