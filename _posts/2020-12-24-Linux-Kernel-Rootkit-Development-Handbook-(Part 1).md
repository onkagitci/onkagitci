---
title: Linux Kernel Rootkit Development Handbook - (Part 1)
layout: post
tags: [Linux,Rootkit,Zararli,syscall,ftrace]
author: 0x00fy
---

# Giriş

Linux'un sunucu ortamlarında Kernel Mode Rootkitlerin kullanımı saniyeden saniyeye artıyor. Yani Linux hacklemek her geçen gün daha ilginç hale geliyor. Bir Linux sistemine erişim sağladıktan sonra kalıcı olmak , ve kolay bir şekilde yönetmek dendiğinde akla gelen ilk yöntem bir **Rootkit Kullanmaktır** . Linux Kernel 'in Loadable Kernel Modules (LKMs) adı verilen özelliği sayesinde, işletim sisteminin hassas kısımlarına erişmemizi sağlayan Kernel Mod 'da çalışan kod yazmak mümkündür. Daha önce LKM Hacking ile ilgili (örneğin Phrack) çok iyi olan bazı makaleler vardı. Yeni fikirler, yeni yöntemler tanıttılar. Ayrıca 1998'deki bazı kamuoyu tartışmaları (Haber Grupları, Posta Listeleri) çok ilginçti.
Bu Seride **LKM Hacking** üzerine daha deyatlı anlatımlar olacak , Peki bu seriyi Neden yazıyorum ?

**Aslında Bunun Birçok Sebebi Var Kaba Taslak Şunlar ;**

* **Bu Konuda Yeterli Türkçe Kaynak Bulunmaması.**
* **Aynı Türkçe Kaynakların Sürekli Farklı yerlerde yeni kaynak gibi Lanse edilmesi**
* **Var olan Türkçe Kaynakların Baştan sona bir Kernel Rootkit geliştirmek için Yeterli Olmaması**

***Lütfen unutmayın: Bu metin sadece eğitim amaçlı yazılmıştır. Bu metne dayalı herhangi bir yasa dışı eylem, sizin kendi probleminizdir.***

# I. Temeller :

### 1. LKM Nedir ?

  **LKM'ler, Linux çekirdeği tarafından işlevselliğini genişletmek için sağlanan Yüklenebilir Çekirdek Modülleri Servisidir. Bu LKM'lerin avantajı: Dinamik olarak yüklenebilir olması ve Yüklendikten sonra Linux Çekirdeğini Yeniden Derlemeyi Gerektirmemesidir.. Bundan önceki dönemlerde Kernel 'a yeni bir işlev kazandırmak için Gereken Eklemeler yapıldıktan sonra Kernel Yeniden Derlenirdi**, ***LKM nin ortaya çıkışı temelde bu soruna çözüm olmuştur..***
  
### 2. LKM Nasıl Yazılır ?   
 
 ```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Baykuş");
MODULE_DESCRIPTION("Kenel Modul");
MODULE_VERSION("0.01");

static int __init example_init(void)
{
    printk(KERN_INFO "Hello, world!\n");
    return 0;
}

static void __exit example_exit(void)
{
    printk(KERN_INFO "Goodbye, world!\n");
}

module_init(example_init);
module_exit(example_exit);
```

Yukarıda basit bir Modül örneği görüyoruz ve bunu birlikte satır satır gözden geçireceğiz;

Öncelikle, her zaman gerekli olacak birkaç kütüphaneyi include ediyoruz, 

 ```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

 ```
 
 
Ardından modülün ne yaptığıyla ilgili ve Modülle ilgili bazı ayrıntılar veren birkaç makro geliyor. Bu bilgi, modülü daha sonra belleğe yüklediğimizde çekirdek tarafından kullanılabilir hale getirilir.

 ```
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Baykuş");
MODULE_DESCRIPTION("Kenel Modul");
MODULE_VERSION("0.01");

 ```


