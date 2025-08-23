# **MVVM — Model–View–ViewModel**

## الترجمة الحرفية  
**Model–View–ViewModel (MVVM)** — **النموذج–العرض–عارض النموذج**

## الوصف العربي المختصر  
نمط لواجهات المستخدم (XAML/سطح مكتب/موبايل) يفصل:  
**Model** بيانات وقواعد، **View** واجهة XAML صِرفة، **ViewModel** يحوّل البيانات لأBindings وأوامر **Command**.

## الشرح المبسّط  
- **View**: لا منطق أعمال؛ فقط XAML وربط بيانات **Binding**.  
- **ViewModel**: خصائص قابلة للإشعار **INotifyPropertyChanged** + أوامر **ICommand**.  
- **Model**: كائنات المجال والتخزين والتحقق.  
- الفائدة: UI نظيف، اختبار أسهل (اختبار ViewModel بدون واجهة)، قابلية إعادة الاستخدام.

## تشبيه  
المُعلّم (ViewModel) يشرح الدرس بلغة يفهمها الجمهور (Bindings وأوامر).  
الكتاب الأصلي (Model) فيه المادة العلمية. شاشة العرض (View) تُظهر الشرح فقط.

---

## مثال عملي مختصر — WPF/.NET (Binding + Command)

### 1) ViewModel: خاصية + أمر (INotifyPropertyChanged / ICommand)
```csharp
// MainViewModel.cs
using System;
using System.ComponentModel;
using System.Runtime.CompilerServices;
using System.Windows.Input;

public class MainViewModel : INotifyPropertyChanged
{
    private int _count;
    public int Count
    {
        get => _count;
        set { if (_count == value) return; _count = value; OnPropertyChanged(); }
    }

    public ICommand IncrementCommand { get; }
    public ICommand ResetCommand     { get; }

    public MainViewModel()
    {
        IncrementCommand = new RelayCommand(_ => Count++);
        ResetCommand     = new RelayCommand(_ => Count = 0, _ => Count > 0); // CanExecute مرتبط بالحالة
    }

    public event PropertyChangedEventHandler? PropertyChanged;
    void OnPropertyChanged([CallerMemberName] string? name = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
}

// أمر بسيط يُستخدم في الـ ViewModel
public class RelayCommand : ICommand
{
    private readonly Action<object?> _exec;
    private readonly Predicate<object?>? _can;
    public RelayCommand(Action<object?> exec, Predicate<object?>? can = null)
        { _exec = exec; _can = can; }

    public bool CanExecute(object? p) => _can?.Invoke(p) ?? true;
    public void Execute(object? p) => _exec(p);
    public event EventHandler? CanExecuteChanged;

    // نادى هذا عند تغيّر شروط التنفيذ (مثلاً عند تغيير Count)
    public void RaiseCanExecuteChanged() => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

> ملاحظة: إن أردت تحديث `CanExecute` تلقائيًا عند تغيّر `Count`، يمكنك استدعاء  
> `((RelayCommand)ResetCommand).RaiseCanExecuteChanged();` داخل Setter الخاص بـ `Count`.

### 2) View: XAML يربط الخصائص والأوامر
```xml
<!-- MainWindow.xaml -->
<Window x:Class="Demo.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="MVVM Demo" Width="300" Height="160">
    <Grid Margin="16" RowDefinitions="Auto,Auto,Auto" RowSpacing="8">
        <TextBlock Text="Counter:" FontWeight="Bold"/>
        <TextBlock Grid.Row="1" Text="{Binding Count}" FontSize="28"/>

        <StackPanel Grid.Row="2" Orientation="Horizontal" HorizontalAlignment="Left" Spacing="8">
            <Button Content="+1"   Command="{Binding IncrementCommand}" Width="60"/>
            <Button Content="Reset" Command="{Binding ResetCommand}"   Width="60"/>
        </StackPanel>
    </Grid>
</Window>
```

### 3) ربط الـ ViewModel بالـ View (مرة واحدة)
```csharp
// MainWindow.xaml.cs
using System.Windows;

namespace Demo
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            DataContext = new MainViewModel(); // ربط مصدر البيانات بالنافذة
        }
    }
}
```

**الإخراج المتوقع (مختصر):**  
نافذة بها عدّاد. زر **+1** يزيد القيمة. **Reset** يعود للصفر ويتعطّل عندما تكون القيمة 0.

> بديل جاهز: حزمة **CommunityToolkit.Mvvm** تختصر `INotifyPropertyChanged` و`RelayCommand` بسمات تلقائية.

---

## خطوات عملية لتطبيق MVVM
- عرّف **Model** نظيفًا بلا ربط واجهة.  
- أنشئ **ViewModel**: خصائص قابلة للإشعار + أوامر **ICommand** + منطق العرض (لا منطق قاعدة بيانات ثقيل).  
- اربط **DataContext** للعرض. استخدم **Binding** و**Converters** و**DataTemplates**.  
- اعزل الوصول للبيانات في **خدمات** تُحقن داخل ViewModel (Dependency Injection).  
- اختبر **ViewModel** بوحدات اختبار (بدون UI).  
- للتحميلات الطويلة: استخدم **async/await** و**CancellationToken** وربط **IsBusy** لتعطيل الأزرار.  
- نظّم المشروع: `Models/ ViewModels/ Views/ Services`.

---

## أخطاء شائعة
- كتابة منطق أعمال في **Code-Behind** للعرض بدل ViewModel.  
- نسيان استدعاء **PropertyChanged** → الواجهة لا تتحدث.  
- استخدام **List<T>** بدل **ObservableCollection<T>** لقوائم مربوطة (لا ترسل إشعارات تغيّر).  
- أوامر **ICommand** بلا **CanExecute** أو بلا تعطيل واجهة عند الانشغال.  
- ربط معقّد جدًا في XAML بدل **تحضير البيانات** داخل ViewModel (تبسيط الـ View).

---

## جدول مقارنة مختصر

| المفهوم                       | الغرض الرئيسي                                   | ملاحظات مهمة                                                 |
| ----------------------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| [MVC](mvc.md)                 | فصل Controller/Model/View لتطبيقات ويب          | المتحكّم ينسّق الطلب/الاستجابة                                 |
| **MVVM**                      | **Binding ثنائي الاتجاه + أوامر عبر ViewModel** | **INotifyPropertyChanged/ICommand؛ مثالي لـ WPF/MAUI/WinUI** |
| [MVP](mvp.md)                 | Presenter ينسّق View “غبيًا”                      | ربط أقل آليّة من MVVM؛ مناسب لبعض المنصّات                     |

---

## ملخص الفكرة  
**MVVM** يفصل واجهة المستخدم عن المنطق عبر **ViewModel** قابل للاختبار،  
مع **Binding** و**Commands** تقلّلان كود الواجهة.  
حافظ على View “رقيقة”، واستخدم `INotifyPropertyChanged`/`ObservableCollection` وعمليات **async** لتجربة سلسة.
