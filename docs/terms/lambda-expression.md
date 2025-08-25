# **Lambda Expression**

## الترجمة الحرفية 
**Lambda Expression** — **تعبير لامبدا** (دالة قصيرة بلا اسم تُمرَّر كقيمة).

## الوصف العربي المختصر  
صيغة مختصرة لتعريف دوال **مجهولة** تُستخدم مع **LINQ**، **الأحداث**، و**EF Core**.  
تُمثَّل بأنواع **Func\<…\> / Action\<…\>** أو كشجرة تعبير **Expression\<Func\<…\>\>**.

## الشرح المبسّط  
- الشكل: `(المعاملات) => جسم`. مثال: `x => x * 2`.  
- إن كان **جسمًا واحدًا** فبدون `{}` و`return`.  
- مع المجموعات في الذاكرة: تُنفَّذ مباشرة (Delegates).  
- مع مزوّد استعلام (EF/Core): تُحوَّل إلى **شجرة تعبير** تُترجَم إلى SQL.

## تشبيه  
**ملاحظة لشرط** تسلّمها لدالة أخرى: “أرجع العناصر التي ينطبق عليها هذا الشرط”.

---

## مثال C# — استعلام EF Core بلامبدا + إسقاط إلى DTO

```csharp
// البحث عن موظف نشِط، وإرجاع حقول محددة (يُترجم إلى SQL)
var dto = await _context.Employees
    .AsNoTracking()
    .Where(e => e.EmpId == id && e.IsActive)      // Expression<Func<Employee,bool>>
    .Select(e => new EmployeeDto { Id = e.EmpId, Name = e.FullName })
    .FirstOrDefaultAsync(ct);                     // يعيد null إن لم يوجد

if (dto is null) return Results.NotFound();
return Results.Ok(dto);
```

**نقاط سريعة:**  
- لا تضع `.ToList()` قبل `Where/Select` مع EF.  
- تجنّب استدعاء دوال غير قابلة للترجمة داخل الشرط (استخدم `EF.Functions` عند الحاجة).

---

## مثال C# — بناء **شجرة تعبير** ديناميكيًا (فلترة مرنة)

```csharp
using System.Linq.Expressions;

Expression<Func<Employee, bool>> BuildPredicate(int? id, string? name)
{
    var p = Expression.Parameter(typeof(Employee), "e");
    Expression body = Expression.Constant(true); // يبدأ دائمًا بصحيح

    if (id is not null)
    {
        var idEq = Expression.Equal(
            Expression.Property(p, nameof(Employee.EmpId)),
            Expression.Constant(id.Value));
        body = Expression.AndAlso(body, idEq);
    }

    if (!string.IsNullOrWhiteSpace(name))
    {
        var nameProp = Expression.Property(p, nameof(Employee.FullName));
        var method = typeof(string).GetMethod(nameof(string.Contains), new[] { typeof(string) })!;
        var call = Expression.Call(nameProp, method, Expression.Constant(name));
        body = Expression.AndAlso(body, call);
    }

    return Expression.Lambda<Func<Employee, bool>>(body, p);
}

// استخدام:
var pred = BuildPredicate(id: 1001, name: "Kareem");
var list = await _context.Employees.Where(pred).ToListAsync(ct);
```

---

## مثال C# — LINQ في الذاكرة + أحداث

```csharp
// 1) في الذاكرة: delegates (Func/Action)
var nums = new[] { 1, 2, 3, 4, 5 };
var evens = nums.Where(n => n % 2 == 0).ToList();  // n => شرط

// 2) حدث بلامبدا
button.Click += (s, e) => Console.WriteLine("Clicked!");

// 3) لامبدا متعددة الأسطر
Func<int, int, int> sum = (a, b) => { var s = a + b; return s; };
```

---

## خطوات عملية للاستخدام الصحيح
1. **اختر النوع المناسب**:  
   - تنفيذ في الذاكرة → `Func<>/Action<>`.  
   - ترجمة (EF/Core) → `Expression<Func<…>>`.  
2. **اجعل الشروط قابلة للترجمة** (لا مناداة `DateTime.Now.ToString()` داخل `Where`).  
3. **أسقط إلى DTO** بـ `Select` لتقليل البيانات.  
4. **تجنّب الإغلاق الخطِر (Closure)**: لا تعتمد على متغيّرات ستتغيّر لاحقًا.  
5. **قصّر اللامبدا**: إن طالت، انقل المنطق إلى دالة مسماة قابلة للاختبار.  
6. **أضِف الإلغاء** مع الاستدعاءات غير المتزامنة (`…Async(ct)`).

---

## أخطاء شائعة
- استعمال `.ToList()` قبل الفلاتر مع EF → تحميل زائد من DB.  
- دوال غير قابلة للترجمة داخل اللامبدا → استثناء أو تقييم كلّي في الذاكرة.  
- خلط `Func<>` مع `Expression<Func<>>` في واجهات تتوقع ترجمة.  
- لامبدا تطول وتتعقّد → صعوبة قراءة واختبار.  
- إغلاق (Closure) يمسك مراجع تتغير → نتائج غير متوقّعة.

---

## جدول مقارنة مختصر

| المفهوم                     | الغرض الرئيسي            | ملاحظات مهمة                      |
| --------------------------- | ------------------------ | --------------------------------- |
| **Lambda Expression**       | تعريف دالة قصيرة بلا اسم | صيغة `=>`، تُستخدم في LINQ/الأحداث |
| **Func / Action**           | مندوب للتنفيذ الفوري     | للذاكرة؛ لا ترجمة إلى SQL         |
| **Expression\<Func\<…\>\>** | شجرة تعبير               | مطلوبة لمزوّدات LINQ (EF)          |
| Anonymous Method            | `delegate(...) { ... }`  | بديل أقدم للامبدا                 |
| Method Group                | تميرير اسم دالة مباشرة   | يتحوّل إلى مندوب تلقائيًا           |

---

## ملخص الفكرة  
**Lambda Expression** = طريقة سريعة لتعريف دالة وتمريرها كقيمة.  
مع **LINQ/EF** تمنحك استعلامات نظيفة وقابلة للترجمة،  
ومع الأحداث/القوائم تُسهّل الكتابة المختصرة—استخدمها بحذر، وبنمط واضح قابِل للاختبار.