Daha sonra, her zaman mevcut olacak çok önemli iki fonksiyon geliyor. `example_init` fonksiyonu, modül yüklendiğinde çalıştırılır ve Modül kaldırıldığında `example_exit` fonksiyonu çalışır. Son iki satır derleyiciye `example_init` ve `example_exit` 'in sahip olduğu rolleri bildirir. (module_init ve module_exit opsiyonları `example_init` ve `example_exit` fonksiyonlarının hangisinin yüklendikten sonra , hangisinin kaldırıldıktan sonra çalışacağını bildirir.).


Bu bölümde `example_init` ve `example_exit` fonksiyonu içerisinde dikkat çeken `printk()` fonksiyonu, kernel modda ekrana yazdırma için kullanılır. Standart C kütüphanesindeki `printf()` fonkisyonu gibi davranır fakat kernel modda ekrana direk olarak çıktı vermez. Yazdırılan değerler `/var/log/syslog` dosyasında ya da `dmesg` komutunun çıktısında görülebilir.

```
static int __init example_init(void)
{
    printk(KERN_INFO "Hello, world!\n");
    return 0;
}`example_exit` fonksiyonu içerisinde 
```

   a)- Derleme İşlemi :


modul.c içeriği :

 ```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Baykuş");
MODULE_DESCRIPTION("Kenel Modul");
MODULE_VERSION("0.01");

static int __init example_init(void)
{
    printk(KERN_INFO "Hello, world!\n");
    return 0;
}

static void __exit example_exit(void)
{
    printk(KERN_INFO "Goodbye, world!\n");
}

module_init(example_init);
module_exit(example_exit);
```



Makefile dosyası içeriği :

