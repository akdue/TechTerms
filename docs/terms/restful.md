# **RESTful**

## الترجمة الحرفية  
**RESTful API** — **واجهة برمجية تطبّق مبادئ REST** على نحوٍ عملي (موارد، دلالات HTTP، تمثيلات، وروابط).

## الوصف العربي المختصر  
واجهة **موجّهة موارد** تستخدم **HTTP** بشكل **عديم الحالة**، تعيد **تمثيلات** (JSON/…)،  
تستخدم **أكواد حالة** صحيحة، **ETag/Cache**، و—عند الاكتمال—توفّر **روابط (HATEOAS)** لاكتشاف التدفّق.

## الشرح المبسّط  
- **المورد له URI ثابت**: `/v1/books/42`.  
- **الأفعال HTTP** تحمل المعنى: `GET/POST/PUT/PATCH/DELETE`.  
- **Stateless**: كل طلب مستقل (Auth/Locale ضمن الرؤوس).  
- **إصدارات** واضحة (`/v1`) وتطوّر متدرّج غير كسّار.  
- **قابلية التخزين المؤقت**: `ETag/If-None-Match`, `Cache-Control`.  
- **HATEOAS** (اختياري لكنه “أكمل”): التمثيل يعيد **روابط** (`_links`) لخطوات قادمة.  
- **سلامة/إديمبوتنس**: `GET` آمن، `PUT/DELETE` **Idempotent**، و`POST` ليس كذلك.

## تشبيه  
مكتبة إلكترونية منظّمة: لكل كتاب عنوان ثابت، لديك أفعال موحّدة (اقرأ/أضِف/عدّل/احذف).  
كل صفحة توصلك بروابط إلى “التالي/السابق/التفاصيل”.

---

## مثال C# — **Minimal RESTful API** مع HATEOAS + ETag + Concurrency

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n RestfulDemo && cd RestfulDemo && dotnet run
using Microsoft.AspNetCore.Mvc;
using System.Security.Cryptography;
using System.Text;

// ====== التركيب والخدمات ======
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IBooksRepo, InMemoryBooksRepo>();
var app = builder.Build();

app.UseHttpsRedirection();

string Base(string path) => $"/v1/books{path}";

// توليد ETag ضعيف (كافٍ للمثال)
static string MakeETag(Book b)
{
    var payload = $"{b.Id}-{b.UpdatedAtUtc.Ticks}";
    var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(payload));
    return $"W/\"{Convert.ToBase64String(bytes)[..16]}\"";
}

// تحويل Book إلى تمثيل RESTful مع روابط
static object ToResource(Book b) => new
{
    b.Id, b.Title, b.Price,
    _links = new {
        self       = new { href = $"/v1/books/{b.Id}", method = "GET" },
        update     = new { href = $"/v1/books/{b.Id}", method = "PUT" },
        delete     = new { href = $"/v1/books/{b.Id}", method = "DELETE" },
        collection = new { href = "/v1/books",         method = "GET" }
    }
};

// ====== المسارات ======
var api = app.MapGroup("/v1/books").WithTags("Books");

// GET /v1/books?skip=0&take=20&q=ddd  (قائمة + روابط تنقّل)
api.MapGet("/", ([FromQuery] int skip, [FromQuery] int take, [FromQuery] string? q, IBooksRepo repo, HttpContext ctx) =>
{
    skip = Math.Max(0, skip);
    take = take is <= 0 or > 100 ? 20 : take;

    var all = repo.All(q).ToList();
    var page = all.Skip(skip).Take(take).Select(ToResource).ToList();

    string self = $"{Base("")}?skip={skip}&take={take}" + (string.IsNullOrEmpty(q) ? "" : $"&q={Uri.EscapeDataString(q)}");
    string? next = (skip + take < all.Count) ? $"{Base("")}?skip={skip + take}&take={take}" + (string.IsNullOrEmpty(q) ? "" : $"&q={Uri.EscapeDataString(q)}") : null;
    string? prev = (skip > 0) ? $"{Base("")}?skip={Math.Max(0, skip - take)}&take={take}" + (string.IsNullOrEmpty(q) ? "" : $"&q={Uri.EscapeDataString(q)}") : null;

    return Results.Ok(new {
        count = all.Count,
        items = page,
        _links = new {
            self = new { href = self, method = "GET" },
            next = next is null ? null : new { href = next, method = "GET" },
            prev = prev is null ? null : new { href = prev, method = "GET" }
        }
    });
});

