---
title: Linux Kernel Rootkit Development - (Part 1)
layout: post
tags: [Linux,Rootkit,Zararli]
author: 0x00fy
---

### Giris

Linux'un sunucu ortamlarında Kernel Mode Rootkitlerin kullanımı saniyeden saniyeye artıyor. Yani Linux hacklemek her geçen gün daha ilginç hale geliyor. Bir Linux sistemine erişim sağladıktan sonra kalıcı olmak , ve kolay bir şekilde yönetmek dendiğinde akla gelen ilk yöntem bir **Rootkit Kullanmaktır** . Linux Kernel 'in Loadable Kernel Modules (LKMs) adı verilen özelliği sayesinde, işletim sisteminin hassas kısımlarına erişmemizi sağlayan Kernel Mod 'da çalışan kod yazmak mümkündür. Daha önce LKM Hacking ile ilgili (örneğin Phrack) çok iyi olan bazı makaleler vardı. Yeni fikirler, yeni yöntemler tanıttılar. Ayrıca 1998'deki bazı kamuoyu tartışmaları (Haber Grupları, Posta Listeleri) çok ilginçti.
Bu Seride **LKM Hacking** üzerine daha deyatlı anlatımlar olacak , Peki bu seriyi Neden yazıyorum ?

**Aslında Bunun Birçok Sebebi Var Kaba Taslak Şunlar ;**

* **Bu Konuda Yeterli Türkçe Kaynak Bulunmaması.**
* **Aynı Türkçe Kaynakların Sürekli Farklı yerlerde yeni kaynak gibi Lanse edilmesi**
* **Var olan Türkçe Kaynakların Baştan sona bir Kernel Rootkit geliştirmek için Yeterli Olmaması**