```
obj-m += modul.o
all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

Makefile dosyamızı kaydettikten sonra derlemek için Terminalimizin komut satırından iki dosyanında aynı yerde bulunduğu dizine gidip `make` komutunu vermemiz yeterli.
bu işlemin ardından birkaç dosya oluşmuş olacak.


   b)- Modülü kernel 'a Load ve Unload Etmek :


`sudo insmod module.ko` Komutu ile modülümüzü kernel 'a Load ediyoruz 

ardından `sudo dmesg -C` komutu ile `example_init` fonksiyonunun işlevini yerine getirdiğini görüyoruz.

hemen ardından `lsmod | grep modul` komutu ile sisteme yüklenmiş modüller içinden kendi modülümüzü ayıklayıp görebiliyoruz

gelelim silme işlemine; `sudo rmmod modul` komutu ile modülümüzü kernel dan siliyoruz..


### II. Sistem Çağrıları

1. Bölümde ilk kernel modülümüzü oluşturduk, ancak şimdi onun harika bir şey yapmasını istiyorsunuz - çalışan kernel 'ın davranışını değiştirmek gibi bir şey. Bunu yapmamız için gereken şey kernel mod Fonksiyonlarını hook etmektir , ancak soru şu - hangi fonksiyonları hooklayacağımızı nasıl bileceğiz?

Neyse ki bizim için Torvalds amcamızın hazırladığı harika bir potansiyel hedef listesi var: *sistem çağrıları!*

**Sistem çağrıları (Syscalls) :**  userspace (ring3) 'den çağrılabilen ve Kernel da neredeyse her şey için gerekli olan Kernel Land (ring0) fonksiyonlarıdır.

İlerleyen Bölümlerde sıklıkla karşımıza çıkacak oalan bazı sistem çağrıları :

   *open
   *read
   *write
   *close
   *execve
   *fork
   *kill
   *mkdir


X86_64 sistem çağrılarının tam listesini [Burada](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl) görebilirsiniz. Bu fonksiyonlardan herhangi birinin kendi isteklerimiz ve pis işlerimiz doğrultusunda çalışması ne güzel olurdu değilmi. Bazı dosyalara yapılan okuma çağrılarını yakalayabilir ve farklı bir şey döndürebilir veya execve ile özel envirionment variables ekleyebiliriz. Bazı eylemleri gerçekleştirmek için rootkit'imize komutlar göndermek için kill fonksiyonunda kullanılmayan bazı sinyalleri bile kullanabiliriz. (Daha önce Ghost Rootkit , Antiy4 vs. Rootkitlerde yapıldı)

Ama önce, user landdan nasıl sistem çağrısı yaptığımıza dair daha iyi bir fikre sahip olmak faydalı olacaktır - sonuçta, müdahale etmeyi istediğimiz şey bir process!


Yukarıdaki [Sistem Çağrısı Tablosuna ](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl) bir göz atarsanız, her sistem çağrısının kendisine atanmış bir numara olduğunu görürsünüz (bu numaralar aslında oldukça akıcıdır ve farklı mimariler ve kernel sürümleri arasında değişiklik gösterir.

Bir sistem çağrısı yapmak istiyorsak, istediğimiz sistem çağrı numarasını `rax` kaydına kaydetmemiz ve ardından yazılım interrupt syscall ile Kernel 'a teslim etmemiz `(int 0x80)` gerekir. Kesmeyi kullanmadan önce sistem çağrısının belirli yazmaçlara yüklenmesi gereken argümanlar ve dönüş değeri neredeyse her zaman `rax`'a yerleştirilir.

Bu, en iyi bir örnekle açıklanır - syscall 0, `sys_read`'ı inceleyelim (tüm sistem çağrılarının önünde `sys_` bulunur). Bu sisteme adam 2 okurken bakarsak, şu şekilde tanımlandığını görürüz:

```ssize_t read(int fd, void *buf, size_t count);```

`*fd` dosya tanımlayıcısıdır ve `open()` çağrısından döndürülür, `buf` okunan veriyi saklamak için bir arabellektir ve `count` okunacak bayt sayısıdır. Dönüş değeri, başarıyla okunan bayt sayısıdır ve hata durumunda -1 değerini alır.

https://i.imgur.com/SzlZhQs.jpg


Böylece, `rdi` file pointer 'ı, `rsi` buffer 'a bir pointer ve `rdx` okunacak bayt sayısını alır.`0x00`'ı rax'a kaydetmiş olduğumuz sürece, devam edip Kernel'a teslim edebiliriz ve sistem çağrımız gerçekleşecektir! Örnek bir NASM parçası şu lekildedir:


```

mov rax, 0x0
mov rdi, 5
mov rsi, buf
mov rdx, 10
syscall

```

Bu kod, `file pointer 5`'ten (rastgele seçilen) 10 bayt okur ve içerikleri buffer'a gösterilen bellek konumunda depolar. Oldukça basit, değil mi?


Hepsi usermod için iyidir, peki ya kernel? Rootkitlerimiz çekirdek bağlamında çalışacak, bu nedenle çekirdeğin sistem çağrılarını nasıl işlediğini biraz anlamalıyız.

Ne yazık ki, işlerin biraz farklı olmaya başladığı yer burası. 64 bitlik çekirdek sürümleri `4.17.0` ve üzeri, sistem çağrılarının kernel tarafından işlenme biçimi değişti. İlk olarak, eski yönteme bakacağız çünkü bu,` Ubuntu 16.04` gibi dağıtımlar için hala geçerli ve eski yöntem mantığı oturduğunda daha yeni sürümün anlaşılması çok daha kolay.

Çekirdekteki `sys_read` için kaynak koduna bakarsak, şunu görürüz:

```
asmlinkage uzun sys_read (işaretsiz int fd, char __user * buf, size_t sayısı);
```

2016'da, argümanlar sisteme tam olarak göründüğü gibi aktarıldı. `Sys_read` için bir hook yazıyor olsaydık, bu işlev bildirimini kendimiz taklit etmemiz gerekirdi ve (kancayı bir kez yerine koyduğumuzda), bu argümanlarla istediğimiz gibi oynayabilirdik.

(64 bit) kernel sürüm `4.17.0` ile bu durum değişti. İlk önce kullanıcı tarafından yazmaçlarda depolanan argümanlar `pt_regs` adlı özel bir yapıya kopyalanır ve sonra sistem çağrısina iletilen tek şey budur. Sistem çağrısı daha sonra ihtiyaç duyduğu argümanları bu yapıdan çıkarmaktan sorumludur. Ptrace.h dosyasına göre aşağıdaki biçime sahiptir:

```
struct pt_regs {
    işaretsiz uzun bx;
    işaretsiz uzun cx;
    işaretsiz uzun dx;
    işaretsiz uzun si;
    işaretsiz uzun di;
    / * anlaşılır olması için yeniden düzenlendi * /
};

