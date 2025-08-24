# **Produce API (Kafka)**

## الترجمة الحرفية  
**Produce API** — **واجهة “إنتاج/إرسال” الرسائل** إلى موضوعات (Topics) كافكا.

## الوصف العربي المختصر  
طلب (Request) يضيف سجلات **Records** إلى **Partitions** داخل Topic.  
تتحكّم في **الاستلام** عبر `acks=0|1|all`، وتدعم **التجزئة بالمفتاح**، **الدمج/الضغط**،  
وخيارات موثوقية مثل **Idempotence** و**Transactions** (للـ Exactly-Once عبر موضوعات/أقسام).

## الشرح المبسّط  
- كل Topic مقسّم إلى **Partitions** (أشرطة نقل متوازية).  
- تضع رسالة تحمل **Key**؛ نفس المفتاح يذهب غالبًا لنفس القسم → **ترتيب** داخل القسم.  
- `acks` = إيصال الاستلام:  
  - `0` بلا إيصال (أسرع/أخطر)، `1` من القائد فقط، `all` من كل النسخ (أكثر أمانًا).  
- **Idempotent Producer** يمنع التكرار عند إعادة المحاولة.  
- **Transactions** تجعل مجموعة رسائل **تُكتب أو تُلغى معًا** (Atomic).

## تشبيه  
شركة شحن: تختار **سيرًا** (Partition) حسب **الملصق/المفتاح**.  
تطلب **إيصال استلام** من القائد فقط أو من جميع الفروع (acks).  
يمكنك شحن **دفعة واحدة** وتؤكدها بالكامل أو تلغيها (معاملة).

---

## مثال C# — منتِج Kafka بإعداد موثوقية (Idempotence + acks=all) ومعاملات

```csharp
// .NET 8/9  —  dotnet add package Confluent.Kafka
using System;
using System.Threading.Tasks;
using Confluent.Kafka;

class ProduceDemo
{
    static async Task Main()
    {
        var topic = "orders";
        var config = new ProducerConfig
        {
            BootstrapServers = "localhost:9092",
            Acks = Acks.All,                    // إيصال من كل النسخ
            EnableIdempotence = true,          // منع التكرار مع Retries
            // إعدادات أداء/زمن:
            LingerMs = 10,                     // دمج رسائل ضمن نافذة قصيرة
            BatchSize = 64 * 1024,             // حجم دفعة
            CompressionType = CompressionType.Lz4,
            MessageTimeoutMs = 30000           // مهلة التسليم
            // (القيم الافتراضية الحديثة تضبط max.in.flight/retries بما يناسب idempotence)
        };

        using var producer = new ProducerBuilder<string, string>(config).Build();

        // إرسال رسالة مفردة بمفتاح (يحافظ على ترتيب المفتاح داخل القسم)
        var dr = await producer.ProduceAsync(topic,
            new Message<string, string>
            {
                Key = "customer-42",
                Value = "{ \"orderId\": 1001, \"amount\": 120.0 }",
                Headers = new Headers { new Header("type", System.Text.Encoding.UTF8.GetBytes("created")) }
            });

        Console.WriteLine($"Delivered to {dr.TopicPartitionOffset} (partition={dr.Partition}, offset={dr.Offset})");

        // ====== معاملات (Exactly-Once عبر عدة موضوعات/أقسام) ======
        var txConfig = new ProducerConfig(config) { TransactionalId = "orders-tx-producer-1" };
        using var txProducer = new ProducerBuilder<string, string>(txConfig).Build();

        txProducer.InitTransactions(TimeSpan.FromSeconds(10));
        try
        {
            txProducer.BeginTransaction();

            txProducer.Produce(topic, new Message<string, string> { Key="customer-42", Value="{\"orderId\":1002,\"amount\":75.0}" });
            txProducer.Produce("audit", new Message<string, string> { Key="order-1002", Value="{\"action\":\"created\"}" });

            // يمكنك أيضًا إرسال offsets المستهلكة لتحقيق Exactly-Once end-to-end
            // txProducer.SendOffsetsToTransaction(offsets, consumerGroupMetadata, TimeSpan.FromSeconds(5));

            txProducer.CommitTransaction();  // إمّا الكل أو لا شيء
        }
        catch (ProduceException<string, string> ex)
        {
            Console.Error.WriteLine($"Produce error: {ex.Error.Reason}");
            txProducer.AbortTransaction();   // إلغاء الدفعة بالكامل
        }
    }
}
```

**الفكرة:**  
- `EnableIdempotence=true` + `Acks=All` = أمان أعلى ضد فقد/تكرار الرسائل.  
- **Transactions** تجعل إرسال عدة رسائل/مواضيع **ذرّيًا**.

---

## خطوات عملية لضبط **Produce API** بذكاء
1. **استراتيجية المفتاح**: اختر `Key` يحافظ على محليّة البيانات (نفس العميل = نفس القسم) لتضمن **ترتيبًا** مفيدًا.  
2. **الأمان مقابل الزمن**:  
   - استخدم `acks=all` للإنتاج الحرج.  
   - فعّل **Idempotence** مع **Retries** تلقائيًا.  
3. **المعاملات** عند الحاجة إلى **اتساق عابر للمواضيع/الأقسام** أو مع القراءة (EOS).  
4. **الضبط للأداء**: `linger.ms` و`batch.size` و`compression` لرفع Throughput، مع مراقبة **الزمن**.  
5. **المراقبة**: اعتمد على **تقارير التسليم** (Delivery Reports) وسجّل الأخطاء/الأسباب.  
6. **الحجم والحدود**: انتبه لـ `message.max.bytes` على الوسيط والعميل؛ قسّم الرسائل الكبيرة (Chunking) أو خزن خارجًا.  
7. **التسلسل/المخططات**: وحّد شكل الرسائل (Avro/JSON/Protobuf) مع **Schema Registry** لتطوّر آمن.

---

## أخطاء شائعة
- إرسال بـ `acks=0` أو `1` لتطبيقات حرجة → احتمال فقد.  
- إهمال **Idempotence** مع **Retries** → رسائل مكرّرة.  
- الاعتماد على ترتيب **عبر أقسام مختلفة** (الترتيب مضمون داخل القسم فقط).  
- الانتظار المتزامن لكل `ProduceAsync` → Throughput ضعيف؛ استخدم **الدمج** و**Callbacks**.  
- رسائل ضخمة تتجاوز الحد → فشل تسليم متكرر.  
- استخدام **Transactions** دون ضبط المستهلك على `isolation.level=read_committed`.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Produce API (Kafka)** | **نشر رسائل إلى Topic/Partition** | `acks`، **Key→Partition**، **Idempotence/Transactions** |
| [Consume-API](consume-api.md) | استهلاك الرسائل مع إدارة Offsets | مجموعات مستهلكين، إعادة التوازن |


---

## ملخص الفكرة  
**Produce API** هو طريقك لـ **إلحاق رسائل** بمواضيع كافكا مع ضبط دقيق للموثوقية والأداء.  
اختر **مفتاحًا** مناسبًا، فعّل **acks=all + idempotence**، واستخدم **Transactions** عند الحاجة—  
تحصل على إرسال **متين**، **مرن**، ويوازن بين **الزمن** و**السلامة** وفق احتياج منتجك.
