---
layout: post
author: Rhotav
title: Ninja SBS Writeup ~ MalwareNinja CTF
---

Selamlar, bu yazıda MalwareNinja CTF'inde "Ninja Seviye Ölçme Yerleştirme Sınavı" kategorisinde yayınlanan NSÖYS adlı sorunun çözümünü yazacağım.

Soru yayınlandıktan birkaç dakika sonra flagi alarak fb attık yanlış hatırlamıyorsam soruyu bizimle beraber 2 takım çözmüştü. Harika bir ctf'ti özellikle ***pyarmor içermeyen*** bazı sorular çok eğlenceli ve öğreticiydi çözerken öğrendim :D

### Göz Gezdirelim

Sorumuz bir exe dosyası halinde içerisinde bir de "Samples" klasörü bulundurarak geldi.

![2](https://user-images.githubusercontent.com/54905232/103215367-31516400-4924-11eb-99d2-fd707644f310.png)
![3](https://user-images.githubusercontent.com/54905232/103215375-38787200-4924-11eb-9b3e-66eff931d9d1.png)

## EXE Dosyası Analiz

Dosyayı DIE (Detect It Easy) ile tarattığımızda olduğu bilgisi geliyor.

![4](https://user-images.githubusercontent.com/54905232/103215382-3f9f8000-4924-11eb-980f-626f79c28b18.png)

Daha detaylı analiz için x32dbg üzerinde başlatıp stringlerine bakacağım.

![1](https://user-images.githubusercontent.com/54905232/103215354-28609280-4924-11eb-9723-61319b195dee.png)

burdan dosyanın aslında pyInstaller ile packlenmiş bir py dosyası olduğunu söyleyebiliriz.

bunun için hemen "pyinstxtractor" kullanacağım. 

{% highlight python %}
PS C:\Users\root\Desktop> python .\pyinstxtractor.py .\Ninja_SBS\Ninja_SBS.exe
[+] Processing .\Ninja_SBS\Ninja_SBS.exe
[+] Pyinstaller version: 2.1+
[+] Python version: 38
[+] Length of package: 8686872 bytes
[+] Found 953 files in CArchive
[+] Beginning extraction...please standby
[+] Possible entry point: pyiboot01_bootstrap.pyc
[+] Possible entry point: pyi_rth_multiprocessing.pyc
[+] Possible entry point: pyi_rth__tkinter.pyc
[+] Possible entry point: question.pyc
[+] Found 225 files in PYZ archive
[+] Successfully extracted pyinstaller archive: .\Ninja_SBS\Ninja_SBS.exe

You can now use a python decompiler on the pyc files within the extracted directory
{% endhighlight %}

Şimdi extracted klasörüne gidip "question.pyc" dosyasını uncompyle6 kullanarak pyc to py işlemine sokalım.

![5](https://user-images.githubusercontent.com/54905232/103215414-51812300-4924-11eb-98b3-d1ec373c66da.png)

ehehehehh pyarmor. Pyarmor unpack için yazılmış harika yazılar var bunları EN ve TR olarak en aşağıda bulabilirsiniz ancak ben mantığını çözemedim yarışma esnasında kullanacağım diğer taktik bana daha kolay geldi açıkçası... Ama mutlaka buna vakit ayıracağım

## Çözelim

Pyarmor var. Bunu unpack edemiyorum dolayısıyla runtime string dumplayacağım şimdi buna bakalım.
Dosyayı açtığımız zaman bize Samples klasöründe bulunan zararlılar hakkında sorular soruyor ve en sonunda da 10 sorudan kaç doğru yaptığımızı söylüyor eğer 10/10 yapamadıysak flag döndürmüyor.

![6](https://user-images.githubusercontent.com/54905232/103215423-5940c780-4924-11eb-9520-713bc47c4585.png)

HxD 'yi açıp memory modunda Ninja_SBS.exe 'yi seçiyorum.

![7](https://user-images.githubusercontent.com/54905232/103215437-6067d580-4924-11eb-834d-ef0d8c4d5122.png)

Modülüm yüklendikten sonra CTRL+F kombinasyonu ile "You Failed" stringini aratıyorum.

![8](https://user-images.githubusercontent.com/54905232/103215441-66f64d00-4924-11eb-99c3-16defa57b302.png)

![9](https://user-images.githubusercontent.com/54905232/103215459-6fe71e80-4924-11eb-96d0-e6487b0a2f25.png)

ehehehe You Failed stringinin hemen altında YOU WIN stringini ve flagi görebiliyorum :

> "48 49 47 48 5f 53 4b 49 4c 4c 45 44 5f 55 4c 54 49 4d 41 54 45 5f 50 52 4f 5f 4e 49 4e 4a 41 5f 50 4c 55 53"

Hex to String işlemine sokalım

"HIGH_SKILLED_ULTIMATE_PRO_NINJA_PLUS"

> flag : malwareninja{HIGH_SKILLED_ULTIMATE_PRO_NINJA_PLUS}

### --- Unpacking PyArmor
> https://anal.school/2020/07/30/Unpacking-pyarmor-Soder-CTF/
> https://devilinside.me/blogs/unpacking-pyarmor