```


Bu, `sys_read` durumunda şuna benzer bir şey yapmamız gerektiği anlamına gelir:

```
asmlinkage uzun sys_read (const struct pt_regs * regs)
{
    int fd = regs-> di;
    char __user * buf = regs-> si;
    size_t count = regs-> d;
    / * işlevin geri kalanı * /
}

```

Tabii ki, kernel bizim için işi yaptığından, gerçek `sys_read`'in bunu yapmasına gerek yoktur. Ancak bir hook fonksiyonu yazarken argümanları bu şekilde ele almamız gerekecek.

Bu kadar temel bilgiden sorna, artık rootkitimizin yapacağı temel işlevleri olulşturmaya başlıyabiliriz. Bu makalenin sonunda tam işlevli bir rootkit yazmış olacağız.


# II.Rootkit İşlevleri


*syscallhooking*
*function hooking*
*process shellcode injection*



### 1. Syscallhooking

Tüm bunların dışında, bir hadi bir `function hook` yazmaya başlayalım! `Sys_mkdir` için, oluşturulmakta olan dizinin adını kernel buffer ına yazdıran çok basit bir hook oluşturmak için yukarıdaki iki yöntemi dikkate alacağız. Daha sonra gerçek `sys_mkdir` yerine bu hook fonksiyonu geçireceğiz.

Öncelikle, hangi kernel sürümünü derlediğimizi kontrol etmemiz gerekecek - `linux/version.h` bu konuda bize yardımcı olacaktır. Ardından işleri bizim için basitleştirmek için bir dizi `önişlemci makrosu` kullanacağız.

```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/syscalls.h>
#include <linux/version.h>
#include <linux/namei.h>

MODULE_LICENSE("GPL");
MODULE_VERSION("0.01");

#if defined(CONFIG_X86_64) && (LINUX_VERSION_CODE >= KERNEL_VERSION(4,17,0))
#define PTREGS_SYSCALL_STUBS 1
#endif

#ifdef PTREGS_SYSCALL_STUBS
static asmlinkage long (*orig_mkdir)(const struct pt_regs *);

asmlinkage int hook_mkdir(const struct pt_regs *regs)
{
    char __user *pathname = (char *)regs->di;
    char dir_name[NAME_MAX] = {0};

    long error = strncpy_from_user(dir_name, pathname, NAME_MAX);

    if (error > 0)
        printk(KERN_INFO "rootkit: trying to create directory with name: %s\n", dir_name);

    orig_mkdir(regs);
    return 0;
}
#else
static asmlinkage long (*orig_mkdir)(const char __user *pathname, umode_t mode);

asmlinkage int hook_mkdir(const char __user *pathname, umode_t mode)
{
    char dir_name[NAME_MAX] = {0};

    long error = strncpy_from_user(dir_name, pathname, NAME_MAX);

    if (error > 0)
        printk(KERN_INFO "rootkit: trying to create directory with name %s\n", dir_name);

    orig_mkdir(pathname, mode);
    return 0;
}
#endif
```




