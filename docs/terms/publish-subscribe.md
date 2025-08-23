# **Publish–Subscribe**

## الترجمة الحرفية  
**Publish–Subscribe (Pub/Sub)** — **نمط النشر والاشتراك**.

## الوصف العربي المختصر  
يرسل **الناشر** رسائل إلى **مواضيع/قنوات** دون معرفة المستلمين.  
يتلقّى **المشتركون** الرسائل وفق اشتراكاتهم، غالبًا **بشكل غير متزامن** عبر وسيط (Broker).

## الشرح المبسّط  
- **Topics/Channels**: مسارات منطقية للرسائل (مثل: `orders.created`).  
- **Publisher**: يرسل الحدث للوسيط، ثم ينسى. لا يحتاج عنوان المستلم.  
- **Subscriber**: يسجّل على موضوع واحد أو أكثر ويتلقى ما يُنشر.  
- **وسيط (Broker)**: يوزّع الرسائل، وقد يقدّم تخزينًا، إعادة المحاولة، وقوائم ميتة (DLQ).  
- **فك الارتباط**: المنتجون والمستهلكون مستقلّون زمنيًا ومكانيًا. مناسب للتوسّع والمرونة.

## تشبيه  
مثل **محطة إذاعية**: المذيع ينشر على موجة، وكل جهاز مضبوط على التردد المناسب يسمع.  
المذيع لا يعرف عدد أو أماكن المستمعين.

## مثال كود C# بسيط (Broker ذاكرة صغير)
```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;

public class InMemoryBroker
{
    private readonly ConcurrentDictionary<string, List<Action<string>>> _subs = new();

    public IDisposable Subscribe(string topic, Action<string> handler)
    {
        var list = _subs.GetOrAdd(topic, _ => new List<Action<string>>());
        lock (list) list.Add(handler);

        return new Unsubscriber(() =>
        {
            lock (list) list.Remove(handler);
        });
    }

    public void Publish(string topic, string message)
    {
        if (_subs.TryGetValue(topic, out var list))
        {
            Action<string>[] handlers;
            lock (list) handlers = list.ToArray();
            foreach (var h in handlers) h(message); // تبسيط: بدون ضمانات تسليم/تكرار
        }
    }

    private sealed class Unsubscriber : IDisposable
    {
        private readonly Action _dispose;
        public Unsubscriber(Action dispose) => _dispose = dispose;
        public void Dispose() => _dispose();
    }
}

class Program
{
    static void Main()
    {
        var broker = new InMemoryBroker();

        using var sub1 = broker.Subscribe("orders.created", m => Console.WriteLine($"[A] {m}"));
        using var sub2 = broker.Subscribe("orders.created", m => Console.WriteLine($"[B] {m}"));

        broker.Publish("orders.created", "{ \"orderId\": 123 }");
        // الإخراج المتوقع:
        // [A] { "orderId": 123 }
        // [B] { "orderId": 123 }
    }
}
```
> ملاحظة: الوسطاء الحقيقيون (Kafka/RabbitMQ/Redis/Service Bus) يوفّرون متانة، تقسيمًا (Partitions)، إعادة المحاولة، وترتيبًا لكل قسم.

## خطوات عملية لتصميم Pub/Sub قوي
- **عرّف الأحداث والمواضيع** وأسماء ثابتة (مع إصدار عند تغييرات غير متوافقة).  
- **حدّد مخطط الرسالة** (JSON/Avro/Protobuf) مع قواعد تطوّر المخطط (Schema Evolution).  
- اختر وسيطًا مناسبًا:
  - **Kafka**: تدفّق عالي، الاحتفاظ طويل، تقسيم قوي.  
  - **RabbitMQ**: أنماط توجيه مرنة، صفوف، تبادل رسائل.  
  - **Redis Pub/Sub**: خفيف وفوري (بدون احتفاظ افتراضيًا).  
  - **Service Bus/Topic** (سحابي): إدارة ورسائل متينة + DLQ.  
- **دلالات التسليم**: على الأقل مرة (At-least-once) مع **Idempotency** في المستهلك.  
- **الملاحظة (Observability)**: سجّل معرف الرسالة، تتبّع (Tracing)، مقا
