# **Consume API (Kafka)**

## الترجمة الحرفية  
**Consume API** — **واجهة “استهلاك/قراءة” الرسائل** من موضوعات (Topics) كافكا عبر **مجموعات مستهلكين**.

## الوصف العربي المختصر  
يقرأ رسائل من **Partitions**، يدير **Offsets** تلقائيًا أو يدويًا،  
يدعم **إعادة التوازن** (Rebalance)، و**عزل المعاملات** `read_committed` عند استخدام منتجين بمعاملات.

## الشرح المبسّط  
- **Consumer Group**: كل رسالة تُعالَج مرّة واحدة لكل مجموعة (تقسيم الأقسام على المستهلكين).  
- **Offset**: موقعك داخل القسم. إمّا **Auto-Commit** أو **Manual Commit** بعد نجاح المعالجة.  
- **الترتيب** مضمون **داخل القسم** فقط.  
- **Auto Offset Reset** عند عدم وجود Offset محفوظ: `earliest` أو `latest`.  
- **Isolation Level**: `read_committed` لتجاهل رسائل مُلغاة من منتجين بمعاملات.

## تشبيه  
طابور صناديق في مستودع (Partitions). فريق عمّال (Consumer Group) يقتسم الصناديق.  
كل عامل يتقدّم بمؤشّر (Offset). لا تُعلن انتهاء الصندوق حتى تُنهي فرزه (Manual Commit).

---

## مثال C# — مستهلك موثوق (Manual Commit + DLT اختياري + read_committed)

```csharp
// .NET 8/9 — dotnet add package Confluent.Kafka
using System;
using System.Threading;
using Confluent.Kafka;

class ConsumerDemo
{
    static void Main()
    {
        var topic = "orders";
        var dlt   = "orders.DLT"; // موضوع أخطاء اختياري (Dead-Letter Topic)

        var config = new ConsumerConfig
        {
            BootstrapServers = "localhost:9092",
            GroupId = "orders-workers",
            EnableAutoCommit = false,                   // سنعتمد على Manual Commit
            AutoOffsetReset = AutoOffsetReset.Earliest,
            // زمنيات مهمة:
            SessionTimeoutMs = 10000,                   // نبضات حياة
            MaxPollIntervalMs = 300000,                 // مهلة المعالجة قبل اعتبارك متجمّدًا
            // في حال استخدام Producer Transactions:
            IsolationLevel = IsolationLevel.ReadCommitted
            // إستراتيجية إعادة توازن افتراضية "cooperative-sticky" مدعومة في الوسطاء الحديثين
        };

        using var consumer = new ConsumerBuilder<string, string>(config)
            .SetErrorHandler((c, e) => Console.Error.WriteLine($"[ERR] {e.Reason}"))
            .SetPartitionsAssignedHandler((c, parts) =>
            {
                Console.WriteLine($"[ASSIGN] {string.Join(", ", parts)}");
                // يمكنك استدعاء c.Assign(parts) يدويًا لتخصيص مخصص
            })
            .SetPartitionsRevokedHandler((c, parts) =>
            {
                Console.WriteLine($"[REVOKE] {string.Join(", ", parts)}");
                // احرص على إنهاء المعالجة الحالية قبل الرفض
            })
            .Build();

        consumer.Subscribe(topic);

        using var cts = new CancellationTokenSource();
        Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

        // منتِج اختياري لإرسال الرسائل الفاسدة إلى DLT
        using var dltProducer = new ProducerBuilder<string, string>(new ProducerConfig { BootstrapServers = "localhost:9092", Acks = Acks.All }).Build();

        Console.WriteLine("Consuming… press Ctrl+C to stop.");
        try
        {
            while (!cts.IsCancellationRequested)
            {
                var cr = consumer.Consume(cts.Token); // يسحب دفعة واحدة
                try
                {
                    // === المعالجة الفعلية ===
                    Console.WriteLine($"[{cr.TopicPartitionOffset}] key={cr.Message.Key} val={cr.Message.Value}");

                    // مثال التحقق
                    if (string.IsNullOrWhiteSpace(cr.Message.Value))
                        throw new InvalidOperationException("empty_payload");

                    // TODO: نفّذ منطقك (DB/HTTP). اجعلها "idempotent" عبر مفتاح فريد.
                    // ========

                    consumer.Commit(cr); // أعلن النجاح (يتقدّم الـ Offset)
                }
                catch (Exception ex)
                {
                    Console.Error.WriteLine($"[PROCESS-ERR] {ex.Message}");

                    // سياسة معالجة الأخطاء:
                    // 1) أرسل إلى DLT ثم تخطَّ الرسالة:
                    try
                    {
                        dltProducer.Produce(dlt, new Message<string, string> {
                            Key = cr.Message.Key,
                            Value = cr.Message.Value,
                            Headers = new Headers {
                                { "origin-topic", System.Text.Encoding.UTF8.GetBytes(cr.Topic) },
                                { "origin-tpo",   System.Text.Encoding.UTF8.GetBytes(cr.TopicPartitionOffset.ToString()) },
                                { "error",        System.Text.Encoding.UTF8.GetBytes(ex.Message) }
                            }
                        });
                        dltProducer.Flush(TimeSpan.FromSeconds(5));
                        consumer.Commit(cr); // تخطّ الرسالة بعد إرسالها للـ DLT
                    }
                    catch (Exception dltEx)
                    {
                        Console.Error.WriteLine($"[DLT-ERR] {dltEx.Message}");
                        // 2) أو: لا تُكمِت → ستُعاد المعالجة بعد إعادة التوازن/إعادة التشغيل
                        // يمكن أيضًا Seek لتجاوزها يدويًا بعد تسجيلها.
                    }
                }
            }
        }
        catch (OperationCanceledException) { }
        finally
        {
            consumer.Close(); // يغلق بأمان ويُلتقط offsets الملتزم بها
        }
    }
}
```

