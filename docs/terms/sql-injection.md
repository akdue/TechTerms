# **SQL Injection**

## الترجمة الحرفية  
**SQL Injection** — **حقن أوامر/استعلامات SQL** داخل تطبيقك.

## الوصف العربي المختصر  
ثغرة تحدث عند **دمج مدخلات المستخدم** داخل استعلام SQL **كنصّ خام**.  
يسمح للمهاجم بتنفيذ استعلامات غير مقصودة: **تجاوز تسجيل الدخول، قراءة/تعديل/حذف بيانات**.

## الشرح المبسّط  
- التطبيق يبني استعلامًا باستخدام **سلاسل نصية** من المستخدم.  
- المُدخل الخبيث يُكمل الاستعلام ويغيّر معناه.  
- قاعدة البيانات تنفّذ الاستعلام كما وصلها.  
- النتيجة: تسريب بيانات أو تعديلها أو السيطرة على الحسابات.

## تشبيه  
كتابة أمرٍ في ورقةٍ مع **فراغات** يملؤها أحدهم. لو كتب عبارات إضافية،  
يتحوّل المعنى تماما. الحل؟ **نموذج ثابت** يضع القيم في **خانات آمنة** فقط.

## مثال كود C# (ضعيف ثم آمن)

### ❌ ضعيف: دمج النصوص (لا تفعل)
```csharp
string user = txtUser.Text;
string pass = txtPass.Text;

string sql = $"SELECT Id FROM Users WHERE Username = '{user}' AND Password = '{pass}'";
// هكذا يمكن لمدخل خبيث أن يغيّر معنى الاستعلام كله

using var con = new SqlConnection(connString);
using var cmd = new SqlCommand(sql, con);
con.Open();
var id = cmd.ExecuteScalar();
```

### ✅ آمن: معاملات (Prepared/Parameterized)
```csharp
using var con = new SqlConnection(connString);
using var cmd = new SqlCommand(
    "SELECT Id FROM Users WHERE Username = @u AND PasswordHash = @p", con);

cmd.Parameters.Add("@u", SqlDbType.NVarChar, 64).Value = txtUser.Text;
// مرّر القيم كمعاملات — لا تُدمج داخل نصّ الاستعلام
cmd.Parameters.Add("@p", SqlDbType.VarChar, 64).Value = ComputeHash(txtPass.Text);

con.Open();
var id = cmd.ExecuteScalar();
```

### ✅ آمن أيضًا: ORM (EF Core يستخدِم معاملات تلقائيًا)
```csharp
// EF Core يحوِّل التعبير التالي إلى استعلام مُعَلْمن (Parameterized)
var u = await db.Users
    .SingleOrDefaultAsync(x => x.Username == user && x.PasswordHash == hash);
```

> ملاحظة: **لا تخزّن كلمات المرور كنصّ**. استخدم **Hash + Salt** وخوارزميات مثل PBKDF2/Argon2/Bcrypt.

## خطوات عملية للتخفيف (Checklist)
- استخدم **Parameterized Queries / Prepared Statements** دائمًا.  
- اعتمد **ORM** موثوق (EF Core/Dapper مع معاملات).  
- **Least Privilege**: حساب قاعدة البيانات بصلاحيات محدودة.  
- **تبييض الإدخال (Allowlist)** للحقول ذات القيم المعروفة (مثل SortField).  
- فعّل **ORM/DB Escaping** الافتراضي؛ لا تعتمد على `Replace` اليدوي.  
- **سجّل محاولات غير طبيعية** وراقب أنماط الاستعلام.  
- أخفِ تفاصيل الأخطاء؛ أرسل رسائل عامة للمستخدم، ودوّن التفاصيل داخليًا.  
- اختبر بـ **أدوات فحص** (SAST/DAST) وأنشئ اختبارات وحدات ضد الحقن.

## أخطاء شائعة
- الاعتماد على **تصفية نصّية** يدوية بدل المعاملات.  
- بناء أجزاء SQL من خيارات واجهة المستخدم مباشرة (مثل `ORDER BY " + sort`).  
- استعمال مستخدم DB بامتيازات عالية (dbo).  
- كشف رسائل أخطاء SQL التفصيلية للمستخدم النهائي.

## جدول مقارنة مختصر

| المفهوم                                   | الغرض الرئيسي                         | ملاحظات مهمة                                       |
| ----------------------------------------- | ------------------------------------- | -------------------------------------------------- |
| XSS                          | حقن سكربت في المتصفح                  | يستهدف المتصفّح/DOM؛ العلاج بـ CSP وتطهير HTML      |
| **SQL Injection**                         | **التلاعب باستعلامات قاعدة البيانات** | **حلّه الأساس: معاملات/Prepared + Least Privilege** |
| Command Injection | حقن أوامر نظام التشغيل                | يحدث عند تمريب مدخلات لأوامر Shell دون ضبط         |

## ملخص الفكرة  
**جذر المشكلة**: دمج المدخلات في الاستعلام كنص.  
**العلاج**: معاملات مُعَلْمنة، صلاحيات دنيا، اختبار وفحص آلي.

## سؤال تثبيت
*أنت تبني استعلامًا يعتمد على مدخلات المستخدم. ما الإجراء الأهم لتجنّب SQL Injection؟*
<details>
  <summary><strong>إظهار الإجابة</strong></summary>
  استخدام **Prepared/Parameterized Queries** (أو ORM يطبّقها تلقائيًا)، مع **Least Privilege** على مستوى قاعدة البيانات.
</details>
