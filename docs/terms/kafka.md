# **Kafka**

## الترجمة الحرفية  
**Apache Kafka** — **نظام سجّل (Log) موزّع لبثّ البيانات والرسائل** بزمن شبه لحظي.

## الوصف العربي المختصر  
Kafka يخزّن الأحداث في **Topics** مقسّمة إلى **Partitions** مرتّبة.  
منتجون **يكتبون**، مستهلكون **يقرؤون** ضمن **مجموعات** مع تحمّل أعطال وتوسّع أفقي.  
يدعم **الاحتفاظ** (Retention)، **ضغط السجل** (Compaction)، و**Semantics**: At-least-once، Idempotent، وExactly-once بالمعاملات.

## الشرح المبسّط  
- **Topic** = ملفّ منطقي للأحداث.  
- **Partition** = تسلسل مرتّب فقط داخله؛ يحدّد التوازي.  
- **Key** يوجّه الحدث لقسم محدّد → يحافظ على ترتيب هذا المفتاح.  
- **Replication** لحماية البيانات (عامل تكرار `RF` + مجموعة ISR).  
- **Brokers** (عُقد) + **Controller/KRaft** لإدارة الميتاداتا والقيادة.  
- **Consumer Groups**: كل رسالة تُعالج مرّة واحدة لكل مجموعة.  
- **Retention** بالوقت/الحجم، و**Compaction** يبقي **آخر قيمة لكل مفتاح**.

## تشبيه  
“شريط تسجيل” ضخم موزّع: الكتابة **تلصق** في النهاية، القراءة تتحرّك بمؤشّر (**Offset**).  
كل مفتاح (عميل 42) يسير على مساره الخاصّ (قسم) بترتيب محفوظ.

---

## مثال C# عملي — منتِج + مستهلك (Confluent.Kafka) بإعدادات موثوقية

```csharp
// .NET 8/9 — dotnet add package Confluent.Kafka
using System;
using System.Threading;
using System.Threading.Tasks;
using Confluent.Kafka;

class KafkaQuick
{
    static async Task Main()
    {
        var topic = "orders";

        // ===== Producer: acks=all + Idempotence =====
        var pConfig = new ProducerConfig
        {
            BootstrapServers = "localhost:9092",
            Acks = Acks.All,
            EnableIdempotence = true,
            LingerMs = 5,                  // دمج رسائل قصيرة
            CompressionType = CompressionType.Lz4
        };

        using var producer = new ProducerBuilder<string, string>(pConfig).Build();

        // إرسال غير متزامن مع مفتاح للحفاظ على ترتيب العميل
        for (int i = 1; i <= 3; i++)
        {
            var key = "customer-42";
            var val = $"{{\"orderId\":{1000+i},\"amount\":{i*50}}}";
            var dr = await producer.ProduceAsync(topic, new Message<string,string> { Key = key, Value = val });
            Console.WriteLine($"Produced → {dr.TopicPartitionOffset}");
        }

        // ===== Consumer: Manual commit + read_committed =====
        var cConfig = new ConsumerConfig
        {
            BootstrapServers = "localhost:9092",
            GroupId = "orders-workers",
            EnableAutoCommit = false,                  // نكمّت يدويًا بعد نجاح المعالجة
            AutoOffsetReset = AutoOffsetReset.Earliest,
            IsolationLevel = IsolationLevel.ReadCommitted
        };

        using var consumer = new ConsumerBuilder<string, string>(cConfig).Build();
        consumer.Subscribe(topic);

        Console.WriteLine("Consuming… (Ctrl+C to stop)");
        using var cts = new CancellationTokenSource();
        Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

        try
        {
            while (!cts.IsCancellationRequested)
            {
                var cr = consumer.Consume(cts.Token);
                Console.WriteLine($"Consumed [{cr.TopicPartitionOffset}] key={cr.Message.Key} val={cr.Message.Value}");

                // TODO: منطق معالجة Idempotent (مثلاً Upsert حسب orderId)
                consumer.Commit(cr); // إعلان النجاح
            }
        }
        catch (OperationCanceledException) { }
        finally { consumer.Close(); }
    }
}
```

