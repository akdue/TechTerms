# **Stored Procedure**

## الترجمة الحرفية  
**Stored Procedure** — **إجراء مُخزَّن** (روتين SQL محفوظ داخل قاعدة البيانات).

## الوصف العربي المختصر  
كتلة أوامر **SQL** يتم حفظها وتشغيلها باسم ثابت وبمعاملات.  
تفيد في **تقليل تنقّل الشبكة**، **إعادة الاستخدام**، **الأمن** (صلاحية `EXECUTE`)، و**الأداء** (خطط تنفيذ مُخزَّنة).

## الشرح المبسّط  
- تفكّر بها كـ **دالة** على مستوى قاعدة البيانات، لكن يمكنها **تعديل البيانات** وتشغيل **معاملات**.  
- تستدعيها بـ `EXEC` وتمرير **معاملات** (Input/Output).  
- قاعدة البيانات تدير **الخطة**، **الأذونات**، و**المعاملة** داخليًا.

## تشبيه  
“إجراء إداري” جاهز في خزنة الشركة: تُسلّم الطلب مع **نموذج** (المعاملات) ويُنفَّذ السيناريو الكامل بسرعة وبصلاحيات مضبوطة.

---

## مثال SQL (T-SQL) — إنشاء، استدعاء، ومعالجة أخطاء

```sql
-- 1) إجراء قراءة: إرجاع إجمالي طلب
CREATE OR ALTER PROCEDURE dbo.GetOrderTotal
    @OrderId INT,
    @Total DECIMAL(18,2) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT @Total = SUM(Price * Qty)
    FROM dbo.OrderLines
    WHERE OrderId = @OrderId;

    IF (@Total IS NULL) SET @Total = 0;
END
GO

-- 2) إجراء كتابة بمعاملة + معالجة خطأ
CREATE OR ALTER PROCEDURE dbo.PlaceOrder
    @OrderId  INT,
    @Customer NVARCHAR(64)
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRAN;

        INSERT INTO dbo.Orders(Id, CustomerId) VALUES(@OrderId, @Customer);

        -- مثال: عناصر مُحضّرة مسبقًا في جدول مؤقت #Items
        INSERT INTO dbo.OrderLines(OrderId, Sku, Qty, Price)
        SELECT @OrderId, Sku, Qty, Price FROM #Items;

        COMMIT;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK;
        THROW; -- يرفع الخطأ للمستدعي مع رقم/حالة واضحين
    END CATCH
END
GO

-- استدعاء وإخراج قيمة
DECLARE @t DECIMAL(18,2);
EXEC dbo.GetOrderTotal @OrderId = 1001, @Total = @t OUTPUT;
SELECT Total = @t;
```

---

## مثال C# — استدعاء إجراء مخزّن عبر `SqlClient` (مدخلات/مخرجات)

```csharp
// .NET 8/9 — dotnet add package Microsoft.Data.SqlClient
using Microsoft.Data.SqlClient;
using System.Data;

var cs = "Server=localhost;Database=Shop;Trusted_Connection=True;TrustServerCertificate=True";

// مثال: قراءة مخرَج OUTPUT
using (var cn = new SqlConnection(cs))
{
    await cn.OpenAsync();

    using var cmd = new SqlCommand("dbo.GetOrderTotal", cn) { CommandType = CommandType.StoredProcedure };
    cmd.Parameters.Add(new SqlParameter("@OrderId", SqlDbType.Int) { Value = 1001 });
    var outParam = new SqlParameter("@Total", SqlDbType.Decimal)
    {
        Precision = 18, Scale = 2, Direction = ParameterDirection.Output
    };
    cmd.Parameters.Add(outParam);

    await cmd.ExecuteNonQueryAsync();
    Console.WriteLine($"Total = {(decimal)outParam.Value:0.00}");
}

// مثال: كتابة داخل معاملة تطبيقية تستدعي SP
using (var cn = new SqlConnection(cs))
{
    await cn.OpenAsync();
    using var tx = cn.BeginTransaction();

    try
    {
        using var cmd = new SqlCommand("dbo.PlaceOrder", cn, tx) { CommandType = CommandType.StoredProcedure };
        cmd.Parameters.AddWithValue("@OrderId", 1002);
        cmd.Parameters.AddWithValue("@Customer", "abdalkreem");

        await cmd.ExecuteNonQueryAsync();
        await tx.CommitAsync();
    }
    catch
    {
        await tx.RollbackAsync();
        throw;
    }
}
```

