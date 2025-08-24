# **XML**

## الترجمة الحرفية  
**Extensible Markup Language (XML)** — **لغة ترميز قابلة للامتداد**.

## الوصف العربي المختصر  
تنسيق نصّي **منظَّم** به عناصر/سمات وقيم، مع **مساحات أسماء** (Namespaces)  
وإمكان **التحقق** عبر مخطّطات (**XSD**) لتبادل بيانات **قابلة للآلات والإنسان**.

## الشرح المبسّط  
- كل وثيقة لها **جذر** (Root) وعُقد (Elements) و**سمات** (Attributes).  
- **Namespace** يميّز الأسماء (`xmlns`) لتجنّب التعارض.  
- يمكن وصف البنية بـ **XSD** ثم **التحقق** قبل المعالجة.  
- القراءة/الكتابة عبر **DOM/LINQ to XML** أو **تسلسل (Serialization)** إلى كائنات.

## تشبيه  
بطاقة نموذج رسمي: خانات ثابتة (**عناصر**) وحقول صغيرة فوقها (**سمات**).  
يمكنك التحقق من صحتها بقواعد الشركة (**XSD**).

---

## مثال C# (1) — إنشاء/قراءة XML مع Namespace (LINQ to XML)

```csharp
// .NET 8/9
using System;
using System.Xml.Linq;
using System.Xml.XPath;

class XmlDemo1
{
    static void Main()
    {
        XNamespace ns = "urn:shop";
        var doc = new XDocument(
            new XElement(ns + "Order",
                new XAttribute("id", 123),
                new XElement(ns + "Customer", "abdalkreem"),
                new XElement(ns + "Items",
                    new XElement(ns + "Item", new XAttribute("sku","KB-01"), new XAttribute("qty",1), 120),
                    new XElement(ns + "Item", new XAttribute("sku","MS-77"), new XAttribute("qty",2), 60)
                )
            )
        );
        doc.Save("order.xml"); // كتابة جميلة افتراضيًا

        // قراءة العناصر وحساب الإجمالي
        decimal total = 0;
        foreach (var it in doc.Root!.Element(ns + "Items")!.Elements(ns + "Item"))
        {
            decimal price = (decimal)it;      // قيمة العنصر
            int qty = (int)it.Attribute("qty")!;
            total += price * qty;
        }
        Console.WriteLine($"Total = {total}"); // 240

        // مثال XPath سريع بدون تعريف NamespaceManager (local-name hack)
        var kb = doc.XPathSelectElement("/*[local-name()='Order']/*[local-name()='Items']/*[local-name()='Item' and @sku='KB-01']");
        Console.WriteLine(kb?.Value); // 120
    }
}
```

---

## مثال C# (2) — تسلسل/فكّ تسلسل XML إلى كائنات

```csharp
using System;
using System.IO;
using System.Xml.Serialization;

[XmlRoot("User")]
public class User
{
    [XmlAttribute] public int Id { get; set; }
    public string Name { get; set; } = "";
}

class XmlDemo2
{
    static void Main()
    {
        var u = new User { Id = 1, Name = "Kreem" };

        var ser = new XmlSerializer(typeof(User));
        using var sw = new StringWriter();
        ser.Serialize(sw, u);
        string xml = sw.ToString();
        Console.WriteLine(xml); // <User Id="1"><Name>Kreem</Name></User>

        using var sr = new StringReader(xml);
        var u2 = (User)ser.Deserialize(sr)!;
        Console.WriteLine(u2.Name); // Kreem
    }
}
```

---

## مثال C# (3) — التحقق من XML بمخطّط **XSD**

