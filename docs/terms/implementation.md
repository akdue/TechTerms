# **Implementation**

## الترجمة الحرفية  
**Implementation** — **تنفيذ/تحقيق** التصميم إلى **برمجية عاملة** (كود + إعداد + سكربتات + IaC).

## الوصف العربي المختصر  
تحويل **المتطلّبات والتصميم** إلى حلول تعمل:  
كتابة الكود، ضبط الإعدادات، الهجرة (Migrations)، الاختبارات، والتعبئة/النشر القابل للتكرار.

## الشرح المبسّط  
- نأخذ **ما يجب أن يحدث** (Requirements/Design) ونبنيه فعليًا.  
- يشمل: هياكل كود، **حرس أخطاء**، أداء، أمن، **Observability** (Logs/Metrics/Tracing).  
- يعكس **NFRs**: مهلات، ذاكرة، توفّر، تشفير…  
- ينتهي بـ **Artifact** جاهز: حزمة/صورة/ملف تنفيذ + اختبارات خضراء ووثائق محدثة.

## تشبيه  
المخطّط المعماري للبيت شيء، و**البناء** على الأرض شيء آخر. التنفيذ = العمال والخرسانة والكهرباء حتى يسكن الناس.

---

## مثال C# قصير — تنفيذ متطلب أمان: **تخزين كلمات المرور** (PBKDF2)
> **المتطلب (NFR/SEC-001):**  
> - هاش بكِلفة قابلة للضبط (Iterations).  
> - ملح فريد لكل كلمة (Salt).  
> - مقارنة **ثابتة الوقت**.  
> - صيغة نصية قابلة للنقل: `pbkdf2:sha256$iter$salt_base64$hash_base64`

```csharp
// .NET 8/9
using System;
using System.Security.Cryptography;
using System.Text;

public interface IPasswordHasher
{
    string Hash(string password);                          // إنشاء تمثيل آمن
    bool Verify(string password, string encodedHash);      // تحقق ثابت الوقت
}

public class Pbkdf2PasswordHasher : IPasswordHasher
{
    private const int SaltBytes = 16;
    private const int HashBytes = 32;                      // لـ sha256
    private readonly int _iterations;

    public Pbkdf2PasswordHasher(int iterations = 100_000)  // قابل للضبط حسب الأداء
        => _iterations = iterations;

    public string Hash(string password)
    {
        byte[] salt = RandomNumberGenerator.GetBytes(SaltBytes);
        byte[] hash = PBKDF2(password, salt, _iterations);
        return $"pbkdf2:sha256${_iterations}${Convert.ToBase64String(salt)}${Convert.ToBase64String(hash)}";
    }

    public bool Verify(string password, string encoded)
    {
        // parsing format: algo$iter$salt$hash
        var parts = encoded.Split('$', StringSplitOptions.RemoveEmptyEntries);
        if (parts.Length != 4 || !parts[0].StartsWith("pbkdf2:sha256")) return false;

        if (!int.TryParse(parts[1], out int iters)) return false;
        var salt = Convert.FromBase64String(parts[2]);
        var expected = Convert.FromBase64String(parts[3]);

        var actual = PBKDF2(password, salt, iters);
        return FixedTimeEquals(actual, expected);
    }

    private static byte[] PBKDF2(string pwd, byte[] salt, int iter)
    {
        using var pbkdf2 = new Rfc2898DeriveBytes(pwd, salt, iter, HashAlgorithmName.SHA256);
        return pbkdf2.GetBytes(HashBytes);
    }

    private static bool FixedTimeEquals(ReadOnlySpan<byte> a, ReadOnlySpan<byte> b)
    {
        if (a.Length != b.Length) return false;
        int diff = 0;
        for (int i = 0; i < a.Length; i++) diff |= a[i] ^ b[i];
        return diff == 0;
    }
}

class Demo
{
    static void Main()
    {
        var hasher = new Pbkdf2PasswordHasher(120_000);
        var encoded = hasher.Hash("N3wP@ssw0rd!");
        Console.WriteLine(encoded);            // pbkdf2:sha256$120000$...$...

        Console.WriteLine(hasher.Verify("N3wP@ssw0rd!", encoded)); // True
        Console.WriteLine(hasher.Verify("wrong", encoded));        // False
    }
}
```

**لماذا هذا “تنفيذ” جيّد؟**  
- يلبّي **المتطلب الأمني** المعلن.  
- **قابل للتهيئة** (Iterations).  
- صيغة **قابلة للنقل/النسخ الاحتياطي**.  
- حماية مقارنة **ثابتة الوقت**.

---

## خطوات عملية لتنفيذ فعّال
1. **Definition of Done** واضح: اختبارات خضراء، تغطية دنيا، وثائق، مقياس/لوج، Artifact قابل للنشر.  
2. ابدأ من **العقد** (Interfaces/Ports) ثم طبّق **Adapters**.  
3. طبّق **حرس الأخطاء** (Args/Null/Boundary) و**Cancellation/Timeouts** للعمليات الخارجية.  
4. أدر **الإعدادات بأمان** (Options/Secrets) ولا تثبّتها في الكود.  
5. أضِف **Telemetry** منذ البداية (Logs بنمط موحّد، قياسات أساسية، Trace IDs).  
6. اختبر: وحدات + تكامل + قبول. افصل **اختبارات بطيئة**.  
7. انشر عبر **CI/CD**؛ أنتج صورة/حزمة **OCI** موقّعة مع SBOM وفحص ثغرات.  
8. راجع الأداء/الأمان مقابل **NFRs** (ملفات تعريف/تحمّل).

---

## أخطاء شائعة
- القفز للكود قبل تثبيت **المتطلبات/القبول**.  
- ربط التنفيذ بمكتبة/مزود مباشرة بدل **واجهة** → صعوبة الاستبدال.  
- تجاهل **NFRs** (مهلات، ذاكرة، أمن) حتى آخر لحظة.  
- غياب **Telemetry** → لا نرى الفشل في الإنتاج.  
- عدم معالجة الأخطاء القابلة لإعادة المحاولة (شبكة/HTTP 5xx/429).  
- أسرار داخل المستودع/الطبقات.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [Requirements](requirements.md) | ماذا يجب أن يفعله النظام | مصدر الحقيقة للقبول |
| [Design/Architecture](architecture.md) | كيف نُقسّم ونربط المكوّنات | حدود وعقود وقرارات كبرى |
| **Implementation** | **تحويل التصميم إلى كود عامل** | **يلبّي NFRs + اختبارات + Telemetry** |
| [Deployment](deployment.md) | وضع المخرجات في بيئة تشغيل | إصدارات/إستراتيجيات طرح/تشغيل |
| PoC/Spike | اختبار فكرة بسرعة | قد لا يلبّي NFRs؛ ليس منتجًا |

---

## ملخص الفكرة  
**Implementation** هو مرحلة **بناء الحقيقة**: كود وإعداد وتشغيل يلبي المتطلبات وسمات الجودة.  
ابدأ من **العقود**، طبّق بوعي **NFRs**، أضِف **Telemetry** منذ اليوم الأول، واجعل المخرجات قابلة للنشر والقياس—تحصل على تنفيذ موثوق وقابل للصيانة. 
