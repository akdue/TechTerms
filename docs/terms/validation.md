# **Validation**

## الترجمة الحرفية  
**Validation** — **التحقّق** (هل البيانات **صحيحة** ومسموح بها؟).

## الوصف العربي المختصر  
عملية تفحص المدخلات وفق **قواعد تركيبية** (الشكل/النمط) و**قواعد دلالية** (المعنى/القواعد التجارية)،  
قبل قبولها أو تخزينها أو تمريرها. هدفها **منع الأخطاء** و**سدّ الثغرات** مبكرًا.

## الشرح المبسّط  
- **Syntactic**: الشكل صحيح؟ (بريد، رقم، طول، Regex).  
- **Semantic**: المعنى صحيح؟ (المجموع > 0، الحالة تسمح، التاريخ منطقي).  
- **Server-side أو Client-side**: نعرض أخطاء مبكرًا على الواجهة، لكن **الخادم هو الحكم**.  
- **Defence-in-depth**: تحقّق في الطبقات (API → Domain → DB constraints).

## تشبيه  
موظف استقبال يأخذ استمارة: يتحقّق من **ملء الحقول بشكل صحيح** (تركيبي)،  
ثم يتأكد أن **الشروط تنطبق** (دلالي) قبل ختم “مقبول”.

---

## مثال C# — Minimal API مع **DataAnnotations** + تحقّق دلالي + أخطاء موحّدة

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n ValidationDemo && cd ValidationDemo && dotnet run
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapPost("/orders", ([FromBody] OrderCreate dto) =>
{
    // 1) فحص DataAnnotations (تركيبي)
    var errors = Validate(dto);

    // 2) فحص دلالي عبر قواعد المجال (Cross-field/Business)
    if (dto.Items.Sum(i => i.Qty * i.Price) <= 0)
        AddError(errors, nameof(OrderCreate.Items), "total_must_be_positive");

    if (dto.Items.Count > 100)
        AddError(errors, nameof(OrderCreate.Items), "too_many_items");

    if (errors.Count > 0) return Results.ValidationProblem(errors);

    // نجاح: تابع المعالجة/الحفظ
    var id = Guid.NewGuid().ToString("N");
    return Results.Created($"/orders/{id}", new { id, total = dto.Items.Sum(i => i.Qty * i.Price) });
});

app.Run();

// ====== النماذج مع DataAnnotations (تحقق تركيبي) ======
public class OrderCreate
{
    [Required, EmailAddress] public string CustomerEmail { get; init; } = "";
    [Required, RegularExpression(@"^[A-Z]{3}$")] public string Currency { get; init; } = "USD"; // ISO-4217
    [MinLength(1)] public List<OrderItem> Items { get; init; } = new();
}

public class OrderItem
{
    [Required, RegularExpression(@"^[A-Z0-9-]{3,20}$")] public string Sku { get; init; } = "";
    [Range(1, int.MaxValue)] public int Qty { get; init; }
    [Range(typeof(decimal), "0.01", "79228162514264337593543950335")] public decimal Price { get; init; }
}

// ====== أدوات مساعدة لأخطاء موحّدة ======
static Dictionary<string, string[]> Validate(object model)
{
    var ctx = new ValidationContext(model);
    var results = new List<ValidationResult>();
    _ = Validator.TryValidateObject(model, ctx, results, validateAllProperties: true);

    // تحقّق متداخل للعناصر (Items)
    if (model is OrderCreate oc)
        foreach (var it in oc.Items)
        {
            var r = new List<ValidationResult>();
            _ = Validator.TryValidateObject(it, new ValidationContext(it), r, true);
            foreach (var e in r) results.Add(new ValidationResult($"Items: {e.ErrorMessage}", e.MemberNames.Select(m => $"Items.{m}")));
        }

    return results
        .GroupBy(e => e.MemberNames.FirstOrDefault() ?? "")
        .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage ?? "invalid").ToArray());
}

