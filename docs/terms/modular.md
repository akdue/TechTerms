# **Modular**

## الترجمة الحرفية  
**Modular / Modularity** — **معياري / قابل للتقسيم إلى وحدات مستقلّة**.

## الوصف العربي المختصر  
تصميم يجزّئ النظام إلى **وحدات (Modules)** صغيرة **عالية التماسك** و**ضعيفة الترابط**،  
لكل وحدة **واجهة واضحة** ومسؤولية محدّدة يمكن تطويرها/اختبارها/نشرها بمعزل.

## الشرح المبسّط  
- كل **Module** يملك: *نطاقًا*، *واجهات*، *بياناته/خداماته*، و**نقاط دخول**.  
- التواصل عبر **عقود** (Interfaces, DTOs, Events)، لا عبر اختراق الطبقات.  
- يسهل **التبديل** وإعادة الاستخدام والاختبار، ويقلّل أثر التغييرات (Blast Radius).  
- ينطبق على: **مشروع واحد معياري (Modular Monolith)** أو **Microservices**.

## تشبيه  
مدينة مقسّمة إلى **أحياء**: لكل حي خدماته (مدرسة/مركز صحي) وطريق رئيسي يربطه بباقي الأحياء.  
لا تبني أنبوبًا عشوائيًا بين البيوت—بل تستخدم **شبكات وواجهات** محدّدة.

---

## مثال C# — “Monolith معياري”: كل Module يعرّف خدماته ونقاطه

> الفكرة: **واجهة `IModule`** يطبّقها كل Module.  
> `Program.cs` يجمع الوحدات ويُشغّلها دون معرفة تفاصيلها.

```csharp
// Contracts: العقد الذي يلتزم به أي Module
public interface IModule
{
    void RegisterServices(IServiceCollection services);          // تسجيل DI
    void MapEndpoints(IEndpointRouteBuilder endpoints);          // تعريف المسارات
}
```

```csharp
// Orders Module (مسؤول عن الطلبات فقط)
public interface IOrderRepo { IEnumerable<Order> All(); Order Add(OrderCreate dto); }
public record Order(int Id, string Item, decimal Amount);
public record OrderCreate(string Item, decimal Amount);

internal class InMemoryOrderRepo : IOrderRepo
{
    private readonly List<Order> _db = new() { new(1,"Keyboard",120) };
    private int _next = 2;
    public IEnumerable<Order> All() => _db;
    public Order Add(OrderCreate dto){ var o=new Order(_next++,dto.Item,dto.Amount); _db.Add(o); return o; }
}

public sealed class OrdersModule : IModule
{
    public void RegisterServices(IServiceCollection s)
        => s.AddSingleton<IOrderRepo, InMemoryOrderRepo>();

    public void MapEndpoints(IEndpointRouteBuilder e)
    {
        var grp = e.MapGroup("/orders").WithTags("Orders");
        grp.MapGet("/", (IOrderRepo repo) => Results.Ok(repo.All()));
        grp.MapPost("/", (OrderCreate dto, IOrderRepo repo) =>
        {
            if (string.IsNullOrWhiteSpace(dto.Item) || dto.Amount <= 0)
                return Results.ValidationProblem(new(){{"Item/Amount",["قيمة غير صالحة"]}});
            var created = repo.Add(dto);
            return Results.Created($"/orders/{created.Id}", created);
        });
    }
}
```

```csharp
// Payments Module (منفصل بعقد خاص به)
public interface IPaymentGateway { bool Charge(decimal amount); }
internal class FakeGateway : IPaymentGateway { public bool Charge(decimal a)=> a>0; }

public sealed class PaymentsModule : IModule
{
    public void RegisterServices(IServiceCollection s)
        => s.AddSingleton<IPaymentGateway, FakeGateway>();

    public void MapEndpoints(IEndpointRouteBuilder e)
    {
        var grp = e.MapGroup("/payments").WithTags("Payments");
        grp.MapPost("/charge", (decimal amount, IPaymentGateway gw)
            => gw.Charge(amount) ? Results.Ok(new { ok=true }) : Results.BadRequest(new { ok=false }));
    }
}
```

