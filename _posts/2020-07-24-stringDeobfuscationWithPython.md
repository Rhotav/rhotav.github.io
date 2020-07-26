---
layout: post
author: Rhotav
title: String Deobfuscation with Invoke ~ Python [TR]
---

Selamlar, bu yazımda "Crypto Obfuscator" adlı yazılımın string korumasını Python'da dnlib ve Reflection ikilisini kullanarak nasıl deobfuscate edebileceğimizi anlattım. Maksat değişiklik olsun...

## Dosya Analizi

Crypto Obfuscator'un deneme sürümü ile sadece string encryption işlemine soktuğumuz dosyayı dnSpy üzerine açalım.

Rastgele bir event'e gittiğimizde içinde bulunan stringlerin şifrelendiğini ve junk kodlar bıraktığını gördük (Junk kodların temizlenme işlemi bu yazıda yapılmayacak):
![Screenshot_1](https://user-images.githubusercontent.com/54905232/88428211-1de22000-cdfd-11ea-890e-19cf5fdb815f.png)

Anahtar değeri decryption methoduna parametre olarak gönderiyor ve geri dönen değer ile işini sürdürüyor.

### Yapılacaklar
- Python kütüphanelerinin kurulumu (ve dnlib.dll dosyasının dahil edilmesi)
- String decryption metodunun ve sınıfının otomatik olarak bulunması
- String encrypted yerleri saptamak ve INVOKE atarak decrypted değerin alınması
- Alınan decrypted değerin yerleştirilmesi
- Kaydetme

### Import ve Load

Aynı klasör içerisinde tutuyorum bütün dosyalarımı. Önce gerekli kütüphaneyi indirelim.

> pip install pythonnet

Reflection kütüphanelerini import edelim.

{% highlight python %}
import clr
from System.Reflection import Assembly, MethodInfo, BindingFlags
from System import Type
{% endhighlight %}

Main bloğunda dnlib.dll dosyamızı referans olarak ekleyelim. module ve assembly adlı değişkenlerimize de obfuscated uygulamayı load edelim.

{% highlight python %}
def DecryptStrings():
    MethodIdentifier()


def Save():
    module.Write("deobfuscated.exe")


def MethodIdentifier():
    pass


if __name__ == "__main__":
    clr.AddReference(r"DNLİB PATH")
    import dnlib
    from dnlib.DotNet.Emit import OpCodes
    global module # her fonksiyondan erişebilir olması için
    global assembly
    module = dnlib.DotNet.ModuleDefMD.Load("WindowsFormsApp3.exe")
    assembly = Assembly.LoadFrom("WindowsFormsApp3.exe")
    MethodIdentifier()
    DecryptStrings()
    Save()
{% endhighlight %}

> Neden ayrı ayrı iki modül yüklüyoruz konusuna ufak bir açıklama geçeyim. assembly, Reflection kütüphanesinden Invoke işlemi için yüklendi (dnlib'de bu özellik yok). module, düzenleme işlemleri için.

## Decryption Metodu

Bu kısımda sadece programa decryption metodunu bulduracağız. Zaten invoke işlemi ile decrypt edeceğimiz için decryption algoritmasını yazmamıza gerek yok.

dnSpy ile tekrar açıp analiz edelim, decryption metodunun nasıl ayırt edici özellikleri varmış bakalım.
Decryption metodumuz burada: 
![Screenshot_3](https://user-images.githubusercontent.com/54905232/88432077-33a71380-ce04-11ea-9788-3ca06917535d.png)

Şimdi programın okuyabileceği şekil olan IL kısmına geçelim. (Sağ Tık > Edit IL Instructions)
![Screenshot_4](https://user-images.githubusercontent.com/54905232/88432269-897bbb80-ce04-11ea-8f12-252681cbbf37.png)

En ayırt edici özellikler olarak methodun 
- IL instruction'larının uzunluğu
- 0. ve 2. indexteki OpCode

> IL uzunluğu = 107
> 0. OpCode = LdcI4
> 2. OpCode = Ldsfld

{% highlight python %}
def MethodIdentifier():
    global stringMethodName #1
    global invokeMethod #1
    eFlags = BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic #2
    for type in module.Types: #3
        for method in type.Methods:
            if not method.HasBody: pass #4
            if len(method.Body.Instructions) < 107: pass #5
            i = 0
            while i < len(method.Body.Instructions): #6
                if(method.Body.Instructions[i].IsLdcI4() and
                     method.Body.Instructions[i + 2].OpCode == OpCodes.Ldsfld): #7
                    stringMethodName = str(method.Name)
                    stringTypeName = str(type.Name)
                    for types in assembly.GetTypes(): #8 ...
                        if types.Name == stringTypeName:
                            classInstance = types
                            break

                    for methods in classInstance.GetMethods(eFlags):
                        if methods.Name == stringMethodName:
                            invokeMethod = methods
                            return
                    break
                i += 1
{% endhighlight %}

Kodumuzu açıklayalım :

- 1) Diğer fonksiyonlarımızın da erişebilmesi için iki değişkenimizi global anahtar kelimesi ile tanımlıyoruz.
- 2) https://docs.microsoft.com/en-US/dotnet/api/system.reflection.bindingflags?view=netcore-3.1
- 3) Yüklediğimiz modülün (yani obfuscated binary'nin) Type'ları yani Class'ları arasında dolaşıyoruz. Hemen altında da Type'ların methodları içerisinde...
- 4) Kodun devamında method.BODY.INSTRUCTIONS diyoruz. Dolayısıyla hata çıkartmaması için eğer bir body'si yoksa atla işlemini sağlıyoruz.
- 5) Evet, burda ilk koşulumuz olan eğer bulunduğun method'un IL sayısı 107'den az ise atla işlemini sağlıyoruz.
- 6) İkinci koşulumuzu sağlamak üzere IL'lerin arasında dolaşıyoruz.
- 7) Eğer bulunduğun instruction LdcI4 (LdcI4.1, LdcI4.2 diye gidiyor, basite indirgemek için IsLdcI4() fonksiyonunu kullanıyoruz.) ve bundan iki sonra Ldsfld OpCode'u varsa ...
- 8) classInstance atama işlemini daha sonrasında decryption metodunun bulunabilmesi için atıyoruz. "for type in module.Types" ile yaptığımızın aynısının Reflection ile olan hali. Invoke edilecek metodu yani Decryption metodunu "MethodInfo" türünde alabilmek adına burdaki işlemleri yapıyoruz.