**الفكرة:**  
- المنتج: `acks=all` + **Idempotence** للوقاية من فقد/تكرار.  
- المستهلك: **Manual Commit** بعد نجاح المعالجة لتحقيق **At-least-once** بأمان.

---

## خطوات عملية لتصميم Kafka سليم
1. **نَمذِج الـ Topics/Keys**: المفتاح يحدّد القسم → ترتيب لكل مفتاح.  
2. اختر **عدد الأقسام** وفق التوازي المطلوب، و**RF ≥ 3** للحرج.  
3. المنتج: `acks=all`، **Idempotence**، و**Transactions** عند الحاجة لتناسق عبر مواضيع/أقسام.  
4. المستهلك: **Manual Commit**، واضبط `max.poll.interval.ms` حسب زمن العمل الفعلي.  
5. **Retention**: بالوقت/الحجم؛ فعّل **Compaction** لموضوع “حالة” (آخر نسخة لكل مفتاح).  
6. **Schema Registry** (Avro/Protobuf/JSON Schema) لتطوّر مخططات آمن.  
7. **مراقبة**: Lag، Throughput، أخطاء إنتاج/استهلاك، ISR، واستخدام القرص.  
8. **أمن**: TLS/mTLS، SASL (OAuth/OIDC أو SCRAM)، وACLs على المواضيع.

---

## أخطاء شائعة
- توقّع **ترتيب عالمي** عبر الأقسام (الترتيب داخل القسم فقط).  
- `acks=0` أو `1` لتطبيقات حرجة → خطر فقد.  
- نسيان **Idempotence/Retries** في المنتج → تكرارات عند إعادة المحاولة.  
- Auto-Commit في مسارات حسّاسة → احتمال فقد/تكرار.  
- رسائل ضخمة تتجاوز `message.max.bytes` → فشل تسليم.  
- أقسام قليلة مع حمولة عالية → اختناق، أو كثيرة بلا حاجة → إدارة أصعب.

---

## أوامر سريعة (مرجع)
```bash
# إنشاء موضوع بـ 6 أقسام وRF=3
kafka-topics.sh --create --topic orders --partitions 6 --replication-factor 3 --bootstrap-server localhost:9092

# وصف موضوع ومراقبة
kafka-topics.sh --describe --topic orders --bootstrap-server localhost:9092

# منتِج/مستهلك سطر أوامر للتجارب
kafka-console-producer.sh --topic orders --bootstrap-server localhost:9092
kafka-console-consumer.sh --topic orders --from-beginning --bootstrap-server localhost:9092
```

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Kafka** | **Log موزّع للأحداث** مع احتفاظ وتوسّع أفقي | أقسام/تكرار/Groups، ترتيب داخل القسم |
| RabbitMQ (AMQP) | وسيط رسائل موجّه مسارات/طوابير | تأكيدات دقيقة ومسارات؛ أقل ملاءمة لبثّ كثيف طويل الأمد |
| Apache Pulsar | مواضيع/مشتركين + تخزين منفصل | تخزين BookKeeper، Functions مدمجة |
| AWS Kinesis | خدمة مُدارة شبيهة بالتدفّق | قيود شَرد/تكلفة؛ تكامل AWS |
| Redis Streams | تيارات داخل Redis | ملائم لوَرْكفْلو خفيف؛ ليس بديلاً كاملًا لـ Kafka |

---

## ملخص الفكرة  
**Kafka** = “سجل أحداث موزّع” عالي الاعتمادية والتوسّع.  
اختر **المفاتيح/الأقسام** بعناية، فعّل **acks=all + idempotence**، واستخدم **Manual Commit** (والمعاملات عند اللزوم)—  
تحصل على خطوط بيانات **سريعة**، **متينة**، ومهيّأة للنمو والتكامل. 
