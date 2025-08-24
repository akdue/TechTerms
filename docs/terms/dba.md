# **DBA — Database Administrator**

## الترجمة الحرفية  
**Database Administrator (DBA)** — **مسؤول قاعدة البيانات**.

## الوصف العربي المختصر  
شخص/دور مسؤول عن **تثبيت** قواعد البيانات، **أمنها**، **النسخ الاحتياطي/الاسترجاع**،  
**الضبط والأداء**، و**التوافر العالي**. يراقب ويؤتمت ويخطّط للسعة ويعالج الأعطال.

## الشرح المبسّط  
- **التوافر**: تجمّعات/نسخ، مراقبة، تنبيهات.  
- **الحماية**: صلاحيات دقيقة، تشفير، تدقيق.  
- **السلامة**: نسخ احتياطي **مُجرّب الاسترجاع**.  
- **الأداء**: فهارس، إحصاءات، ضبط استعلامات.  
- **التغييرات**: ترحيل مخططات بآمان (Migrations).  
- **السعة**: تخطيط مساحة وIOPS.  
- **الأتمتة**: مهام مجدولة وتقارير صحّة.

## تشبيه  
أمين مكتبة + حارس أمن + ميكانيكي:  
يحافظ على النظام، يمنع الدخول غير المصرّح، ويُصلّح/يضبط الآلة لتبقى سريعة.

---

## مثال كود C# — سكربت فحص سريع (نسخ احتياطي حديث + طلبات طويلة)

```csharp
// .NET 8/9 — dotnet add package Microsoft.Data.SqlClient
using System;
using System.Data;
using Microsoft.Data.SqlClient;

class DbaQuickCheck
{
    static void Main()
    {
        var connStr = "Server=localhost;Database=master;Trusted_Connection=True;TrustServerCertificate=True";
        using var cn = new SqlConnection(connStr);
        cn.Open();

        // 1) آخر نسخ احتياطي كامل لكل قاعدة (يوم/ساعة مضت)
        using (var cmd = cn.CreateCommand())
        {
            cmd.CommandText = @"
SELECT d.name AS [db],
       DATEDIFF(hour, MAX(b.backup_finish_date), SYSUTCDATETIME()) AS hours_since_full
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset b ON b.database_name = d.name AND b.type = 'D'
GROUP BY d.name
ORDER BY hours_since_full DESC";
            using var r = cmd.ExecuteReader();
            Console.WriteLine("== Backup age (hours) ==");
            while (r.Read())
                Console.WriteLine($"{r["db"],-20}  {r["hours_since_full"],5}");
        }

        // 2) أعلى 5 طلبات جارية زمنًا
        using (var cmd = cn.CreateCommand())
        {
            cmd.CommandText = @"
SELECT TOP 5 r.session_id, r.status, r.command, r.wait_type, r.wait_time,
       r.cpu_time, r.total_elapsed_time, SUBSTRING(t.text,1,200) AS sql_text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id <> @@SPID
ORDER BY r.total_elapsed_time DESC";
            using var r = cmd.ExecuteReader();
            Console.WriteLine("\n== Long running requests ==");
            while (r.Read())
                Console.WriteLine($"sid={r["session_id"]}  elapsed={r["total_elapsed_time"]}ms  sql={r["sql_text"]}");
        }
    }
}
```

> ماذا يفيد؟  
> - يُنذر لو تأخّر نسخ احتياطي.  
> - يُظهر الطلبات البطيئة الآن لتبدأ التشخيص.

---

## أمثلة T-SQL سريعة يستخدمها الـDBA

```sql
-- نسخة احتياطية كاملة (SQL Server)
BACKUP DATABASE MyDb TO DISK = 'E:\backups\MyDb_full.bak' WITH INIT, COMPRESSION, CHECKSUM;

-- استرجاع (اختبره دوريًا!)
RESTORE VERIFYONLY FROM DISK = 'E:\backups\MyDb_full.bak';
-- RESTORE DATABASE MyDb FROM DISK = '...' WITH MOVE ..., REPLACE;

-- إحصاءات واستيفاء الفهارس (تحسين الأداء)
UPDATE STATISTICS MyDb WITH FULLSCAN;
ALTER INDEX ALL ON dbo.Orders REBUILD WITH (ONLINE = ON);

-- أقل صلاحية لازمة (مبدأ الأقل امتيازًا)
CREATE USER appUser FOR LOGIN appLogin;
EXEC sp_addrolemember 'db_datareader', 'appUser';
EXEC sp_addrolemember 'db_datawriter', 'appUser';
```

---

## خطوات عملية يقوم بها الـDBA
1. **الأساسيات**: تثبيت/ترقية محكومة، إعداد التخزين والـTempDB، والإعدادات العامة.  
2. **الأمن**: مصادقة، أدوار، تشفير (TDE/At-Rest)، أسرار في Vault.  
3. **النسخ الاحتياطي/الاسترجاع**: سياسة `Full + Diff + Log`، وجدولة، **اختبارات استرجاع**.  
4. **المراقبة**: مؤشرات (CPU/IO/Latch/Waits/Lag)، وتتبع استعلامات بطيئة.  
5. **الأداء**: تصميم فهارس، تحديث إحصاءات، مراجعة خطط التنفيذ، حصر N+1.  
6. **التوافر العالي**: Always On / Replication / Failover / Backups جغرافية.  
7. **التغييرات**: مسار ترحيل منظم (Migrations)، مراجعة تأثير، نافذة صيانة.  
8. **التوثيق**: خرائط قواعد، سياسات، إجراءات أعطال، RPO/RTO.

---

## أخطاء شائعة
- نسخ احتياطي **بدون** اختبار استرجاع.  
- حسابات تطبيق بصلاحيات **sysadmin**.  
- فهارس كثيرة/قليلة بلا قياس؛ إحصاءات قديمة.  
- تعديل في الإنتاج دون نافذة/خطة رجوع.  
- تجاهل مراقبة **Waits/IO** والتركيز فقط على CPU.  
- الاعتماد على أدوات GUI فقط؛ غياب سكربتات قابلة للإعادة.  
- خلط مسؤوليات DBA مع تطبيق قواعد العمل داخل DB (Triggers معقدة بلا داعٍ).

---

## جدول مقارنة مختصر

| المفهوم           | الغرض الرئيسي                          | ملاحظات مهمة                      |
| ----------------- | -------------------------------------- | --------------------------------- |
| **DBA**           | **تشغيل/أمن/توفر/أداء قواعد البيانات** | نسخ احتياطي، مراقبة، ضبط، استرجاع |
| Database Engineer | تصميم/بناء حلول DB وأتمتة              | قريب من التطوير/الأدوات           |
| Data Engineer     | خطوط بيانات وتحويلات/ETL               | يستهلك DBs؛ ليس مشغّلها الأساسي    |
| SysAdmin/DevOps   | بنية تحتية/نشر عام                     | يتعاون مع DBA لتوفير المنصّة       |
| SRE               | موثوقية/قابلية ملاحظة                  | يقيس SLO/SLI مع فرق الـDB         |

---

## ملخص الفكرة  
**DBA** يحافظ على قاعدة بياناتك **آمنة، متاحة، وسريعة**.  
يؤتمت النسخ الاحتياطي و**يختبر الاسترجاع**، يراقب، يضبط الفهارس والاستعلامات،  
ويُدير الصلاحيات والتوافر—لتحصل على بيانات **موثوقة** تدعم العمل دون انقطاع. 