**الفكرة:**  
- **Manual Commit** بعد نجاح المعالجة يحقق **At-Least-Once** بأمان.  
- لـ **Exactly-Once** من طرفك: اجعل المعالجة **Idempotent** (مثلاً مفتاح فريد + Upsert) أو نمط **Outbox**.  
- عند استخدام منتجين بمعاملات، اضبط `IsolationLevel=read_committed` لتجاهل ما أُلغي.

---

## خطوات عملية لضبط **Consume API** بذكاء
1. **Manual Commit** بعد نجاح العمل الحقيقي. لا تعتمد على `EnableAutoCommit` للمسارات الحرجة.  
2. اضبط **`max.poll.interval.ms`** وفق أسوأ زمن معالجة؛ وإلا سيعتبرك الكلاستر متوقّفًا ويعيد التوازن.  
3. اجعل المعالجة **Idempotent** (مفتاح عمل فريد، Upsert، أو Outbox) لتجنب التكرارات.  
4. عالج الأخطاء: **DLT** لرسائل “سمّية”، مع **قياسات** ونِسَب إعادة المحاولة.  
5. راقب **Lag** (الفارق بين آخر Offset وموضعك)، و**سجّل** `TopicPartitionOffset` في كل خطوة.  
6. اختر **`auto.offset.reset`** بعناية عند أول تشغيل (`earliest` لتاريخ قديم، `latest` للإحداثيات الأحدث فقط).  
7. لو كان المنتج **Transactional**: استخدم `read_committed`، وإلا `read_uncommitted` أسرع لكنه يرى الرسائل الملغاة.  
8. اضبط **التوازي** عبر زيادة **عدد الأقسام** أو عدد المستهلكين في المجموعة (مستهلك واحد لكل قسم كحد أقصى).

---

## أخطاء شائعة
- الاعتماد على **Auto-Commit** ثم فقدان رسائل عند الفشل بين المعالجة والالتزام.  
- افتراض **ترتيب عالمي** عبر الأقسام (الترتيب داخل القسم فقط).  
- مهلة معالجة أطول من `max.poll.interval.ms` → **Rebalance** مفاجئ وفقد تقدّم عمل لم يُكمَت.  
- تجاهل DLT أو تسجيل سبب الفشل → “رسائل عالقة” تتكرّر بلا نهاية.  
- عدم إغلاق المستهلك بـ `Close()` → ضياع آخر الالتزامات.  
- تفعيل `read_uncommitted` مع منتجين بمعاملات ثم الاستغراب من “اختفاء” رسائل (كانت مُلغاة).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Consume API** | **قراءة رسائل وإدارة Offsets ضمن مجموعة** | Manual Commit بعد نجاح المعالجة |
| [Produce-API](produce-api.md)| **نشر رسائل إلى Topic/Partition** | `acks`، **Key→Partition**، **Idempotence/Transactions** |

---

## ملخص الفكرة  
**Consume API** = قراءة منظّمة عبر **Consumer Groups** مع تحكّم دقيق في **Offsets**.  
اعتمد **Manual Commit**، اجعل المعالجة **Idempotent**، أضف **DLT**، واضبط **زمنيات/عزل** مناسب—  
تحصل على استهلاك **موثوق**، **قابل للتوسّع**، وبأقل فقد وتكرار ممكن. 
