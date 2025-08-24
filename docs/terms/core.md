# **Core**

## الترجمة الحرفية  
**Core** — **النواة/القلب (الطبقة الأساسية)**.

## الوصف العربي المختصر  
أكثر جزء **ثابت** في النظام: يحوي **نموذج المجال** (Entities/Value Objects)،  
و**العقود** (Interfaces/Ports) التي تحتاجها بقية الطبقات.  
لا يعتمد على أطر أو تفاصيل بنية (EF/HTTP) — **اتجاه التبعيات نحوه فقط**.

## الشرح المبسّط  
- ماذا يوجد في **Core**؟
  - **Domain Model**: كيانات، كائنات قيمة، خدمات مجال، أحداث مجال.  
  - **Contracts**: واجهات للمخازن/الوقت/الرسائل… يطبّقها الخارج.  
  - (اختياري) **Use Cases** الخفيفة إن اتّبعت أسلوب “Application داخل الـCore”.
- ماذا **لا** يوجد؟  
  - لا EF/SQL، لا ASP.NET، لا IO مباشر.  
- لماذا؟  
  - لتبقى قواعد العمل قابلة للاختبار والتطوير دون كسر بقية النظام.

## تشبيه  
قلب المحرّك: حيث تُولَد القوة (قواعد العمل).  
الكوابل والأنابيب (البنية/الأطر) تتبدّل، لكن القلب يبقى **مستقلاً وثابتًا**.

---

## مثال C# — مشروع **Core** فقط (مجال + عقود بلا تبعيات خارجية)

```csharp
// Shop.Core (مكتبة صنفية) — بدون مراجع لـ EF/ASP.NET
namespace Shop.Core.Domain;

// ===== Value Object =====
public readonly record struct Money(decimal Amount)
{
    public static Money Zero => new(0);
    public static Money operator +(Money a, Money b) => new(a.Amount + b.Amount);
    public static Money operator *(Money a, int m)   => new(a.Amount * m);
    public override string ToString() => Amount.ToString("0.00");
}

// ===== Entity =====
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();
    public int Id { get; }
    public string CustomerId { get; }
    public IReadOnlyList<OrderLine> Lines => _lines;

    public Order(int id, string customerId)
    {
        if (id <= 0) throw new ArgumentOutOfRangeException(nameof(id));
        if (string.IsNullOrWhiteSpace(customerId)) throw new ArgumentException("customer_required");
        Id = id; CustomerId = customerId;
    }

    public void AddLine(string sku, int qty, Money price)
    {
        if (string.IsNullOrWhiteSpace(sku)) throw new ArgumentException(nameof(sku));
        if (qty <= 0 || price.Amount <= 0) throw new ArgumentOutOfRangeException();
        _lines.Add(new OrderLine(sku, qty, price));
    }

    public Money Subtotal() => _lines.Aggregate(Money.Zero, (t, l) => t + (l.Price * l.Qty));
}

public sealed record OrderLine(string Sku, int Qty, Money Price);

// ===== Domain Service =====
public interface IPricingPolicy
{
    Money DiscountFor(Order order);
}

public sealed class SimplePricingPolicy : IPricingPolicy
{
    public Money DiscountFor(Order order)
    {
        var subtotal = order.Subtotal();
        return subtotal.Amount >= 500 ? new Money(subtotal.Amount * 0.05m) : Money.Zero;
    }
}

// ===== Ports/Contracts (تُنفّذ في Infrastructure) =====
public interface IOrderRepository
{
    Task<Order?> GetAsync(int id, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    Task SaveChangesAsync(CancellationToken ct = default);
}

public interface IClock { DateTime UtcNow { get; } }

// ===== Use Case (اختياري داخل الـCore) =====
public sealed class PlaceOrder
{
    private readonly IOrderRepository _repo;
    private readonly IPricingPolicy _pricing;
    private readonly IClock _clock;
    public PlaceOrder(IOrderRepository repo, IPricingPolicy pricing, IClock clock)
        { _repo = repo; _pricing = pricing; _clock = clock; }

    public async Task<Result> HandleAsync(Command cmd, CancellationToken ct = default)
    {
        var order = new Order(cmd.OrderId, cmd.CustomerId);
        foreach (var it in cmd.Items) order.AddLine(it.Sku, it.Qty, new Money(it.Price));

        var subtotal = order.Subtotal();
        var discount = _pricing.DiscountFor(order);
        await _repo.AddAsync(order, ct);
        await _repo.SaveChangesAsync(ct);

        return new Result(order.Id, subtotal.Amount, discount.Amount, _clock.UtcNow);
    }

    public sealed record Command(int OrderId, string CustomerId, List<Item> Items);
    public sealed record Item(string Sku, int Qty, decimal Price);
    public sealed record Result(int OrderId, decimal Subtotal, decimal Discount, DateTime PlacedAtUtc);
}
```

> لاحظ: لا يوجد EF/HTTP/ملفات. الـ **Infrastructure** لاحقًا يطبّق `IOrderRepository`،  
> و**Presentation** يستدعي `PlaceOrder` عبر DI — **اتجاه التبعيات إلى الـCore**.

---

## خطوات عملية لتصميم **Core** قوي
1. **عرّف نموذج المجال** (Entities/Value Objects) بأ invariant واضحة.  
2. **اكتب العقود** كواجهات (Ports) لما تحتاجه من الخارج: تخزين/وقت/رسائل.  
3. **امنع التبعيات** على أي إطار — اجعل Core خفيفًا وقابلاً للاختبار.  
4. **اختبارات وحدة** مكثّفة لـ Core (من دون DB/شبكة).  
5. **لا تسرب** مفاهيم البنية (DbContext/HttpRequest) إلى Core.  
6. إن احتجت **Use Cases**، اجعلها رفيعة وتنسّق فقط (Validation/Policies).

---

## أخطاء شائعة
- إدخال EF/DbContext داخل الكيانات أو الخدمات.  
- جعل الـCore مجرّد **DTOات** بلا سلوك (Anemic Domain).  
- ربط الـCore بوقت النظام مباشرة (استخدم `IClock`).  
- خلط سياسات عرض/HTTP في الـCore (Status Codes، ModelState…).  
- عقود غامضة أو عامة جدًا (`IRepository<T>` بلا حدود مجال).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Core (Domain/Core Layer)** | **قواعد العمل + عقود** | بلا تبعيات أطر؛ اتجاه التبعيات نحوه |
| Application Layer | تنسيق Use Cases/Transactions | قد يكون جزءًا من Core أو خارجَه حسب الأسلوب |
| Infrastructure | تنفيذ العقود (DB/IO/HTTP) | يعتمد على Core ولا العكس |
| Presentation | نقل/HTTP/UI | يستدعي Use Cases؛ بلا قواعد عمل |
| **.NET/ASP.NET Core** | **إطار عمل** | مختلف عن مفهوم “Core” كنواة تصميم |

---

## ملخص الفكرة  
**Core** = **القلب** الذي يحتوي قواعد عملك وعقودك الثابتة.  
أبعد عنه الأطر والتفاصيل، واجعل كل التبعيات **تتجه إليه**—  
تحصل على نظام **متين**، **قابل للاختبار**، ويتحوّر بسرعة دون كسر البقية. 
