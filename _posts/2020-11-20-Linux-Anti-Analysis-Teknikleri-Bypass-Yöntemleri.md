---
title: Linux Anti Analysis Teknikleri & Bypass Yöntemleri
layout: post
tags: [Linux,Analysis]
author: 0x00fy
---


#### Linux Anti Analysis Teknikleri & Bypass Yöntemleri

Merhaba Dostlar Yeni yazıma Hoşgeldiniz ,

Günümüzde zararlı yazılım geliştiricileri yazdıkları zararlı yazılımın 3. Kişiler ve Malware Analist ler tarafından incelenmesini zorlaştırmak veya engelleyebilmek için **Anti Analysis** tekniklerinden yararlanırlar. Malware Analist lerin kullandığı araç ve yöntemlerden bazıları;

***disassembling, debugging, patching, function hooking, virtual machine olarak isimlendirilebilir…***

Bu yazımda en yaygın **Anti Analysis** tekniklerini ve bypass yöntemlerini anlatacağım. Herşeyden önce bu yazıdaki bypass yöntemlerini daha iyi anlayabilmeniz açısından [Linux Load Time Function Hijacking](http://modebit0.ifo) yazısını okumanızı tavsiye ediyorum, bu yazıda kullanacağımız bypass teknikleri **Linux Load Time Function Hijacking (via shared object injection)** methodu üzerine kuruludur. Artık Gerekli şeyleri öğrendiğinizi farzederek Anti Analysis tekniklerini ve Bypass Methodlarını Sırayla anlatmaya başlıyorum ;


## **1) - Anti Ptrace Tekniği**

Binary analizlerde sıkça kullanılan yöntemlerden bir tanesi debugging dir. Debugger adı verilen araçlar bize herhangi bir programı en ince ayrıntısına kadar inceleme fırsatı verir. Programı disassemble etme, breakpoint koyarak programın çalışmasını istediğimiz noktada durdurma, herhangi bir anda registerların değerini görme/değiştirme, programa ait bellek alanını inceleme gibi oldukça işe yarayan özellikleri bulunur.

Peki nasıl oluyorda bir programın çalışmasına bu denli müdahale edebiliyoruz derseniz, tam bu noktada process tracing konusu devreye giriyor. Bu işlem için Linux’da ptrace adlı bir sistem çağrısı bulunmakta. ( Bu sistem Çağrısının işlevine Önceki Yazımda detaylı şekilde değinmiştim )

Linux ortamında debuggerlar bu sistem çağrısını kullanarak işlemi takip ederler. Bir program aynı anda yanlızca bir process tarafından trace edilebilir!

Anti Ptrace Source Kodu :


```
#include <stdio.h>
#include <stdlib.h>
#include <sys/ptrace.h>

int check_ptrace()
{
	if ( ptrace(PTRACE_TRACEME) == -1 )  // Program Kendini trace edememe durumu varsa
		exit(0); // Bu durumda program kapansın
}

int main()
{
	check_ptrace();
	puts("Ptrace Tespit Edilemedi , Zararli Kodlar Calisabilir..");
	return 0;
}



```

Source Kodun bu Bölümde `ptrace()` sistem çağrısının `PTRACE_TRACEME` parametresi kullanılarak Ana process 'e TRACEME sinyali gönderilir, bu sinyali alan process kendisini trace etmeye başlar. Bu sayede process herhangi bir debugger tarafından trace edilmeye çalışıldığında program otomatik olarak kendini sonlandıracaktır…

`if ( ptrace(PTRACE_TRACEME) == -1 )`

**Anti ptrace Bypass :**

Gelelim bu Anti Analysis tekniğinin bypass edilişine , bu yazıda hemen hemen bütün teknikleri aynı şekilde bypass edeceğiz. Olayımızın kilit noktası ve mantığı özetle ; **Load Time Function Hijacking** ile kilit fonksiyonları Hijack Ederek, bu teknikleri bypass etmek. Bunun için izlememiz gereken adımlar sırası ile ;

* Aşşağıdaki Source Kodu Shared Object (.SO) şeklinde derliyoruz.
* LD_PRELOAD dış değişkenine atıyoruz.
* Programı debugger ile yeniden çalıştırıyoruz.
* Anti ptrace Bypass Source (bypass.c) :

```
long ptrace(int request, int pid, int addr, int data)
{
    return 0;
}
```
Derlemek : `$ gcc bypass.c -o bypass.so -fPIC -shared -ldl -D_GNU_SOURCE`

Çalıştırmak : `$ export LD_PRELOAD="/home/user/bypass.so" && gdb -q antidebugger`


**Burada ne yaptık ?**

Özetle anlatmak gerekirse; Anti debugger source (bypass.c) içerisinde ptrace fonksiyonunun yerine geçecek bir sahte fonksiyon hazırladık , ardından Shared Object Library halinde bu dosyayı derledik ve **LD_PRELOAD** ortam değikeni içerisine atarak programın içine dahil olmasını sağladık ve bizim sahte fonksiyonumuz gerçek olanın yerine geçti , bu sayede debugger ımızın çalışmasını engelleyen ptrace fonksiyonunu işlevsiz hale getirmiş olduk.

Ne yaptığımızı anlamayan arkadaşlar için ufak bir video hazırladım, izleyerek uygulayabilirler…

[https://www.youtube.com/watch?v=LmM_8O26Xgw](https://www.youtube.com/watch?v=LmM_8O26Xgw)


**NOT :** Bundan sonraki Anti Analysis Teknikleri için herhangi bir uygulamada bulunmayacağım , Teknik hakkında detay ve hijack için gereken library 'i verip geçeceğim. Zira videodan sonra uygulamayı artık öğrenmiş olmamız gerek…






## **2) - Anti Breakpoint Tekniği**

Bir program debug edilirken programın belirli noktalarına breakpoint koyarak istenilen noktalarda durmasını sağlar ve programın o anki durumunu inceleriz.

Breakpoint koymanın debuggerler tarafından kullanılan en popüler yolu **int 3** instruction’ı kullanmaktır. Bu instruction çalıştığında kernel’a ***SIGTRAP*** sinyali yollanıp programın çalışması durdurulur. Programın durmasını istediğimiz yere ***int 3*** instructionının opcode’u olan ***0xCC*** değerini yazarak breakpoint koymuş oluruz.

Bu programda ilk fonksiyonun adresinden son fonksiyonun adresine kadar olan bellek alanı taranarak ***0xCC*** değerinin var olup olmadığına bakılır. Şayet belirtilen alan içerisinde bu değer yer alıyorsa bu breakpoint varlığına işarettir.

Anti Breakpoint Source :
```
#include <stdio.h>
#include <stdlib.h>

void dummy();

void check_breakpoint()
{
	for (int i = &check_breakpoint; i <= &dummy+3; i++ ) // "check_breakpoint" fonksiyonundan "dummy" fonksiyonuna kadar olan kısmı
  {                                                    // yani diğer bir deyişle programımızı baştan sonra tarıyoruz.
		char *p = i;
		if ( (*p) & 0xff == 0xcc) // şayet herhangi bir adresin değeri "int 3" (breakpoint) opcode'una eşit ise bu debugger kullanıldığına işarettir.
			exit(0); // bu durumda program kapansın.
	}

}

int main()
{
	check_breakpoint();
	puts("No breakpoints detected! Starting evil stuff..");

	return 0;
}

void dummy()
{
	__asm__("nop");
}
```

**Anti Breakpoint Bypass :** Burada programın içinde bulunan fonksiyonları objdump aracı ile tespit edip **dummy() , check_breakpoint() , exit()** fonksiyonlarından birini Hijack edebiliriz. Veya daha daha çılgınca davranıp fonksiyonların kullandığı **syscal** lari hijack etmeye çalışabiliriz, burdan sonrası hayal gücünüze kalmış…

Ben burada **exit()** fonksiyonunu hijack ederek **main()** fonksiyonuna yönlendirmeyi tercih ettim;

Hijack Library ( Anti Breakpoint) :

```
long exit(int request, int pid, int addr, int data)
{
    main();
    return 0;
}
```




## **3)- Anti Binary Modification Tekniği**

Analistler inceledikleri bir programda bulunan kontrolleri aşmak adına programda bazı yerleri modifiye edip bu kontrolleri bypass edebilirler. Bu yönteme binary modification veya patching adı verilir. **Anti-patching** tekniğinin mantığı program çalışmaya başlamadan önce kodların checksum değerini hesaplayıp önceden hesaplanmış esas checksum değeri ile karşılaştırarak programın değiştirilip değiştirilmediğini bulmaya çalışmaktır.

Orijinal checksum değerinin depolandığı yer checksum hesabını etkilememesi adına *text* segmentinden farklı bir segmentte bulunmalıdır. Dolayısıyla örnek programda bu değeri depolayacak değişken static char olarak belirlenmiştir. Bu değer debugger aracılığıyla bulunabilir veya elle opcode değerlerinin toplanması yoluyla hesaplanabilir. Checksum hesabı için de çok basit bir algoritma kullanılmıştır.

Source Code :
```
#include <stdio.h>
#include <stdlib.h>

void dummy();

void check_patching()
{
	char chk;
	static char checksum = 0x64; // Olması gereken checksum değeri. Debugger ile elde edilebilir. Farklı segmentte yer alması için static char olarak tanımlandı.

	for (int i = &check_patching; i <= &dummy+3; i++ ) // anti-breakpoint tekniğinde olduğu gibi bütün kod alanını tarıyoruz
	{
		char *p = i;
		chk += (*p) & 0xff; // Çok basit bir checksum algoritması, byte'ları toplayarak gidiyor.
	}

	if (chk != checksum) // Byte'ların toplamı asıl checksum değerine eşit değilse bu programın modifiye edildiğine işaret
		exit(0); // Bu durumda programı kapatalım.
}

int main()
{
	check_patching();
	puts("No patching detected! Starting evil stuff.."); // Pis işlerimiz..

	return 0;
}

void dummy()
{
	__asm__("nop");
}
```
**Anti Binary Modification Bypass :** Bu yöntemin program üzerinde herhangi bir patchleme işlemi yapılmıyacaksa bypass edilmesine gerek yok fakat , bypass etmek istersek objdump aracı ile fonksiyon isimlerini belirleyip hijack library hazırlayabilirsiniz. Veya **exit()** fonksiyonunu hijack etmek işinizi görecektir…



## **4)- Anti Hook Tekniği**

Gelelim işin en trajikomik kısımına , zararlı yazılım geliştiricilerinin fonksiyonları hooklamamızı engelleyeceğini düşündüğü kod parçasına, bu yöntem bizim Fonksiyonları hijack etmemizi engellemek üzerine kuruludur,

Source Kod :

```
#include <stdio.h>
#include <stdlib.h>

void check_hook()
{
	if (getenv("LD_PRELOAD") != NULL) // LD_PRELOAD ortam değişkeninin içi doluysa bu function hook yapıldığına işaret.
		exit(0); // Bu durumda programı kapatalım.
}

int main()
{
	check_hook();
	puts("No hooks detected! Starting evil stuff..");

	return 0;
}

```

Anti Hook Bypass : Programın çalışma sırasında **getenv()** fonksiyonu kullanarak *LD_PRELOAD* dış değişkeninin boş olup olmadığı kontrol edilir, bunun Analistleri engelliyeceğini sanmışlar, asıl komedi de burada başlıyor. Kontrol programın çalışma sırasında yapılırken, biz programa fonksiyonu **Load Time** 'da iken inject ediyoruz. Yani bu demek oluyor ki biz kontrolü atlamak için **getenv()** fonksiyonuna müdehale edebiliriz :)))

Kullanacağımız Library :

```
long getenv(int request, int pid, int addr, int data)
{
    return 0;
}
```

**Örneklerin Derlenmiş Halleri , Bypass Libraryleri ve python ile otomatize edilmiş hallerini bir Repository 'de topladım.

AntiAnalysis uygulamasını derledikten sonra , **“python bypass.py ./anti-ptrace”** şekinde çalıştırmanız yeterli olacaktır.

**NOT : Çalışması için GDB kurulu olmalıdır , başka debugger kullanmak isterseniz *bypass.py 'nin 34. Satırındaki* *gdb -q* başlatıcısını değiştirmeniz yeterli olacaktır…**

**Repository :**
[GITHUB REPOSITORY (ANTI ANALYSIS TEKNIKLERI VE OTOMATIZE EDILMIS BYPASS UYGULAMALARI](https://github.com/0x00fy/Linux-Anti-Analysis-Bypass)


## THE END

Makale Hakkında Kafanıza Takılan Soruları , Yanlış veya Eksik gördüğünüz yerleri vs. Yorumlaeda belirtebilirsiniz…

Birdahaki yazımda görüşmek üzere , esen kalın…
