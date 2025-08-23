# **Encryption**

## الترجمة الحرفية  
**Encryption** — **تشفير البيانات** (تحويلها إلى صيغة غير مفهومة إلا بمفتاح).

## الوصف العربي المختصر  
تقنية تحمي السريّة عبر **تحويل النصّ الواضح** إلى **نصّ مُعمّى** باستخدام **مفتاح**.  
الفكّ يتم بالمفتاح الصحيح، وقد يشمل **توثيق الرسالة** لمنع العبث (AEAD).

## الشرح المبسّط  
- لديك **رسالة** + **مفتاح** → تخرج **بيانات غير مفهومة**.  
- من يملك المفتاح فقط يستطيع **إعادتها** إلى نصّها الأصلي.  
- نوعان شائعان:  
  - **متماثل (Symmetric):** نفس المفتاح للتشفير والفك (AES).  
  - **غير متماثل (Asymmetric):** مفتاح عام للتشفير وخاص للفك (RSA/EC).  
- الأفضل اليوم مع التماثل: **AES-GCM** (تشفير + سلامة في خطوة واحدة).

## تشبيه  
قفل وصندوق. تضع الرسالة في الصندوق وتُقفلها.  
من لديه **المفتاح** فقط يفتحه ويقرأ الرسالة. لو حاول أحد **تغيير المحتوى**، القفل الذكي يكتشف ذلك.

## مثال كود C# بسيط (AES-GCM مع اشتقاق مفتاح من كلمة مرور)
```csharp
using System;
using System.Security.Cryptography;
using System.Text;

public static class CryptoDemo
{
    // يدمج: SALT(16) + NONCE(12) + TAG(16) + CIPHERTEXT(..)
    public static string Encrypt(string plaintext, string password)
    {
        // 1) اشتقاق مفتاح 256-بت من كلمة مرور عبر PBKDF2
        byte[] salt  = RandomNumberGenerator.GetBytes(16);
        using var kdf = new Rfc2898DeriveBytes(password, salt, 100_000, HashAlgorithmName.SHA256);
        byte[] key = kdf.GetBytes(32);

        // 2) توليد Nonce فريد لكل رسالة (12 بايت)
        byte[] nonce = RandomNumberGenerator.GetBytes(12);

        // 3) تشفير + وسم سلامة (AEAD)
        byte[] pt = Encoding.UTF8.GetBytes(plaintext);
        byte[] ct = new byte[pt.Length];
        byte[] tag = new byte[16];
        using var aes = new AesGcm(key);
        aes.Encrypt(nonce, pt, ct, tag); // يمكن إضافة AAD عند الحاجة

        // 4) التجميع للتخزين/النقل
        byte[] pack = new byte[salt.Length + nonce.Length + tag.Length + ct.Length];
        Buffer.BlockCopy(salt,  0, pack, 0,               salt.Length);
        Buffer.BlockCopy(nonce, 0, pack, salt.Length,     nonce.Length);
        Buffer.BlockCopy(tag,   0, pack, salt.Length+nonce.Length, tag.Length);
        Buffer.BlockCopy(ct,    0, pack, salt.Length+nonce.Length+tag.Length, ct.Length);

        return Convert.ToBase64String(pack);
    }

    public static string Decrypt(string b64, string password)
    {
        byte[] pack = Convert.FromBase64String(b64);

        byte[] salt  = new byte[16];
        byte[] nonce = new byte[12];
        byte[] tag   = new byte[16];
        byte[] ct    = new byte[pack.Length - salt.Length - nonce.Length - tag.Length];

        int o = 0;
        Buffer.BlockCopy(pack, o, salt, 0, salt.Length); o += salt.Length;
        Buffer.BlockCopy(pack, o, nonce, 0, nonce.Length); o += nonce.Length;
        Buffer.BlockCopy(pack, o, tag, 0, tag.Length); o += tag.Length;
        Buffer.BlockCopy(pack, o, ct, 0, ct.Length);

        using var kdf = new Rfc2898DeriveBytes(password, salt, 100_000, HashAlgorithmName.SHA256);
        byte[] key = kdf.GetBytes(32);

        byte[] pt = new byte[ct.Length];
        using var aes = new AesGcm(key);
        aes.Decrypt(nonce, ct, tag, pt); // يرمي استثناء عند العبث (فشل التحقق)

        return Encoding.UTF8.GetString(pt);
    }
}

// مثال استخدام:
// string token = CryptoDemo.Encrypt("Secret message", "Strong#Passw0rd!");
// string msg   = CryptoDemo.Decrypt(token, "Strong#Passw0rd!");
```
**الإخراج المتوقع:** سلسلة Base64 مشفّرة. عند الفك بالمفتاح الصحيح، تعود `"Secret message"`. إن تغيّر البايتات أو استُخدمت كلمة مرور خاطئة → **استثناء** (فشل سلامة).

