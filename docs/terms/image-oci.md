# **Image (OCI)**

## الترجمة الحرفية  
**Open Container Initiative Image (OCI Image)** — **صورة حاوية قياسية** قابلة للتبادل بين الأدوات.

## الوصف العربي المختصر  
حزمة **محتوى قابلة للعنونة بالـ Digest** تتكوّن من:  
**Manifest** (يصف الطبقات)، **Config** (بيئة التشغيل/التاريخ)، و**Layers** (فروق نظام الملفات).  
تُخزَّن/تُنقل عبر **Registry** (مثل Docker Hub/ACR/ECR/GCR) وتعمل بأي Runtime متوافق مع **OCI**.

## الشرح المبسّط  
- **Layers**: أرشيفات (tar.gz/zstd) تُطبَّق بالتسلسل لتُنشئ الجذر (`/`) داخل الحاوية.  
- **Config**: معلومات التشغيل (Entrypoint/Cmd/Env/WorkingDir/OS/Arch وDiffIDs).  
- **Manifest**: يربط الـ Config والـ Layers مع **Media Types** وأحجام و**Digests** (`sha256:...`).  
- **Image Index (Manifest List)**: حاوية ميتا تُشير إلى **إصدارات منصّات متعددة** (amd64/arm64).  
- **Content-addressable**: كل Blob يُعرَّف بــ **Digest**؛ النزول/التخزين يعتمد على المضمون لا الاسم.  
- **متوافق**: Docker/Containerd/Podman تبادلها لأن **OCI** حدّد الصيغ.

## تشبيه  
**كعكة طبقات**: كل طبقة تضيف ملفات.  
ورقة تعليمات (Manifest) تقول: “استعمل هذه الطبقات، وهذا ملف الإعداد (Config)”.

---

## مثال عملي سريع — Dockerfile (بناء متعدد المراحل + ملصقات OCI)
```dockerfile
# مرحلة البناء
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app /p:PublishTrimmed=true /p:PublishSingleFile=true

# مرحلة التشغيل (صورة أصغر)
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app ./
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
# ملصقات OCI قياسية (تُساعد في التعقّب/التوثيق)
LABEL org.opencontainers.image.title="demo-api" \
      org.opencontainers.image.description="Minimal API" \
      org.opencontainers.image.source="https://example.com/repo" \
      org.opencontainers.image.licenses="MIT"
ENTRYPOINT ["./Program"]
```

**أوامر شائعة**
```bash
# بناء ودفع
docker build -t ghcr.io/acme/demo-api:1.0 .
docker push ghcr.io/acme/demo-api:1.0

# فحص المانيفست والطبقات
docker buildx imagetools inspect ghcr.io/acme/demo-api:1.0
docker image inspect ghcr.io/acme/demo-api:1.0 | jq '.[0].RootFS.Layers'

# إنشاء صورة متعددة المعماريات
docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/acme/demo-api:1.1 --push .
```

---

## مثال C# قصير — التحقق من Digest (فكرة العنوانة بالمحتوى)
```csharp
// dotnet add package System.Text.Json
using System;
using System.IO;
using System.Security.Cryptography;

class OciDigest
{
    // يحسب sha256:... لملف (مثلاً layer.tar) لمقارنته بما في الـ Manifest
    public static string Sha256Digest(string path)
    {
        using var s = File.OpenRead(path);
        using var sha = SHA256.Create();
        var hash = sha.ComputeHash(s);
        return "sha256:" + Convert.ToHexString(hash).ToLowerInvariant();
    }

    static void Main(string[] args)
    {
        var file = args.Length > 0 ? args[0] : "layer.tar";
        Console.WriteLine(Sha256Digest(file)); // مثال: sha256:3b1e...
    }
}
```
**الفكرة:** الـ Registry وRuntime يعتمدون الـ **Digest** للتأكّد أن الـ Blob (طبقة/Config) **هو نفسه** المتوقَّع في الـ Manifest.

---

## كيف تُغ包 الصورة حسب معيار OCI (مختصر)
- **Manifest (`application/vnd.oci.image.manifest.v1+json`)**:  
  يشير إلى **Config** و**Layers** (MediaType/Size/Digest).  
- **Config (`application/vnd.oci.image.config.v1+json`)**:  
  حقل `config` (Entrypoint/Cmd/Env/Volumes/ExposedPorts) + `rootfs.diff_ids` (هاشات الطبقات *بعد فك الضغط*).  
- **Layers**: أرشيفات لنظام الملفات تُطبّق بالترتيب.

> العملي: لا تحتاج لصنع هذه الملفات يدويًا—الأدوات (Docker/BuildKit/Buildx/Podman) تولّدها وفق **OCI**.

---

## خطوات عملية/نصائح
- **صغّر الصورة**: Multi-stage، حذف أدوات البناء، استخدام صور أساس نحيفة (alpine/distroless/aspnet).  
- **ثبّت الإصدارات** (Pin) وتجنّب `:latest` في الإنتاج.  
- أضِف **Labels OCI** (العنوان/المصدر/النسخة/الرخّص).  
- **لا تضع أسرارًا** في الطبقات (تظلّ في التاريخ). استخدم **Build Secrets**/Runtime Env.  
- فعّل **SBOM** و**التوقيع**:  
  - SBOM (spdx/cyclonedx) + **cosign attest**.  
  - توقيع **cosign sign** لتوثيق المصدر (Sigstore/SLSA).  
- **إعادة الإنتاج**: ثبّت الوقت/المناطق/أوامر `COPY` المتّسقة لتقليل تغيّر الطبقات.  
- لمنصّات متعددة: استخدم **`buildx --platform`** وادفع **Manifest List**.  
- افحص الثغرات: **trivy/grype** قبل النشر.

---

## أخطاء شائعة
- استخدام `:latest` → تكرارات غير متحكَّم بها وصعوبة الرجوع.  
- صور **ضخمة** بسبب ترتيب خطوات خاطئ (وضع `COPY . .` قبل `restore` يُبطل الكاش).  
- تضمين **Secrets** (مفاتيح/توكنات) داخل الطبقات أو السجلّ.  
- إهمال **Labels/SBOM/Signatures** → صعوبة التعقّب والثقة.  
- بناء صورة لمنصّة واحدة فقط ثم تشغيلها على **معمارية** مختلفة.  
- عدم احترام **OCI Media Types** عند التعامل بالأدوات المنخفضة (oras/skopeo).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Image (OCI)** | **حزمة طبقات + Manifest/Config بعنونة Digest** | **قياسي، قابل للنقل بين Docker/Containerd/Podman/Registries** |
| [Container](container.md) | تنفيذ حيّ لصورة مع طبقة كتابة | عمر قصير، يشارك نواة المضيف |
| [Image Index](image-index.md) | قائمة مانيفستات لمنصّات متعددة | لــ amd64/arm64…؛ يدعمه buildx/registries |
| [VM Image](virtual-machine.md) | صورة نظام تشغيل كامل (AMI/VHD) | أثقل؛ ليست وفق OCI؛ لا تُدار بالـ Registry نفسه |

---

## ملخص الفكرة  
**OCI Image** = **طبقات محتوى + وصف قياسي** يُنقل ويُشغَّل في أي بيئة متوافقة.  
ابنِ صورًا **صغيرة وموقَّعة وبلا أسرار**، أضِف **Labels/SBOM**، وادفع **Manifest List** للمنصّات المختلفة—تحصل على نشر أسرع، موثوق، وقابل للتتبع.
