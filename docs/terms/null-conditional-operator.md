# **Null-Conditional Operator**

##  المعامل الشرطي الآمن (`?.`, `?[]`, `?.Invoke`)

## الترجمة الحرفية  
**Null-Conditional Operator** — **معامل التوقّف الآمن عند `null`**.

## الوصف العربي المختصر  
يسمح بالوصول إلى الخاصيّات/الأساليب/الفهارس **بشكل آمن**:  
إذا كان الطرف الأيسر `null`، **يُعيد `null` فورًا** دون رمي **NullReferenceException** ودون تقييم بقية السلسلة.

## الشرح المبسّط  
- اكتب `obj?.Prop` بدلًا من فحص `obj == null` يدويًا.  
- إن كان `obj` غير `null` → يُكمِل الوصول؛ وإلا → النتيجة `null`.  
- يعمل مع: الخاصيّات (`?.`)، الاستدعاء (`?.Method()`)، المؤشِّر (`arr?[i]`)، والأحداث (`Event?.Invoke(...)`).  
- غالبًا يُدمَج مع `??` لإعطاء **قيمة بديلة** بدل `null`.

## تشبيه  
طريق فيه بوابات. كل بوابة تتأكد: إن لم تُفتَح، **نرجع بلطف** بدل أن نصطدم بالحائط.  
وهكذا لا ينكسر البرنامج عند غياب جزء من السلسلة.

## مثال كود C# بسيط
```csharp
using System;
using System.Collections.Generic;

class Address { public string? City { get; set; } }
class User
{
    public string? Name { get; set; }
    public Address? Address { get; set; }
    public event EventHandler? LoggedIn;

    public void RaiseLogin() => LoggedIn?.Invoke(this, EventArgs.Empty); // آمن عند عدم وجود مشترِك
}

class Program
{
    static void Main()
    {
        User? user = null;

        // 1) خاصية متداخلة: بدون استثناء حتى لو user أو Address = null
        string city = user?.Address?.City ?? "Unknown";
        Console.WriteLine(city); // "Unknown"

        // 2) طول مصفوفة مع نوع قابل للإلغاء:
        string[]? items = null;
        int length = items?.Length ?? 0; // Length تعود int? ثم نعطي 0 كبديل

        // 3) فهرسة آمنة (arr?[i]) — يحمي من arr = null (ليس من خطأ الفهرس)
        int[]? nums = null;
        int? first = nums?[0]; // null إن كانت nums = null

        // 4) قاموس: يحمي من dict = null، لكن ليس من مفتاح غير موجود
        var dict = new Dictionary<string, int> { ["x"] = 10 };
        int? val = dict?["x"]; // 10
        // int? missing = dict?["y"]; // سيَرمي KeyNotFound إن كان dict ليس null والمفتاح غير موجود

        // 5) استدعاء دالة أو تفويض بشكل آمن
        Func<int, int>? op = null;
        int? result = op?.Invoke(5); // null إن كان op = null

        // 6) نمط تحقق من منطقية Nullable<bool>
        bool? isActive = null;
        if (isActive == true && user?.Name is { Length: > 0 })
            Console.WriteLine("Active user with name");

        // 7) مثال حدث من داخل الصنف
        var u = new User();
        u.RaiseLogin(); // لا مستمعين → آمن
    }
}
```
الإخراج المتوقع (مختصر):  
- `"Unknown"`، ثم `length = 0`.  
- `first = null` عند غياب المصفوفة.  
- الاستدعاءات الآمنة لا ترمي NullReferenceException.

## خطوات عملية لاستخدامه بفعالية
- ادمجه مع **`??`**: `user?.Address?.City ?? "N/A"`.  
- تذكّر أن الوصول إلى **قيمة** (مثل `int`) عبر `?.` يُنتج **Nullable** (`int?`). استخدم `??` أو تحويلًا مناسبًا.  
- عند الأحداث/التفويضات: استخدم **`?.Invoke(...)`** لتفادي السباق على `null`.  
- استخدمه لسلاسل وصول طويلة بدل فحوص `if (x != null)` المتكررة.  
- أضِف أقواسًا عند المزج مع معاملات أخرى لزيادة الوضوح.

## أخطاء شائعة
- افتراض أنه يمنع **كل** الاستثناءات: هو يمنع فقط **NullReference** للطرف الأيسر؛  
  ما يزال بإمكانك الحصول على **IndexOutOfRange** أو **KeyNotFound** إلخ.  
- استخدام `dict?["k"]` ظنًا أنه يحمي من المفتاح غير الموجود — يحمي فقط من **القاموس = null**.  
- نسيان أنه يعيد **`null`**؛ يجب التعامل مع النتيجة لاحقًا (مثلًا عبر `??`).

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [Ternary Operator](ternary-operator.md) | اختيار قيمة بناءً على **شرط منطقي عام** | مرن لعدة شروط؛ قد يصبح معقدًا بالتعشيش |
| **Null-Conditional Operator** | **إيقاف السلسلة عند `null` بأمان (`?.`, `?[]`)** | **يُعيد `null` بدل الرمي؛ ادمجه مع `??` لقيمة افتراضية** |
| [Null-Coalescing Operator](null-coalescing-operator.md) | تعيين قيمة بديلة عند `null` | `??` يعيد البديل، و`??=` يُسنِد عند `null` فقط |

## ملخص الفكرة  
`?.` و`?[]` و`?.Invoke` تمنحك **تنقلًا آمنًا** في السلاسل المعرضة لـ `null`.  
استخدمها مع `??` لتعيين بدائل واضحة، وتذكر أنها لا تمنع سوى **NullReference** للطرف الأيسر.
