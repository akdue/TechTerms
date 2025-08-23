# **Decryption**

## الترجمة الحرفية  
**Decryption** — **فكّ التشفير** (إرجاع النصّ المُعمّى إلى نصٍّ واضح باستخدام مفتاح صحيح).

## الوصف العربي المختصر  
عملية معاكسة للتشفير: تأخذ **البيانات المشفّرة** و**المفتاح/المفاتيح** وتُعيد **المحتوى الأصلي**.  
في المخططات الحديثة (AEAD) يتحقق فكّ التشفير أيضًا من **سلامة الرسالة** و**أصالتها**.

## الشرح المبسّط  
- لديك **صندوق مقفول** (ciphertext) و**مفتاح**.  
- تفتح الصندوق فتستعيد الرسالة كما كانت.  
- إن غيّر أحدٌ محتوى الصندوق أو كان المفتاح خاطئًا → **تفشل العملية**.  
- نوعان شائعان:
  - **تماثلي (Symmetric):** نفس المفتاح للتشفير والفك (مثل **AES-GCM**).  
  - **غير تماثلي (Asymmetric):** مفتاح عام للتشفير وخاص للفك (مثل **RSA-OAEP**).

## تشبيه  
جهاز خزنة بكود سرّي: إن أدخلت الكود الصحيح تفتح الخزنة.  
لو أخطأت بالكود أو حاولت العبث بالقفل، لن تُفتح—وأحيانًا يطلق إنذار (فشل تحقق السلامة).

## مثال كود C# بسيط (AES-GCM — فكّ فقط للحِمل المجمّع)
> نفترض أن الحِمل بصيغة: `SALT(16) | NONCE(12) | TAG(16) | CIPHERTEXT(...)`
```csharp
using System;
using System.Security.Cryptography;
using System.Text;

public static class Decryptor
{
    public static string DecryptAesGcm(string base64, string password)
    {
        byte[] pack = Convert.FromBase64String(base64);

        byte[] salt  = new byte[16];
        byte[] nonce = new byte[12];
        byte[] tag   = new byte[16];
        byte[] ct    = new byte[pack.Length - salt.Length - nonce.Length - tag.Length];

        int o = 0;
        Buffer.BlockCopy(pack, o, salt, 0, salt.Length);                o += salt.Length;
        Buffer.BlockCopy(pack, o, nonce, 0, nonce.Length);              o += nonce.Length;
        Buffer.BlockCopy(pack, o, tag,   0, tag.Length);                o += tag.Length;
        Buffer.BlockCopy(pack, o, ct,    0, ct.Length);

        using var kdf = new Rfc2898DeriveBytes(password, salt, 100_000, HashAlgorithmName.SHA256);
        byte[] key = kdf.GetBytes(32); // 256-bit

        byte[] pt = new byte[ct.Length];
        using var aes = new AesGcm(key);
        // يرمي CryptographicException عند كلمة مرور خاطئة أو عبث بالبيانات
        aes.Decrypt(nonce, ct, tag, pt);

        return Encoding.UTF8.GetString(pt);
    }
}

// مثال استخدام:
// string plaintext = Decryptor.DecryptAesGcm(tokenBase64, "Strong#Passw0rd!");
```

## مثال كود C# مختصر (RSA-OAEP — فكّ بمفتاح خاص)
```csharp
using System.Security.Cryptography;
using System.Text;

// pemPrivateKey: نص PEM للمفتاح الخاص (PKCS#8)
string DecryptRsaOaep(string base64, string pemPrivateKey)
{
    byte[] ct = Convert.FromBase64String(base64);

    using var rsa = RSA.Create();
    rsa.ImportFromPem(pemPrivateKey);

    // OAEP باستخدام SHA-256 (أفضل من PKCS#1 v1.5)
    byte[] pt = rsa.Decrypt(ct, RSAEncryptionPadding.OaepSHA256);
    return Encoding.UTF8.GetString(pt);
}
```

## خطوات عملية لفكّ آمن (Checklist)
- استخدم مخطط حديث **AEAD** (مثل **AES-GCM** أو **ChaCha20-Poly1305**) لسلامة+سريّة معًا.  
- تحقّق من **صيغة الحِمل** (ترتيب الحقول: salt/nonce/tag/ct) وثبّت تنسيقًا موحّدًا.  
- لا تُعيد استخدام **Nonce/IV** مع نفس المفتاح في التماثلي.  
- خزّن المفاتيح في **Key Vault/HSM**، ولا تُضمّنها في الكود/المستودع.  
- عالِج **الاستثناءات** كفشل أمني محتمل وسجّل الحدث دون تسريب بيانات حسّاسة.  
- خطّط لـ **تدوير المفاتيح** وترحيل البيانات القديمة تدريجيًا.  
- في **RSA**، استخدم دائمًا **OAEP** بدل المخطط القديم (PKCS#1 v1.5).

## أخطاء شائعة
- الخلط بين **فكّ التشفير** و**فكّ الترميز** (Base64/UTF-8) — الترميز ليس أمانًا.  
- استخدام خوارزميات قديمة/أوضاع غير آمنة (**AES-ECB**، أو **RSA بدون OAEP**).  
- تجاهل فشل التحقق في AEAD والتعامل معه كـ “تحذير” بدل **منع القراءة**.  
- إعادة استخدام **Nonce** أو مفاتيح ساكنة لوقت طويل دون تدوير.  
- الاعتماد على كلمة مرور مباشرة دون **KDF** (PBKDF2/Argon2) و**Salt**.

## جدول مقارنة مختصر

| المفهوم                     | الغرض الرئيسي                               | ملاحظات مهمة                                 |
| --------------------------- | ------------------------------------------- | -------------------------------------------- |
| [Encryption](encryption.md) | إخفاء المحتوى ليسترجعه من يملك المفتاح      | يفضَّل AEAD (مثل AES-GCM) وإدارة مفاتيح سليمة  |
| **Decryption**              | **استرجاع النصّ الواضح والتحقق من السلامة**  | **يفشل عند مفتاح خاطئ/عبث بالبيانات (AEAD)** |
| [Encoding](encoding.md)     | تمثيل البيانات للنقل/التخزين (Base64/UTF-8) | ليس أمانًا؛ لا يمنع القراءة                   |
| [Hashing](hashing.md)       | بصمة أحادية الاتّجاه للتحقق                  | لا يُفك؛ استخدم مع Salt/KDF لكلمات المرور     |

## ملخص الفكرة  
**Decryption** يعيد البيانات إلى أصلها فقط عند امتلاك **المفتاح الصحيح** ونجاح **تحقّق السلامة**.  
التزم بمخططات حديثة (AEAD/RSA-OAEP)، صيغ حِمل واضحة، وإدارة مفاتيح آمنة لتفادي الفشل والتسريب.
