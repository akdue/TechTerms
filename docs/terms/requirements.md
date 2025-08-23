# **Requirements**

## الترجمة الحرفية  
**Requirements** — **المتطلّبات** (ما يجب أن يفعله النظام وكيف يجب أن يكون أداؤه).

## الوصف العربي المختصر  
تفصيل **ما يحتاجه أصحاب المصلحة** من النظام:  
- **وظيفية (Functional):** سلوكيات وميزات.  
- **لا-وظيفية (NFRs):** أداء، أمان، توافُر، قابلية صيانة…  
مع **معايير قبول** واضحة و**إمكانية تتبّع** من التحليل حتى الاختبار.

## الشرح المبسّط  
- نكتب المتطلبات بصيغة **قابلة للاختبار** (Given/When/Then أو معايير قبول).  
- نميّز بين **Must/Should/Could/Won’t** (**MoSCoW** للأولوية).  
- نوثّق القيود (Constraints) والافتراضات (Assumptions).  
- نحافظ على **تتبّع (Traceability)**: كل متطلب ↔ تصميم ↔ كود ↔ اختبار.  
- المتطلبات تتغيّر؛ لذا نحتاج **حكم تغيير** وإصدارات.

## تشبيه  
عقد بناء شقة: يحدد **عدد الغرف**، **المساحات**، **العزل الحراري**، و**موعد التسليم**.  
المقاول لا يبدأ دون عقد واضح يمكن **فحصه** عند التسليم.

---

## مثال عملي مختصر — من متطلب إلى اختبار إلى تنفيذ (C# + xUnit)

> **REQ-PR-001**: *يستطيع المستخدم إعادة تعيين كلمة المرور بشرط أن يكون رمز الاستعادة صحيحًا وغير منتهي ولم يُستخدم سابقًا.*  
> **قبول:**  
> - إذا كان الرمز **سليمًا** → يرجع **Success** وتُحدّث كلمة المرور.  
> - إذا كان **منتهيًا** أو **مستعملًا** أو **غير مطابق للمستخدم** → **Failure** برسالة مناسبة.

### 1) اختبار (يعبّر عن معايير القبول)

```csharp
// dotnet add package xunit
// dotnet add package xunit.runner.visualstudio
using System;
using System.Threading.Tasks;
using Xunit;

public class PasswordResetTests
{
    [Fact] // حالة نجاح
    public async Task Reset_Succeeds_When_Token_Valid_And_Owned()
    {
        var store = new InMemoryResetStore();
        var svc = new PasswordResetService(store);

        var token = await store.IssueAsync("user@example.com", expiresInMinutes: 15);
        var res = await svc.ResetAsync("user@example.com", token, "N3wP@ssw0rd!");

        Assert.True(res.Success);
        Assert.True(await store.IsPasswordSetAsync("user@example.com"));
    }

    [Theory] // حالات فشل
    [InlineData("other@example.com", "TokenNotForUser")]
    [InlineData("user@example.com",  "Expired")]
    [InlineData("user@example.com",  "Used")]
    public async Task Reset_Fails_When_Token_Invalid(string email, string mode)
    {
        var store = new InMemoryResetStore();
        var svc = new PasswordResetService(store);

        var token = await store.IssueAsync("user@example.com", expiresInMinutes: 0); // ينتهي فورًا
        if (mode == "Used") await store.MarkUsedAsync(token);
        if (mode == "TokenNotForUser") email = "other@example.com";

        var res = await svc.ResetAsync(email, token, "N3wP@ssw0rd!");
        Assert.False(res.Success);
        Assert.NotEmpty(res.Error!);
    }
}
```

