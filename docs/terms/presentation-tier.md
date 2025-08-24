# **Presentation Tier**

## الترجمة الحرفية  
**Presentation (Web/UI) Tier** — **طبقة العرض**.

## الوصف العربي المختصر  
الطبقة المواجهة للمستخدم/العميل: **HTTP/UI**، **نماذج طلب/رد**، **المصادقة/المعدّات الوسيطة**،  
و**تحويل** (Mapping) بين العالم الخارجي وطبقة الأعمال. لا تحتوي قواعد عمل معقّدة.

## الشرح المبسّط  
- تستقبل الطلبات، **تتحقق** من المدخلات، تتفاوض على المحتوى (JSON/…)، وتتعامل مع **الهوية**.  
- **تحوّل DTO ↔ Use Case**، وتعيد **أكواد حالة** ورسائل أخطاء موحّدة.  
- تتواصل مع **Business Tier** عبر **عقود** (Interfaces) — بلا SQL أو منطق مجال داخلي.

## تشبيه  
واجهة المطعم (الاستقبال): تستقبل الزبون، تتحقق من الحجز، تكتب الطلب **بنموذج موحّد**،  
ثم ترسله للمطبخ (طبقة الأعمال). لا تطبخ بنفسها.

---

## مثال C# — طبقة عرض **Minimal API** تتحدث إلى **Business Tier** عبر DTOs + أخطاء موحّدة

```csharp
// Web (Presentation Tier) — .NET 8/9
// dotnet new web -n WebApi && cd WebApi
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;

// ====== عقود طبقة الأعمال (يتم تنفيذها في Business/Data Tiers) ======
public interface IOrderService
{
    Task<PlaceOrderResult> PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct = default);
}
public record PlaceOrderCommand(int OrderId, string CustomerId, List<OrderItem> Items);
public record OrderItem(string Sku, int Qty, decimal Price);
public record PlaceOrderResult(int OrderId, decimal Total, string Status);

// ====== بدء تشغيل طبقة العرض ======
var builder = WebApplication.CreateBuilder(args);

// Presentation Tier duties: Filters/Middleware/Auth/RateLimit/ProblemDetails…
builder.Services.AddProblemDetails();
builder.Services.AddAuthorization();
builder.Services.AddEndpointsApiExplorer().AddSwaggerGen();

// حقن عقد الأعمال — التنفيذ الحقيقي يأتي من مشروع آخر
builder.Services.AddScoped<IOrderService, OrderServiceStub>(); // مؤقتًا

var app = builder.Build();
app.UseHttpsRedirection();
app.UseAuthorization();
app.UseExceptionHandler(); // يستخدم ProblemDetails تلقائيًا
app.UseSwagger().UseSwaggerUI();

// نقطة إنشاء طلب — تتحقق وتحوّل ثم تستدعي الأعمال
app.MapPost("/v1/orders", async ([FromBody] PlaceOrderHttp dto, IOrderService svc, HttpContext ctx, CancellationToken ct) =>
{
    // 1) تحقق تركيبي سريع
    var errs = Validate(dto);
    if (errs.Count > 0) return Results.ValidationProblem(errs);

    // 2) تحويل DTO (عرض) → Command (أعمال)
    var cmd = new PlaceOrderCommand(
        dto.OrderId,
        dto.CustomerId!,
        dto.Items!.Select(i => new OrderItem(i.Sku!, i.Qty, i.Price)).ToList());

    // 3) نداء حالة الاستخدام وإرجاع أكواد حالة دقيقة
    try
    {
        var res = await svc.PlaceOrderAsync(cmd, ct);
        return Results.Created($"/v1/orders/{res.OrderId}", res);
    }
    catch (ArgumentException ex)        { return Results.ValidationProblem(new() { [""] = new[] { ex.Message } }); }
    catch (InvalidOperationException ex){ return Results.BadRequest(new { error = ex.Message }); }
})
.Produces<PlaceOrderResult>(StatusCodes.Status201Created)
.ProducesProblem(StatusCodes.Status400BadRequest)
.ProducesValidationProblem(StatusCodes.Status422UnprocessableEntity)
.WithTags("Orders");

app.Run();


// ====== DTOs طبقة العرض + تحقق تركيبي بسيط ======
public class PlaceOrderHttp
{
    [Range(1, int.MaxValue)] public int OrderId { get; init; }
    [Required, MinLength(3)] public string? CustomerId { get; init; }
    [MinLength(1)] public List<PlaceOrderItemHttp>? Items { get; init; }
}
public class PlaceOrderItemHttp
{
    [Required, RegularExpression(@"^[A-Z0-9-]{3,20}$")] public string? Sku { get; init; }
    [Range(1, int.MaxValue)] public int Qty { get; init; }
    [Range(typeof(decimal), "0.01", "79228162514264337593543950335")] public decimal Price { get; init; }
}

static Dictionary<string, string[]> Validate(object model)
{
    var ctx = new ValidationContext(model);
    var results = new List<ValidationResult>();
    Validator.TryValidateObject(model, ctx, results, true);
    if (model is PlaceOrderHttp po && po.Items is not null)
        foreach (var it in po.Items)
            Validator.TryValidateObject(it, new ValidationContext(it), results, true);

    return results
        .GroupBy(r => r.MemberNames.FirstOrDefault() ?? "")
        .ToDictionary(g => g.Key, g => g.Select(r => r.ErrorMessage ?? "invalid").ToArray());
}

// ====== تنفيذ مؤقت لعقد الأعمال (Stub) — لأغراض العرض فقط ======
public class OrderServiceStub : IOrderService
{
    public Task<PlaceOrderResult> PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct = default)
    {
        if (cmd.Items.Sum(i => i.Price * i.Qty) <= 0) throw new InvalidOperationException("total_must_be_positive");
        return Task.FromResult(new PlaceOrderResult(cmd.OrderId, cmd.Items.Sum(i => i.Price * i.Qty), "confirmed"));
    }
}
```