// GET /v1/books/{id}  (يدعم ETag/If-None-Match)
api.MapGet("/{id:int}", (int id, HttpRequest req, IBooksRepo repo) =>
{
    var b = repo.Find(id);
    if (b is null) return Results.NotFound();

    var etag = MakeETag(b);
    if (req.Headers.IfNoneMatch == etag)
        return Results.StatusCode(StatusCodes.Status304NotModified);

    var res = Results.Ok(ToResource(b));
    res.EnableBuffering(); // للسماح بإضافة رأس قبل الإرسال (اختياري)
    return res.WithETag(etag);
});

// POST /v1/books  (201 + Location + تمثيل)
api.MapPost("/", ([FromBody] BookCreate dto, IBooksRepo repo, HttpContext ctx) =>
{
    if (string.IsNullOrWhiteSpace(dto.Title) || dto.Price <= 0)
        return Results.ValidationProblem(new(){{"title/price",["عنوان مطلوب وسعر > 0"]}});

    var b = repo.Add(dto);
    var etag = MakeETag(b);
    var url = $"{ctx.Request.Scheme}://{ctx.Request.Host}{Base($"/{b.Id}")}";
    return Results.Created(url, ToResource(b)).WithETag(etag);
});

// PUT /v1/books/{id}  (Idempotent + If-Match لتفادي تعارض النسخ)
api.MapPut("/{id:int}", (int id, [FromBody] BookUpdate dto, HttpRequest req, IBooksRepo repo) =>
{
    var cur = repo.Find(id);
    if (cur is null) return Results.NotFound();

    var currentEtag = MakeETag(cur);
    var ifMatch = req.Headers.IfMatch.ToString();
    if (!string.IsNullOrEmpty(ifMatch) && ifMatch != currentEtag)
        return Results.StatusCode(StatusCodes.Status412PreconditionFailed);

    if (string.IsNullOrWhiteSpace(dto.Title) || dto.Price <= 0)
        return Results.ValidationProblem(new(){{"title/price",["عنوان مطلوب وسعر > 0"]}});

    var updated = repo.Replace(id, dto);
    return Results.Ok(ToResource(updated)).WithETag(MakeETag(updated));
});

// DELETE /v1/books/{id}  (Idempotent — 204)
api.MapDelete("/{id:int}", (int id, IBooksRepo repo)
    => repo.Delete(id) ? Results.NoContent() : Results.NotFound());

app.Run();

// ====== النماذج/المستودع ======
public record Book(int Id, string Title, decimal Price, DateTime UpdatedAtUtc);
public record BookCreate(string Title, decimal Price);
public record BookUpdate(string Title, decimal Price);

public interface IBooksRepo
{
    IEnumerable<Book> All(string? q = null);
    Book? Find(int id);
    Book Add(BookCreate dto);
    Book Replace(int id, BookUpdate dto);
    bool Delete(int id);
}

public class InMemoryBooksRepo : IBooksRepo
{
    private readonly List<Book> _db = new() {
        new(1,"DDD Quickly",120, DateTime.UtcNow),
        new(2,"Clean Code",  90, DateTime.UtcNow)
    };
    private int _next = 3;

    public IEnumerable<Book> All(string? q = null)
        => string.IsNullOrWhiteSpace(q) ? _db : _db.Where(b => b.Title.Contains(q, StringComparison.OrdinalIgnoreCase));

    public Book? Find(int id) => _db.FirstOrDefault(p => p.Id == id);

    public Book Add(BookCreate dto)
    {
        var b = new Book(_next++, dto.Title, dto.Price, DateTime.UtcNow);
        _db.Add(b); return b;
    }

    public Book Replace(int id, BookUpdate dto)
    {
        var i = _db.FindIndex(p => p.Id == id);
        var updated = new Book(id, dto.Title, dto.Price, DateTime.UtcNow);
        _db[i] = updated; return updated;
    }

