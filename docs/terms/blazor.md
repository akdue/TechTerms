# **Blazor**

## الترجمة الحرفية  
**Blazor (ASP.NET Core)** — **إطار مكوّنات ويب بـ C#** يعمل على **الخادم** أو **WebAssembly** داخل المتصفّح.

## الوصف العربي المختصر  
تكتب واجهاتك بـ **مكوّنات Razor** (ملفات `.razor`) باستخدام **C#** بدل JavaScript.  
نموذجان للتشغيل: **Blazor Server** (Rendering على الخادم عبر SignalR) و**Blazor WebAssembly** (C# يركض داخل المتصفّح).

## الشرح المبسّط  
- **المكوّن** = ملف `.razor` يجمع **Markup + C#** مع ربط بيانات وأحداث.  
- **التحديث** ذكي عبر DOM افتراضي (diff) مثل React.  
- **التواصل** مع JS عند الحاجة عبر **JS Interop**.  
- **البيانات** تُجلب عبر `HttpClient` أو خدمات DI قياسية.

## تشبيه  
تخيل React لكن بدلاً من JavaScript تكتب **C#**، وتختار أين ينفّذ:  
على **الخادم** (خفيف تحميل أولي) أو داخل **المتصفّح** (بدون اتصال مستمر).

---

## مثال سريع — مكوّن Razor + حقن خدمة + حدث

```razor
@* Components/Counter.razor *@
<h3>Counter</h3>

<p>Count: @count</p>
<button class="btn btn-primary" @onclick="Increment">+1</button>

@code {
    private int count;
    void Increment() => count++;
}
```

---

## مثال **Blazor Server** — تشغيل فوري واتصال SignalR

```csharp
// Program.cs  (.NET 8/9)
// dotnet new blazorserver -n BlazorSrv && dotnet run
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorComponents().AddInteractiveServer(); // Blazor Server
var app = builder.Build();

if (!app.Environment.IsDevelopment()) { app.UseExceptionHandler("/Error"); app.UseHsts(); }
app.UseHttpsRedirection();
app.UseStaticFiles();
app.MapRazorComponents<App>()
   .AddInteractiveServerRenderMode(); // SignalR
app.Run();

// App.razor
@using Microsoft.AspNetCore.Components.Routing
<Router AppAssembly="@typeof(App).Assembly">
  <Found Context="routeData">
    <RouteView RouteData="@routeData" />
  </Found>
  <NotFound><h1>404</h1></NotFound>
</Router>
```

**الفكرة:**  
- عرض تفاعلي يُحسب على الخادم.  
- يحتاج اتصال **مستمر** (WebSocket/SignalR).  
- تحميل أولي **سريع** وحجم أصغر.

---

## مثال **Blazor WebAssembly** — تشغيل داخل المتصفّح (بدون خادم مستمر)

```csharp
// Program.cs  (.NET 8/9)
// dotnet new blazorwasm -n BlazorWasm && dotnet run
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");

// خدمات الواجهة (HttpClient للـ APIs)
builder.Services.AddScoped(sp => new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });

await builder.Build().RunAsync();

// index.html (المضيف) يتضمن <div id="app"></div> وتحميل blazor.webassembly.js
```

**الفكرة:**  
- يعمل كليًا داخل المتصفّح عبر **WebAssembly**.  
- يمكنه العمل **دون اتصال** (PWA).  
- تحميل أولي أكبر (حِزم Runtime + Assemblies) لكن بدون تبعية اتصال مباشر.

---

## مثال **JS Interop** — نداء JavaScript من Blazor (سيرفر/واسِم)

```razor
@* Components/Clipboard.razor *@
@inject IJSRuntime JS
<button class="btn" @onclick="Copy">Copy URL</button>
<p>@status</p>

@code {
    string status = "";
    async Task Copy()
    {
        await JS.InvokeVoidAsync("navigator.clipboard.writeText", NavigationManager.Uri);
        status = "Copied!";
    }
    [Inject] NavigationManager NavigationManager { get; set; } = default!;
}
```

---

## خطوات عملية للاختيار والبناء
1. **اختر النموذج**:  
   - **Server**: لوحات داخلية، اتصالات موثوقة، زمن بدء سريع.  
   - **WASM**: عمل دون اتصال/PWA، تقليل ضغط الخادم، دعم CDN.  
2. **تقسيم الكود** إلى **مكوّنات صغيرة** قابلة لإعادة الاستخدام.  
3. **DI** للخدمات (`AddScoped`/`AddSingleton`) واستخدم `HttpClient` للـ APIs.  
4. **الملاَحظية**: سجّل الأخطاء (ErrorBoundary)، وتتبّع الأداء (Telemetry).  
5. **الهوية**: دمج Identity/OIDC؛ في WASM استخدم **Access Token** بحذر.  
6. **الأداء**:  
   - Server: تقليل تكرار State، تجنّب تحديثات كثيفة عبر SignalR.  
   - WASM: تمكين **AOT** أو **Native AOT** عند الحاجة، ضغط Brotli، Lazy-load.  
7. **الاختبار**: اختبر المنطق بـ xUnit، والمكوّنات بـ `bUnit`.

---

## أخطاء شائعة
- وضع منطق أعمال ثقيل داخل المكوّنات بدل خدمات منفصلة.  
- إعادة **State** ضخمة في Blazor Server → حمل SignalR.  
- تجاهل Lazy-loading في WASM → زمن بدء طويل.  
- JS Interop بلا تنظيف/تحقق من وجود عناصر DOM.  
- دمج سيء للهوية (Tokens في `localStorage` بدون حذر من XSS).  
- ربط ثنائي الاتجاه كثيف على عناصر كثيرة دون افتراض **Virtualize**.

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **Blazor Server** | تفاعلية C# على الخادم | اتصال SignalR مستمر، بدء سريع، حمل خادم أعلى |
| **Blazor WebAssembly** | تفاعلية C# داخل المتصفّح | حجم أولي أكبر، يعمل دون اتصال، يقلّل ضغط الخادم |
| Razor Pages | صفحات Forms/CRUD تقليدية | بدون تفاعلية لحظية كثيفة |
| **SPA (React/Vue/Angular)** | مكوّنات JS | نظام بيئي JS أوسع؛ ليس C# |
| Minimal APIs | خدمات REST فقط | واجهات بلا UI؛ تُستهلك من Blazor/SPA |

---

## ملخص الفكرة  
**Blazor** يتيح لك بناء واجهات **تفاعلية** بـ **C#** ومكوّنات Razor.  
اختر **Server** لبدء سريع واتصال مستمر، أو **WASM** للاستقلالية ودعم عدم الاتصال،  
واستخدم **DI + JS Interop + تحسينات أداء**—تحصل على واجهة حديثة بقوّة نظام .NET. 
