# **DevOps**

## الترجمة الحرفية  
**DevOps** — **Development + Operations** (التطوير + التشغيل).

## الوصف العربي المختصر  
ثقافة وممارسات توحّد **التطوير** و**التشغيل** عبر **أتمتة** و**تعاون** و**مراقبة** مستمرة.  
النتيجة: **سرعة إصدار أعلى**، **جودة أفضل**، و**استقرار** مع قدرة سريعة على **الاسترجاع**.

## الشرح المبسّط  
- كسر الحواجز بين الفرق. **فريق واحد** يملك الكود والإصدار والتشغيل.  
- **CI/CD**: بناء، اختبار، نشر تلقائي مع حواجز جودة.  
- **Infra as Code**، **المراقبة/التتبّع**، و**Feedback سريع** من الإنتاج.  
- هدفان توأم: **تدفّق تغييرات سريع** + **موثوقية عالية**.

## تشبيه  
سير شحن آلي: خط إنتاج يبني الحاويات، يفحصها، يوسمها، ثم يطلقها للسفن مع تتبّع GPS وتنبيهات فورية.

---

## مثال عملي مختصر — خدمة C# + خطّ CI/CD (GitHub Actions)

### 1) خدمة .NET Minimal API (صحيّة + إصدار)
```csharp
// Program.cs  (.NET 8/9)
using Microsoft.AspNetCore.Diagnostics.HealthChecks;

var b = WebApplication.CreateBuilder(args);
b.Services.AddHealthChecks();
var app = b.Build();

app.MapGet("/", () => new {
    service = "orders-api",
    version = typeof(Program).Assembly.GetName().Version?.ToString() ?? "0.0.0",
    time = DateTime.UtcNow
});
app.MapHealthChecks("/healthz");      // فحص حيوية للنشر/الموازِن
app.Run();
```

### 2) Dockerfile بسيط (Multi-stage)
```dockerfile
# build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app /p:PublishTrimmed=true /p:PublishSingleFile=true

# run
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app ./
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["./Program"]
```

### 3) CI/CD (GitHub Actions) — بناء/اختبار/حاوية/دفع للسجل
```yaml
# .github/workflows/ci.yml
name: ci-cd
on:
  push: { branches: [ "main" ] }
  pull_request: { branches: [ "main" ] }

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}   # ghcr.io/<owner>/<repo>

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: "8.0.x" }
      - run: dotnet restore
      - run: dotnet build --no-restore -warnaserror
      - run: dotnet test  --no-build --collect:"XPlat Code Coverage"

  docker:
    needs: build-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```

> **الفكرة:** كل دفع إلى `main` يبني، يختبر، يُنشئ صورة، ويدفعها لـ **GHCR**.  
> يمكن إضافة خطوة نشر لاحقًا (Kubernetes/VM/PAAS) مع **Blue/Green** أو **Canary**.

---

## خطوات عملية لتبنّي DevOps
1. **نسق عمل**: Git معيار واحد، فروع خفيفة، مراجعات PR سريعة.  
2. **CI**: بناء + اختبارات وحدة/تكامل + تحليل ساكن + تغطية كحد أدنى.  
3. **تعليب**: حاويات **OCI** موقّعة (cosign) + SBOM + فحص ثغرات.  
4. **CD**: نشر تلقائي ببوابات (اختبارات/موافقة) ونهج **Blue/Green/Canary**.  
5. **أسرار**: مدير أسرار (Vault/KMS) وبيئات `dev/stage/prod` مفصولة.  
6. **Infra as Code**: Terraform/Bicep/Ansible؛ تتبّع تغييرات تحت Git.  
7. **الملاحظة**: Logs/Metrics/Tracing و**SLIs/SLOs** وتنبيهات وRunbooks.  
8. **التعلّم المستمر**: Postmortems بلا لوم، تتبّع MTTR/Change Failure Rate.

---

## أخطاء شائعة
- اعتبار DevOps **أداة** لا **ثقافة + عملية**.  
- نشر يدوي/غير مكرر، وعدم وجود **Rollback** واضح.  
- أسرار داخل المستودع/الحاوية.  
- اختبارات هشة وبطيئة تعرقل التدفق.  
- تجاهل **المراقبة** وغياب مؤشرات **SLI/SLO**.  
- بنية تحتية خارج Git (لا يمكن تكرارها).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **DevOps** | **ثقافة + ممارسات لأتمتة وبناء/نشر/تشغيل مشترك** | يشمل CI/CD وIaC والمراقبة |
| CI/CD | أتمتة بناء/اختبار/نشر | جزء من DevOps |
| SRE | موثوقية تشغيلية بقياس SLO/خطط طوارئ | تركيز على التشغيل/الاعتمادية |
| [Agile](agile.md)  | إدارة تكرارية للمنتج | يكمل DevOps (سرعة + تعلّم) |
| GitOps | تشغيل بالبُنى التعريفية عبر Git | تطبيق DevOps للنشر المستمر للبنية |

---

## ملخص الفكرة  
**DevOps** يوحّد التطوير والتشغيل عبر الأتمتة والتعاون والقياس.  
ابنِ **خط CI/CD** صلبًا، احفظ البنية تحت **كود**، راقب الإنتاج بأرقام واضحة،  
واعمل بثقافة **تحسين مستمر**—ستحصل على سرعة **وأمان** واستقرار في آن واحد.
