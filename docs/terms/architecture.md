# **Architecture**

## الترجمة الحرفية  
**(Software / System) Architecture** — **المعماريّة (البرمجيات/النُّظم)**

## الوصف العربي المختصر  
خارطة عليا **تقسّم النظام** إلى مكوّنات وحدود وتدفقات.  
قراراتها موجّهة بـ **سمات الجودة**: الأداء، القابلية للصيانة، الاختبارية، الأمان، القابلية للتوسّع.

## الشرح المبسّط  
- تحدد **المكوّنات**، **الواجهات**، و**آلية التواصل** بينها.  
- تختار **نمطًا معماريًا**: طبقات (Layered)، سداسي/منافذ ومحوّلات (Hexagonal/Ports & Adapters)، أحداث (Event-Driven)، خدمات صغرى (Microservices)، أحادي (Monolith)…  
- **المعماريّة ≠ التصميم التفصيلي**: الأولى *مبادئ وحدود*، الثانية تفاصيل الأصناف والخوارزميات.  
- توثَّق عبر **C4**، **ADR** (سجلات قرار)، وخرائط تدفّق.

## تشبيه  
مخطط **مهندس معماري** للمول: مناطق، مداخل، مخارج، مصاعد، ولوائح أمان.  
التفاصيل الداخلية للمتاجر = **تصميم تفصيلي**.

---

## مثال C# عملي (Ports & Adapters بأسلوب بسيط على ASP.NET Core)

> الفكرة: **المجال/التطبيق** يعرف **منافذ** (Interfaces).  
> البنية التحتية توفّر **محوّلات** لها. القوائم والاعتمادات **تُحقن عبر DI**.

```csharp
// Program.cs  (.NET 8/9)  —  dotnet new web -n Shop
using System.Linq;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// تسجيل المنافذ (Ports) بتنفيذات البنية التحتية (Adapters)
builder.Services.AddScoped<IPaymentGateway, StripeGateway>();
builder.Services.AddScoped<CheckoutHandler>();

var app = builder.Build();

// حواف (Adapters) دخول: HTTP Endpoints تستدعي حالات الاستخدام
app.MapPost("/checkout", async (CheckoutRequest req, CheckoutHandler useCase)
    => await useCase.HandleAsync(req));

app.Run();

// ====== Domain (Entities + Ports) ======
public record Money(decimal Amount, string Currency = "USD");
public record CartItem(string Sku, int Qty, Money UnitPrice);

public interface IPaymentGateway  // Port
{
    Task<string> ChargeAsync(Money amount, string customerId, CancellationToken ct = default);
}

// ====== Application (Use Case) ======
public record CheckoutRequest(string CustomerId, CartItem[] Items);

public class CheckoutHandler
{
    private readonly IPaymentGateway _payments;
    public CheckoutHandler(IPaymentGateway payments) => _payments = payments;

    public async Task<IResult> HandleAsync(CheckoutRequest req, CancellationToken ct = default)
    {
        var total = req.Items.Sum(i => i.UnitPrice.Amount * i.Qty);
        var txId = await _payments.ChargeAsync(new Money(total), req.CustomerId, ct);
        return Results.Ok(new { total, txId });
    }
}

// ====== Infrastructure (Adapter) ======
public class StripeGateway : IPaymentGateway
{
    public Task<string> ChargeAsync(Money amount, string customerId, CancellationToken ct = default)
        => Task.FromResult($"tx_{Guid.NewGuid():N}"); // تمثيل بسيط؛ الحقيقي يستدعي HTTP
}
```

**الإخراج المتوقع (مختصر):**  
`POST /checkout` بجسم يحوي عناصر السلّة → JSON فيه `total` و`txId`.  
التطبيق **لا يعرف Stripe** مباشرة؛ يعرف **المنفذ** فقط → قابلية استبدال عالية.

---

## خطوات عملية لبناء معماريّة نافعة
- اكتب **الهدف وسِمات الجودة** (NFRs): زمن الاستجابة، معدل الطلبات، مستويات توفّر…  
- ارسم **C4 (Context/Container/Component)** سريعًا.  
- اختر **النمط** الأنسب (Monolith-Modular أولًا، ثم Microservices عند الحاجة الحقيقية).  
- ثبّت الحدود بعقود واضحة (DTOs/Interfaces/Events).  
- وثّق قراراتك في **ADR** قصيرة (قرار/بدائل/سبب/تأثير).  
- طبّق **DI**، و**افصل المجال** عن المنصّات (HTTP/DB/Message Bus).  
- أضِف **مراقبة** منذ اليوم الأول: سجلات، قياسات، تتبّع موزّع.  
- اختبر **المرونة**: فشل خدمة خارجية، بطء قاعدة البيانات، انقطاع شبكة.

---

## أخطاء شائعة
- **معماريّة سابقة لأوانها**: تقسيم مبالغ فيه قبل فهم المجال (YAGNI).  
- **خلط الأدوار**: منطق المجال يتسرّب إلى Controllers أو Repositories.  
- **اعتماد إطار = معماريّة** (اختيار Framework لا يغني عن الحدود).  
- **Microservices مبكّرًا** بدون منصة مراقبة/أتمتة → تعقيد وتشظّي.  
- تجاهل **NFRs** والتركيز على “الشكل” فقط.  
- انعدام **الانضباط على التبعيات** (دوائر بين المكوّنات).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Software Architecture** | **حدود ومكوّنات وتدفّقات لتحقيق سمات الجودة** | أنماط: Layered, Hexagonal, Event-Driven, Microservices |
| System Architecture | تصور أعلى يضم العتاد والشبكات وأنظمة خارجية | خرائط نشر، شبكات، أمن، ربط أنظمة |
| Solution/High-Level Design | تفصيل معماريّة الحل لمنتج محدّد | جسْر بين المعماريّة والتنفيذ؛ قرارات تقنية |
| CPU/ISA Architecture | مجموعة التعليمات والمعمارية (x86-64/ARM64) | تؤثّر على بناء الصور (OCI) والتوافق والسرعة |
| [Framework](framework.md) | أدوات/هياكل للتنفيذ داخل المكوّنات | **ليس** معماريّة بحد ذاته؛ يُستخدم داخل الحدود |

---

## ملخص الفكرة  
**المعماريّة** تحدّد *الحدود والواجهات والقرارات الكبرى* التي تجعل النظام قابلًا للتغيير والاختبار والتشغيل.  
ابدأ بسيطًا، ركّز على **سمات الجودة**، افصل **المجال** عن **البنية التحتية**، ووثّق قراراتك بـ **ADR**—ثم دع التفاصيل تتبع الحدود لا العكس.