## Decryptor Method

Decryption methodumuzu bulduktan sonra Decrypt metodunun yapacağı tek şey şifrelenmiş alanları bularak, işe yarayan parametreyi decryption metoduna gönderip invoke ediyoruz. Çıkan sonucu yerine yerleştiriyoruz.

Yapmamız gereken ilk şey obfuscated bölümleri tanımak. dnSpy'ı açıp bakıyoruz.
![Screenshot_5](https://user-images.githubusercontent.com/54905232/88434258-7ff45280-ce08-11ea-95a1-ef0682f21df7.png)

Obfuscated bir yerin IL instruction'larına baktığımızda sadece metoda sayısal bir ifade gönderildiğini göreceğiz. 
Ama birden çok metot böyle çalışabilir. İşte bu kısımda önceden tanımladığımız "stringMethodName" değişkeni işimize yarayacak. "call" OpCode'lu kısmın Operand'ında eğer atadığımız "stringMethodName" geçiyorsa burası obfuscated bir alandır diyebiliriz.

O zaman bunun ayırt edici özellikleri :
- Bulunduğun index LdcI4 ve bundan bir sonraki instruction OpCode'u call ise
- call OpCode'una sahip instruction'ının Operand'ı stringMethodName'i içeriyorsa

Kodu yazalım :

{% highlight python %}
    decryptedStrings = 0 # Ne kadar stringi decrypt ettiğini görelim burda
    for type in module.Types: # Klasik dolaşma işlemleri
        if not type.HasMethods: pass # Eğer metodu yoksa atla
        
        for method in type.Methods:
            if not method.HasBody: pass
            i = 0
            operText = ""
            while i < len(method.Body.Instructions):
                operText = str(method.Body.Instructions[i].Operand).encode()
                if(method.Body.Instructions[i].OpCode == OpCodes.Call and
                        operText.find(str(stringMethodName).encode()) != -1):
.........
{% endhighlight %}

Şimdi yapmamız gereken şey argüman olarak göndereceğimiz şeyin değerini çekmek ve Invoke edip gelen değeri yerine yerleştirmek :

{% highlight python %}
  keyValue = method.Body.Instructions[i - 1].GetLdcI4Value()
  result = str(invokeMethod.Invoke(None, [keyValue]))
  method.Body.Instructions[i - 1].OpCode = OpCodes.Nop
  method.Body.Instructions[i].OpCode = OpCodes.Ldstr
  method.Body.Instructions[i].Operand = result
  decryptedStrings += 1
i += 1
{% endhighlight %}

Gerekli dosyalar ve kodun tamamı :
> https://github.com/Rhotav/Python-Method-Invoker

