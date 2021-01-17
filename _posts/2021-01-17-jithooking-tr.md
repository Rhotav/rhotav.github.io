---
layout: post
author: Rhotav
title: .NET Just-In-Time (JIT) Compiler Hooking
---

Selamlar. Uzun zamandır yazı yazmamamın(writeuplar hariç) nedeni aslında bu yazıydı. 
Bir süredir .NET Framework ile yazılmış bir yazılımın JIT ile iletişime geçişini ve bunun "hooklanması" hakkında çalışıyordum. Sonunda bu yazı ortaya çıktı.

Yazıma geçmeden önce dnSpy'ın, .NET dosyalar üzerinde çalışırken bizim işimize en fazla yarayacak yazılım olduğunun ve *işini en iyi yapan* .NET Decompiler/Debugger olduğunu kalın harflerle yazmak istiyorum.

### Yazı Amacı
Bu yazıda bir JIT hooker yazmayı planlıyoruz (C#, C/C++ kullanarak). Peki bunu neden yapıyoruz?

----

Bizim amacımız aslında korumaya yönelik (Anti-Tamper)... .NET Framework ile yazılmış bir dosyayı dnSpy gibi decompile yazılımları ile basitçe decompile/ildasm edebiliriz ama bu her zaman kusursuz şekilde olmaz.
Hedef yazılımın üzerinde bir koruma varsa (örn. ConfuserEx) ve bu korumanın da method yapısını bozmak gibi bir özelliği varsa (Anti-Tamper vb.) dnSpy dosyayı decompile edemez çünkü method IL'leri bozulmuş olacak.

----
### Anti-Tamper Örneği
dnSpy dosyayı okuyamıyor çünkü methodlar bozuk. Uygulamanın düzgün şekilde çalışabilmesi için modül constructor'ı olan .cctor noktasında fonksiyonları düzeltmesi gerekiyor.
> Tabii ki her korumada böyle olmak zorunda değil örneğin her class'ın kendi içeriğini, class'a ait constructor'da da çözdürebilir burada anlatmak istediğim nokta; çalışacak ilk noktada düzeltmesi gerektiği.

Anti Tamper olayının daha iyi kavranması için ConfuserEx üzerinde bir örnek göstereceğim : 
Bir uygulamamız var. Bu uygulamanın "Main()" fonksiyonunu decompile etmek için dnSpy üzerinden modüle sağ tıklayıp entry point noktasına gidiyorum.

![1](https://user-images.githubusercontent.com/54905232/103443089-c0d07d00-4c6c-11eb-986b-516b78c4903a.png)

Entry Point noktasında hiçbir şey gözükmüyor? Bir de fonksiyon adına sağ tıklayıp Edit Method Body... seçeneğinden IL kodlarına bakmayı deneyelim.

![2](https://user-images.githubusercontent.com/54905232/103443109-e2c9ff80-4c6c-11eb-823c-745924688a91.png)

:D Program normal bir şekilde açılıp çalışıyor. O halde bu programın Main() fonksiyonundan dahi önce çalışan bir fonksiyonda program fonksiyonlarını düzelten bir fonksiyon çalışıyor.

Bu da Constructor olacak o hâlde noktam .cctor  hemen modüle sağ tıklayıp .cctor noktasına gidelim.
![3](https://user-images.githubusercontent.com/54905232/103443135-33415d00-4c6d-11eb-94e8-007c3d8a4bce.png)

(Anti-Tamper korumasının detaylı analizini farklı bir yazıda yaparız şimdilik kalsın). Eğer fonksiyon çalıştıktan hemen sonra bütün fonksiyonlar düzeliyorsa normal olarak decompile edilecek anlamına geliyor.

Yapacağımız şey belli dnSpy üzerinde tespit ettiğimiz antitamper fonksiyonuna breakpoint atalım ve o breakpoint çalıştığı zaman "Step Over(F10)" seçeneğinden devam edelim. Debugger tekrar duracak bu aşamada "Modules" sekmesinden programı dumplayalım.

![5](https://user-images.githubusercontent.com/54905232/103443295-b44d2400-4c6e-11eb-8ffa-4df3b4c79e38.png)

Dumpladığımız dosyayı dnSpy'a atıp Main() fonksiyonuna gittiğimizde :
![6](https://user-images.githubusercontent.com/54905232/103443308-d9da2d80-4c6e-11eb-9216-d3a91af6cfcf.png)

Anti-Tamp fonksiyonu bütün fonksiyonları eski haline çevirmiş. Ancak bu aşamada bir problem var dosya çalışmayacaktır.
Bunun nedeni anti-tamper fonksiyonunu silmemiş olmamız. Dosya çalıştığı zaman method body'leri alakalı keylerle tekrar aksiyona sokunca method bodyler tekrardan bozuluyor bu sefer yazılım exception'a düşüp kapanıyor. Bunun için cctor'da çalışan fonksiyonu noplayıp kaydedebiliriz burayı geçiyorum.

## API Hooking

*API :* Application Programming Interface, işletim sisteminin kendisi ile iletişime geçmek isteyen uygulamalara sunduğu bir dizi fonksiyona verilen isimdir.

*API Hooking* ise iletişim kurmak istediğimiz API ile uygulama arasına girerek API'ye giden tüm istekleri, yeniden programımız içerisinde yazdığımız API'yi birebir taklit eden sahte fonksiyona yönlendirmek oluyor. Bunu örnekler ile daha net anlatacağım.
----

### Örnek
Bunu daha net kavramak için daha kolay bir örnek üzerinden API Hooking'i göstereyim.
Bu örnekte "user32.dll" içerisinde bulunan "MessageBoxA" API'sini hooklayacağız.

#### Bir Hooking Nasıl Gerçekleşir?
API Hooker yazarken izleyeceğimiz yol şu şekilde olacak:

- Öncelikle hooklayacağımız API'yı tamamen taklit edebilecek sahte bir fonksiyon yazalım. "MessageBoxA" API'sini taklit edecek fonksiyonu yazabilmek için API'nin MSDN sayfasına giderek fonksiyonu inceleyelim.

Bunu beraber yapalım öncelikle [buradan](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-messageboxa) MSDN dökümantasyonuna gidelim.
{% highlight cpp %}
int MessageBoxA(
  HWND   hWnd,
  LPCSTR lpText,
  LPCSTR lpCaption,
  UINT   uType
);
{% endhighlight %}

Bunu Visual Studio üzerinde açtığım projede tekrar yazıyorum.
> Fonksiyonu yazmadan önce `#include <iostream> #include <Windows.h>` satırlarını eklemeyi unutmayalım.
{% highlight cpp %}
int hookedMessageBox(HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType) {
    //...
	return MessageBoxA(0, "Not Hooked", "Normal", 0);
}
{% endhighlight %}

Fonksiyonumuz tamamdır. Gelen argümanları bana bildirmesi için 
{% highlight cpp %}
std::cout << "Yakalandi \"lpText\": " << lpText << std::endl;
std::cout << "Yakalandi \"lpCaption\": " << lpCaption << std::endl;
{% endhighlight %}

Satırlarını da ekleyip bütün işi bitiriyorum.
----

- Sahte fonksiyonumuzu yazdık şimdi hooklayacağımız API'nin process'imiz üzerindeki adresini tespit edelim.
> Bunu ve gerisini manual hooking yaptıktan hemen sonra yapacağız.

----
> Buraya kadar her şey tamam. Şimdi yapacağımız şey byteları çalışma zamanında düzenlemek. API'nin adresini almıştık dolayısıyla API ile iletişime geçildiği zaman sürecin nerden yönetildiğini biliyoruz. Yapmamız gereken tek şey okunan byteları editleyerek süreci kendi fonksiyonumuza "jumplatmak" (atlatmak) jmp.

- Değiştireceğimiz byteları yedeklemek (bir byte array e kopyalamak). 
> Hemen nedenini de açıklayalım. Yazdığımız taklit fonksiyon (bizim isteğimize göre) gerçek API ile iletişim kurması gerektiğinde sürekli patchlenmiş API fonksiyonuna geleceği için kendi içerisinde sonsuz döngüye girecek. Bunu engellemek için gerçekten iletişime geçmesi gerektiğinde hook işlemini kaldırmamız lazım yani orjinal byteları yerine yazmamız lazım ki işini yerine getirebilsin.

----

### Manual Hooking with x32dbg
İzleyeceğimiz bütün yol bu kadar. Şimdi daha iyi anlaşılması için x32dbg ile örnek bir dosya üzerinden manual hooking işleminin nasıl yapıldığına bakalım. 

- Öncelikle yazdığımız dosyanın kaynak kodu bu şekilde :
{% highlight cpp %}
#include <iostream>
#include <Windows.h>

int hookedMessageBox(HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType) {

	std::cout << "Yakalandi \"lpText\": " << lpText << std::endl;
	std::cout << "Yakalandi \"lpCaption\": " << lpCaption << std::endl;

	return MessageBoxA(0, "Not Hooked", "Normal", 0);
}

int main()
{
	MessageBoxA(0, "Not Hooked", "Normal", 0);
	std::cin.get(); //programı durdurmak için
	return 0;
}
{% endhighlight %}

x32dbg üzerinde dosyayı çalıştırdıktan sonra entry noktasına gelince kesilecektir.
> Eğer kesilmezse Symbols sekmesinden .exe modülünü seçip "_main" fonksiyonuna gidilebilir.

![7](https://user-images.githubusercontent.com/54905232/103454779-5fee8680-4cf8-11eb-8690-1ea25b8a3972.png)

- ilk olarak gidip sahte fonksiyonumuzun adresini kopyalayalım. "Symbols" sekmesinden "sample".exe modülünü seçip sağ kısma fonksiyonumun ismini yazıp aratıyorum ve çıkan sonucun adresini kopyalayıp bir yere not ediyorum.

![8](https://user-images.githubusercontent.com/54905232/103454846-e4410980-4cf8-11eb-8eb3-656ddaef330f.png)

- "Symbols" sekmesine tekrar gidip solda görünen modüllerden "user32.dll" i seçiyorum sağa bu modülün içerdiği export fonksiyonları gelecektir. Buradan MessageBoxA fonksiyonuna gidelim. 

![9](https://user-images.githubusercontent.com/54905232/103454867-0dfa3080-4cf9-11eb-8e04-63e4c963eecd.png)

----
- Burda gördümüz ilk satırı jmp 0xADRES şeklinde düzenleyeceğiz ancak bu düzenleme işlemini dump üzerinde yapacağız. 
> Neden Assemble edip düzenlemiyoruz da dump üzerinden düzenliyoruz? Çünkü hem otomatize hooker yazarken bunu yapacağımız için hem de yapıyı tamamı ile bozacağı için. Bunu kendiniz de deneyebilirsiniz eğer direkt olarak jmp opcode'unu koyup operand kısmına da adresimizi yazarsanız 1. atlatmada problem çıkmayacak ancak düzenlemeye çalıştığınızda iyice şaşacak disasm kısmı...

`jmp 0xADRES` şeklinde düzenlemek yerine 
`push 0xADRES
ret`
şeklinde düzenleyeceğim [neden?](https://stackoverflow.com/questions/30916768/differences-between-call-pushret-and-pushjump-in-assembly)

![10](https://user-images.githubusercontent.com/54905232/103455544-10f81f80-4cff-11eb-9ceb-753a67abfbcc.png)

Uygulamam tam olarak MessageBoxA fonksiyonunun adresinde durunca (bp atıyoruz buraya öncesinden) `MOV EDI, EDI` satırına sağ tıklayıp dökümde takip ediyoruz.

> Bu arada sağ kısımda stdcall(winapi çağırılırken kullanılan) MessageBoxA fonksiyonuna gönderdiğimiz argümanları stack üzerinde görebilirsiniz. 

![11](https://user-images.githubusercontent.com/54905232/103455588-764c1080-4cff-11eb-8e03-d20a9b86e3e7.png)

Önce [buraya](https://defuse.ca/online-x86-assembler.htm#disassembly) gidip istediğimiz asm satırlarının byte karşılığını alalım.

`0:  68 10 1c e9 00  push   0xe91c10
5:  c3      ret`

`68 10 1C E9 00 C3`

> Yalnız buraya dikkat! Düzenleme işlemine geçmeden önce ne kadar byte düzenleyeceksek orjinal yedeğini almamız gerekiyor. Bizim düzenleyeceğimiz byte sayısı 6 byte dolayısıyla ilk 6 byte'ı yedekleyelim.

`8B FF 55 8B EC 83`

![12](https://user-images.githubusercontent.com/54905232/103455859-d3e15c80-4d01-11eb-9994-cc5b3e873ebf.png)

Düzenleyeceğimiz kadar byte seçip CTRL+E kombinasyonu veya resimde gördüğümüz yolu takip ederek edit panelini açalım.
![13](https://user-images.githubusercontent.com/54905232/103455894-1acf5200-4d02-11eb-95ac-eeab1c9cbfdc.png)

Hex kısmında en sola tıklayıp kopyaladığımız byteları yapıştıralım.

![14](https://user-images.githubusercontent.com/54905232/103455970-c678a200-4d02-11eb-85d6-b79be94edd07.png)

![15](https://user-images.githubusercontent.com/54905232/103456061-b1504300-4d03-11eb-8637-c3004bf0b9ce.png)

Satırımız bu hale dönecek. Şimdi bir kere devam ettiriyorum start buttonundan breakpoint'e tekrar çarpınca program ne çıktı vermiş diye bakıp hook işlemini kaldıracağım.

![16](https://user-images.githubusercontent.com/54905232/103456074-d9d83d00-4d03-11eb-8518-66817e2044b6.png)

vuhuuu çıktımız geldi. Şimdi asıl messageBox işini yapabilmesi için byte'ları yedeklediğimiz bytelara yani eski haline döndürelim.
![17](https://user-images.githubusercontent.com/54905232/103456137-6aaf1880-4d04-11eb-9ae7-55c84c5c45a1.png)

![18](https://user-images.githubusercontent.com/54905232/103456142-7f8bac00-4d04-11eb-8e2d-f3176b5aa8c6.png)

Evet, Manual hooking bu kadardı şimdi işi otomatizeye dökelim...

![21](https://user-images.githubusercontent.com/54905232/103456326-0db46200-4d06-11eb-83c9-1f35d6eeb28a.png)

>[C# İmplementasyonu](https://gist.github.com/Rhotav/4a9adeb322bdcf753eb3c46eb3546ca4)

{% highlight cpp %}
#include <iostream>
#include <Windows.h>

FARPROC messageBoxAddress;
char backup[6] = { 0 };
SIZE_T written = 0;

void unHook() {
	WriteProcessMemory(GetCurrentProcess(), (LPVOID)messageBoxAddress, backup, sizeof(backup), &written);
}

int hookedMessageBox(HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType) {

	std::cout << "Yakalandi \"lpText\": " << lpText << std::endl;
	std::cout << "Yakalandi \"lpCaption\": " << lpCaption << std::endl;
	unHook();
	return MessageBoxA(0, lpText, lpCaption, 0);
}

void installHook() {
	void* hookedMessageBoxAddress = &hookedMessageBox; //Yazdığımız sahte fonksiyonun adresini aldık.
	char patch[6] = { 0 }; //Patch edeceğimiz byte'ları yazıyoruz örn: 68 10 1C E9 00 C3
	memcpy_s(patch, 1, "\x68", 1);
	memcpy_s(patch + 1, 4, &hookedMessageBoxAddress, 4);
	memcpy_s(patch + 5, 1, "\xC3", 1);

	WriteProcessMemory(GetCurrentProcess(), (LPVOID)messageBoxAddress, patch, sizeof(patch), &written);
}

int main(void)
{
	HINSTANCE library = LoadLibraryA("user32.dll");
	messageBoxAddress = GetProcAddress(library, "MessageBoxA"); //API adresini bulduk.
	SIZE_T bytesRead = 0; //ReadProcessMemory fonksiyonu için gerekli. MSDN dökümantasyonunu ziyaret edin.
	ReadProcessMemory(GetCurrentProcess(), messageBoxAddress, backup, 6, &bytesRead);

	installHook();
	MessageBoxA(0, "AA MERHABAA", "bakalim ne olacak", 0);
	return 0;
}
{% endhighlight %}


## JIT HACKING

Geldik yazımızın ana konusuna. Bir .NET yazılımının yapısını tam olarak burda yazamayacağım onu da farklı bir yazıya bırakalım.

----

Normal API Hooking işleminde hangi yolu izliyorsak aynı yolu izleyerek compileMethod() fonksiyonunu sahte fonksiyonumuza yönlendirip gelen ILBody'i almak ve yazdırmak.
> .NET Dilleri .NET bytecode'ları şeklinde compile edilir. Runtime şekilde CLR sanal makinesinde yorumlanır ve JIT ile çalıştırılır. JIT içerisindeki compileMethod() fonksiyonu, gelen ILBody'i native dile çevirmek için kullanılıyor...

Bu işlem eğer hedef uygulamanın framework sürümü 4.0'dan yüksekse "clrjit.dll", eğer düşükse "mscorjit.dll" içerisinde runtime şekilde yapılıyor.
Yazacağımız örnek dosyanın framework sürümü de 4.0'dan yüksek olduğu için işlemlerimizi clrjit.dll üstünden yürüteceğiz. 

----

CFF Explorer yardımı ile clrjit.dll dosyasının hangi fonksiyonları extern ettiğine bakalım detaylı analizi GitHub reposu üzerinden yapacağız.
![Screenshot_1](https://user-images.githubusercontent.com/54905232/104369067-f8bda700-552d-11eb-864e-15af6e52fcc7.png)
2 adet fonksiyon extern ediyor "getJit" ve "jitStartup" 
Hangisini hooklayacağımız konusuna karar verebilmek için repoyu inceleyelim.
[dotnet/runtime](https://github.com/dotnet/runtime/blob/eb03112e4e715dc0c33225670c60c6f97e382877/src/coreclr/inc/corjit.h#L93) 

![Screenshot_2](https://user-images.githubusercontent.com/54905232/104371133-ac279b00-5530-11eb-8a67-30bc2ad5404a.png)

93. satırda extern ettiği "getJit" fonksiyonunu 88. satırda da jitStartup fonksiyonunu görüyoruz. jitStartup fonksiyonu void bir fonksiyon iken getJit ICorJitCompiler* döndürüyor. Yapıyı inceleyelim.
{% highlight cpp %}
extern "C" ICorJitCompiler* __stdcall getJit();

// #EEToJitInterface
// ICorJitCompiler is the interface that the EE uses to get IL bytecode converted to native code. Note that
// to accomplish this the JIT has to call back to the EE to get symbolic information.  The code:ICorJitInfo
// type passed as 'comp' to compileMethod is the mechanism to get this information.  This is often the more
// interesting interface.
//
//
class ICorJitCompiler
{
public:
    // compileMethod is the main routine to ask the JIT Compiler to create native code for a method. The
    // method to be compiled is passed in the 'info' parameter, and the code:ICorJitInfo is used to allow the
    // JIT to resolve tokens, and make any other callbacks needed to create the code. nativeEntry, and
    // nativeSizeOfCode are just for convenience because the JIT asks the EE for the memory to emit code into
    // (see code:ICorJitInfo.allocMem), so really the EE already knows where the method starts and how big
    // it is (in fact, it could be in more than one chunk).
    //
    // * In the 32 bit jit this is implemented by code:CILJit.compileMethod
    // * For the 64 bit jit this is implemented by code:PreJit.compileMethod
    //
    // Note: Obfuscators that are hacking the JIT depend on this method having __stdcall calling convention
    virtual CorJitResult __stdcall compileMethod (
    ICorJitInfo *comp,       /* IN */
    struct CORINFO_METHOD_INFO  *info,       /* IN */
    unsigned /* code:CorJitFlag */   flags,  /* IN */
    BYTE**nativeEntry,       /* OUT */
    ULONG       *nativeSizeOfCode    /* OUT */
    ) = 0;
...
{% endhighlight %}


compileMethod fonksiyonunu bulduk. Hedefimiz getJit(). Yorum satırlarında yazılmış kısımları da özetleyerek compileMethod'un amacını iyice oturtalım.

> compileMethod, JIT derleyicisinden bir method için native kod oluşturmasını isteyen ana fonksiyondur. Derlenecek method 'info' parametresinde "CORINFO_METHOD_INFO" struct yapısı olarak geçirilir.

Bu işlem için de izleyeceğimiz yolu yazalım. Yazımızın başında yaptığımız API hooking işleminden birazcık farklı. Burada byte'ları doğrudan düzenlemeyeceğiz. getJit() fonksiyonunun döndürdüğü pointer, bir [VTable](https://en.wikipedia.org/wiki/Virtual_method_table) döndürüyor bizim yapacağımız şey de bu VTable'ın gösterdiği ilk pointer'ı patchlemek, kendi fonksiyonumuzun adresi ile değiştirmek.
> Çünkü VTable'daki ilk pointer compileMethod fonksiyonumuzu işaret ediyor.

CORINFO_METHOD_INFO yapısını da bırakalım şöyle:
{% highlight cpp %}
struct CORINFO_METHOD_INFO
{
    CORINFO_METHOD_HANDLE       ftn;
    CORINFO_MODULE_HANDLE       scope;
    BYTE *      ILCode;
    unsigned    ILCodeSize;
    unsigned    maxStack;
    unsigned    EHcount;
    CorInfoOptions      options;
    CorInfoRegionKind   regionKind;
    CORINFO_SIG_INFO    args;
    CORINFO_SIG_INFO    locals;
};

struct CORINFO_SIG_INFO
{
    CorInfoCallConv callConv;
    CORINFO_CLASS_HANDLE    retTypeClass;   // if the return type is a value class, this is its handle (enums are normalized)
    CORINFO_CLASS_HANDLE    retTypeSigClass;// returns the value class as it is in the sig (enums are not converted to primitives)
    CorInfoType     retType : 8;
    unsignedflags   : 8;    // used by IL stubs code
    unsignednumArgs : 16;
    struct CORINFO_SIG_INST sigInst;  // information about how type variables are being instantiated in generic code
    CORINFO_ARG_LIST_HANDLE args;
    PCCOR_SIGNATURE pSig;
    unsignedcbSig;
    CORINFO_MODULE_HANDLE   scope;  // passed to getArgClass
    mdToken token;

    CorInfoCallConv     getCallConv()       { return CorInfoCallConv((callConv & CORINFO_CALLCONV_MASK)); }
    boolhasThis()   { return ((callConv & CORINFO_CALLCONV_HASTHIS) != 0); }
    boolhasExplicitThis()   { return ((callConv & CORINFO_CALLCONV_EXPLICITTHIS) != 0); }
    unsigned    totalILArgs()       { return (numArgs + hasThis()); }
    boolisVarArg()  { return ((getCallConv() == CORINFO_CALLCONV_VARARG) || (getCallConv() == CORINFO_CALLCONV_NATIVEVARARG)); }
    boolhasTypeArg(){ return ((callConv & CORINFO_CALLCONV_PARAMTYPE) != 0); }
};
{% endhighlight %}

### Local JIT Hook

Adımlarımız : 
- Clrjit.dll için DLL Import işlemi ve extern ettiği getJit fonksiyonunu almak.
{% highlight csharp %}
[DllImport("Clrjit.dll", CallingConvention = CallingConvention.StdCall, PreserveSig = true)]
static extern IntPtr getJit();
{% endhighlight %}

- getJit fonksiyonumuzun döndürüğü IntPtr değerini bir değişkene alalım ve içerisinde tuttuğu ilk pointer değerini okuyalım.
{% highlight csharp %}
var vTable = getJit(); //Döndürdüğü değeri vTable adlı değişkene tanımladık.
var compileMethodPtr = Marshal.ReadIntPtr(vTable); //vTable'ın ilk sanal fonksiyonunu okuyup compileMethodPtr adlı değişkene attık.
{% endhighlight %}

- Burdan sonrası için önce sahte fonksiyonumuzu hazırlamamız gerekiyor. Öncelikle tamamen taklit edebilecek bir delegate hazırlamamız gerekiyor. Bu işlemleri farklı bir class üzerinde yapacağım.
{% highlight csharp %}
[UnmanagedFunctionPointer(CallingConvention.StdCall, SetLastError = true)]
public unsafe delegate int delCompileMethod(
IntPtr thisPtr, [In] IntPtr corJitInfo, [In] CorMethodInfo* methodInfo, CorJitFlag flags,
    [Out] IntPtr nativeEntry, [Out] IntPtr nativeSizeOfCode);
{% endhighlight %}
(SJITHook Kullanılabilir)
Bunu yaptıktan sonra kullandığımız bazı şeyler hata verecek hemen struct yapılarını da alalım corjit.h ve corinfo.h içerisinden.

{% highlight csharp %}
[StructLayout(LayoutKind.Sequential)]
public unsafe struct CorinfoSigInst
{
    public uint classInstCount;
    uint dummy;
    public IntPtr* classInst;
    public uint methInstCount;
    uint dummy2;
    public IntPtr* methInst;
}

public enum CorJitFlag
{
    CORJIT_FLG_SPEED_OPT = 0x00000001,
    CORJIT_FLG_SIZE_OPT = 0x00000002,
    CORJIT_FLG_DEBUG_CODE = 0x00000004, // generate "debuggable" code (no code-mangling optimizations)
    CORJIT_FLG_DEBUG_EnC = 0x00000008, // We are in Edit-n-Continue mode
    CORJIT_FLG_DEBUG_INFO = 0x00000010, // generate line and local-var info
    CORJIT_FLG_LOOSE_EXCEPT_ORDER = 0x00000020, // loose exception order
    CORJIT_FLG_TARGET_PENTIUM = 0x00000100,
    CORJIT_FLG_TARGET_PPRO = 0x00000200,
    CORJIT_FLG_TARGET_P4 = 0x00000400,
    CORJIT_FLG_TARGET_BANIAS = 0x00000800,
    CORJIT_FLG_USE_FCOMI = 0x00001000, // Generated code may use fcomi(p) instruction
    CORJIT_FLG_USE_CMOV = 0x00002000, // Generated code may use cmov instruction
    CORJIT_FLG_USE_SSE2 = 0x00004000, // Generated code may use SSE-2 instructions
    CORJIT_FLG_PROF_CALLRET = 0x00010000, // Wrap method calls with probes
    CORJIT_FLG_PROF_ENTERLEAVE = 0x00020000, // Instrument prologues/epilogues
    CORJIT_FLG_PROF_INPROC_ACTIVE_DEPRECATED = 0x00040000,
    // Inprocess debugging active requires different instrumentation
    CORJIT_FLG_PROF_NO_PINVOKE_INLINE = 0x00080000, // Disables PInvoke inlining
    CORJIT_FLG_SKIP_VERIFICATION = 0x00100000,
    // (lazy) skip verification - determined without doing a full resolve. See comment below
    CORJIT_FLG_PREJIT = 0x00200000, // jit or prejit is the execution engine.
    CORJIT_FLG_RELOC = 0x00400000, // Generate relocatable code
    CORJIT_FLG_IMPORT_ONLY = 0x00800000, // Only import the function
    CORJIT_FLG_IL_STUB = 0x01000000, // method is an IL stub
    CORJIT_FLG_PROCSPLIT = 0x02000000, // JIT should separate code into hot and cold sections
    CORJIT_FLG_BBINSTR = 0x04000000, // Collect basic block profile information
    CORJIT_FLG_BBOPT = 0x08000000, // Optimize method based on profile information
    CORJIT_FLG_FRAMED = 0x10000000, // All methods have an EBP frame
    CORJIT_FLG_ALIGN_LOOPS = 0x20000000, // add NOPs before loops to align them at 16 byte boundaries
    CORJIT_FLG_PUBLISH_SECRET_PARAM = 0x40000000,
    // JIT must place stub secret param into local 0.  (used by IL stubs)
};

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct CorMethodInfo
{
    public IntPtr methodHandle;
    public IntPtr moduleHandle;
    public IntPtr ilCode;
    public uint ilCodeSize;
    public ushort maxStack;
    public ushort EHCount;
    public uint corInfoOptions;
    public CorinfoSigInst args;
    public CorinfoSigInst locals;
}
{% endhighlight %}

- Şimdi class ile işimiz bitti şimdi main kısmına geçip işlemlerimizi yapalım gerekli bütün açıklamalar yorum satırlarında yazıyor ama olur da bir yerde takılırsanız bana ulaşabilirsiniz
{% highlight csharp %}
unsafe static void Main(string[] args)
{
    uint old;
    Context.delCompileMethod hookedCompileMethod = HookedCompileMethod;
    var vTable = getJit(); //ICorJitCompiler pointer'ı alındı
    var compileMethodPtr = Marshal.ReadIntPtr(vTable); //İçerisindeki ilk pointer okundu.
    OrigCompileMethod = (Context.delCompileMethod)Marshal.GetDelegateForFunctionPointer(Marshal.ReadIntPtr(compileMethodPtr), typeof(Context.delCompileMethod)); //Orjinal compileMethod fonksiyonu Delegate türünde yüklendi.
    //Bizim iznimizde tekrardan çalıştırmak istersek orjinal fonksiyonu yerine koymak zorunda olduğumuz için
    if (!VirtualProtect(compileMethodPtr, (uint)IntPtr.Size, 0x40, out old)) //VirtualProtect ile bölgenin izinleri execute read write izni olarak değiştirildi
    return;

    RuntimeHelpers.PrepareDelegate(hookedCompileMethod);//Belirtilen temsilcinin kısıtlanmış bir yürütme bölgesine (CER) eklenmek üzere hazırlanması gerektiğini gösterir.
    RuntimeHelpers.PrepareDelegate(OrigCompileMethod);
    //Bunları koymadan çalıştırırsanız göreceksiniz ki program stackoverflow exception'a düşecek. Sonsuz döngüye girmemesi için koyuyoruz.
    Marshal.WriteIntPtr(compileMethodPtr, Marshal.GetFunctionPointerForDelegate(hookedCompileMethod)); //Fake fonksiyonumuzun adresini alıp compileMethod pointer'ının yerine yazdırdık.
    VirtualProtect(compileMethodPtr, (uint)IntPtr.Size,
old, out old);//İzinleri eski haline döndürüyoruz.

    Console.WriteLine(testFunc()); //Bakalım çalışıyor mu

    if (!VirtualProtect(compileMethodPtr, (uint)IntPtr.Size, 0x40, out old)) //VirtualProtect ile bölgenin izinleri execute read write izni olarak değiştirildi
return; //Şimdi orjinal compileMethod'u yazdıracağımız için tekrar izinleri düzenliyoruz execute read write olarak.

    Marshal.WriteIntPtr(compileMethodPtr, Marshal.GetFunctionPointerForDelegate(OrigCompileMethod)); //Orjinal compileMethod'u yazdırdık fonksiyonumuzu normal çalıştırmak için.

    Console.WriteLine("Şuan çalışmıyor");
    Console.ReadKey();
}

public static string testFunc()
{
    return "Çalışıyorrr";
}

private static unsafe int HookedCompileMethod(IntPtr thisPtr, [In] IntPtr corJitInfo,
 [In] Context.CorMethodInfo* methodInfo, Context.CorJitFlag flags,
[Out] IntPtr nativeEntry, [Out] IntPtr nativeSizeOfCode)
{
    int token;
    Console.WriteLine("Compilation:\r\n");
    Console.WriteLine("Token: " + (token = (0x06000000 + *(ushort*)methodInfo->methodHandle)).ToString("x8"));//Token hesaplaması. dnSpy üzerinden tekrardan teyit edersek doğru olduğunu göreceğiz.
    Console.WriteLine("Name: " + typeof(Program).Module.ResolveMethod(token).Name);
    Console.WriteLine("Body size: " + methodInfo->ilCodeSize);

    var bodyBuffer = new byte[methodInfo->ilCodeSize]; //ilCodeSize tam da burda işimize yarıyor. ne kadar byte allocate edeceğimizi ona göre seçiyoruz.
    Marshal.Copy(methodInfo->ilCode, bodyBuffer, 0, bodyBuffer.Length); //ilCode yapısını değişkenimize yazdırıyoruz.

    Console.WriteLine("Body: " + BitConverter.ToString(bodyBuffer));

    return OrigCompileMethod(thisPtr, corJitInfo, methodInfo, flags, nativeEntry, nativeSizeOfCode); //Asıl fonksiyonu çalıştırıyoruz.
}

{% endhighlight %}

![1](https://user-images.githubusercontent.com/54905232/104839465-66712680-58d2-11eb-9a95-65a4a939cd6b.png)

> Yazım gerçekten uzun olmuş olabilir. Tüm ayrıntılarıyla detaylı bir yazı ortaya çıkartmaya çalıştım. Takıldığınız bir nokta olursa bana her zaman yorumlar kısmından veya Twitter'dan ulaşabilirsiniz. Remote JIT Hooker'ı da çok yakında yayınlayacağım aralarında fazla açıklık olmayacak :D İngilizce versiyonu da yakında...

[Local JIT Hooker GitHub](https://github.com/Rhotav/Local-JIT-Hooker)

#### References
[https://github.com/dotnet/coreclr](https://github.com/dotnet/coreclr)
[https://xoofx.com/blog](https://xoofx.com/blog/2018/04/12/writing-managed-jit-in-csharp-with-coreclr/#)
[https://www.mono-project.com/news/2018/09/11/csharp-jit/](https://www.mono-project.com/news/2018/09/11/csharp-jit/)
[SJITHook](https://github.com/maddnias/SJITHook)