static void AddError(Dictionary<string, string[]> dict, string key, string message)
{
    if (!dict.TryGetValue(key, out var arr)) dict[key] = new[] { message };
    else dict[key] = arr.Concat(new[] { message }).ToArray();
}
```

**الفكرة:**  
- **DataAnnotations** تغطي الشكل (Email/Regex/Range).  
- نضيف **قواعد عمل** (مجموع موجب، حد أقصى للعناصر).  
- نرجع **ValidationProblem** موحّدًا يفيد الواجهة.

---

## خطوات عملية لبناء تحقق قوي
1. **قسّم القواعد**: تركيبي (Annotations/Schema) ثم دلالي (قواعد المجال).  
2. **اعرض الأخطاء مبكرًا** على الواجهة، لكن **تحقق على الخادم** دائمًا.  
3. **هيّئ رسائل موحّدة** (رموز أخطاء قابلة للترجمة) بدل نصوص حرّة.  
4. **ضع حدودًا** للأحجام/الطول/الملفات (MIME/حجم) لمنع إساءة الخدمة.  
5. **Whitelist** بدلاً من **Blacklist** (اسمح بما هو متوقع فقط).  
6. **فرّق بين** Validation (القبول/الرفض) و**Sanitization** (تنظيف/تطبيع مثل Trim/Unicode Normalization).  
7. **تحقق متعدد الطبقات**: API → Domain → DB (قيود فريدة/أنواع/FK).  
8. **اختبر الحواف**: قيم قصوى/صفرية/Unicode/RTL/JSON كبير/ترميز.

---

## أخطاء شائعة
- الاعتماد على **عميل فقط** (JS) بلا تحقق خادمي.  
- Regex معقّد للبريد/الأسماء يرفض الصحيح أو يقبل الخطأ.  
- **Blacklist** لسلاسل “خطرة” بدل قيود إيجابية (Whitelist).  
- رسائل غير موحّدة تصعّب تعريب/تجربة المستخدم.  
- رمي استثناءات عامّة بدل **422 Unprocessable Entity** (أو ValidationProblem).  
- خلط **التنظيف** (Encoding/HTML escape) مع **التحقق** (قرار قبول/رفض).  
- تجاهل **الترميز**/التطبيع → تباينات Unicode (NFC/NFD).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Validation** | **قبول/رفض البيانات وفق قواعد** | تركيبي + دلالي، على الخادم أساسًا |
| Sanitization | تنظيف/تطبيع البيانات | Trim/Normalize/Encode؛ لا يثبت الصحة |
| Client-side | تجربة أفضل فوريًا | لا يُعتمد عليه وحده |
| Server-side | قرار نهائي موثوق | أعِد 400/422 مع تفاصيل |
| DataAnnotations | تحقق بسيط تركيبي | Range/Regex/Required… |
| FluentValidation | قواعد غنية وقابلة للاختبار | DSL مرن، اندماج ممتاز مع ASP.NET |
| DB Constraints | خط دفاع أخير | فريد/Not Null/Check/FK |

> إن أردت DSL أقوى، استخدم **FluentValidation**:
> ```csharp
> // dotnet add package FluentValidation.AspNetCore
> builder.Services.AddValidatorsFromAssemblyContaining<OrderCreateValidator>();
> public class OrderCreateValidator : AbstractValidator<OrderCreate>
> {
>   public OrderCreateValidator()
>   {
>     RuleFor(x => x.CustomerEmail).NotEmpty().EmailAddress();
>     RuleFor(x => x.Currency).Matches("^[A-Z]{3}$");
>     RuleFor(x => x.Items).NotEmpty().Must(list => list.Sum(i => i.Qty * i.Price) > 0);
>   }
> }
> ```

---

## ملخص الفكرة  
**Validation** يمنع الأخطاء والثغرات عند **الباب**.  
ابدأ بتحقق **تركيبي**، ثم **دلالي**، وارجع **أخطاء موحّدة**، واحتفظ بقيود قاعدة البيانات—  
تحصل على نظام **متين**، **سهل الاستخدام**، وآمن ضد مدخلات معطوبة.
