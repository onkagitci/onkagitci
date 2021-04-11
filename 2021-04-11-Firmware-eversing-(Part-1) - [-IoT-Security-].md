---
title: Firmware Reversing (Part 1) - [ IoT Security ]
layout: post
tags: [Linux,Reversing]
author: 0x00fy
---




Merhaba dostlar yeni yazıma hoşgeldiniz, bu yazımda elimizde olan bir firmware’ın analizinden bahsedeceğim.
Firmware Nedir ?

Donanım yazılımı, sayısal veri işleme yeteneği bulunan her tür donanımın kendisinden beklenen işlevleri yerine getirebilmesi için kullandığı yazılımlara verilen addır. Elektronikte ve bilişimde donanım yazılımı, kalıcı bellek, program kodu ve veri deposudur. Özetle Firmware, donanım için yazılmış bir yazılımdır. Cihazda, programların çalışmasını sağlar.
Firmware Dosyalarını Elde Etmek

Firmware Dosyalarını elde etmek için en çok bilinen yöntemler şunlardır ;

    Cihaza Fiziksel erişim varsa , Cihazın UART portları üzerinden çeşitli yöntemler ile dosyaları elde edebiliriz.

    Üreticinin Websitesinden dosyaları indirebiliriz.

Firmware Dosyalarını elde ettikten sonra , artık analiz etme aşamasına geçebiliriz.
Analize Başlıyalım

Bu yazımızda uygulamalı örnek olması için D-Link 'in DCS-932L isimli WiFi Camera ürününün 1.14.4 versiyonuna sahip Firmware’ı nı analiz edeceğiz.

Burada Analiz işlemimizi kolaylaştıracak binwalk isimli bir araç kullanacağız. (GitHub - ReFirmLabs/binwalk: Firmware Analysis Tool 1)

Binwalk özetle , dosyayı ayrıştırır ve bulduklarına göre bir içindekiler tablosu döndürür.

İlk olarak dosyayı .zip den çıkaralım.


![](https://vvhack.org/uploads/default/original/2X/c/c71ff5c38a5f8a4b7901fa35dfbcc8bd97e9fcc8.png)

ve adrından ,

`$ binwalk dcs932l_v1.14.04.bin` Komutunu çalıştırarak çıktıyı görelim. Çıktı üzerinde ;

    Ondalık formatta bir dosya konumu
    Onaltılık (Hexa Decimal) formatta bir dosya konumu
    O konumda bulunan şeyin açıklaması

İlk satıra baktığımızda binwalk 106352’de bir U-Boot dizesi bulduğunu görüyoruz. U-Boot popüler bir bootloader’dır. Bir aygıt açıldığında, işletim sistemini yüklemek bootloader 'In görevidir. Ve tabii ki 327680’de, 327744’te başlayan bir LZMA arşivinde OS çekirdek imajını bulacağımızı söyleyen bir uImage başlığı görebiliriz.

![](https://vvhack.org/uploads/default/original/2X/e/ea1409b9dfdba9a3faac9605b6f1b4cdc2fa79a4.png)

Bu LZMA arşivini açmadan ve derinlemesine incelemeden önce, onu .bin dosyadan ayırmamız gerekiyor. Şu Komutu çalıştırarak bu işlemi gerçekleştiriyoruz :

`$ dd if=dcs932l_v1.14.04.bin skip=327744 bs=1 of=kernel.lzma`

![](https://vvhack.org/uploads/default/original/2X/2/2f6dc78379fd334382a5ff3fbcca4ef31b5e6d3f.png)

LZMA arşivini elde ettiğimize göre artık dosyaları arşivden çıkartabiliriz. unlzma kernel.lzma komutu ile bu işlemi gerçekleştiriyoruz. Ve arşivden çıkan dosyanın bir data dosyası olduğunu görüyoruz.

Arşivden çıkan dosyayı tekrar binwalk ile analiz ettiğimizde 4038656 adresinde , birtane daha LZMA arşivi olduğunu görüyoruz. İç içe geçmiş 2 LZMA arşivi…

Şimdi orada gördüğümüz LZMA’yı da data dosyasından ayıralım…

`$ dd if=kernel skip=4038656 bs=1 of=temizcikti.lzma`

ardından çıkan LZMA arşivinden çıkartalım…

`$ unlzma temizcikti.lzma`

komtu çalıştırdıktan sonra , çıkan dosyanın bir cipo arşivi olduğunu görüyoruz. Bu arşiv içinden çıkacak dosyaların , Linux Sistem dosyaları olma ihtimali çok yüksek.

Şimdi linux sistem dosyalarını bir klasör oluşturalım ve bu klasöre çıkartalım.

`$ mkdir cpio; cd cpio`

Ardından cipo arşivi içindeki dosyaları , çıkartalım ;

`$ cpio -idm --no-absolute-filenames < ../temizcikti`

Dosyaları çıkartma işleminin ardından çıkan dosyaların , Linux Sistem dosyaları olduğunu görüyoruz.

![](https://vvhack.org/uploads/default/original/2X/4/4e2e70f52d4f2c6a8dbb1defc80f22bb8f416798.png)

Bundan sonraki adımda , configuration dosyaları , Binary dosyaları üzerinde veya ullandığınız Firmware Http kullanıyorsa Web tarafında PHP/ASP kodları üzerinde Resarch Yapabilirsiniz. 2. yazımda bu kısımlara daha detaylı şekilde göz atıp. Firmware üzerinde modification yaptıktan sonra tekrar .bin haline getirmek ve Firmware a Backdoor gömmek üzerinde duracağız.

Bir sonraki yazımda görüşmek üzere , Esen Kalın…
