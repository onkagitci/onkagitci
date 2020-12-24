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

 Tüm LKM ler Temelde 2 Fonksiyondan oluşur(Minimum) Bunlar, **init** ve **exit** 'dir
 
 ```
 int init_module(void) /*Yüklendikten sonra çalışacak kısım*/
{

.. Bla Bla ..

}

void cleanup_module(void) /* Uninstall Sırasında Çalışacak Kısım */
{
.. Bla Bla..
}
```
