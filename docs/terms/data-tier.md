# **Data Tier**

## الترجمة الحرفية  
**Data Tier** — **طبقة البيانات/التخزين**.

## الوصف العربي المختصر  
الطبقة المسؤولة عن **تخزين واسترجاع** البيانات (DB/Cache/Files/Queues) عبر **عقود** واضحة.  
تُنفّذ الوصول للبيانات (ORM/SQL) وتضمن **المعاملات**، **الفهارس**، و**القيود**—من دون قواعد عمل.

## الشرح المبسّط  
- تتعامل مع **مصادر بيانات**: قواعد SQL/NoSQL، ملفات، Redis، رسائل.  
- توفّر **مستودعات/وحدات عمل** (Repositories/UoW) تُخفي تفاصيل التخزين.  
- تضمن **نزاهة** البيانات (قيود وفهارس) و**اتساقًا** أثناء الفشل (Transactions).  
- تُحقن في طبقة الأعمال كـ **Interfaces** تُنفَّذ هنا.

## تشبيه  
“المخزن” في الشركة: فيه رفوف منظّمة (جداول وفهارس)، وسياسات إدخال/إخراج (قيود/معاملات).  
الموظفون في المكاتب (الأعمال) لا يتعاملون مع الرفوف مباشرة؛ يطلبون عبر بوابة منضبطة.

---

## مثال C# — تنفيذ **Repository** بـ **EF Core** + تكوين DB + معاملة

> يعتمد عقد `IOrderRepository` من طبقة الأعمال، ويُخفي EF/SQL خلفه.

```csharp
// Infrastructure.Data — .NET 8/9
// dotnet add package Microsoft.EntityFrameworkCore
// dotnet add package Microsoft.EntityFrameworkCore.SqlServer

using Microsoft.EntityFrameworkCore;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

// ====== (1) عقود الأعمال (مرجع من مشروع Business) ======
public interface IOrderRepository
{
    Task<Order?> GetAsync(int id, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    Task SaveChangesAsync(CancellationToken ct = default);
}

// كيانات مجال مبسّطة (تعريفها في Business عادةً)
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
        if (qty <= 0 || price <= 0) throw new ArgumentOutOfRangeException();
        _lines.Add(new OrderLine(sku, qty, price));
    }
}
public record OrderLine(string Sku, int Qty, decimal Price);

// ====== (2) نماذج EF (جداول) ======
[Table("Orders")]
public class OrderRow
{
    [Key] public int Id { get; set; }
    [Required, MaxLength(64)] public string CustomerId { get; set; } = "";
    public DateTime CreatedUtc { get; set; } = DateTime.UtcNow;
    public List<OrderLineRow> Lines { get; set; } = new();
    [Timestamp] public byte[]? RowVersion { get; set; } // تفادي تعارضات تحديث
}

[Table("OrderLines")]
public class OrderLineRow
{
    [Key] public int Id { get; set; }
    [ForeignKey(nameof(Order))] public int OrderId { get; set; }
    [Required, MaxLength(32)] public string Sku { get; set; } = "";
    [Range(1, int.MaxValue)] public int Qty { get; set; }
    [Column(TypeName = "decimal(18,2)")] public decimal Price { get; set; }
    public OrderRow? Order { get; set; }
}

// ====== (3) DbContext + خرائط ======
public class AppDb : DbContext
{
    public AppDb(DbContextOptions<AppDb> options) : base(options) { }
    public DbSet<OrderRow> Orders => Set<OrderRow>();
    public DbSet<OrderLineRow> OrderLines => Set<OrderLineRow>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<OrderRow>()
         .HasMany(o => o.Lines)
         .WithOne(l => l.Order!)
         .HasForeignKey(l => l.OrderId)
         .OnDelete(DeleteBehavior.Cascade);

        b.Entity<OrderLineRow>()
         .HasIndex(l => new { l.OrderId, l.Sku }); // فهرس شائع للبحث

        // قيود تحقق على مستوى DB
        b.Entity<OrderLineRow>()
         .ToTable(t => t.HasCheckConstraint("CK_OrderLine_Qty", "[Qty] > 0"));
    }
}

// ====== (4) التحويل بين المجال وEF ======
static class Map
{
    public static OrderRow ToRow(this Order o) => new OrderRow
    {
        Id = o.Id,
        CustomerId = o.CustomerId,
        Lines = o.Lines.Select(l => new OrderLineRow { Sku = l.Sku, Qty = l.Qty, Price = l.Price }).ToList()
    };

    public static Order ToDomain(this OrderRow r)
    {
        var o = new Order(r.Id, r.CustomerId);
        foreach (var l in r.Lines) o.AddLine(l.Sku, l.Qty, l.Price);
        return o;
    }
}

// ====== (5) Repository بتنفيذ EF Core + معاملة ======
public class EfOrderRepository : IOrderRepository
{
    private readonly AppDb _db;
    public EfOrderRepository(AppDb db) => _db = db;

    public async Task<Order?> GetAsync(int id, CancellationToken ct = default)
    {
        var row = await _db.Orders
            .Include(o => o.Lines)
            .AsNoTracking()
            .FirstOrDefaultAsync(o => o.Id == id, ct);
        return row?.ToDomain();
    }

    public async Task AddAsync(Order order, CancellationToken ct = default)
        => await _db.Orders.AddAsync(order.ToRow(), ct);

    public async Task SaveChangesAsync(CancellationToken ct = default)
        => await _db.SaveChangesAsync(ct);

    // مثال معاملة يدوية لعدة عمليات
    public async Task<bool> SaveManyAtomicAsync(IEnumerable<Order> orders, CancellationToken ct = default)
    {
        using var tx = await _db.Database.BeginTransactionAsync(ct);
        try
        {
            foreach (var o in orders) await _db.Orders.AddAsync(o.ToRow(), ct);
            await _db.SaveChangesAsync(ct);
            await tx.CommitAsync(ct);
            return true;
        }
        catch
        {
            await tx.RollbackAsync(ct);
            return false;
        }
    }
}
```

