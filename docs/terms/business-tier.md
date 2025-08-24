# **Business Tier**

## الترجمة الحرفية  
**Business (Application) Tier** — **طبقة الأعمال/التطبيق**.

## الوصف العربي المختصر  
الطبقة الوسطى التي تحتوي **قواعد العمل وحالات الاستخدام** (Use Cases).  
تنسّق العمليات، تفرض السياسات، وتتعامل مع طبقة البيانات/الخدمات الخارجية عبر **عقود** واضحة.

## الشرح المبسّط  
- **ماذا تفعل؟** تنفيذ منطق المجال: تسعير، خصومات، صلاحيات، تدفّق طلبات.  
- **ماذا لا تفعل؟** لا UI، ولا SQL مباشر. تتعامل مع DB عبر **واجهات** (Repositories/UoW).  
- **كيف تتحدث؟** تستقبل **DTOs** من الواجهة، وتعيد **نتائج/أحداث**، وتستدعي تكاملات (Email/Payments) عبر عقود.

## تشبيه  
في مطعم: **المطبخ** = طبقة الأعمال. لا يكلّم المخزن مباشرة؛ يرسل طلبات عبر بوابة الاستلام.  
يطبخ وفق الوصفة (قواعد العمل) ويُنسّق بين الشيفات (خطوات/حالات استخدام).

---

## مثال C# — خدمة أعمال في طبقة مستقلّة + حدود واضحة

```csharp
// مشروع Business (طبقة الأعمال) — .NET 8/9
// يعتمد فقط على عقود (Interfaces) ونماذج مجال بسيطة. لا مرجع لـ ASP.NET أو EF.

// ====== عقود البنية التحتية (تعريفها هنا، تنفيذها في طبقة البيانات) ======
public interface IOrderRepository
{
    Task<Order?> GetAsync(int id, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    Task SaveChangesAsync(CancellationToken ct = default);
}
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(string customerId, decimal amount, CancellationToken ct = default);
}
public record PaymentResult(bool Success, string? Error = null);

// ====== نماذج مجال مبسّطة ======
public class Order
{
    private readonly List<OrderLine> _lines = new();
    public int Id { get; }
    public string CustomerId { get; }
    public IReadOnlyList<OrderLine> Lines => _lines;
    public decimal Total => _lines.Sum(l => l.Price * l.Qty);

    public Order(int id, string customerId) { Id = id; CustomerId = customerId; }

    public void AddLine(string sku, int qty, decimal price)
    {
        if (qty <= 0) throw new ArgumentOutOfRangeException(nameof(qty));
        if (price <= 0) throw new ArgumentOutOfRangeException(nameof(price));
        _lines.Add(new OrderLine(sku, qty, price));
    }
}
public record OrderLine(string Sku, int Qty, decimal Price);

// ====== DTOs لطبقة الأعمال ======
public record PlaceOrderCommand(int OrderId, string CustomerId, List<OrderItemDto> Items);
public record OrderItemDto(string Sku, int Qty, decimal Price);
public record PlaceOrderResult(int OrderId, decimal Total, string Status);

// ====== خدمة الأعمال (Use Case) ======
public class OrderService
{
    private readonly IOrderRepository _repo;
    private readonly IPaymentGateway _payments;

    public OrderService(IOrderRepository repo, IPaymentGateway payments)
    { _repo = repo; _payments = payments; }

    public async Task<PlaceOrderResult> PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct = default)
    {
        // تحقق دلالي (قواعد عمل)
        if (string.IsNullOrWhiteSpace(cmd.CustomerId)) throw new ArgumentException("customer_required");
        if (cmd.Items is null || cmd.Items.Count == 0) throw new ArgumentException("items_required");

        var order = new Order(cmd.OrderId, cmd.CustomerId);
        foreach (var i in cmd.Items) order.AddLine(i.Sku, i.Qty, i.Price);

        // سياسة: حدّ أقصى للطلب (مثال)
        if (order.Total > 10_000m) throw new InvalidOperationException("limit_exceeded");

        // تحصيل المبلغ عبر عقد خارجي
        var pay = await _payments.ChargeAsync(order.CustomerId, order.Total, ct);
        if (!pay.Success) throw new InvalidOperationException($"payment_failed:{pay.Error}");

        // حفظ (عبر المستودع/الوحدة)
        await _repo.AddAsync(order, ct);
        await _repo.SaveChangesAsync(ct);

        return new PlaceOrderResult(order.Id, order.Total, "confirmed");
    }
}
```

