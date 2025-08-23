# **Hashing**

## الترجمة الحرفية  
**Hashing / Hash Function** — **تجزئة/دالة هاش** تنتج **بصمة ثابتة الطول** لبيانات إدخال.

## الوصف العربي المختصر  
تحويل حتمي لأي بيانات إلى **بصمة قصيرة** ثابتة الطول.  
الهدف: **التحقق** و**الفهرسة** و**سلامة البيانات** — **ليس** لإخفاء المحتوى ولا يمكن عكسه عمليًا.

## الشرح المبسّط  
- نفس الإدخال → نفس البصمة دائمًا.  
- تغيّر بسيط في الإدخال → بصمة مختلفة جذريًا (Avalanche).  
- خصائص مطلوبة: **مقاومة التصادم**، **مقاومة الصورة الأولى/الثانية**.  
- استخدامات: التحقق من الملفات، بنى البيانات (HashTable)، تخزين كلمات المرور عبر **KDF + Salt**.

## تشبيه  
آلة تصنع **ختمًا** لورقة. الختم يثبت أنها نفس الورقة لاحقًا،  
لكن لا يمكنك إعادة الورقة من الختم وحده.

## مثال كود C# بسيط (SHA-256 + HMAC + تخزين كلمات المرور بـ PBKDF2)
```csharp
using System;
using System.Security.Cryptography;
using System.Text;

class HashingDemo
{
    // 1) هاش عام (SHA-256) — للتحقق من سلامة ملف/نص
    public static string Sha256(string text)
    {
        using var sha = SHA256.Create();
        byte[] bytes = sha.ComputeHash(Encoding.UTF8.GetBytes(text));
        return Convert.ToHexString(bytes); // بصمة سداسية
    }

    // 2) HMAC-SHA256 — بصمة مع مفتاح سرّي (سلامة + أصالة رسالة)
    public static string HmacSha256(string text, byte[] key)
    {
        using var hmac = new HMACSHA256(key);
        return Convert.ToHexString(hmac.ComputeHash(Encoding.UTF8.GetBytes(text)));
    }

    // 3) تخزين كلمة مرور: PBKDF2 (KDF بطيء) + Salt
    // التخزين المقترح: Base64( salt | iterations(4B) | hash )
    public static string HashPassword(string password, int iterations = 100_000)
    {
        byte[] salt = RandomNumberGenerator.GetBytes(16);
        using var kdf = new Rfc2898DeriveBytes(password, salt, iterations, HashAlgorithmName.SHA256);
        byte[] hash = kdf.GetBytes(32); // 256-bit

        byte[] pack = new byte[salt.Length + 4 + hash.Length];
        Buffer.BlockCopy(salt, 0, pack, 0, salt.Length);
        BitConverter.GetBytes(iterations).CopyTo(pack, salt.Length);
        Buffer.BlockCopy(hash, 0, pack, salt.Length + 4, hash.Length);
        return Convert.ToBase64String(pack);
    }

    public static bool VerifyPassword(string password, string stored)
    {
        byte[] pack = Convert.FromBase64String(stored);
        byte[] salt = new byte[16];
        Buffer.BlockCopy(pack, 0, salt, 0, salt.Length);
        int iterations = BitConverter.ToInt32(pack, salt.Length);
        byte[] hash = new byte[32];
        Buffer.BlockCopy(pack, salt.Length + 4, hash, 0, hash.Length);

        using var kdf = new Rfc2898DeriveBytes(password, salt, iterations, HashAlgorithmName.SHA256);
        byte[] candidate = kdf.GetBytes(32);
        return CryptographicOperations.FixedTimeEquals(hash, candidate); // مقارنة آمنة زمنياً
    }

    static void Main()
    {
        Console.WriteLine(Sha256("hello")); // 2CF24DBA...

        var key = Encoding.UTF8.GetBytes("secret");
        Console.WriteLine(HmacSha256("msg", key)); // HMAC

        string rec = HashPassword("P@ssw0rd!");
        Console.WriteLine(VerifyPassword("P@ssw0rd!", rec)); // True
        Console.WriteLine(VerifyPassword("wrong", rec));     // False
    }
}
```
**الإخراج المتوقع (مختصر):**  
- بصمة SHA-256 لـ`hello`.  
- بصمة HMAC للنص `msg`.  
- **True/False** للتحقق من كلمة المرور.

## خطوات عملية لاستخدام التجزئة بأمان
- اختر دوال حديثة: **SHA-256/512** أو **BLAKE2/3** (حسب الدعم).  
- لكلمات المرور: **لا تستخدم SHA مباشرة**؛ استخدم **KDF بطيء**: PBKDF2/Argon2/bcrypt مع **Salt** فريد.  
- للتوثيق مع سرّ مشترك: **HMAC** بدل الهاش العادي.  
- دوّن **تنسيق التخزين** (salt/iterations/hash) وخطّة **تدوير** المعاملات والمفاتيح.  
- قارن البصمات عبر دالة **زمن ثابت** لتجنّب قنوات جانبية.  
- تجنّب **SHA-1/MD5** للأمان (غير موصى بهما بسبب تصادمات).

## أخطاء شائعة
- الخلط بين **Hashing** و**Encryption** و**Encoding**.  
- تخزين كلمات المرور بـ **SHA-256 فقط** دون **Salt/KDF**.  
- إعادة استخدام **Salt** أو تجاهل زيادة **iterations** بمرور الوقت.  
- استخدام هاش عادي للتحقق من الرسائل بينما يجب **HMAC** أو **توقيع رقمي**.  
- مقارنة بصمات بسذاجة (`==`) بدل مقارنة زمن ثابت.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [Encryption](encryption.md) | إخفاء المحتوى ليسترجعه صاحب المفتاح | قابل للعكس بالمفتاح؛ يوفّر سريّة |
| **Hashing** | **بصمة أحادية الاتّجاه للتحقّق/الفهرسة** | **غير قابل للعكس؛ استخدم KDF+Salt لكلمات المرور** |
| [Encoding](encoding.md) | تمثيل البيانات للنقل (Base64/UTF-8) | ليس أمانًا؛ لا يمنع القراءة |

## ملخص الفكرة  
**Hashing** ينتج **بصمة** ثابتة لا تُعكس عمليًا.  
للكلمات السرّية: استعمل **KDF بطيئًا مع Salt**؛ للرسائل الموقَّعة سرّيًا: **HMAC**.  
تذكّر: **Hashing ≠ Encryption ≠ Encoding** — لكل غرضه.
