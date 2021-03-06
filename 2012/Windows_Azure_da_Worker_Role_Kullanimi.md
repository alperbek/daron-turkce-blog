---
FallbackID: 2788
Title: Windows Azure'da Worker Role Kullanımı
PublishDate: 18/9/2012
EntryID: Windows_Azure_da_Worker_Role_Kullanimi
IsActive: True
Section: software
MinutesSpent: 0
Tags: Windows Azure
---
Bugüne kadar birçok detaya bakmış olsak da Worker Role'lere hiç göz
atmadık. Bunun nedenlerinden biri tabi ki Worker Role'lerin aslında
özüne bakılacak olursa süper basit olmasıydı :) Ama yine de hızlıca bir
göz atalım istiyorum. Hatta Worker Role'lere bakarken genel Azure
senaryosunu da devreye alaray WebRole, Worker Role ve Storage Queue
yapılarını yan yana getirip güzel bir resim çizelim ne dersiniz? ;)

![Yeni bir Worker
Role...](media/Windows_Azure_da_Worker_Role_Kullanimi/worker.png)\
*Yeni bir Worker Role...*

### Taze bir Worker Role

Yeni bir Azure projesi yaratıp içine sadece bir Worker Role eklerseniz
Solution Explorer içerisinde sizi pek de kalabalık bir dosya sayısı
beklemeyecek :) Worker Role'ler biliyorsunuzki Azure ortamında host
edilirken içerisinde IIS olmayan sanal makinelerde barındırılıyorlar. O
nedenle ASP.NET'in de doğal olarak bütün application lifecycle'ından
kurtulmuş durumdalar.

![Yeni bir Worker Role projesinde tek bir CS dosyası
var.](media/Windows_Azure_da_Worker_Role_Kullanimi/worker2.png)\
*Yeni bir Worker Role projesinde tek bir CS dosyası var.*

Yukarıdaki ekran görüntüsünden de inceleyebileceğiniz üzere :) yeni bir
worker role projesinde sadece bir tane kod dosyası var. Onun dışında
hiçbir şey yok! :) Tertemiz bir manzara söz konusu. Worker Role'lerin
kullanımı genelde arkaplanda yapılacak işlemlerle ilgili oluyor. Web
Role ön tarafta kullanıcının veri girişini alıp sistemdeki veriyi
kullanıcıya göstermekle yükümlüyken Worker Role ise arkada diğer
işlemleri yapmakla ve süreçleri takip etmekle yükümlü.

**[C\#]**

<span style="color:blue;">public</span> <span
style="color:blue;">override</span> <span
style="color:blue;">void</span> Run()\
 {\
    <span
style="color:green;">// This is a sample worker implementation. Replace with your logic.</span>\
     <span style="color:#2b91af;">Trace</span>.WriteLine(<span
style="color:#a31515;">"\$projectname\$ entry point called"</span>, <span
style="color:#a31515;">"Information"</span>);\
\
    <span style="color:blue;">while</span> (<span
style="color:blue;">true</span>)\
     {\
        <span style="color:#2b91af;">Thread</span>.Sleep(10000);\
         <span style="color:#2b91af;">Trace</span>.WriteLine(<span
style="color:#a31515;">"Working"</span>, <span
style="color:#a31515;">"Information"</span>);\
     }\
}

Sıfırdan yaratılan bir worker role'ün tek kod dosyasında da yukarıdaki
kod yazıyor :) Şimdi hemen bu noktadan "While(True)" gördüğünüz gibi
"Nasıl yani?" diyor olabilirsiniz :) Konseptimiz şu, eğer bir Worker
Role'un içerisinde Run metodundan çıkılırsa worker role çatlamış oluyor
/ kapanmış oluyor. Böyle bir durumda doğal olarak FC Guest Agent da
gidip "Bu Worker kapandı" diyip Worker'ı tekrar açıyor :) Yani özetle
bir worker role'un Run metodunun hiçbir zaman bitmemesi gerekiyor. Bunun
için de basit bir While(True) işi görmeye yetiyor. Tabi çılgın looplara
girmemek adına arada Thread'i Sleep'e almak mantıklı olacaktır ama yoğun
işlemler yaptığınızda Sleep süresini de yönetmenizde fayda var. Aşırı
yoğun senaryolarda Sleep yapmayabilirsiniz de, çünkü zaten Worker'ın
yapacağı birçok iş vardır. Tüm bu işler / işlemler de tahmin
edebileceğiniz üzere Whie(True) döngümüz içerisinde gerçekleştirilir.

