# **Image Index (OCI Manifest List)**

## الترجمة الحرفية  
**OCI Image Index / Docker Manifest List** — **فهرس صور متعدد المنصّات** تحت **وسم واحد**.

## الوصف العربي المختصر  
قطعة وصف (Artifact) تُشير إلى **عدّة مانيفستات صور** لكل **نظام/معمارية**.  
عند السحب `pull`، يختار العميل تلقائيًا **المانيفست المناسب** (مثلاً `linux/amd64` أو `linux/arm64`) من **نفس الوسم**.

## الشرح المبسّط  
- الوسم (Tag) لا يشير لصورة واحدة، بل إلى **Image Index** يحوي قائمة `manifests[]`.  
- كل عنصر يحدد: **Digest** للصورة، **mediaType**، **الحجم**، و`platform.os/architecture`.  
- يدعم **منصّات متعددة** (Multi-Arch) و**أنظمة** مختلفة (Linux/Windows).  
- الصيغ القياسية:  
  - `application/vnd.oci.image.index.v1+json` (OCI).  
  - `application/vnd.docker.distribution.manifest.list.v2+json` (Docker).

## تشبيه  
فهرس في مقدمة كتاب يوجّهك إلى **الفصل المناسب** حسب لغتك.  
العميل يقرأ الفهرس، ثم يفتح **الفصل** الملائم لمنصته.

---

## مثال JSON مبسّط (فهرس OCI)
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:aaa...",
      "size": 901234,
      "platform": { "os": "linux", "architecture": "amd64" }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:bbb...",
      "size": 889123,
      "platform": { "os": "linux", "architecture": "arm64" }
    }
  ]
}
```

---

## مثال كود C# — قراءة Image Index من Registry وعرض المنصّات
> يتطلب توكن عند بعض المستودعات (Docker Hub/خاص). ضع `REGISTRY`, `REPO`, `TAG`, `TOKEN`.

```csharp
// dotnet add package System.Text.Json
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Threading.Tasks;

record OciPlatform([property:JsonPropertyName("os")] string Os,
                   [property:JsonPropertyName("architecture")] string Arch,
                   [property:JsonPropertyName("variant")] string? Variant);

record OciManifestRef(
    [property:JsonPropertyName("mediaType")] string MediaType,
    [property:JsonPropertyName("digest")] string Digest,
    [property:JsonPropertyName("size")] long Size,
    [property:JsonPropertyName("platform")] OciPlatform Platform);

record OciIndex(
    [property:JsonPropertyName("schemaVersion")] int SchemaVersion,
    [property:JsonPropertyName("mediaType")] string MediaType,
    [property:JsonPropertyName("manifests")] OciManifestRef[] Manifests);

class Program
{
    static async Task Main()
    {
        string registry = Environment.GetEnvironmentVariable("REGISTRY") ?? "ghcr.io";
        string repo     = Environment.GetEnvironmentVariable("REPO")     ?? "acme/demo-api";
        string tag      = Environment.GetEnvironmentVariable("TAG")      ?? "latest";
        string token    = Environment.GetEnvironmentVariable("TOKEN")     ?? ""; // اختياري

        using var http = new HttpClient { BaseAddress = new Uri($"https://{registry}/") };
        http.DefaultRequestHeaders.Accept.ParseAdd("application/vnd.oci.image.index.v1+json");
        http.DefaultRequestHeaders.Accept.ParseAdd("application/vnd.docker.distribution.manifest.list.v2+json");
        if (!string.IsNullOrEmpty(token))
            http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);

        var url = $"v2/{repo}/manifests/{tag}";
        using var res = await http.GetAsync(url);
        res.EnsureSuccessStatusCode();

        var json = await res.Content.ReadAsStringAsync();
        var idx = JsonSerializer.Deserialize<OciIndex>(json);

        if (idx?.Manifests is null) { Console.WriteLine("Not an index/empty."); return; }

        Console.WriteLine($"MediaType: {idx.MediaType}  | Items: {idx.Manifests.Length}");
        foreach (var m in idx.Manifests)
        {
            string plat = $"{m.Platform.Os}/{m.Platform.Arch}" + (m.Platform.Variant is {Length:>0} v ? $"/{v}" : "");
            Console.WriteLine($"- {plat}  -> {m.Digest}  ({m.Size} bytes)");
        }
    }
}
```
**الفكرة:** نطلب الـ Index بقبول (Accept) نوعَي **OCI/Docker**. نطبع المنصّات والدجست لكل صورة.

---

## خطوات عملية لإنشاء ودفع Image Index (Multi-Arch)  
- **باستخدام Docker Buildx:**
  ```bash
  docker buildx create --use
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t ghcr.io/acme/demo-api:1.0 \
    --push .
  ```
  سينتج **Image Index** تلقائيًا للوسم `1.0`.

- **التحقق:**
  ```bash
  docker buildx imagetools inspect ghcr.io/acme/demo-api:1.0
  ```
  سترى المنصّات المتاحة تحت الوسم نفسه.

- **نصائح:**
  - وفّر **Base Images** لكلا المعماريتين.  
  - راعِ **Windows** بإضافة `os.version` عند صور Windows.  
  - استخدم **Labels** و**توقيع (cosign)** للفهرس أيضًا.

---

## أخطاء شائعة
- دفع صور منفصلة لكل معماريّة **بنفس الوسم** دون فهرس → آخر دفع يطغى.  
- نسيان `--platform` → تُبنى صورة واحدة فقط (لا يوجد Index).  
- الاعتماد على `:latest` في الإنتاج دون تجميد الإصدارات.  
- فهرس يشير إلى مانيفستات **بـ mediaType غير متوافق** مع الرن-تايم المستهدف.  
- تجاهل `os.version/os.features` لصور Windows → سحب فاشل على بعض الإصدارات.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| [Image (OCI)](image-oci.md) | صورة لمنصّة واحدة (OS/Arch) | طبقات + مانيفست واحد |
| **Image Index** | **تجميع مانيفستات لمنصّات متعددة تحت وسم واحد** | **يختار العميل المنصّة المناسبة تلقائيًا** |
| Multi-Arch Build| طريقة بناء صور عدة منصّات | يولّد عادةً **Image Index** عند `--push` |

---

## ملخص الفكرة  
**Image Index** يجعل الوسم الواحد يخدم **عدّة منصّات** بشكل شفاف.  
ابنِ بصيغة **Multi-Arch**، ادفع الفهرس، وتحقق عبر `imagetools inspect`—ستحصل على سحب تلقائي للصورة المناسبة لكل جهاز.
