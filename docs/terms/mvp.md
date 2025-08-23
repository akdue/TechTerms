# **MVP — Model–View–Presenter**

## الترجمة الحرفية  
**Model–View–Presenter (MVP)** — **النموذج–العرض–المُقدِّم**

## الوصف العربي المختصر  
نمط لواجهات المستخدم يفصل منطق العرض داخل **Presenter**.  
**View** واجهة بسيطة (Passive View). **Model** بيانات/قواعد المجال.

## الشرح المبسّط  
- **View**: طبقة واجهة فقط. تعرض بيانات. ترفع أحداث للمُقدِّم.  
- **Presenter**: يمسك المنطق. يستمع لأحداث الواجهة. ينسّق مع **Model/خدمات**. ثم يُحدّث الواجهة عبر **Interface**.  
- **Model**: كائنات المجال والتخزين. بلا تفاصيل UI.  
- الفائدة: **اختبار سهل** للمُقدِّم. الواجهة تبقى رقيقة.

## تشبيه  
المُقدِّم في مؤتمر. الجمهور (View) يشاهد. فريق البحث (Model) يزوّد البيانات.  
المُقدِّم وحده يقرّر ماذا يُعرض ومتى.

---

## مثال كود C# بسيط (WinForms/WPF — MVP بأسلوب أحداث وواجهات)
> الفكرة: **العرض** يعرّف واجهة `ILoginView`.  
> **Presenter** يربط الأحداث بالخدمات، ثم يحدّث العرض دون معرفة تفاصيله.

```csharp
// العقدة بين العرض والمُقدِّم — لا UI هنا
public interface ILoginView
{
    string Username { get; }
    string Password { get; }

    event EventHandler LoginClicked;        // يرفعه العرض عند ضغط زر الدخول

    void SetBusy(bool on);
    void ShowMessage(string text, bool isError = false);
}

// خدمة مصادقة — تمثيل مبسّط (Model/Service)
public interface IAuthService
{
    Task<bool> SignInAsync(string user, string pass, CancellationToken ct = default);
}

public class FakeAuthService : IAuthService
{
    public Task<bool> SignInAsync(string user, string pass, CancellationToken ct = default)
        => Task.FromResult(user == "admin" && pass == "1234");
}

// Presenter — يمسك منطق العرض، لا يعرف تفاصيل WinForms/WPF
public class LoginPresenter
{
    private readonly ILoginView _view;
    private readonly IAuthService _auth;

    public LoginPresenter(ILoginView view, IAuthService auth)
    {
        _view = view;
        _auth = auth;
        _view.LoginClicked += OnLoginClicked;    // ربط الحدث
    }

    private async void OnLoginClicked(object? sender, EventArgs e)
    {
        _view.SetBusy(true);
        try
        {
            var ok = await _auth.SignInAsync(_view.Username, _view.Password);
            _view.ShowMessage(ok ? "Welcome!" : "Invalid credentials", isError: !ok);
        }
        catch (Exception ex)
        {
            _view.ShowMessage($"Error: {ex.Message}", isError: true);
        }
        finally
        {
            _view.SetBusy(false);
        }
    }
}
```

### ربط Presenter مع View (مثال هيكلي)
```csharp
// في كود واجهتك (WinForms: Form / WPF: Window/UserControl)
// نفّذ ILoginView بخصائص وأحداث تربط عناصر UI (TextBox/Button).
public partial class LoginForm : Form, ILoginView
{
    public string Username => txtUser.Text;
    public string Password => txtPass.Text;

    public event EventHandler? LoginClicked;

    public LoginForm()
    {
        InitializeComponent();
        btnLogin.Click += (s, e) => LoginClicked?.Invoke(this, EventArgs.Empty);
        // إنشاء Presenter وحقن الخدمة
        var presenter = new LoginPresenter(this, new FakeAuthService());
    }

    public void SetBusy(bool on) => UseWaitCursor = on;

    public void ShowMessage(string text, bool isError = false)
        => MessageBox.Show(text, isError ? "Error" : "Info");
}
```

**الإخراج المتوقع (مختصر):**  
- عند الضغط على **Login**، يستدعي Presenter خدمة المصادقة.  
- يعرض رسالة نجاح/فشل. الواجهة لا تعرف منطق التحقق.

---

## خطوات عملية لتطبيق MVP
- عرّف واجهات **View** (خصائص/أحداث/دوال) بدون أي منطق أعمال.  
- اكتب **Presenter** يحقن **خدمات/Model**، ويستجيب للأحداث، ويحدث الواجهة عبر الواجهة (Interface).  
- اختبر **Presenter** بوحدات اختبار عبر **View مزيفة (Mock/Fake)**.  
- أبقِ **Code-Behind** رقيقًا: توجيه أحداث فقط.  
- للتزامن: مرّر **CancellationToken** وراعِ **الواجهة الرئيسية** عند تحديث UI (Dispatcher/Invoke).

---

## أخطاء شائعة
- **Fat View**: منطق داخل الواجهة بدل Presenter.  
- ربط Presenter بتفاصيل UI (فقدان قابلية الاختبار).  
- عدم فصل **Model/Service** عن Presenter → صعوبة التبديل/الاختبار.  
- تسريبات أحداث: عدم إلغاء الاشتراك عند التخلص (Dispose).  
- منطق Async داخل UI بدون حماية السياق → تجمّد الواجهة.

---

## جدول مقارنة مختصر

| المفهوم         | الغرض الرئيسي                             | ملاحظات مهمة                               |
| --------------- | ----------------------------------------- | ------------------------------------------ |
| [MVC](mvc.md)   | فصل Controller/Model/View                 | مناسب للويب؛ المتحكّم يعالج طلبات HTTP      |
| **MVP**         | **Presenter يدير منطق العرض، View سلبية** | **واجهات وأحداث؛ اختبار Presenter بسهولة** |
| [MVVM](mvvm.md) | Binding وCommands عبر ViewModel           | ممتاز لـ XAML/WPF/MAUI؛ يقلّل كود الـ View  |

---

## ملخص الفكرة  
**MVP** يجعل واجهتك **رقيقة** والمُنطق في **Presenter**.  
اختبر Presenter بسهولة، وابقِ الواجهة “غبية” ترفع أحداثًا وتعرض نتائج—تحصل على تطبيق نظيف وسهل الصيانة.