Eeehh :) Worker olayı bu kadar :) Basit olacağını söylemiştim. Web
Role'lerde olan tüm Role özellikleri Worker Role'lerde de var. Yani yine
vm size ve instance count gibi değişkenler ayarlanarak Worker Role'ler
de genişletilebiliyor. Şimdi isterseniz gelin Web Role ve Worker Role'ün
beraber kullanımına göz atalım.

### Web Role ve Worker Role el ele!

Yeni bir Cloud Service projesi yaratıp içine bir Worker ve bir de Web
Role koyuyoruz. Web Role'ün hemen global.asax'da
ConfigurationPublisher'ı set ediyoruz ki Storage Account'umuzu da
kullanabilelim rahatlıkla.

**[C\#]**

<span style="color:blue;">void</span> Application\_Start(<span
style="color:blue;">object</span> sender, <span
style="color:#2b91af;">EventArgs</span> e)\
 {\
    <span
style="color:green;">// Code that runs on application startup</span>\
     <span
style="color:#2b91af;">CloudStorageAccount</span>.SetConfigurationSettingPublisher(\
                               
(configName, configSettingPublisher) =\>\
    {\
        <span style="color:blue;">var</span> connectionString = <span
style="color:#2b91af;">RoleEnvironment</span>.GetConfigurationSettingValue(configName);\
         configSettingPublisher(connectionString);\
    }\
    );\
}

Publisher hazır olduktan sonra web sitesine bir ASPX dosyası, sayfaya da
bir TextBox ve Button ekleyin :) Sonra da Button'un Click'e aşağıdaki
kodu yazalım.

