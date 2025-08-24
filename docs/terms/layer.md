# **Layer**

## الترجمة الحرفية  
**Layer** — **طبقة** (وحدة منطقية على مستوى تجريد محدّد داخل بنية النظام).

## الوصف العربي المختصر  
تقسيم للكود إلى **مستويات أفقية** (Presentation / Application / Domain / Infrastructure).  
الهدف: **فصل المسؤوليات**، **تقليل الترابط**، وتحديد **اتجاه التبعيات** بوضوح.

## الشرح المبسّط  
- كل طبقة تركّز على **نوع واحد** من العمل:  
  - **Presentation**: HTTP/UI ونماذج الطلب/الرد.  
  - **Application**: حالات استعمال (Use Cases) وتنسيق العمليات.  
  - **Domain**: قواعد العمل الخالصة (كيانات/قيم/خدمات مجال).  
  - **Infrastructure**: التفاصيل التقنية (DB/Files/Email/HTTP).  
- **اتجاه التبعيات**: إلى **الأسفل** أو **إلى الداخل** نحو المجال فقط. لا عكس.  
- **عقود** بين الطبقات (Interfaces/DTOs) تمنع التسريب.

## تشبيه  
مبنى من أربع طوابق: استقبال (عرض)، إدارة العمليات (تطبيق)، القسم القانوني (المجال)، الخدمات اللوجستية (البنية).  
الزائر لا يذهب للمخزن مباشرة. يمر عبر استقبال→إدارة→قانوني عند الحاجة.

---

## مثال C# — تطبيق طبقي بسيط (API ↔ Application ↔ Domain ↔ Infrastructure)

```csharp
// .NET 8/9 — مشروع واحد للتوضيح، لكن الفكرة عادةً 4 مشاريع (طبقة لكل مشروع)
// API: Presentation   | Application: Use Cases   | Domain: Business   | Infrastructure: DB/IO

// ========== Domain ==========
namespace Shop.Domain;

// كيان مجال + قواعد بسيطة
public record OrderId(int Value);
public record Money(decimal Amount)
{
    public static Money operator +(Money a, Money b) => new(a.Amount + b.Amount);
    public static Money Zero => new(0);
}
public class Order
{
    public OrderId Id { get; }
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;

    public Order(OrderId id) => Id = id;

    public void AddLine(string sku, int qty, Money price)
    {
        if (qty <= 0) throw new ArgumentOutOfRangeException(nameof(qty));
        if (price.Amount <= 0) throw new ArgumentOutOfRangeException(nameof(price));
        _lines.Add(new OrderLine(sku, qty, price));
    }

    public Money Total() => _lines.Aggregate(Money.Zero, (t, l) => t + new Money(l.Price.Amount * l.Qty));
}
public record OrderLine(string Sku, int Qty, Money Price);

// عقد تخزين يُعرَّف قرب المجال
public interface IOrderRepository
{
    Task<Order?> GetAsync(OrderId id, CancellationToken ct = default);
    Task SaveAsync(Order order, CancellationToken ct = default);
}

// ========== Application ==========
namespace Shop.Application;
using Shop.Domain;

// حالة استعمال (Use Case) — تنسيق خطوات بدون تفاصيل تخزين
public class CreateOrderHandler
{
    private readonly IOrderRepository _repo;
    public CreateOrderHandler(IOrderRepository repo) => _repo = repo;

    public async Task<Order> HandleAsync(CreateOrder cmd, CancellationToken ct = default)
    {
        var order = new Order(new OrderId(cmd.OrderId));
        foreach (var l in cmd.Lines)
            order.AddLine(l.Sku, l.Qty, new Money(l.Price));

        await _repo.SaveAsync(order, ct);
        return order;
    }
}

// DTO خاص بالطبقة التطبيقية
public record CreateOrder(int OrderId, List<CreateOrderLine> Lines);
public record CreateOrderLine(string Sku, int Qty, decimal Price);

// ========== Infrastructure ==========
namespace Shop.Infrastructure;
using Shop.Domain;

// مستودع في الذاكرة (بديل DB) — تفاصيل تقنية معزولة هنا
public class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<int, Order> _db = new();
    public Task<Order?> GetAsync(OrderId id, CancellationToken ct = default)
        => Task.FromResult(_db.TryGetValue(id.Value, out var o) ? o : null);

    public Task SaveAsync(Order order, CancellationToken ct = default)
    {
        _db[order.Id.Value] = order;
        return Task.CompletedTask;
    }
}

// ========== Presentation (Minimal API) ==========
namespace Shop.Api;
using Microsoft.AspNetCore.Mvc;
using Shop.Application;
using Shop.Domain;
using Shop.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

// تكوين DI وفق الطبقات (Presentation يعرف Application/Domain فقط عبر العقود)
builder.Services.AddSingleton<IOrderRepository, InMemoryOrderRepository>(); // Infrastructure
builder.Services.AddScoped<CreateOrderHandler>();                             // Application

var app = builder.Build();
app.UseHttpsRedirection();

// Endpoint طبقة العرض — لا يعرف تفاصيل التخزين
app.MapPost("/orders", async ([FromBody] CreateOrder dto, CreateOrderHandler handler, CancellationToken ct) =>
{
    var order = await handler.HandleAsync(dto, ct);
    return Results.Created($"/orders/{order.Id.Value}", new {
        id = order.Id.Value,
        total = order.Total().Amount,
        lines = order.Lines.Select(l => new { l.Sku, l.Qty, price = l.Price.Amount })
    });
});

app.Run();
```

