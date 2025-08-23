# **ETL/ELT**

## الترجمة الحرفية  
**ETL** — **اِستِخراج → تحويل → تحميل**  
**ELT** — **اِستِخراج → تحميل → تحويل**
**Extract–Transform–Load / Extract–Load–Transform**
## الوصف العربي المختصر  
طريقتان لبناء مواسير بيانات (Data Pipelines):  
- **ETL**: نُحَوِّل البيانات قبل إدخالها للمخزن.  
- **ELT**: نحمّل الخام سريعًا إلى المخزن/البحيرة، ثم نُحَوِّلها داخله (SQL/DBT).

## الشرح المبسّط  
- **ETL** يناسب التحويلات الثقيلة خارج المستودع، أو حين نحتاج **تنظيفًا صارمًا** قبل التخزين.  
- **ELT** يستفيد من **قوة المعالجة داخل المستودع** (DW/Lakehouse) ويُسرِّع الوصول للبيانات الخام.  
- كلاهما يحتاج: **عقود سكيمة**، **تتبّع نسخ/إصدارات**، **تحقّق جودة**، **توقيت** (دُفعات/شبه لحظي)، و**Idempotency**.

## تشبيه  
تنقية ماء النهر:  
- **ETL** = ننقّي الماء ثم نخزّنه في الخزان.  
- **ELT** = نخزّن الماء كما هو، ثم نشغّل فلاتر الخزان داخله كلما احتجنا.

---

## مثال عملي مختصر — **ETL** بـ C# (CSV → تنظيف → `SqlBulkCopy`)

> **افتراض**: CSV بسيط مفصول بفواصل بدون اقتباسات معقّدة: `Id,Name,Price,CreatedUtc`  
> **الهدف**: تجاهل الصفوف السيئة، تنسيق الأرقام/التواريخ، ثم **Bulk Load** إلى SQL Server.

```csharp
// dotnet add package Microsoft.Data.SqlClient
using System;
using System.Data;
using System.Globalization;
using System.IO;
using Microsoft.Data.SqlClient;

class EtlCsvToSql
{
    static void Main()
    {
        string csv = Environment.GetEnvironmentVariable("CSV_PATH") ?? "products.csv";
        string connStr = Environment.GetEnvironmentVariable("SQL_CONN") ?? "Server=.;Database=Demo;Trusted_Connection=True;Encrypt=False";

        var table = new DataTable("Products");
        table.Columns.Add("Id", typeof(int));
        table.Columns.Add("Name", typeof(string));
        table.Columns.Add("Price", typeof(decimal));
        table.Columns.Add("CreatedUtc", typeof(DateTime));

        using var sr = new StreamReader(csv);
        _ = sr.ReadLine(); // تخطّي العنوان
        string? line;
        var inv = CultureInfo.InvariantCulture;

        while ((line = sr.ReadLine()) is not null)
        {
            var cols = line.Split(',');
            if (cols.Length < 4) continue;

            if (!int.TryParse(cols[0], out var id)) continue;
            var name = cols[1].Trim();
            if (string.IsNullOrWhiteSpace(name)) continue;
            if (!decimal.TryParse(cols[2], NumberStyles.Any, inv, out var price)) continue;
            if (!DateTime.TryParse(cols[3], CultureInfo.InvariantCulture, DateTimeStyles.AssumeUniversal, out var ts)) continue;

            table.Rows.Add(id, name, price, ts.ToUniversalTime());
        }

        using var con = new SqlConnection(connStr);
        con.Open();
        using var bulk = new SqlBulkCopy(con) { DestinationTableName = "dbo.Products" };
        bulk.WriteToServer(table);
        Console.WriteLine($"ETL loaded rows: {table.Rows.Count}");
    }
}
```

**جدول الوجهة (T-SQL):**
```sql
CREATE TABLE dbo.Products(
  Id         INT PRIMARY KEY,
  Name       NVARCHAR(100) NOT NULL,
  Price      DECIMAL(18,2) NOT NULL,
  CreatedUtc DATETIME2     NOT NULL
);
```

---

## مثال عملي مختصر — **ELT** (تحميل خام → T-SQL تحويل داخل المستودع)

> **الخطوة 1 (E+L)**: حمّل **كما هو** إلى جدول وسيط `RawProducts` ثم حوِّل داخل SQL.

```csharp
// نفس القراءة لكن إلى DataTable خام (NVARCHAR فقط)، ثم BulkCopy إلى dbo.RawProducts
var raw = new DataTable("RawProducts");
raw.Columns.Add("Id", typeof(string));
raw.Columns.Add("Name", typeof(string));
raw.Columns.Add("Price", typeof(string));
raw.Columns.Add("CreatedUtc", typeof(string));
// … املأ الصفوف مباشرة من CSV دون تحقق صارم …
using var bulk2 = new SqlBulkCopy(con) { DestinationTableName = "dbo.RawProducts" };
bulk2.WriteToServer(raw);
```

