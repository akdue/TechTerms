# **Functionality Testing**

## الترجمة الحرفية  
**Functionality (Functional) Testing** — **اختبار وظيفي/اختبار الوظائف**

## الوصف العربي المختصر  
يتحقق أن **الميّزات تعمل حسب المتطلّبات**: مدخلات صحيحة/خاطئة، مسارات نجاح/فشل، رسائل الأخطاء، وقواعد العمل.

## الشرح المبسّط  
- يركّز على **ماذا** يفعل النظام (العقد والسلوك)، لا على الأداء أو الاستهلاك.  
- يشمل: **اختبار قبول**، **تكامل**، **دخولي (Smoke)**، **مرجعي (Regression)**، **حالات حواف/ حدود**.  
- أساليب مفيدة: **تقسيم مكافئ**، **تحليل القيم الحدّية**، **جداول قرار**.

## تشبيه  
ريموت التلفاز: نجرّب كل زر—قناة، صوت، كتم—ونتأكد أن **النتيجة المتوقَّعة** تحدث. لا يهمنا كيف صُنِع الريموت داخليًا.

---

## مثال C# — اختبار وظيفي لسيناريو تسجيل الدخول (قبول + سالب)

> **المتطلّب:**  
> - إذا كانت بيانات الدخول صحيحة ولم يكن المستخدم مقفولًا → **نجاح** + **Token**.  
> - إن كانت كلمة المرور خاطئة أو المستخدم مقفولًا → **فشل** برسالة مناسبة.  
> - حساب محاولات خاطئة لإقفال مؤقت بعد 3 محاولات.

```csharp
// LoginService.cs
using System;
using System.Collections.Concurrent;
using System.Threading.Tasks;

public record LoginResult(bool Success, string? Token = null, string? Error = null);

public interface IUserStore
{
    Task<bool> VerifyPasswordAsync(string email, string password);
    Task<bool> IsLockedAsync(string email);
    Task IncrementFailuresAsync(string email);
    Task ResetFailuresAsync(string email);
}

public class InMemoryUserStore : IUserStore
{
    private readonly ConcurrentDictionary<string, (string Pwd, int Failures, DateTimeOffset? LockedUntil)> _db = new();
    public InMemoryUserStore()
        => _db["user@example.com"] = ("N3wP@ss", 0, null);

    public Task<bool> VerifyPasswordAsync(string email, string password)
        => Task.FromResult(_db.TryGetValue(email, out var u) && u.Pwd == password);

    public Task<bool> IsLockedAsync(string email)
        => Task.FromResult(_db.TryGetValue(email, out var u) && u.LockedUntil is DateTimeOffset t && t > DateTimeOffset.UtcNow);

    public Task IncrementFailuresAsync(string email)
    {
        _db.AddOrUpdate(email,
            addValueFactory: _ => ("", 1, null),
            updateValueFactory: (_, u) =>
            {
                int f = u.Failures + 1;
                var locked = f >= 3 ? DateTimeOffset.UtcNow.AddMinutes(5) : u.LockedUntil;
                return (u.Pwd, f, locked);
            });
        return Task.CompletedTask;
    }

    public Task ResetFailuresAsync(string email)
    {
        if (_db.TryGetValue(email, out var u)) _db[email] = (u.Pwd, 0, null);
        return Task.CompletedTask;
    }
}

public class LoginService
{
    private readonly IUserStore _store;
    public LoginService(IUserStore store) => _store = store;

    // تنفيذ المتطلب الوظيفي
    public async Task<LoginResult> LoginAsync(string email, string password)
    {
        if (await _store.IsLockedAsync(email))
            return new LoginResult(false, Error: "locked");

        if (!await _store.VerifyPasswordAsync(email, password))
        {
            await _store.IncrementFailuresAsync(email);
            return new LoginResult(false, Error: "invalid_credentials");
        }

        await _store.ResetFailuresAsync(email);
        var token = Convert.ToBase64String(Guid.NewGuid().ToByteArray()); // تمثيل بسيط
        return new LoginResult(true, Token: token);
    }
}
```