### 2) التنفيذ (بساطة تكفي لاجتياز الاختبارات)
```csharp
using System;
using System.Collections.Concurrent;
using System.Threading.Tasks;

public record ResetResult(bool Success, string? Error = null);

public interface IResetStore
{
    Task<string> IssueAsync(string email, int expiresInMinutes);
    Task<(string Email, DateTimeOffset Expiry, bool Used)?> LookupAsync(string token);
    Task MarkUsedAsync(string token);
    Task SetPasswordAsync(string email, string newPassword);
    Task<bool> IsPasswordSetAsync(string email);
}

public class PasswordResetService
{
    private readonly IResetStore _store;
    public PasswordResetService(IResetStore store) => _store = store;

    // تنفيذ متطلب REQ-PR-001
    public async Task<ResetResult> ResetAsync(string email, string token, string newPassword)
    {
        var info = await _store.LookupAsync(token);
        if (info is null) return new ResetResult(false, "invalid_token");
        if (!string.Equals(info.Value.Email, email, StringComparison.OrdinalIgnoreCase))
            return new ResetResult(false, "token_not_for_user");
        if (info.Value.Used) return new ResetResult(false, "token_used");
        if (DateTimeOffset.UtcNow > info.Value.Expiry) return new ResetResult(false, "token_expired");

        // (تبسيط) تحقق حد أدنى لنFR الأمن
        if (newPassword.Length < 8) return new ResetResult(false, "weak_password");

        await _store.SetPasswordAsync(email, newPassword);
        await _store.MarkUsedAsync(token);
        return new ResetResult(true);
    }
}

// تخزين داخل الذاكرة للاختبار فقط
public class InMemoryResetStore : IResetStore
{
    private record Token(string Email, DateTimeOffset Expiry, bool Used);
    private readonly ConcurrentDictionary<string, Token> _tokens = new();
    private readonly ConcurrentDictionary<string, string> _passwords = new();

    public Task<string> IssueAsync(string email, int expiresInMinutes)
    {
        var token = Guid.NewGuid().ToString("N");
        _tokens[token] = new Token(email, DateTimeOffset.UtcNow.AddMinutes(expiresInMinutes), false);
        return Task.FromResult(token);
    }
    public Task<(string Email, DateTimeOffset Expiry, bool Used)?> LookupAsync(string token)
        => Task.FromResult(_tokens.TryGetValue(token, out var t)
           ? (string Email, DateTimeOffset Expiry, bool Used)? (t.Email, t.Expiry, t.Used) : null);
    public Task MarkUsedAsync(string token)
    { if (_tokens.TryGetValue(token, out var t)) _tokens[token] = t with { Used = true }; return Task.CompletedTask; }
    public Task SetPasswordAsync(string email, string newPassword)
    { _passwords[email] = newPassword; return Task.CompletedTask; }
    public Task<bool> IsPasswordSetAsync(string email) => Task.FromResult(_passwords.ContainsKey(email));
}
```

**الفكرة:** بدأنا بمتطلب **قابل للاختبار**، كتبنا اختبارًا (قبول)، ثم تنفيذًا يحقّق الشرط.  
أي تغيير على المتطلب يجب أن ينعكس في **الاختبارات** قبل الكود.

---

## خطوات عملية لكتابة متطلبات ممتازة
- اكتب **هدف العمل** و**نطاقًا واضحًا** لكل متطلب.  
- صِغ **قصص مستخدم** مع **معايير قبول** (Given/When/Then).  
- غطِّ **NFRs** بقيم قابلة للقياس:  
  - **Performance:** P95 ≤ 200ms عند 200rps.  
  - **Availability:** 99.9% شهريًا.  
  - **Security:** MFA، تشفير TLS1.2+، سجلات تدقيق.  
- استخدم **MoSCoW** للأولوية.  
- حافظ على **التتبّع** (RTM): كل متطلب له اختبارات، تصميم، وتذاكر.  
- راجع المتطلبات مع أصحاب المصلحة و**اقفل النسخة** (Version) قبل التنفيذ.  
- ضع **سياسة تغيير** (Change Control) وتأثيرها على الجدول/التكلفة.

### مثال RTM صغير (Traceability)
| Req ID     | الوصف المختصر                     | تصميم/كود              | اختبار                 |
| ---------- | --------------------------------- | ---------------------- | ---------------------- |
| REQ-PR-001 | إعادة تعيين كلمة المرور برمز صالح | `PasswordResetService` | `PasswordResetTests.*` |

---

## أخطاء شائعة
- متطلبات **غامضة** (“سريع”، “آمن”) بدون **مقاييس**.  
- خلط المتطلبات مع **الحلّ** (“استعمل PostgreSQL”) بدل **النتيجة**.  
- عدم وجود **قبول** قابل للاختبار.  
- نسيان **NFRs** ثم اكتشافها بعد فوات الأوان.  
- غياب **التتبّع** → صعوبة معرفة ما الذي كُسر عند التغيير.  
- تغييرات بدون **حكم تغيير** وإصدارات.

---

## جدول مقارنة مختصر

| المفهوم                         | الغرض الرئيسي                                | ملاحظات مهمة                              |
| ------------------------------- | -------------------------------------------- | ----------------------------------------- |
| **Requirements**                | **ما يجب أن يفعله النظام (Functional/NFRs)** | تُكتب بمعايير قبول وقابلة للاختبار         |
| Acceptance Criteria             | تعريف النجاح لكل قصة/ميزة                    | تُحوَّل إلى اختبارات قبول                    |
| [Approach](approach.md)         | مقاربة/خطة لتحقيق المتطلبات                  | بدائل ومفاضلات وخطوات تنفيذ               |
| [Architecture](architecture.md) | حدود ومكوّنات وتدفقات                         | تُبنى لدعم المتطلبات/NFRs                  |
| [SDLC](sdlc.md)                 | مراحل التطوير                                | المتطلبات تدخل من البداية وتُتابَع          |
| [QA](qa.md)                     | ضمان العملية والجودة                         | يتحقق أن المتطلبات مُلبّاة باختبارات/بوابات |

---

## ملخص الفكرة  
**المتطلّبات** هي **العقد** الذي يقود التصميم والكود والاختبار.  
اجعلها **واضحة وقابلة للقياس** مع **معايير قبول** و**تتبّع** كامل—  
وبذلك تتحوّل إلى اختبارات تحرس المنتج عند كل تغيير.
