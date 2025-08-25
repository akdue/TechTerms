# **Lambda Expiration**

## الترجمة الحرفية  
**Lambda Expiration** — **انتهاء وقت التنفيذ** أو **انقضاء صلاحية الحدث/الروابط** في AWS Lambda.

## الوصف العربي المختصر  
في AWS يُقصد بـ “Lambda Expiration” عادةً:  
1) **Timeout**: الحدّ الأقصى لزمن تنفيذ الدالة (حتى **15 دقيقة**).  
2) **Maximum Event Age**: العمر الأقصى للحدث في طابور الاستدعاء غير المتزامن (حتى **6 ساعات**).  
3) **SQS Visibility Timeout** المرتبط بالمعالجة.  
4) صلاحية عناصر تُنشئها الدالة مثل **Presigned URLs**. :contentReference[oaicite:0]{index=0}

## الشرح المبسّط  
- **Function Timeout**: إن لم تُنهِ الدالة خلال المهلة تُنهى بـ **Timeout**. الحد الأقصى 900 ثانية. :contentReference[oaicite:1]{index=1}  
- **Async Invocations**: تضع Lambda الأحداث في طابور داخلي. لو فشلت/تأخرت: تُعاد المحاولة حتى حد **المرّات** وحد **عمر الحدث** (MaximumEventAgeInSeconds). عند تجاوزهما تُرسل للـ **DLQ/Destination**. أقصى عمر 21600 ثانية (6 ساعات). :contentReference[oaicite:2]{index=2}  
- **SQS Triggers**: اضبط **Visibility Timeout** ≥ **6×** مهلة الدالة لتسمح بإعادات المحاولة عند الخنق/التباطؤ. :contentReference[oaicite:3]{index=3}  
- **Presigned URLs**: صلاحيتها تحددها أنت عند إنشائها. إن استُخدمت **اعتمادات مؤقتة** (Roles) قد تُقيَّد بحد جلسة (كثيرًا ≈ ساعة). :contentReference[oaicite:4]{index=4}

## تشبيه  
“بيتزا في الفرن”:  
- **Timeout** = أقصى وقت للخبز قبل أن تُخرجها المنظومة.  
- **Event Age** = أقصى زمن مسموح للطلب في طابور الأفران قبل التخلص منه.  
- **SQS Visibility** = مدة إخفاء الطلب عن طابور الانتظار أثناء خبزه.  
- **Presigned URL** = كوبون خصم له **تاريخ انتهاء**.

---

## مثال C# — Lambda تُنهي العمل بأمان قبل الـ **Timeout** وتراعي الإلغاء

```csharp
// .NET 8 — Amazon.Lambda.AspNetCoreServer أو Minimal Lambda
// dotnet add package Amazon.Lambda.Core
using Amazon.Lambda.Core;
using System.Net.Http;

public class Function
{
    private static readonly HttpClient _http = new() { Timeout = TimeSpan.FromSeconds(5) };

    // handler: يعتمد توقيع مشروعك (API Gateway/Function URL/Custom)
    public async Task<string> FunctionHandler(object input, ILambdaContext ctx, CancellationToken ct)
    {
        // كم تبقّى من المهلة؟
        var remaining = ctx.RemainingTime; // مثال: 00:00:07.123

        // إن تبقى أقل من هامش أمان، أعدّ ردًا مبكرًا بدل أن تتسبب بـ Timeout
        if (remaining < TimeSpan.FromSeconds(3))
            return "will-timeout-soon";

        using var linked = CancellationTokenSource.CreateLinkedTokenSource(ct);
        // هامش أمان 2 ث قبل نهاية Lambda
        linked.CancelAfter(remaining - TimeSpan.FromSeconds(2));

        // مثال: نداء خارجي سريع
        var res = await _http.GetAsync("https://example.com/health", linked.Token);
        return res.IsSuccessStatusCode ? "ok" : "upstream-error";
    }
}
```

**الفكرة:** استخدم `ctx.RemainingTime` + `CancellationToken` لتتجنّب **Timeout** غير المتحكَّم فيه.

---

## مثال إعداد — تحديد **Maximum Event Age** و**Retries** (CloudFormation/SAM)