    public bool Delete(int id) => _db.RemoveAll(p => p.Id == id) > 0;
}

// ====== امتداد بسيط لإضافة ETag ======
public static class ResultExtensions
{
    public static IResult WithETag(this IResult res, string etag)
        => Results.Extensions(res, ctx => ctx.Response.Headers.ETag = etag);
}
```

**تجربة سريعة (curl):**
```bash
# قائمة + روابط تنقل
curl -s http://localhost:5000/v1/books

# عنصر واحد + ETag
curl -i http://localhost:5000/v1/books/1
# جرّب التخزين المؤقت
curl -i http://localhost:5000/v1/books/1 -H 'If-None-Match: W/"abc..."'   # 304

# إنشاء (201 + Location + روابط)
curl -i -X POST http://localhost:5000/v1/books \
  -H 'Content-Type: application/json' \
  -d '{ "title":"The Pragmatic Programmer", "price":150 }'

# تحديث محمي بـ If-Match (تفادي الكتابة المفقودة)
ET=$(curl -si http://localhost:5000/v1/books/2 | grep ETag | cut -d' ' -f2)
curl -i -X PUT http://localhost:5000/v1/books/2 \
  -H "If-Match: $ET" -H 'Content-Type: application/json' \
  -d '{ "title":"Clean Code (2nd)", "price":95 }'
```

---

## خطوات عملية لبناء API **RESTful**
1. **نمذجة الموارد** بأسماء جمع واضحة وروابط بينية (`_links`).  
2. **استخدم دلالات HTTP الصحيحة** + أكواد حالة دقيقة (`201/204/304/409/412/422`).  
3. **التخزين المؤقت والتزامن**: `ETag/If-None-Match` للقراءة، و`If-Match` للتحديث.  
4. **إصدار** (`/v1`) وتطوّر **غير كسّار** (إضافة خصائص بدل حذفها).  
5. **الأمن**: HTTPS، Auth (JWT/OAuth2)، حدود معدل (429)، CORS.  
6. **التوثيق**: OpenAPI/Swagger + أمثلة واقعية وأخطاء موحّدة.  
7. **المراقبة**: Logs/Trace/Correlation-Id ومقاييس (`2xx/4xx/5xx`, زمن).

---

## أخطاء شائعة
- مسارات أفعال (`/createBook`) بدل **موارد** (`/books`).  
- تجاهل **HATEOAS/روابط** كليًا ثم تعقيد الاكتشاف.  
- خلط الدلالات (تعديل بـ `GET` أو حذف بـ `POST`).  
- عدم دعم **ETag/If-Match** → سباقات وتحديثات مفقودة.  
- كسر التوافق بإزالة حقول دون إصدار جديد.  
- ردّ 200 دائمًا بدل أكواد دقيقة (201/204/404/409/412).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **RESTful** | **تطبيق عملي لمبادئ REST على واجهة حقيقية** | موارد، دلالات HTTP، أكواد حالة، **(اختياري) HATEOAS** |
| [REST](rest.md) | أسلوب/نمط معماري | مبادئ عامة؛ لا يلزمك تنفيذًا محددًا |
| RPC/JSON-RPC | استدعاء إجراءات مباشرة | أسماء أفعال؛ ليس موجّه موارد |
| GraphQL | جلب مرن بكيان واحد | تخزين مؤقت أعقد؛ نقطة واحدة |
| gRPC | أداء عالٍ وStreaming عبر HTTP/2 | ملفات `.proto` وتوليد كود |
| SOAP | رسائل XML بعقد WSDL | أثقل؛ معايير مؤسسية (WS-*) |

---

## ملخص الفكرة  
**RESTful** يعني أن APIّك **يعيش مبادئ REST**: موارد واضحة، أفعال صحيحة، أكواد دقيقة،  
تمثيلات غنية وروابط، مع **ETag/Versioning** وأمن ومراقبة—فتحصل على واجهة **متينة**،  
**قابلة للتوسع** و**سهلة الاستهلاك** عبر المنصّات. 
