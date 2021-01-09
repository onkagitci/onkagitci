---
title: Linux Kernel Rootkit Development - (Part 1)
layout: post
tags: [Linux,Rootkit,Zararli]
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

ardından `dmesg` komutu ile `example_init` fonksiyonunun işlevini yerine getirdiğini görüyoruz.

hemen ardından `lsmod | grep modul` komutu ile sisteme yüklenmiş modüller içinden kendi modülümüzü ayıklayıp görebiliyoruz

gelelim silme işlemine; `sudo rmmod modul` komutu ile modülümüzü kernel dan siliyoruz..




