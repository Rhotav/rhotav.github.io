---
layout: post
author: Rhotav
title: Windows Internals Notlarım #1 ~ Process&Thread
---

Selamlar, büyük sınav için az vaktim kaldı. Bu yüzden de ne projelerime ne de yazılara devam edemiyorum. Çok kısıtlı vakitte yazdığım bu yazı gerçekten aceleye geldi bu yüzden hatalar fazlaysa affola... Yazının asıl yazılma amacı CreateProcess'in irdelenmesi üzerine olacaktı normalde detaysız olan yazı hoşuma gitmiyor ama ilk kısmı kısa tutmam gerekiyor CreateProcess'i iyice irdeleyebilmek için.

#### Bu Yazıda Kullanılanlar
 - WinDBG
 - Process Hacker
 - MSDN
 
Bu yazıyı "Windows Internals" (part1 pdf linkini en altta bulabilirsiniz) kitabındaki bir kısmın ve daha birçok kaynaktan derlenmiş bilgilerin türkçeleştirme ve ekleme yapılmış hali olarak düşünebilirsiniz.

## Process Internals
### Data Structures
Windows üzerinde her süreç bir "Yürütme Süreci" (EPROCESS - Executive Process) olarak temsil edilir.
WinDbg üzerinde ```dt nt!_eprocess``` komutu ile nt!_eprocess 'i dump edelim.
 > [dökümün tamamı için buraya tıklayın](https://paste.ubuntu.com/p/x5hvz6Y8Jj/)

    ntdll!_EPROCESS
       +0x000 Pcb              : _KPROCESS
       +0x298 ProcessLock      : _EX_PUSH_LOCK
       +0x29c UniqueProcessId  : Ptr32 Void
       +0x2a0 ActiveProcessLinks : _LIST_ENTRY
       +0x2a8 RundownProtect   : _EX_RUNDOWN_REF
       +0x2ac VdmObjects       : Ptr32 Void
       +0x2b0 Flags2           : Uint4B
       +0x2b0 JobNotReallyActive : Pos 0, 1 Bit
       (...)
       +0x660 SecurityDomain   : Uint8B
       +0x668 ParentSecurityDomain : Uint8B
       +0x670 CoverageSamplerContext : Ptr32 Void
       +0x674 MmHotPatchContext : Ptr32 Void
       +0x678 DynamicEHContinuationTargetsTree : _RTL_AVL_TREE
       +0x67c DynamicEHContinuationTargetsLock : _EX_PUSH_LOCK
       +0x680 DynamicEnforcedCetCompatibleRanges : _PS_DYNAMIC_ENFORCED_ADDRESS_RANGES 

  
 Bir EPROCESS, bir süreçle ilgili birçok özniteliği içermesinin yanı sıra, başka ilgili veri yapılarını içerir. Örneğin her süreç bir veya daha fazla "thread" yani iş parçacığı içerir (daha sonraki notlarda daha derine ineceğiz). Bunların her biri "Yürütme İş Parçacığı" (ETHREAD) olarak temsil edilir.
 - Thread:
 ![1](https://user-images.githubusercontent.com/54905232/121421621-05c4b200-c977-11eb-884b-3a0c31ae71cd.png)

> Bu kısımda EPROCESS ve bağlantılı diğer veri yapılarının PEB üzerinde bulunduğu, bazı yapıların "user-mode" kod üzerinden erişilebilir olmasını sağlamak amacıyla Process Address Space üzerinde bulunması, "Win32 Subsystem Process"(Csrss)'in her bir süreç için CSR_PROCESS adı verilen bir yapıyı süreç için sağlaması ve bunun detayı gibi konuların çok fazla detayına inmeyeceğim bu yazıda çok daha detaylı olarak process yapısını inceleyeceğimiz başka bir yazı gelebilir belki.

## Process Initalization Sequence (İşlem Tanımlama Sırası)
Özellikle Windows üzerinde (linux üzerinde çok iyi değilim o yüzden yorum yapamıyorum) reverse konusunu daha iyi anlayabilmek için bir Process(süreç) başladığında ne olur? nasıl başlar? gibi sorulara cevap aramalıyız.
> Şimdi bir sistemde süreç oluşturma sırasında yapılan işlemlerin kısa bir açıklamasını vereceğim adım adım.

- Yeni bir süreç objesi ve yeni bir adres alanı oluşturmak : Herhangi bir süreç bir Win32 API'si olan `CreateProcess`'e çağrı yaptığında; yeni bir süreç objesi oluşturulur ve süreç için RAM üzerinden alan tahsis edilir. (CreateProcess'i ayriyeten irdeleyeceğiz)

- `CreateProcess`, süreç için tahsis ettiği alan içerisinde `NTDLL.DLL` ve .EXE dosyasını birbiri ile eşler.
- `CreateProcess`, süreç için ilk iş parçacığını (thread) ve onun için stack alanını tahsis eder (RAM'den alan ayırır).
> Oluşturulan iş parçacığı oluşturulur yalnız çalıştırılmaz. Çalıştırılma işlemi şimdi yapılacak.
- Sürecin önceden oluşturulan ilk iş parçacığı bu aşamada çalıştırılır ve `NTDLL.DLL` arkasında `LdrpInitialize` adlı fonksiyon çalışır.
- `LdrpInitialize` fonksiyonunun temel amacı : birincil çalıştırılabilir dosyanın yani çalıştırdığımız dosyanın import table içerisindeki bütün fonksiyonları yinelemeli olarak atlayıp, dosyanın çalışabilmesi için gerekli olan ikincil dosyaları hafızaya eşler.
> Bu aşamaya kadar sürecimiz için hafızada bir alan açıldı. Bu alan içerisinde  ilk iş parçacığı oluşturuldu ve bu iş parçacığı için bir stack alanı açıldı. `LdrpInitialize` fonksiyonu ile exe dosyamızın çalışabilmesi için gerekli olan ikincil dosyalar hafızaya yüklendi ve eşlendi.
- Bu noktada kontrol, hafızaya statik olarak yüklenen DLL dosyalarının başlatılmasından sorumlu tutulan `LdrpRunInitializeRoutines` fonksiyonuna geçirilir. Başlatma işlemi her DLL'nin giriş noktasının `DLL_PROCESS_ATTACH` ile başlatmasıyla sürdürülür.
- Tüm DLL'ler başlatıldığında kontrol tekrardan `LdrpInitialize` fonksiyonuna geçer. `KERNEL32.DLL` içerisinden `BaseProcessStart` fonksiyonu çağırılır ve birincil dosyanın `WinMain` giriş noktası aranır ve çağırılır.
> Olabildiğince açıklamalı ve kısa anlatmaya çalıştım daha güzel ve detaylı gözükmesi için bir de şema çizmeye çalışacağım.
![2](https://user-images.githubusercontent.com/54905232/121745752-7ac9f000-cb0d-11eb-8009-b79fb7e83664.png)


## CreateProcess'i İrdeleyelim

En temelden en derine bir irdeleme yapacağımız için öncelikle CreateProcess hakkında temel bilgileri edinmek için MSDN sayfasına gidelim. [burası](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa)

{% highlight cpp %}
BOOL CreateProcessA(
  LPCSTR                lpApplicationName,
  LPSTR                 lpCommandLine,
  LPSECURITY_ATTRIBUTES lpProcessAttributes,
  LPSECURITY_ATTRIBUTES lpThreadAttributes,
  BOOL                  bInheritHandles,
  DWORD                 dwCreationFlags,
  LPVOID                lpEnvironment,
  LPCSTR                lpCurrentDirectory,
  LPSTARTUPINFOA        lpStartupInfo,
  LPPROCESS_INFORMATION lpProcessInformation
);
{% endhighlight %}
Bunu cpp ile programa dökelim disassemble falan edeceğiz.

{% highlight cpp %}
#include <iostream>
#include <Windows.h>
int main()
{
    std::cout << "Process Olusturuyorum!\n";
    STARTUPINFOW process_startup_info{ 0 };
    process_startup_info.cb = sizeof(process_startup_info);
    PROCESS_INFORMATION process_info{ 0 };
    wchar_t commandline_args[] = L"calc.exe";
    if (CreateProcessW(NULL, commandline_args, NULL, NULL, TRUE, 0, NULL, NULL, &process_startup_info, &process_info))
    {
        std::cout << "Process olustu.\n";
    }
    std::cin.get();
}
{% endhighlight %}