```csharp
// Program.cs (تكوين طبقة البيانات في Web/Composition Root)
// dotnet add package Microsoft.EntityFrameworkCore.SqlServer
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<AppDb>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Sql"))); // أو من Secrets/Env
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
```

**نموذج SQL (توليد مبدئي إن لم تستخدم Migrations):**
```sql
CREATE TABLE Orders (
  Id           INT        NOT NULL PRIMARY KEY,
  CustomerId   NVARCHAR(64) NOT NULL,
  CreatedUtc   DATETIME2  NOT NULL DEFAULT SYSUTCDATETIME(),
  RowVersion   ROWVERSION
);
CREATE TABLE OrderLines (
  Id       INT IDENTITY PRIMARY KEY,
  OrderId  INT NOT NULL REFERENCES Orders(Id) ON DELETE CASCADE,
  Sku      NVARCHAR(32) NOT NULL,
  Qty      INT NOT NULL CHECK (Qty > 0),
  Price    DECIMAL(18,2) NOT NULL
);
CREATE INDEX IX_OrderLines_Order_Sku ON OrderLines(OrderId, Sku);
```

> **الفكرة:**  
> - تَعرِض طبقة البيانات **تنفيذًا** للعقد فقط (`IOrderRepository`).  
> - تُدير **المعاملات**، **القيود**، و**الفهارس**؛ وتُخفي ORM/SQL عن الطبقات العليا.

---

## خطوات عملية لبناء **Data Tier** قوي
1. **العقود أولًا**: عرّف `IRepository/UoW` قرب المجال؛ نفّذها هنا.  
2. **المخطط**: صمّم جداول/مفاتيح/فهارس وقيود (NOT NULL/UNIQUE/CHECK/FK).  
3. **المعاملات**: حدود واضحة لكل Use Case؛ افصل القراءة/الكتابة (CQRS) إن لزم.  
4. **الأداء**: فهارس محسوبة، تتبّع خطط التنفيذ، Paginate، وتجنّب N+1 (`Include`/`SELECT … WHERE`).  
5. **التزامن**: `rowversion`/ETag، و**تفادي الكتابة المفقودة** (If-Match/ConcurrencyToken).  
6. **الأسرار والاتصالات**: Connection Pool، Timeouts، Retry مع Backoff، وأسرار في Vault.  
7. **الهجرة/الإصدار**: EF Migrations/DbUp مع خطط رجوع (Rollback).  
8. **المراقبة**: قياسات I/O، Latency، Deadlocks، وErrors؛ سجّل استعلامات بطيئة.

---

## أخطاء شائعة
- وضع **قواعد العمل** داخل طبقة البيانات (Triggers/Procedures ثقيلة) بدل الأعمال.  
- تسريب ORM/DbContext إلى الطبقات العليا (كسر الحدود).  
- نسيان **الفهارس** أو كتابة استعلامات بلا حدود/ترقيم.  
- تجاهل **Timeout/Retry/Transactions** → تجمّد/عطب عند الضغط.  
- الاعتماد على **Auto-Migrations** في الإنتاج دون سيطرة.  
- عدم التعامل مع **التزامن** (RowVersion/ETag) → تعارضات صامتة.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Data Tier** | **تخزين/استرجاع مع معاملات وقيود** | يُخفي ORM/SQL خلف عقود |
| [Business Tier](business-tier.md) | منطق المجال وحالات الاستخدام | يستدعي المستودعات/العقود |
| [Presentation Tier](presentation-tier.md) | HTTP/UI وتحويل الطلب/الرد | بلا SQL أو منطق مجال |
| ORM (EF Core) | إنتاجية عالية وخريطة كائن-علاقة | راقب الأداء وتفادى N+1 |
| Dapper/SQL خام | تحكم/أداء أعلى | مسؤولية يدويّة للفحوص/الخرائط |

---

## ملخص الفكرة  
**Data Tier** = حيث تُحفظ الحقيقة الدائمة.  
أخفِ تفاصيل التخزين خلف **عقود**، اضبط **المعاملات/القيود/الفهارس**، وتعامل مع **التزامن**—  
تحصل على قاعدة متينة تدعم طبقاتك العليا بأمان وأداء واستدامة. 