**ماذا فعلت طبقة العرض؟**  
- تحقّق **تركيبي** + تحويل **DTO → Command**.  
- رجّعت **أكواد حالة** صحيحة (201/400/422).  
- لم تنفّذ قواعد العمل ولا SQL — كل ذلك في الطبقات الخلفية.

---

## خطوات عملية لبناء **Presentation Tier** سليم
1. **أكواد حالة واتفاق أخطاء موحّد** (`ProblemDetails`, رموز قابلة للترجمة).  
2. **Validation** على المدخلات، و**Sanitization** (Trim/Normalize) قبل التحويل.  
3. **AuthN/AuthZ**، **CORS**، **Rate Limiting**، و**Content Negotiation**.  
4. **Mapping** واضح بين DTO ↔ Use Case/Domain، مع **Versioning** (`/v1`).  
5. **Observability**: سجّل `Correlation-Id`، زمن الاستجابة، ونتائج التفويض.  
6. **لا منطق أعمال**: أي قاعدة غير مرتبطة بالنقل/العرض تُرحَّل للأعمال.

---

## أخطاء شائعة
- كتابة **قواعد العمل** داخل Controllers/Endpoints.  
- إرجاع **200** دائمًا بدل أكواد دقيقة (201/204/400/401/403/404/409/422).  
- تمرير **Entities DB** مباشرة للعميل بدل **DTO**.  
- تجاهل **التفاوض على المحتوى/الترميز** و**ETag/Cache** حيث يفيد.  
- عدم توحيد **أخطاء التحقق** → تجربة سيئة للواجهات.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Presentation Tier** | **استقبال الطلب/الهوية/التحقق/التحويل/الرد** | بلا قواعد عمل؛ يتحدث مع الأعمال عبر عقود |
| [Business Tier](business-tier.md) | تنفيذ حالات الاستخدام وقواعد المجال | ينسّق، يفرض السياسات، لا UI/HTTP |
| [Data Tier](data-tier.md) | تخزين واستعلام وتنفيذ العقود | DB/Cache/Files؛ لا تتعامل مع HTTP |

---

## ملخص الفكرة  
**Presentation Tier** = “باب النظام” على الويب.  
تحقّق من المدخلات، طبّق الهوية والسياسات السطحية، **حوّل** لـ **Business Tier**،  
وأعد استجابات **دقيقة** وموحّدة—تحصل على واجهة **نظيفة**، **آمنة**، وسهلة التطوير.
