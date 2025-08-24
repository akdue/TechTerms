# **SOAP — Simple Object Access Protocol**

## الترجمة الحرفية  
**SOAP** — **بروتوكول الوصول البسيط للكائنات** (تبادل رسائل **XML** وفق غلاف معياري).

## الوصف العربي المختصر  
بروتوكول رسائل يعتمد **XML** مع **غلاف (Envelope)** يحوي **Header** اختياريًا و**Body** إجباريًا.  
يعتمد غالبًا عبر **HTTP/HTTPS**، ويوصف بالعقد عبر **WSDL**، ويدعمه معيار **WS-\*** (أمن/عنونة/موثوقية).

## الشرح المبسّط  
- الرسالة = **Envelope → Header → Body/Fault**.  
- **SOAP 1.1**: `Content-Type: text/xml` + رأس **SOAPAction**.  
- **SOAP 1.2**: `Content-Type: application/soap+xml; action="..."`.  
- أساليب الأنماط: **RPC** أو **Document/Literal** (الموصى به للتوافق).  
- **WSDL** يحدد العمليات/أنواع XML لتوليد عميل تلقائيًا (Proxies).

## تشبيه  
طرد بريدي رسمي: ظرف (Envelope)، عليه بيانات إضافية (Header: توقيع/عنوان)، وداخل الظرف المحتوى (Body) أو ورقة خطأ (Fault).

---

## مثال C# — استهلاك خدمة SOAP يدويًا عبر `HttpClient`

> نرسل طلبًا لعملية **Add(a,b)** إلى نقطة `/soap` (أسلوب 1.1).  
> عدّل العناوين/الأسماء حسب خدمتك الحقيقية (Namespace/Action/Endpoint).

```csharp
// .NET 8/9
using System;
using System.Net.Http;
using System.Text;
using System.Xml.Linq;

class SoapDemo
{
    static async System.Threading.Tasks.Task Main()
    {
        var endpoint = "https://example.com/soap";          // ← غيّره
        var ns = "http://tempuri.org/";                     // ← غيّره حسب WSDL
        var action = "http://tempuri.org/ICalc/Add";        // ← SOAPAction

        // 1) بناء المغلّف (Envelope) — SOAP 1.1
        var soapEnv = $@"<?xml version=""1.0"" encoding=""utf-8""?>
<soap:Envelope xmlns:xsi=""http://www.w3.org/2001/XMLSchema-instance""
               xmlns:xsd=""http://www.w3.org/2001/XMLSchema""
               xmlns:soap=""http://schemas.xmlsoap.org/soap/envelope/"">
  <soap:Header/>
  <soap:Body>
    <Add xmlns=""{ns}"">
      <a>7</a>
      <b>5</b>
    </Add>
  </soap:Body>
</soap:Envelope>";

        using var http = new HttpClient();
        var content = new StringContent(soapEnv, Encoding.UTF8, "text/xml");
        content.Headers.Add("SOAPAction", $"\"{action}\""); // مهم في 1.1

        // 2) إرسال واستقبال
        var resp = await http.PostAsync(endpoint, content);
        var xml = await resp.Content.ReadAsStringAsync();

        // 3) معالجة Fault أو استخراج النتيجة
        var doc = XDocument.Parse(xml);
        XNamespace soap = "http://schemas.xmlsoap.org/soap/envelope/";
        if (doc.Root?.Element(soap + "Body")?.Element(soap + "Fault") is XElement fault)
        {
            var code = fault.Element("faultcode")?.Value;
            var msg  = fault.Element("faultstring")?.Value;
            throw new Exception($"SOAP Fault: {code} - {msg}");
        }

        XNamespace svc = ns;
        var result = doc.Root?
            .Element(soap + "Body")?
            .Element(svc + "AddResponse")?
            .Element(svc + "AddResult")?.Value;

        Console.WriteLine($"AddResult = {result}");
    }
}
```

> بديلًا: استخدم **WSDL** لتوليد عميل قوي (Proxy) بأداة توليد (مثل `dotnet-svcutil`) بدل بناء XML يدويًا.

---

## خطوات عملية لاستهلاك/إنشاء SOAP
1. **اقرأ WSDL**: حدّد **Binding/Port/Operations** و**Namespaces** و**SOAPAction**.  
2. **اختر الإصدار**: 1.1 (مع SOAPAction) أو 1.2 (`application/soap+xml`).  
3. **ولّد عميلًا** من WSDL (أفضلية) أو ابنِ **Envelope** يدويًا كما بالمثال.  
4. **أمن**: إن طُلب، أضف **WS-Security** (UsernameToken/توقيع/تشفير) ضمن **Header**، واستخدم **TLS**.  
5. **تعامل مع Fault**: افحص `Fault` دائمًا وسجّل **Reason/Detail**.  
6. **اختبر التوافق**: الأسماء ومساحات الأسماء **Namespace** حسّاسة؛ راقب الـ **XML Schema** بعناية.

---

## أخطاء شائعة
- خطأ في **Namespace** أو العنصر الجذري (يكسر التفاسير).  
- نسيان **SOAPAction** في 1.1 أو وضعه في 1.2 بطريقة خاطئة.  
- تجاهل عنصر **Fault** والاكتفاء برمز HTTP.  
- تضمين كلمات مرور خام في Header بدلاً من **WS-Security** المناسب أو **TLS**.  
- خلط **RPC** و**Document/Literal** في العقد/الرسالة.  
- الاعتماد على زمن نظام غير متزامن عند استخدام **Timestamps** في WS-Security (انحراف الساعة).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **SOAP** | **تبادل رسائل XML بعقد WSDL ومعايير WS-\*** | يدعم أمن/موثوقية/معاملات؛ أثقل من REST |
| [REST](rest.md) | واجهات موارد عبر HTTP | JSON شائع؛ بساطة/أداء؛ بلا عقد صارم افتراضيًا |
| gRPC | استدعاء إجراء ثنائي عبر HTTP/2 | بروتوبَف، عالي الأداء، Streaming، يحتاج توليد كود |
| WSDL | وصف عمليات/أنواع SOAP | يولّد Clients/Servers متوافقة |
| WS-Security | توقيع/تشفير/Token | يضيف Headers أمنية للرسائل |

---

## ملخص الفكرة  
**SOAP** = رسائل **XML** معيارية بغلاف محدّد، تُبنى على **WSDL** وتتكامل مع **WS-\*** للأمن والموثوقية.  
عند الاستهلاك: احترم **Namespaces/Actions**، تعامل مع **Fault**، واستخدم **TLS/WS-Security**—  
تحصل على تكامل صارم العقد مناسب للأنظمة المؤسسية واللوائح الصارمة.
