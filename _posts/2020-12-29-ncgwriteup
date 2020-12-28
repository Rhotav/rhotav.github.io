---
layout: post
author: Rhotav
title: NCG Writeup ~ MalwareNinja CTF 
---

Selamlar, bu yazımda MalwareNinja CTF'inde "cringe" kategorisinde yayınlanan NCG adlı sorunun çözümünü yazacağım. Önceki yazı ile birleşik olmamasının sebebi soruların ben önceki writeup'ı yazdıktan sonra açılmış olması  ::(
(Soru yarışma sırasında patchlendi bir sebep yüzünden. Patchlenmiş soruya yazılmış bir writeup'tır.)

###Göz Gezdirelim
Soruyu indirdiğimiz zaman gelen tek şey bir .exe dosyası hemen DIE atarak analiz edelim.

![1](https://user-images.githubusercontent.com/54905232/103244241-6e8e1400-496d-11eb-9eae-a317bc990442.png)

>(Sorunun patchlenmemiş halinde normalde direkt pyInstaller ile packlenmiş şekilde geliyordu.)

Yupiii .NET geldi hemen dnSpy...

Dosyanın entry point noktasına gittikten sonra programda gömülü olan resources'lar içerisinde bazı xor decrypt işlemleri yapıldığını görüyoruz.
Bu işlemden sonra bir byte array'e atılıp Assembly.Load kullanılarak bir assembly'e dönüştürülüyor ve **export edilmiş type**'lar alınıyor. Tam olarak burası, çıkan resources'ın .dll dosyası olduğunu belirtir.
![2](https://user-images.githubusercontent.com/54905232/103244302-9d0bef00-496d-11eb-9fc5-93557918fb56.png)

Yapmamız gereken şey belli. İçerden çıkan .dll dosyasını analiz edebilmek için çalıştırma aşamasına bp atıp gelen array'i bellekten dump alacağız.
Satıra bp atıp debug işlemini başlatıyorum.

![3](https://user-images.githubusercontent.com/54905232/103244431-0d1a7500-496e-11eb-8797-11cd31e6b808.png)

Breakpoint kilitlendiği zaman Locals sekmesinden array değişkenini dump alıyoruz.

![4](https://user-images.githubusercontent.com/54905232/103244466-32a77e80-496e-11eb-9949-4814687c2d17.png)

### Kaydettiğimiz DLL Dosyası...

Şimdi dosyayı DIE atalım.

![5](https://user-images.githubusercontent.com/54905232/103244510-5074e380-496e-11eb-98dd-825bb4a76c24.png)

Bu da .NET :::):)):):):):):)

yapacağımız şey belli direkt dnSpy... Bu arada bir açıklama geçeyim her yazımda özellikle dnSpy kullanmamın sebebi aslında belli ama ben yine de söyleyeceğim **dnSpy'ın harika bir araç olduğunun tekrardan altını çizmek istiyorum** GitHub projesi arşivlendi ancak forkladığım projeden zaman buldukça geliştirmeye devam edeceğim...

![6](https://user-images.githubusercontent.com/54905232/103244607-9cc02380-496e-11eb-8d8a-70c1f221cf3b.png)

Çalıştırdığı kısım burası. Yukarıda EXE'ye baktığımız kısma tekrar dönerseniz "GetExportedTypes()[0]" kullanılmıştı. Burdaki [0] indisi constructor'ı belirtiyor ve argüman olarak verilen "typeof(Program)" kısmı da programın entry point type'ını belirtiyor.

ilk başta "Asm" adlı değişkende argüman olarak gönderilen Type'ın assembly'sinin atandığını görüyoruz. Geçelim direkt sonraki satırlarda ne yaptığını görmek için kullandığı fonksiyonlarla birlikte kopyalayarak visual studio üzerinde oluşturduğum projede çalıştırıp debuglayacağım.

{% highlight csharp %}
static void Main(string[] args)
{
    List<string> list = new List<string>();
    list.Add("BgAAAAFrAWICZGYBdAJzZgACZm4UZ2FtZV9mb250X3VwZGF0ZS5leGUBZQF5A3Jfaw5oaW5nZXBlYXJsLmRhdA==");
    list.Add("BgAAAAFrAWICZGYCYWQCc2YAAmZuCG1haW4uZXhlAWUBeQNyX2sNZGVmZWN0aW9uLmRhdA==");
    List<Dictionary<string, string>> list2 = new List<Dictionary<string, string>>();
    foreach (string s in list)
    {
        list2.Add(Deserialize(s));
    }

Console.ReadKey();

}

private static Dictionary<string, string> Deserialize(string s)
{
  byte[] buffer = Convert.FromBase64String(s);
	Dictionary<string, string> dictionary = new Dictionary<string, string>();
	using (MemoryStream memoryStream = new MemoryStream(buffer))
	{
	using (BinaryReader binaryReader = new BinaryReader(memoryStream, Encoding.UTF8))
	{
	  for (int i = binaryReader.ReadInt32(); i > 0; i--)
	  {
	    string key = binaryReader.ReadString();
	    string value = binaryReader.ReadString();
			dictionary.Add(key, value);
		}
	}
}
	return dictionary;
}
{% endhighlight %}

![7](https://user-images.githubusercontent.com/54905232/103244752-0d674000-496f-11eb-835e-1e811530c960.png)

![8](https://user-images.githubusercontent.com/54905232/103244794-2a9c0e80-496f-11eb-8c0e-7918d6bf56d4.png)

Herhangi bir decrypt işlemi yapmadığını da gördük. Yaptığı tek decrypt işlemi en baştaki xor işlemiydi o da geçtiğine göre artık resource dökümü alabiliriz.

İşimize yarayacak olan resource "hingepearl.dat" adlı olan. Save edelim dnSpy üzerinden ve DIE atalım.

![9](https://user-images.githubusercontent.com/54905232/103244894-72bb3100-496f-11eb-913e-9613566510ab.png)

Burayı kısa keseceğim. Gelen .dat dosyası direkt pyInstaller ile packlenmiş bir dosya bunun için pyinstxtractor kullanalım pyc dosyalarını alalım sonra da uncompyle6 kullanarak decompile edeceğiz.

{% highlight python %}
PS C:\Users\root\Desktop\upsi\hingepearl.dat_extracted> uncompyle6 -o ..\decompiled.py .\game_font_update.pyc
.\game_font_update.pyc --
# Successfully decompiled file
{% endhighlight %}

decompiled.py olarak kaydettim. Gidip bakalım main noktasında ne oluyormuş?

![10](https://user-images.githubusercontent.com/54905232/103245151-4e138900-4970-11eb-9a70-aab7795dc1f9.png)

Telegram kullanarak @kill_covid_r4t_bot botuna mesaj atalım ve flagi alalımm