```csharp
// Program.cs  (.NET 8/9)
// dotnet new web -n ModularDemo && cd ModularDemo && ضَع الملفات ثم dotnet run
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(o => o.SwaggerDoc("v1", new OpenApiInfo{Title="Modular API",Version="v1"}));

// التركيب (Composition Root): ضف Modules دون معرفة داخلها
var modules = new IModule[] { new OrdersModule(), new PaymentsModule() };
foreach (var m in modules) m.RegisterServices(builder.Services);

var app = builder.Build();
if (app.Environment.IsDevelopment()) { app.UseSwagger(); app.UseSwaggerUI(); }
app.UseHttpsRedirection();

foreach (var m in modules) m.MapEndpoints(app);

app.MapGet("/", ()=> new { service="modular-demo", modules = modules.Select(x=>x.GetType().Name) });
app.Run();
```

**الإخراج المتوقع (مختصر):**  
- `GET /orders` ، `POST /orders`، `POST /payments/charge`.  
- كل وحدة تسجّل خدماتها ومساراتها **بدون تسرّب تفاصيل** للآخرين.

---

## خطوات عملية لبناء نظام معياري
1. **ارسم الحدود**: ماذا يملك كل Module؟ ما واجهته؟ ما الذي **ليس** من اختصاصه؟  
2. **عالي التماسك / ضعيف الترابط**: اجعل مسؤوليات واضحة، وامنع الاستدعاءات الجانبية العشوائية.  
3. **عقود ثابتة**: Interfaces/DTOs/Events. عدّلها عبر **نسخ إصدار** (v1→v2).  
4. **طبّق DI**: كل Module يحقن ما يحتاجه فقط.  
5. **إخفاء التفاصيل**: اجعل الأنواع الداخلية `internal`، وصدّر القليل فقط.  
6. **اختبار مستقل**: اختبارات وحدة/تكامل لكل Module + عقود بينية (Contract Tests).  
7. **مراقبة**: قياسات/سجلات **لكل Module** لسهولة المتابعة والضبط.

---

## أخطاء شائعة
- **دوائر اعتماد** بين الوحدات (A يعتمد على B وB على A).  
- مشاركة **نموذج بيانات داخلي** بين الوحدات بدل DTOs.  
- “God Module” يجمع كل شيء.  
- تجاوز الحدود: وحدة تقرأ جدول وحدة أخرى مباشرة.  
- عدم توثيق العقود/الإصدارات → كسر متبادل عند التغيير.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Modular** | **تقسيم النظام إلى وحدات مستقلّة بعقود واضحة** | **High Cohesion / Low Coupling**، اختبار وتبديل أسهل |
| [Monolith](monolith.md) | تطبيق واحد متماسك | يمكن أن يكون **معياريًا** داخليًا أو متشابكًا |
| [Modular Monolith](modular-monolith.md) | حزمة واحدة بحدود داخلية صلبة | أداء أبسط من الميكروسيرفس؛ حدود عبر العقود |
| [Microservices](microservices.md) | خدمات مستقلة تُنشر منفصلة | حرية تقنية/نشر؛ تعقيد شبكة/مراقبة أعلى |
| [Plug-in Architecture](plugin.md) | توسيع بملحقات قابلة للتحميل | حدود/نقاط تمديد واضحة (Extensibility) |

---

## ملخص الفكرة  
**التصميم المعياري** يجعل نظامك **أسهل تغييرًا واختبارًا** عبر حدود وعقود واضحة.  
افصل المسؤوليات، استخدم **DI** و**DTOs/Events**، وأخفِ التفاصيل—تحصل على كود **قابل للصيانة** والنمو بأقل مفاجآت. 