**الفكرة:**  
- **Domain** يحوي القواعد فقط.  
- **Application** ينسّق ويعتمد على **عقد** `IOrderRepository`.  
- **Infrastructure** يقدّم التنفيذ الحقيقي للعقد.  
- **Presentation** يستدعي **Use Case** ويحوّل بين DTO وHTTP.

---

## خطوات عملية لتصميم طبقي ناجح
1. **ارسم الحدود**: ما مسؤولية كل طبقة؟ ما الذي **لا** يخصها؟  
2. **عرّف العقود** في الداخل (Domain/Application) و**نفّذها** في الخارج (Infrastructure).  
3. **احفظ اتجاه التبعيات**: Presentation/Application → Domain، وInfrastructure يعتمد على العقود فقط.  
4. **استعمل DI** لتمرير التنفيذ (عكس التبعيات).  
5. **افصل النماذج**: DTO للعرض، Models/Entities للمجال. لا تسرّب `DbContext` للأعلى.  
6. **اختبر الطبقات** بمعزل:  
   - Domain باختبارات وحدة.  
   - Application باختبارات Use Cases مع Repos وهمية.  
   - Infrastructure باختبارات تكامل مع DB حقيقية.

---

## أخطاء شائعة
- استدعاء **ORM/DbContext** من **Controller** مباشرة (تسريب طبقة البنية).  
- **دوران تبعيات** (Presentation يعتمد على Infrastructure والعكس).  
- طبقات **فارغة** بلا قيمة (Layering لأجل التقسيم فقط).  
- وضع قواعد العمل في **Infrastructure** (كسر حدود المجال).  
- **Anemic Domain**: كيانات بلا سلوك، كل المنطق في الخدمات.  
- مشاركة **نموذج قاعدة البيانات** كـ DTO خارجي.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Layer** | **فصل منطقي حسب التجريد** | تبعيات **للأسفل/للداخل** فقط |
| [Tier](tier.md) | فصل **فيزيائي** (Web/App/DB) | قد يطابق طبقات أو يختلف |
| [Module](modular.md) | تجميع مسؤولية بحدود واضحة | قد يحتوي عدّة طبقات داخله |
| Clean Architecture| طبقات متحدة المركز تقودها المجال | تبعيات نحو **Domain** فقط |
| Hexagonal/Onion | منافذ/محولات تعزل البنية | **Ports/Adapters**، يعكس التبعيات |
| [Interface](interface.md) | عقود بين الطبقات | تمنع التسريب وتسهّل الاستبدال |

---

## ملخص الفكرة  
**Layer** تنظّم نظامك إلى مستويات واضحة، وتفرض **حدودًا** واتجاه تبعيات **منضبط**.  
ضع **العقد** في الداخل، و**التنفيذ** في الخارج، ومرّر كل شيء عبر **DI**—  
تحصل على تصميم **قابل للصيانة**، **قابل للاختبار**، ويسهّل النمو دون تشابك.
