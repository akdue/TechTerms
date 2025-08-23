# **SaaS — Software as a Service**

## الترجمة الحرفية  
**Software as a Service (SaaS)** — **البرمجية كخدمة**.

## الوصف العربي المختصر  
تطبيق جاهز **مستضاف في السحابة**. تصل إليه عبر **المتصفح** أو **API** باشتراك.  
المزوّد يدير **البنية والمنصّة والتحديثات**. أنت تدير **البيانات والإعدادات والاستخدام**.

## الشرح المبسّط  
- لا خوادم ولا تحديثات عندك. كل شيء عند المزوّد.  
- تدخل بحسابك وتبدأ العمل.  
- يوجد **واجهة ويب** و/أو **واجهات برمجة (APIs)** للتكامل.  
- غالبًا **متعدّد المستأجرين (Multi-Tenant)** مع عزل بيانات.  
- يدعم **SSO**, **RBAC**, **Logs**, **Backups** حسب الخطة.

## تشبيه  
بدل شراء برنامج وتركيبه، **تستأجره كخدمة جاهزة** على الإنترنت.  
مثل الاشتراك في صالة رياضية كاملة الأدوات بدل بناء صالة في بيتك.

---

## مثال كود C# — استهلاك API لخدمة SaaS (مع مهلات ومحاولات وإيقاع حدود)
> يوضّح المصادقة بـ **Bearer Token**، وإعادة المحاولة عند **429/5xx**، والانتظار حسب **Retry-After**.

```csharp
// dotnet add package System.Net.Http.Json
using System;
using System.Net;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;

class SaaSClient
{
    static readonly HttpClient http = new HttpClient
    {
        BaseAddress = new Uri(Environment.GetEnvironmentVariable("SAAS_BASE_URL")
                              ?? "https://api.example-saas.com"),
        Timeout = TimeSpan.FromSeconds(10)
    };

    static SaaSClient()
    {
        var token = Environment.GetEnvironmentVariable("SAAS_TOKEN") ?? "demo-token";
        http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
        http.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    }

    public static async Task SendMessageAsync(string to, string body)
    {
        var payload = JsonSerializer.Serialize(new { to, body });

        for (int attempt = 1; attempt <= 3; attempt++)
        {
            using var req = new HttpRequestMessage(HttpMethod.Post, "/v1/messages")
            {
                Content = new StringContent(payload, Encoding.UTF8, "application/json")
            };

            using var res = await http.SendAsync(req);
            if (res.IsSuccessStatusCode)
            {
                var json = await res.Content.ReadAsStringAsync();
                Console.WriteLine($"OK: {json}");
                return;
            }

            // 429/5xx: انتظر ثم أعد المحاولة (Exponential Backoff + Retry-After)
            if (res.StatusCode == (HttpStatusCode)429 || (int)res.StatusCode >= 500)
            {
                var delay = res.Headers.RetryAfter?.Delta
                            ?? TimeSpan.FromSeconds(Math.Pow(2, attempt));
                await Task.Delay(delay);
                continue;
            }

            // أخطاء أخرى — اطبع وانهِ
            var err = await res.Content.ReadAsStringAsync();
            throw new HttpRequestException($"SaaS error {(int)res.StatusCode}: {err}");
        }

        throw new TimeoutException("Max retries reached while calling SaaS API.");
    }

    static async Task Main()
    {
        await SendMessageAsync("user@example.com", "Hello from SaaS API!");
    }
}
```

**الإخراج المتوقع (مختصر):**  
- عند نجاح الطلب: يطبع `OK: {...}` مع معرّف الرسالة.  
- عند ضغط المعدّل: ينتظر حسب `Retry-After` ثم يحاول مرة أخرى.

---

## خطوات عملية لاعتماد SaaS
1. حدّد **الفئة**: بريد/إشعارات، CRM، دفعات، دعم فني، تحليلات…  
2. قيّم **الأمان والامتثال**: تشفير، **SSO/SAML/OIDC**، **RBAC**، **DPA/ISO/GDPR**.  
3. افحص **الـ SLA**، التوافر، حدود المعدّل (**Rate Limits**)، وخطة النسخ الاحتياطي.  
4. اختبر **التكامل**: REST/Webhooks/SDKs، وسهولة الاستيراد/التصدير.  
5. خطّط لـ **الخروج**: تصدير بيانات، تنسيق واضح، وثيقة ملكية البيانات.  
6. فعّل **المراقبة**: سجلات، تنبيهات فشل التكامل، قياس تكلفة الاستخدام.  
7. افصل **البيئات**: `dev/staging/prod`، ومفاتيح API مختلفة لكل بيئة.  

---

## أخطاء شائعة
- الاعتماد الكامل دون **خطة خروج** أو **Export** للبيانات.  
- تجاهل **الخصوصية** وموقع تخزين البيانات (Residency).  
- نسيان حدود المعدّل → أخطاء 429 متكرّرة.  
- خلط مفاتيح **الإنتاج** في بيئة الاختبار.  
- صلاحيات واسعة بلا **RBAC** دقيق أو مراجعة وصول دورية.  
- عدم التعامل مع **انقطاع** SaaS بخيارات بديلة/طابور إعادة المحاولة.

---

## جدول مقارنة مختصر

| المفهوم         | الغرض الرئيسي                          | ملاحظات مهمة                                             |
| --------------- | -------------------------------------- | -------------------------------------------------------- |
| [IaaS](iaas.md) | خوادم/شبكات افتراضية تدير نظامها بنفسك | مرونة عالية؛ مسؤولية تحديثات وأمن النظام عليك            |
| [PaaS](paas.md) | تشغيل **الكود** على منصّة مُدارة         | نشر سريع، Autoscale، تحكم أقل بالـ OS                    |
| **SaaS**        | **تطبيق جاهز يُستهلك عبر الويب/API**    | **أسرع تبنّي؛ أقل تحكّم داخلي؛ ركّز على البيانات والتكامل** |

---

## ملخص الفكرة  
**SaaS** يتيح لك استخدام تطبيقات جاهزة فورًا عبر الويب وواجهات برمجة.  
ركّز على **الأمان/الامتثال/التكامل/خطة الخروج**، وتعامل مع **القيود** (Rate Limits/اعتمادية) بذكاء داخل تطبيقك.  
الكود أعلاه يوضح نمط استهلاك API آمنًا مع مهلات، وإعادة المحاولة، واحترام سياسات المزوّد.