## خطوات عملية لتشفير آمن (Checklist)
- اختر خوارزمية حديثة: **AES-GCM** (تماثلي) أو **ChaCha20-Poly1305**.  
- لا تعِد استخدام **Nonce/IV** مع نفس المفتاح؛ ولّده عشوائيًا لكل رسالة.  
- اشتق المفاتيح من كلمات مرور عبر **KDF** قوي (PBKDF2/Argon2) مع **Salt** و**عدد دورات** عالٍ.  
- خزّن المفاتيح في **Secret Manager/Key Vault/HSM**، ولا تضعها في الكود/المستودع.  
- دوّن **تنسيق الحِمل** (كيف تحفظ salt/nonce/tag) ووثّقه.  
- نفّذ **تدوير مفاتيح** (Key Rotation) وخطّة ترحيل.  
- امنع **التسريب الجانبي** (Logs، استثناءات، مفاتيح في ذاكرة طويلة العمر).  
- اختبر الأداء والصحّة، وأدرج اختبارات تفشل عند العبث (tamper tests).

## أخطاء شائعة
- استخدام **AES-ECB** أو التشفير دون **توثيق** (لا HMAC/AEAD).  
- إعادة استخدام **IV/Nonce** مع نفس المفتاح.  
- مفاتيح **صلبة** في الكود أو بيئات بدون وصول مقيّد.  
- الخلط بين **التشفير** و**الهاش** أو **الترميز**.  
- تطوير خوارزمية مخصّصة بدل بروتوكولات قياسية مدقّقة.

## جدول مقارنة مختصر

| المفهوم                                   | الغرض الرئيسي                              | ملاحظات مهمة                                           |
| ----------------------------------------- | ------------------------------------------ | ------------------------------------------------------ |
| [Hashing](hashing.md)                     | بصمة أحادية الاتّجاه للتحقّق/التخزين         | لا يُفك؛ استخدمه لكلمات المرور مع **Salt + KDF**        |
| **Encryption**                            | **إخفاء المحتوى ليسترجعه من يملك المفتاح** | **يفضّل AEAD (مثل AES-GCM)، وإدارة مفاتيح/Nonce صحيحة** |
| [Encoding](encoding.md)                   | تمثيل البيانات بصيغة نقل (Base64/UTF-8)    | ليس أمانًا؛ لا يمنع القراءة                             |
| [Digital Signature](digital-signature.md) | إثبات هوية المرسل وسلامة الرسالة           | غير سرّي؛ يتحقق بالمفتاح العام                          |
| [Decryption](decryption.md)               | استرجاع النصّ الواضح والتحقق من السلامة     | يفشل عند مفتاح خاطئ/عبث بالبيانات (AEAD)               |

## ملخص الفكرة  
**التشفير** يحمي السريّة. طبّقه بخوارزميات حديثة (AEAD)، مفاتيح مُدارة جيدًا، وNonces فريدة.  
تذكّر: **Encoding ≠ Encryption ≠ Hashing** — لكل واحد غرض مختلف.