> **ملحوظة:** يمكن أيضًا استدعاؤها من **EF Core** (للتحديثات عبر `ExecuteSqlRaw` أو للقراءة عبر `FromSql`).

---

## خطوات عملية لاستخدام الإجراءات المخزّنة بذكاء
1. **سمِّها بوضوح** وباستخدم **Schema** (`sales.PlaceOrder`)، وتجنّب `sp_` في SQL Server.  
2. مرّر **معاملات Strongly-Typed**، واضبط `Precision/Scale` للأنواع العشرية.  
3. **اجعلها قصيرة ومحدّدة الهدف**؛ تجنّب “إجراءات عملاقة”.  
4. أضِف `SET NOCOUNT ON` لتقليل رسائل الصفوف المتأثرة.  
5. **تحكّم بالأذونات**: امنح `EXECUTE` على الإجراء بدل CRUD مباشر على الجداول.  
6. فعّل **معالجة الأخطاء** (`TRY...CATCH` + `THROW`) وحدود المعاملة داخل الإجراء عند الحاجة.  
7. راقب **الخطط** والإحصاءات، وحدّثها عند تغيّر البيانات.  
8. وثِّق **العقد**: الأسماء، الأنواع، المعاني، وسيناريوهات الأخطاء.

---

## أخطاء شائعة
- وضع **قواعد عمل معقدة** بالكامل في الـ DB → صعوبة اختبار/إصدار.  
- بناء **SQL ديناميكي** غير مُعلّق ⇒ قابل لحقن SQL (استخدم معاملات/قوالب آمنة).  
- نسيان `SET NOCOUNT ON` → التباس لدى العملاء الذين يعتمدون على `RecordsAffected`.  
- تجاهل **أكواد الأخطاء** ورفع نصوص عامة فقط.  
- جعل الإجراء يعتمد على **حالة جلسة** (Temp مفتوحة بلا توثيق/تنظيف).  
- تماثل أسماء ومعاني المعاملات غير واضح ⇒ واجهة هشّة.  
- نقل كل شيء إلى الإجراءات “لتحسين الأداء” دون قياس فعلي.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Stored Procedure** | **تنفيذ حزم SQL باسم وثابت** | يمكن أن تُحدِث وتقرأ، تدير معاملات، وأذونات `EXECUTE` |
| Function (Scalar/TVF) | حساب/إرجاع قيمة/مجموعة | يُفترض أن تكون **خالصة** بدون آثار جانبية (للأداء/الفهرسة) |
| Ad-hoc SQL | استعلام مباشر من التطبيق | مرن؛ لكن أذونات/أداء/أمان أقل إن أسيء استخدامه |
| ORM (EF Core) | خرائط كائن-علاقة وانتاجية | يمكنه استدعاء SP؛ مناسب للكود القابل للاختبار |
| Jobs/Triggers | تشغيل تلقائي داخل DB | تجنّب قواعد عمل ثقيلة فيها؛ استخدم بحذر |

---

## ملخص الفكرة  
**الإجراء المُخزَّن** = روتين **SQL** مُعاد الاستخدام داخل قاعدة البيانات.  
يؤمّن **أداء** (خطة محفوظة)، **أمن** (أذونات تنفيذ)، و**اتساق** (معاملات/أخطاء مُدارة).  
استخدمه لمهام DB واضحة، وبقواعد عمل محدودة، ومع **توثيقٍ** ومعاملاتٍ آمنة — لتحصل على طبقة بيانات **متينة** وقابلة للصيانة.

> ملاحظة: إن كنت تقصد **Prosegur** (شركة أمن إسبانية) وليس **Stored Procedure**، أخبرني لأشرحها بالصيغة نفسها. 