### اختبارات وظيفية (xUnit) — نجاح، خطأ، قفل، إعادة المحاولة
```csharp
// dotnet add package xunit
// dotnet add package xunit.runner.visualstudio
using System.Threading.Tasks;
using Xunit;

public class LoginFunctionalTests
{
    [Fact] // قبول: بيانات صحيحة
    public async Task Login_Succeeds_With_Correct_Credentials()
    {
        var store = new InMemoryUserStore();
        var svc = new LoginService(store);

        var res = await svc.LoginAsync("user@example.com", "N3wP@ss");
        Assert.True(res.Success);
        Assert.False(string.IsNullOrEmpty(res.Token));
    }

    [Fact] // سالب: كلمة مرور خاطئة + رسائل مفهومة
    public async Task Login_Fails_With_Wrong_Password()
    {
        var store = new InMemoryUserStore();
        var svc = new LoginService(store);

        var res = await svc.LoginAsync("user@example.com", "wrong");
        Assert.False(res.Success);
        Assert.Equal("invalid_credentials", res.Error);
    }

    [Fact] // قفل بعد 3 محاولات خاطئة
    public async Task Account_Locks_After_Three_Failures()
    {
        var store = new InMemoryUserStore();
        var svc = new LoginService(store);

        await svc.LoginAsync("user@example.com", "x");
        await svc.LoginAsync("user@example.com", "y");
        await svc.LoginAsync("user@example.com", "z");

        var res = await svc.LoginAsync("user@example.com", "N3wP@ss");
        Assert.False(res.Success);
        Assert.Equal("locked", res.Error);
    }
}
```

**الفكرة:** اختبارات **Black-Box** على عقد الخدمة (مدخلات/مخرجات).  
نغطّي **سيناريو نجاح** + **سالب** + **حالة حدّية** (القفل بعد 3).

---

## خطوات عملية لاختبار وظيفي فعّال
1. **حوّل المتطلبات إلى حالات اختبار** (Given/When/Then + معايير قبول).  
2. غطِّ **المسارات الأساسية** و**الاستثناءات** و**الحدود** (0/1/Many، طول، نطاق).  
3. جهّز **بيانات اختبار** واقعية وآمنة (مجهولة، غير إنتاجية).  
4. ابنِ **حزمة Regression** تُشغَّل آليًا في CI عند كل تغيير.  
5. استعمل **تتبّع** Req ↔ Test (RTM) لمعرفة التغطية.  
6. للواجهات الشبكية: اختبارات **تكامل/E2E** باستخدام عميل HTTP أو أداة UI مناسبة.  
7. وثّق **نتائج التنفيذ** (تمرير/فشل) وخطّة علاج العيوب.

---

## أخطاء شائعة
- الخلط بين **الوظيفي** و**اللا-وظيفي** (أداء/أمان) وفقدان التركيز.  
- إهمال **حالات الحواف** أو رسائل الخطأ القابلة للفهم.  
- اختبارات تعتمد على **حالة مشتركة هشة** (ترتيب التنفيذ).  
- عدم تحديث الاختبارات عند تعديل المتطلبات → نتائج مضلّلة.  
- غياب **Regression** بعد إصلاح العيوب → عودة الأخطاء القديمة.

---

## جدول مقارنة مختصر

| المفهوم                                 | الغرض الرئيسي                     | ملاحظات مهمة                    |
| --------------------------------------- | --------------------------------- | ------------------------------- |
| **Functionality Testing**               | **تحقق من الميزات وفق المتطلبات** | تركيز على العقد والسلوك المتوقع |
| Non-Functional (NFR)                    | أداء/أمان/توافر/قابلية استخدام    | مكمل لاختبار الوظائف            |
| Smoke/Sanity                            | فحص سريع لأساسيات التطبيق         | يسبق أعمق الاختبارات            |
| Regression                              | منع عودة عيوب قديمة               | يُشغَّل آليًا مع كل تغيير           |
| [Black-Box](black-white-box-testing.md) | اختبار مدخلات/مخرجات فقط          | مناسب للقبول والتكامل           |

---

## ملخص الفكرة  
**Functionality Testing** يثبت أن **الميّزات تعمل كما وُصفت** لكل الحالات المهمة،  
مع **حالات حواف/سلبية** وحزمة **Regression** آلية—فتحصل على ثقة حقيقية عند كل إصدار.