**جداول ELT ومُعالجة داخل SQL:**
```sql
-- جدول خام
CREATE TABLE dbo.RawProducts(
  Id         NVARCHAR(50),
  Name       NVARCHAR(200),
  Price      NVARCHAR(50),
  CreatedUtc NVARCHAR(50)
);

-- تحويل داخل المستودع (Stage -> Target) مع تنظيف وأنماط فشل ناعمة
MERGE dbo.Products AS T
USING (
  SELECT
    TRY_CONVERT(INT, Id)             AS Id,
    NULLIF(LTRIM(RTRIM(Name)), '')   AS Name,
    TRY_CONVERT(DECIMAL(18,2), Price) AS Price,
    TRY_CONVERT(DATETIME2, CreatedUtc) AS CreatedUtc
  FROM dbo.RawProducts
) AS S
ON T.Id = S.Id
WHEN MATCHED AND (T.Name <> S.Name OR T.Price <> S.Price OR T.CreatedUtc <> S.CreatedUtc)
  THEN UPDATE SET Name=S.Name, Price=S.Price, CreatedUtc=S.CreatedUtc
WHEN NOT MATCHED BY TARGET AND S.Id IS NOT NULL AND S.Name IS NOT NULL AND S.Price IS NOT NULL AND S.CreatedUtc IS NOT NULL
  THEN INSERT (Id,Name,Price,CreatedUtc) VALUES (S.Id,S.Name,S.Price,S.CreatedUtc)
WHEN NOT MATCHED BY SOURCE THEN /* احتفاظ اختياري */ DO NOTHING; -- أو منطق أرشفة
```

**الفكرة:** في **ELT** نُنجز التنظيف/الدمج داخل المستودع (أقوى/أسرع غالبًا)، ونحتفظ بالخام للأثر الرجعي.

---

## خطوات عملية لبناء بايبلاين متين
- حدّد **المصدر/الوجهة/السكيمة** و**مفتاح التطابق** (Natural/Surrogate Key).  
- اختر **دفعات/تدفق** (Batch/Streaming) وتواتر التحميل.  
- اجعل التحميل **Idempotent**: مفاتيح طلب/تجزئة محتوى/جداول Stage.  
- أضِف **تحقّق جودة**: عدّ الصفوف، حدود Null/نطاقات، إنذارات.  
- سجّل **الأنساب (Lineage)** والإصدارات (Schema Version).  
- للأداء: **Bulk Load**، تقسيم/فهرسة، و**Partitioning** للكبيرة.  
- للأمان: تشفير سر/قنوات، إخفاء/تقنيع بيانات حسّاسة.  
- للأتمتة: **Orchestrator** (Airflow/ADF/SSIS) + **dbt** لـ ELT داخل المستودع.

---

## أخطاء شائعة
- **إعادة تحميل كامل** دون حاجة ⇒ تكلفة/قفل.  
- تجاهل **المناطق الزمنية/التواريخ** ⇒ بيانات مغلوطة.  
- عدم التعامل مع **التكرارات** أو **المفاتيح المكرّرة**.  
- تعويل على CSV معقّد دون مكتبة Parsing مناسبة.  
- السكيمة تتغيّر بلا **Versioning/Migration**.  
- غياب **مراقبة/تنبيهات** للفشل أو انزياحات الحجم.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **ETL** | **تنظيف/تحويل قبل التحميل** | يقلّل “الخام” بالمخزن؛ جيّد للامتثال الصارم قبل التخزين |
| **ELT** | **تحميل الخام ثم التحويل داخل المستودع** | يستفيد من قوة SQL/DW؛ أسرع وصول؛ يحتفظ بالأثر الخام |
| Streaming ETL | قرب لحظي (Kafka/SSE/CDC) | نوافذ ووقت حدث/استلام؛ تعقيد ترتيب وIdempotency |
| Data Quality | اختبارات وثقة البيانات | Checks/SLAs/تنبيهات؛ أساسي لكلا النهجين |

---

## ملخص الفكرة  
**ETL** و**ELT** طريقان لنفس الهدف: **بيانات نظيفة جاهزة للاستهلاك**.  
اختر ETL عندما يلزم **تنظيف قبل التخزين**، وELT عندما تريد **سرعة التحميل** والاستفادة من قوة المستودع.  
أحكمهما بـ **Idempotency، مراقبة، اختبارات جودة، وإدارة سكيمة**—وستحصل على بايبلاين موثوق وقابل للتوسع.