**[C\#]**

<span style="color:blue;">var</span> storageAccount = <span
style="color:#2b91af;">CloudStorageAccount</span>.FromConfigurationSetting(<span
style="color:#a31515;">"DataConnectionString"</span>);\
 <span
style="color:blue;">var</span> Client = storageAccount.CreateCloudQueueClient();\
 <span
style="color:#2b91af;">CloudQueue</span> q = Client.GetQueueReference(<span
style="color:#a31515;">"yenikuyruk"</span>);\
 <span style="color:#2b91af;">CloudQueueMessage</span> yeniMesaj = <span
style="color:blue;">new</span> <span
style="color:#2b91af;">CloudQueueMessage</span>(<span
style="color:blue;">\
                                string</span>.Format(<span
style="color:#a31515;">"text:{0}"</span>,TextBox1.Text));\
 q.AddMessage(yeniMesaj, <span
style="color:#2b91af;">TimeSpan</span>.FromMinutes(1),<span
style="color:#2b91af;">TimeSpan</span>.FromMinutes(2));

Kodumuza baktğımızda DataConnectionString diye bir yerden StorageAccount
connection stringinin alındığını ve "yenikuyruk" adında bir kuyruk
yaratılıp TextBox içerisindeki metnin kuyruğa mesaj olarak konduğunu
görebiliriz. Tabi burada örnek olduğu için konu ve işlemler epey basit
duruyor ama buradaki senaryoyu gerçek hayattaki bir senaryoyla
eşleştirmek gerekirse... bir E-Ticaret sitesinde alışveriş yapan bir
müşterinin alışveriş sepetinin SQL Azure'a tamamen kaydedildikten sonra
sipariş onaylandığında da siparişle ilgili işlemlerin yapılması üzerine
sipariş ID'sini kuyruğa attığını düşünebilirsiniz. Özetle arka tarafta
yapılması gereken bir işlemle ilgili bilgileri SQL Azure ve Blob'lara
koyduktan sonra Web Role yani web sitesi işlemin bir referansını da
kuyruğa atıyor. Böylece bu noktadan sonra Worker Role'ler gelip kuyruğa
bakıp eğer yeni işlem varsa işlemi alıp üzerinde çalışabilecekler.

Yeni Worker Role'ümüze geçtiğimizde ise yine Worker Role içerisinde de
Storage Client'ı kullanacağımız için SettingsPublisher'ı ayarlamamız
gerekiyor.

**[C\#]**

<span style="color:blue;">public</span> <span
style="color:blue;">override</span> <span
style="color:blue;">bool</span> OnStart()\
 {\
    <span
style="color:#2b91af;">ServicePointManager</span>.DefaultConnectionLimit = 12;\
\
    <span
style="color:#2b91af;">CloudStorageAccount</span>.SetConfigurationSettingPublisher(\
                            (configName, configSettingPublisher) =\>\
    {\
        <span style="color:blue;">var</span> connectionString = <span
style="color:#2b91af;">RoleEnvironment</span>.GetConfigurationSettingValue(configName);\
         configSettingPublisher(connectionString);\
    }\
    );\
    <span style="color:blue;">return</span> <span
style="color:blue;">base</span>.OnStart();\
 }

Herşey hazır olduğuna göre artık Worker Role içerisindeki kodumuzu
yazmaya başlayabiliriz. Amacımız Storage Account'a bağlanıp kuyrukta
yapılması gereken işlem var mı? yok mu? kontrol etmek. Eğer işlem varsa,
işlemi yapıp kuyruktan kaldırmamız, eğer işlem yoksa bir süre sonra
tekrar kontrol etmemiz gerekecek.

**[C\#]**

<span style="color:blue;">public</span> <span
style="color:blue;">override</span> <span
style="color:blue;">void</span> Run()\
 {\
    <span style="color:blue;">var</span> storageAccount = <span
style="color:#2b91af;">CloudStorageAccount</span>.FromConfigurationSetting(<span
style="color:#a31515;">"DataConnectionString"</span>);\
     <span
style="color:blue;">var</span> Client = storageAccount.CreateCloudQueueClient();\
     <span
style="color:#2b91af;">CloudQueue</span> q = Client.GetQueueReference(<span
style="color:#a31515;">"yenikuyruk"</span>);\
\
    <span style="color:blue;">while</span> (<span
style="color:blue;">true</span>)\
     {\
        <span
style="color:#2b91af;">CloudQueueMessage</span> mevcutMesaj = q.GetMessage();\
\
        <span style="color:blue;">if</span> (mevcutMesaj != <span
style="color:blue;">null</span>)\
         {\
            <span
style="color:blue;">string</span> s = mevcutMesaj.AsString;\
\
            <span style="color:#2b91af;">Trace</span>.WriteLine(<span
style="color:blue;">string</span>.Format(<span
style="color:#a31515;">"Gelen Mesaj : {0}"</span>, s), <span
style="color:#a31515;">"Information"</span>);\
\
            q.DeleteMessage(mevcutMesaj);\
        }\
        <span
style="color:#2b91af;">Thread</span>.Sleep(10000);               \
     }\
}

Yukarıdaki kod tam da planladığımız şeyi yapıyor. Run metodu
içerisindeki önceki StorageAccount'a sonra da web arayüzünen mesajların
aktarıldığı kuyruğa ulaşıyoruz. While(True) döngüsü içerisinde kuyruktan
sürekli bir mesaj isteyip eğer gelen mesaj null değilse yani mesaj varsa
mesajı alıp Trace olarak yazıyoruz. Bu noktada tabi ki sizin farklı
kodlar yazarak işlemler yapmanız mümkün. Bizim örnek test amaçlı olduğu
için ben sadece Trace'e yazdırdım.

Herşey bittikten sonra artık kuyruk üzerinden **DeleteMessage** diyerek
aldığımız ve işlemlerini tamamladığımız işin mesajını da siliyoruz.

Sanırım şu anda hem Worker Role, Web Role ve Storage Queue ile beraber
taşları yerli yerine oturtabilmişizdir ;) Uygulamanızı çalıştırmadan da
her iki role'ün de CSDEF ve CSCFG'lerinden Storage Account connection
stringlerini yazmayı unutmayın. Uygulamayı çalıştırdıktan sonra Compute
Emulator'ın console görüntülerinden Trace'leri izleyebilirsiniz ;)

Hepinize kolay gelsin.