```csharp
// طبقة العرض (Presentation) — Minimal API تستدعي طبقة الأعمال فقط
// Program.cs في مشروع Web (مرجع إلى مشروع Business + مشروع Data لتنفيذ العقود)
app.MapPost("/orders", async (PlaceOrderCommand cmd, OrderService service, CancellationToken ct) =>
{
    try
    {
        var res = await service.PlaceOrderAsync(cmd, ct);
        return Results.Created($"/orders/{res.OrderId}", res);
    }
    catch (ArgumentException ex)        { return Results.ValidationProblem(new() { [""] = new[] { ex.Message } }); }
    catch (InvalidOperationException ex){ return Results.BadRequest(new { error = ex.Message }); }
});
```

> **الفكرة:**  
> - **Business Tier** يملك القواعد ويعتمد على **عقود** (`IOrderRepository`, `IPaymentGateway`).  
> - طبقة البيانات تنفّذ العقود (EF/Dapper/HTTP)، وطبقة العرض تحوّل HTTP↔DTO وتستدعي الخدمة.

---

## خطوات عملية لتصميم **Business Tier** سليم
1. **حدّد حالات الاستخدام** (Use Cases) كخدمات واضحة (`OrderService.PlaceOrder`).  
2. **اكتب القواعد قرب المجال** (داخل الخدمات/الكيانات) لا في Controllers.  
3. افصل الوصول للبيانات في **عقود** (`IRepository/UoW`) ونفّذها في طبقة البيانات.  
4. استخدم **DTOs** للمدخلات/المخرجات؛ لا تسرّب كيانات DB للخارج.  
5. ضع حدود **معاملات** واضحة (Transaction Boundary) بحول كل Use Case.  
6. أضِف **سياسات** (Authorization/Validation) داخل أو بجوار الخدمات.  
7. فعّل **تتبّع/قياس** (Logs/Metrics) على مستوى Use Case مع `Correlation-Id`.

---

## أخطاء شائعة
- منطق أعمال داخل **Controller/ORM** (تسريب الطبقات).  
- استدعاء **SQL/HTTP** مباشرة من طبقة الأعمال (اقتران قوي).  
- **Anemic Domain**: كيانات بلا سلوك وكل شيء في خدمات سميكة غير متماسكة.  
- غياب حدود المعاملة → حالات نصف منجزة عند الفشل.  
- تمرير نماذج DB كـ DTO للواجهة.  
- عدم اختبار Use Cases بمعزل عن DB (استخدم Mocks/Fakes للعقود).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Business Tier** | **تنفيذ قواعد العمل وحالات الاستخدام** | يتعامل عبر **عقود** مع البيانات/التكاملات |
| [Presentation Tier](presentation-tier.md) | تعامل HTTP/UI وتحويل الطلب/الرد | لا يحوي قواعد عمل معقّدة |
| [Data Tier](data-tier.md) | تخزين/استعلام وتنفيذ العقود | لا يقرّر سياسات/قواعد العمل |

---

## ملخص الفكرة  
**Business Tier** هو **قلب النظام المنطقي**: يطبّق القواعد، ينسّق الخطوات، ويتحدث مع الخارج عبر **واجهات**.  
افصله عن العرض والبيانات، واجعل عقوده واضحة واختباراته مستقلّة—تحصل على طبقة **متينة**،  
**قابلة للتغيير**، و**سهلة الصيانة** دون تشابك مع تفاصيل البنية. 
