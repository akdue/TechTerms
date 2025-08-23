# **HSTS**

## الترجمة الحرفية  
**HTTP Strict-Transport-Security (HSTS)** — **سياسة النقل الآمن الصارمة عبر HTTP**.

## الوصف العربي المختصر  
ترويسة أمان تُجبر المتصفّح على استخدام **HTTPS فقط** لموقعك لمدّة محددة.  
تمنع الرجوع إلى **HTTP** وتخفّف هجمات **SSL Stripping** و**المحتوى المختلط**.

## الشرح المبسّط  
- المرة الأولى عبر **HTTPS** يقرأ المتصفّح ترويسة HSTS.  
- بعدها، أي محاولة للوصول بـ `http://` تُحوَّل داخليًا إلى `https://` قبل الاتصال.  
- خيارات مهمة:  
  - `max-age`: مدّة الإلزام بالثواني (يفضّل ≥ سنة في الإنتاج).  
  - `includeSubDomains`: يشمل كل النطاقات الفرعية.  
  - `preload`: طلب إدراج نطاقك في **قائمة المتصفّحات** المسبقة (لا تحتاج الزيارة الأولى).

## تشبيه  
علامة على باب المبنى: “**دخول عبر البوابة الآمنة فقط**”.  
حتى لو حاول أحدهم توجيهك للبوابة العادية، **سيمنعك الحارس** ويعيدك للبوابة الآمنة.

## أمثلة إعداد (كود قصير)

### 1) ASP.NET Core
```csharp
// Program.cs (.NET 8/9)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHsts(o =>
{
    o.MaxAge = TimeSpan.FromDays(365);   // max-age=31536000
    o.IncludeSubDomains = true;          // includeSubDomains
    o.Preload = true;                    // preload (بعد استيفاء شروط القائمة)
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();              // يضيف ترويسة HSTS
    app.UseHttpsRedirection();  // 308 من HTTP إلى HTTPS
}

app.MapGet("/", () => "Hello HSTS");
app.Run();
```

### 2) Nginx
```nginx
server {
  listen 443 ssl http2;
  server_name example.com;
  # ... SSL/TLS config ...

  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}

# تأكد أن منفذ 80 يحوّل دائمًا إلى HTTPS
server {
  listen 80;
  server_name example.com;
  return 308 https://$host$request_uri;
}
```

### 3) الترويسة نفسها
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

## خطوات عملية لتفعيل HSTS بأمان
- فعّل **HTTPS كامل** أولًا (شهادات صحيحة، سلاسل ثقة مكتملة).  
- أضِف HSTS بقيمة **صغيرة** مبدئيًا (مثلاً أيام) في بيئة إنتاج محدودة، ثم ارفعها تدريجيًا.  
- بعد الاستقرار، استخدم `max-age` ≥ سنة + `includeSubDomains`.  
- أضف `preload` فقط عند التأكد أن **كل النطاقات الفرعية** تعمل على HTTPS و**تحويل 80→443** دائم.  
- راقب انتهاء الشهادات وجدّدها تلقائيًا لمنع حظر الوصول عند الانتهاء.

## أخطاء شائعة
- تفعيل `preload` قبل جاهزية كل النطاقات الفرعية → **قفل** المستخدمين خارج الخدمة.  
- الإبقاء على `max-age` قصيرًا جدًا → فاعلية منخفضة ضد الهجمات.  
- نسيان التحويل من HTTP إلى HTTPS على المنفذ 80.  
- استخدامه في بيئات **تجريبية/محلية** على نطاقات حقيقية.  
- الاعتماد على HSTS وحده دون معالجة **المحتوى المختلط** وسياسات مثل **CSP**.

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [HTTPS](https.md) | قناة ويب مشفّرة وهوية الخادم | يحمي الاتصال؛ لا يجبر المتصفّح دائمًا على HTTPS |
| **HSTS** | **إجبار المتصفّح على HTTPS دومًا** | **max-age/includeSubDomains/preload؛ يمنع الرجوع إلى HTTP** |
| [TLS](tls.md) | تشفير/سلامة/توثيق على طبقة النقل | يستخدم تحت HTTPS وبروتوكولات أخرى |

## ملخص الفكرة  
**HSTS** يضمن أن موقعك يُزار عبر **HTTPS فقط** خلال مدّة محددة،  
فيقضي عمليًا على محاولات الرجوع إلى HTTP ويقوّي وضع الأمان—خصوصًا مع `includeSubDomains` و`preload` بعد التأكد من الجاهزية.