```yaml
Resources:
  MyFunc:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: dotnet8
      Handler: Function::Function.FunctionHandler
      Role: arn:aws:iam::123456789012:role/my-func-role
      Timeout: 60             # 60s (الحد الأقصى 900s)
  MyFuncInvokeCfg:
    Type: AWS::Lambda::EventInvokeConfig
    Properties:
      FunctionName: !Ref MyFunc
      Qualifier: "$LATEST"
      MaximumRetryAttempts: 2             # محاولتان عند Async
      MaximumEventAgeInSeconds: 21600     # 6 ساعات (أقصى حد)
      DestinationConfig:
        OnFailure:
          Destination: arn:aws:sqs:us-east-1:123456789012:dead-letter
```

> **مهم:** لا تجعل وقت **SQS Visibility** أقل من مهلة الدالة—يفضل **≥ 6×**. :contentReference[oaicite:5]{index=5}

---

## خطوات عملية لضبط “الانتهاء/الانقضاء”
1. **اضبط Timeout بدقة**: اجعله قريبًا من الزمن الفعلي + هامش صغير؛ راقب أخطاء `Task timed out`. :contentReference[oaicite:6]{index=6}  
2. **احترم الإلغاء**: مرّر `CancellationToken` لكل I/O، واخرج مبكرًا عند اقتراب انتهاء المهلة.  
3. **Async Invocations**: عيّن **MaximumEventAge** و**Retries**، وفعّل **DLQ/Destination** للأحداث الساقطة. :contentReference[oaicite:7]{index=7}  
4. **SQS Trigger**: اضبط **Visibility Timeout ≥ 6×** مهلة الدالة؛ وفكّر في **DLQ**. :contentReference[oaicite:8]{index=8}  
5. **Presigned URLs**: اختر صلاحية قصيرة تكفي الاستخدام، وتذكّر حدود الجلسات عند الاعتمادات المؤقتة. :contentReference[oaicite:9]{index=9}  
6. **مراقبة**: راقب `AsyncEventAge` و%timeouts، ونبّه عند ارتفاعهما. :contentReference[oaicite:10]{index=10}

---

## أخطاء شائعة
- الاعتماد على وقت تنفيذ طويل دون تقسيم → وصول سريع لحد **15 دقيقة**. :contentReference[oaicite:11]{index=11}  
- نسيان **DLQ/Destinations** في الاستدعاء غير المتزامن → ضياع أحداث بعد انتهاء عمرها. :contentReference[oaicite:12]{index=12}  
- **SQS Visibility** أقصر من مهلة الدالة → إعادة تسليم مبكّرة ورسائل مكررة. :contentReference[oaicite:13]{index=13}  
- توليد **Presigned URL** بعمر أطول من الحاجة أو عدم مراعاة حدود الاعتمادات المؤقتة. :contentReference[oaicite:14]{index=14}

---

## جدول مقارنة مختصر

| المفهوم                       | الغرض الرئيسي                | ملاحظات مهمة                                                                           |
| ----------------------------- | ---------------------------- | -------------------------------------------------------------------------------------- |
| **Function Timeout**          | أقصى زمن تنفيذ للدالة        | حد أقصى **900s** (15m)؛ اضبط قريبًا من الواقع. :contentReference[oaicite:15]{index=15}  |
| **Maximum Event Age** (Async) | عمر الحدث في طابور Lambda    | حد أقصى **6h**؛ بعده تُرسل للـ DLQ/Destination. :contentReference[oaicite:16]{index=16} |
| **SQS Visibility Timeout**    | إخفاء الرسالة أثناء المعالجة | أوصِ بـ **≥ 6×** مهلة الدالة لتفادي التكرار. :contentReference[oaicite:17]{index=17}    |
| **Presigned URL Expiration**  | صلاحية رابط موقّع (مثل S3)    | تحددها عند الإنشاء؛ قد تُقيَّد بجلسة مؤقتة. :contentReference[oaicite:18]{index=18}       |

---

## ملخص الفكرة  
“**Lambda Expiration**” يشمل **Timeout التنفيذ**، **عمر الحدث** في الاستدعاء غير المتزامن،  
وضبط **SQS Visibility** وصلاحية روابط مثل **Presigned URLs**.  
اضبط هذه الحدود بوعي، احترم الإلغاء، وفَعِّل **DLQ/Destinations**—تحصل على نظام **مُوثوق** يتفادى ضياع الرسائل و**Timeouts** غير الضرورية.
