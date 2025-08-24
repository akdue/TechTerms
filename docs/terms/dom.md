# **DOM — Document Object Model**

## الترجمة الحرفية  
**Document Object Model (DOM)** — **نموذج كائني لمستند الويب**.

## الوصف العربي المختصر  
تمثيل **شجري** لصفحة HTML داخل المتصفّح.  
كل وسمٍ يصبح **عقدة/عنصر** لها خصائص وأحداث.  
يمكن للـ JS (أو Blazor/غيره) **قراءة** الشجرة و**تعديلها** و**الاستماع** للأحداث.

## الشرح المبسّط  
- الصفحة = **شجرة** تبدأ من `document` → عناصر (`html` → `body` → …).  
- تستطيع الوصول للعناصر بـ **Selectors** (`getElementById`, `querySelector`).  
- تغيّر النص/الخصائص/الأنماط وتتعامل مع **الأحداث** (click, input…).  
- أي تعديل DOM قد يسبّب **إعادة تدفّق/رسم** (Reflow/Repaint) → انتبه للأداء.

## تشبيه  
DOM مثل **شجرة عائلة** للصفحة: كل عنصر هو **عُقدة** لها أبناء وإخوة.  
عندما تغيّر عقدة، قد تتأثر **فروع** أخرى (إعادة ترتيب/رسم).

---

## مثال سريع (JavaScript) — تحديد عنصر، تعديل نص، حدث، وإدراج عنصر

```html
<div id="msg"></div>
<ul id="list"></ul>
<button id="add">Add Item</button>

<script>
  // قراءات/تعديلات آمنة: استخدم textContent لتفادي XSS
  const msg = document.getElementById('msg');
  msg.textContent = 'Hello DOM!';

  // تفويض أحداث: مستمع واحد لقائمة كاملة
  const list = document.querySelector('#list');
  document.getElementById('add').addEventListener('click', () => {
    const li = document.createElement('li');
    li.textContent = `Item ${list.children.length + 1}`;
    list.appendChild(li); // إدراج في الشجرة
  });
</script>
```

> ملاحظات:  
> - **لا** تستخدم `innerHTML` مع مدخلات المستخدم.  
> - عدّل DOM **دفعة واحدة** لتقليل إعادة التدفّق (DocumentFragment / `requestAnimationFrame`).

---

## مثال C# (Blazor) — نداء DOM عبر **JS Interop** بلا التعامل المباشر من C#
```csharp
// Index.razor
@inject IJSRuntime JS
<h1 id="title">Welcome</h1>
<button @onclick="ChangeTitle">Change</button>

@code {
    private async Task ChangeTitle()
        => await JS.InvokeVoidAsync("setText", "#title", "مرحبا DOM");
}

// wwwroot/js/dom.js  (أضفه في index.html: <script src="js/dom.js"></script>)
window.setText = (selector, text) => {
  const el = document.querySelector(selector);
  if (el) el.textContent = text; // آمن
};
```

> الفكرة: تبقى قواعد العمل في C#، والتغيير الفعلي للـ DOM يتم داخل **وظيفة JS صغيرة**.

---

## خطوات عملية للتعامل مع DOM بذكاء
1. **اختيار عناصر**: `getElementById` للأسرع؛ `querySelector(All)` للمرونة.  
2. **أمان**: استخدم `textContent`/`setAttribute` بدل `innerHTML` مع مدخلات المستخدم.  
3. **أداء**:  
   - جمّع التعديلات (Fragment / `append` دفعة واحدة).  
   - قلّل قراءات/كتابات متبادلة (اقرأ أولًا، ثم اكتب).  
   - استخدم `classList` بدل `style` المتكرر.  
4. **أحداث**: **تفويض** الأحداث على عنصر أب لعدد كبير من الأبناء.  
5. **قياس**: راقب **Reflow/Repaint** وأزمنة المعالجة (Performance tab).  
6. **قابلية الاختبار**: أعطِ العناصر **IDs/roles** واضحة، وتجنّب الاعتماد على بنية HTML الهشّة.

---

## أخطاء شائعة
- استعمال `innerHTML` مع نص غير موثوق → **XSS**.  
- لمس DOM داخل حلقات كثيفة دون تجميع → بطء واضح.  
- الاعتماد على ترتيب عقد معيّن دون تثبيت **Selectors** واضحة.  
- خلط **DOM** و**حالة التطبيق** دون إدارة حالة → سلوك متناقض.  
- تجاهل **التنظيف** عند إزالة عناصر (إلغاء مستمعي الأحداث).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **DOM** | **شجرة عناصر الصفحة** قابلة للقراءة/الكتابة | أساس التفاعل في المتصفّح |
| **Virtual DOM** | طبقة وسيطة لمقارنة الشجرة وتحديثها بكفاءة | شائع في React؛ يقلّل لمس DOM المباشر |
| **Shadow DOM** | عزل نمط/بنية لمكوّنات الويب | يمنع تسرّب CSS والأسماء |
| **CSSOM** | شجرة أنماط (CSS) | تتكامل مع DOM لتحديد التخطيط |
| **BOM** | كائنات المتصفّح (window, location, history) | خارج شجرة عناصر الصفحة |

---

## ملخص الفكرة  
**DOM** هو **الشجرة الحيّة** لصفحتك.  
اختر العناصر بذكاء، عدّلها **آمنًا** و**دفعيًا**، واستخدم تفويض الأحداث—  
تحصل على واجهة **سريعة**، **آمنة**، وسهلة الصيانة. 