```csharp
using System;
using System.IO;
using System.Xml;
using System.Xml.Linq;
using System.Xml.Schema;

class XmlDemo3
{
    static void Main()
    {
        XNamespace ns = "urn:shop";
        var doc = XDocument.Parse(@"<Order xmlns='urn:shop' id='1'>
  <Customer>abdalkreem</Customer>
  <Items><Item sku='KB-01' qty='1'>120</Item></Items>
</Order>");

        string xsd = @"<?xml version='1.0'?>
<xs:schema xmlns:xs='http://www.w3.org/2001/XMLSchema'
           targetNamespace='urn:shop' xmlns='urn:shop'
           elementFormDefault='qualified'>
  <xs:element name='Order'>
    <xs:complexType>
      <xs:sequence>
        <xs:element name='Customer' type='xs:string'/>
        <xs:element name='Items'>
          <xs:complexType>
            <xs:sequence>
              <xs:element name='Item' maxOccurs='unbounded'>
                <xs:complexType>
                  <xs:simpleContent>
                    <xs:extension base='xs:decimal'>
                      <xs:attribute name='sku' type='xs:string' use='required'/>
                      <xs:attribute name='qty' type='xs:int'    use='required'/>
                    </xs:extension>
                  </xs:simpleContent>
                </xs:complexType>
              </xs:element>
            </xs:sequence>
          </xs:complexType>
        </xs:element>
      </xs:sequence>
      <xs:attribute name='id' type='xs:int' use='required'/>
    </xs:complexType>
  </xs:element>
</xs:schema>";

        var schemas = new XmlSchemaSet();
        schemas.Add("urn:shop", XmlReader.Create(new StringReader(xsd)));

        bool ok = true;
        doc.Validate(schemas, (o, e) => { ok = false; Console.WriteLine("XSD error: " + e.Message); });
        Console.WriteLine($"Valid? {ok}");
    }
}
```

---

## خطوات عملية لاستخدام XML بذكاء
1. **حدّد المخطّط** (XSD) مبكرًا: الأنواع، الإلزام، الحدود.  
2. استخدم **Namespaces** دائمًا في المستندات المشتركة.  
3. اختر: **عنصر** للمحتوى/البنى، و**سمة** لميتا بسيطة (معرّفات، وحدات).  
4. لا تبنِ XML بـ **سلاسل نصية**؛ استخدم **XDocument/XElement** أو **XmlWriter**.  
5. عند التبادل: ثبّت **الترميز** (`UTF-8`) وتجنّب الاعتماد على ترتيب غير محدّد بالمخطّط.  
6. فعّل **التحقق** قبل المعالجة، وسجّل أخطاء XSD مع موقعها.

---

## أخطاء شائعة
- تجاهل **Namespaces** ثم فشل الاستعلام/التحقق.  
- خلط عناصر/سمات بلا منطق → صعوبة التحقق/التطوّر.  
- الاعتماد على **String Concatenation** بدل API آمن.  
- نسيان **escape** لـ `<` `&` داخل النص.  
- عدم إغلاق Readers/Writers أو فقدان الترميز الصحيح.  
- عدم التمييز بين **XML** و**HTML** (HTML ليس XML صارمًا).

---

## جدول مقارنة مختصر

| المفهوم | الغرض الرئيسي | ملاحظات مهمة |
|---|---|---|
| **XML** | **تنسيق مُهيكل مع مخطّطات قوية (XSD)** | غنيّ بالتحقق/الامتداد؛ أثقل من JSON |
| JSON | تبادل بيانات خفيف للأغلب | أسهل للويب/JS؛ بلا مخطط صارم افتراضيًا |
| YAML | إعدادات بشرية القراءة | حساس للمسافات؛ أقل صرامة |
| ProtoBuf | ثنائي عالي الأداء | يحتاج تعريف `.proto` وتوليد كود |
| SOAP | بروتوكول رسائل يعتمد XML | عقود WSDL + WS-* |
| HTML | ترميز صفحات الويب | ليس XML بالضرورة (قد يكون HTML5 حرًا) |

---

## ملخص الفكرة  
**XML** قوي عندما تحتاج **عقودًا صارمة**، **مساحات أسماء**، و**تحقق XSD**.  
ابنه عبر **LINQ to XML/Serialization**، وفَعّل **التحقق**، واضبط **Namespaces/Encoding**—  
تحصل على تبادل بيانات متين وقابل للتطوّر بين الأنظمة. 
