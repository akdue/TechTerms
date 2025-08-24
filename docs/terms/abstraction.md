# **Abstraction**

## الترجمة الحرفية  
**Abstraction** — **تجريد** (التركيز على *ما* يجب فعله لا *كيف* يُنفَّذ).

## الوصف العربي المختصر  
مبدأ يُعرِّف **عقدًا واضحًا** (واجهات/أصناف مجرّدة) ويُخفي التفاصيل التنفيذية.  
الهدف: **تقليل التعقيد**، **فصل المخاوف**، وتمكين **الاستبدال** بسهولة.

## الشرح المبسّط  
- نعرض **قدرات** (عمليات/خصائص) دون كشف الآلية الداخلية.  
- نستخدم **Interfaces** و/أو **Abstract Classes** لفرض *ما هو مطلوب تنفيذه*.  
- العملاء يتعاملون مع **العقد** فقط؛ التنفيذ يمكن تغييره دون كسرهم.

## تشبيه  
مقبس كهرباء: يَعِد بـ **تيار 220V** (عقد). لا يهمك مسار الأسلاك أو نوع المولّد (التنفيذ).

---

## مثال C# (1) — واجهة كـ **تجريد** عن التخزين

```csharp
// IFileStorage يعرّف "ما ينبغي" دون "كيف"
public interface IFileStorage
{
    Task SaveAsync(string path, byte[] bytes);
    Task<byte[]> ReadAsync(string path);
}

// تنفيذ داخل الذاكرة (للاختبار)
public class MemoryStorage : IFileStorage
{
    private readonly Dictionary<string, byte[]> _db = new();
    public Task SaveAsync(string path, byte[] bytes) { _db[path] = bytes; return Task.CompletedTask; }
    public Task<byte[]> ReadAsync(string path) => Task.FromResult(_db[path]);
}

// تنفيذ على القرص (مبسّط)
public class DiskStorage : IFileStorage
{
    public Task SaveAsync(string path, byte[] bytes)
    {
        Directory.CreateDirectory(Path.GetDirectoryName(path)!);
        return File.WriteAllBytesAsync(path, bytes);
    }
    public Task<byte[]> ReadAsync(string path) => File.ReadAllBytesAsync(path);
}

// خدمة أعمال تعتمد على "التجريد" فقط
public class AvatarService
{
    private readonly IFileStorage _storage;
    public AvatarService(IFileStorage storage) => _storage = storage;

    public async Task<string> UploadAsync(string userId, byte[] png)
    {
        var path = $"avatars/{userId}.png";
        await _storage.SaveAsync(path, png);
        return path;
    }
}
```

```csharp
// Program.cs  (.NET 8/9)
// تبديل التنفيذ دون لمس AvatarService
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IFileStorage, MemoryStorage>();   // dev
// builder.Services.AddSingleton<IFileStorage, DiskStorage>(); // prod
builder.Services.AddSingleton<AvatarService>();
```

**الفكرة:** `AvatarService` لا يعرف *كيف* يتم التخزين؛ يعرف فقط **العقد** `IFileStorage`.

---

## مثال C# (2) — صنف **مجرّد** + أسلوب القالب (Template Method)

```csharp
using System.Text;

// يحدّد "قالب" التصدير ويترك التفاصيل للمشتقات
public abstract class ReportExporter
{
    public void Export(IEnumerable<string> rows, Stream destination)
    {
        if (rows is null) throw new ArgumentNullException(nameof(rows));
        var bytes = Encode(rows);        // ← يجب على المشتق تنفيذها
        Write(destination, bytes);       // ← سلوك افتراضي قابل للتجاوز
    }

    protected abstract byte[] Encode(IEnumerable<string> rows);
    protected virtual void Write(Stream dest, byte[] bytes) => dest.Write(bytes, 0, bytes.Length);
}

public class CsvExporter : ReportExporter
{
    protected override byte[] Encode(IEnumerable<string> rows)
        => Encoding.UTF8.GetBytes(string.Join('\n', rows.Select(r => $"\"{r.Replace("\"","\"\"")}\"")));
}

public class JsonExporter : ReportExporter
{
    protected override byte[] Encode(IEnumerable<string> rows)
        => Encoding.UTF8.GetBytes(System.Text.Json.JsonSerializer.Serialize(rows));
}
```

**الفكرة:** العميل يستدعي `Export(...)` دون معرفة إن كان التصدير **CSV** أو **JSON**.

---

## خطوات عملية لتطبيق التجريد
1. **ابدأ بالعقد**: ما العمليات/الخصائص التي يحتاجها العميل؟ سمِّها بوضوح.  
2. **افصل التنفيذ** خلف **Interface/Abstract Class**.  
3. **مرّر** التنفيذ عبر **DI** لتمكين التبديل والاختبار.  
4. **قلّل سطح العقد** (Minimal API) وامنع تسرّب التفاصيل.  
5. وثّق **الضمانات** (Exceptions، حدود، حالات حواف) ضمن العقد لا التنفيذ.

---

## أخطاء شائعة
- عقد **منتفخ** يحوي تفاصيل تنفيذية (يكسر التجريد).  
- استخدام التجريد لـ “مجرّد” تغليف بلا حاجة (تعقيد زائد).  
- خلط التجريد مع **التغليف**: الأول يحدد *ما*، الثاني يحمي *كيف/البيانات*.  
- ربط العميل بنوع ملموس بدل العقد → فقدان فائدة التجريد.  
- ترك العقود بلا **اختبارات عقد** (Contract Tests) فتنحرف المشتقات.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Abstraction** | **تعريف ما يجب فعله دون كيف** | عبر **Interfaces/Abstract Classes** |
| [Encapsulation](encapsulation.md) | إخفاء الحالة والتفاصيل | يحمي الثوابت خلف واجهة |
| Interface | عقد سلوك دون حالة | مثالي للتبديل والاختبار |
| Abstract Class | عقد + سلوك/حالة مشتركة | يسمح بقالب افتراضي (Template Method) |
| [Polymorphism](polymorphism.md) | تنفيذ مختلف لنفس العقد | يتحقق عبر الواجهات/الوراثة |

---

## ملخص الفكرة  
**التجريد** يبسّط الأنظمة: تحدّد **العقد** أولًا، ثم تخفي **التنفيذ** خلفها.  
اعتمد **Interfaces/Abstract** + **DI**، واجعل العقد صغيرة وواضحة—تحصل على كود **مرن**، **قابل للاستبدال**، وأسهل فهمًا وصيانة. 
