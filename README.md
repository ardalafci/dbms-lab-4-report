# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [X]  **Blok bazlı disk erişimi** → block_id + offset
- [X]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [X]  VT hangisini kullanır? **Sayfa** okuması


---

### Buffer Pool

- [ ]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [X]  LRU / CLOCK gibi algoritmaları
- [X]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [X]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
- [ ]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
- [ ]  Clustered vs Non-Clustered Index Kavramı
- [ ]  InnoDB satırı diskte nasıl durur?
- [ ]  LSM-tree (LevelDB, RocksDB) farkı
- [ ]  PostgreSQL heap + index ayrımı

DB diske yazarken:

- [X]  WAL (Write Ahead Log) İlkesi
- [ ]  Log disk (fsync vs write) sistem çağrıları farkı

---



### Veri Yönetimi Karşılaştırma Tablosu: 

| Kavram | Bellek (C / Sistem) | Veri Tabanı (PostgreSQL / Disk) | Teknik Açıklama |
| :--- | :--- | :--- | :--- |
| **Adresleme** | RAM Adresi (Pointer) | `ItemPointerData` (Block + Offset) | Bellek byte bazlı, DB ise 8KB'lık sayfa (page) bazlı adresleme yapar. |
| **Erişim Hızı** | $O(1)$ (Doğrudan) | Disk I/O (Milisaniye seviyesi) | Bellek hızı nanosaniye, disk hızı ise fiziksel kafa hareketiyle (I/O) sınırlıdır. |
| **Kimlik (Key)** | Değişken Adı / Adres | İndeks Anahtarı (B+ Tree Key) | RAM'de veriye adresle ulaşılırken, DB'de indeks yapısındaki anahtarlarla ulaşılır. |
| **Veri Yapısı** | Array / Struct / Linked List | B+ Tree / Page Layout | RAM'de veriler CPU dostu yapılarla, diskte ise I/O dostu ağaç yapılarıyla tutulur. |
| **Ön Bellek** | CPU L1/L2/L3 Cache | Buffer Pool (Shared Buffers) | CPU donanımsal cache kullanırken, DB yazılımsal Buffer Pool (Clock Sweep) kullanır. |
| **Güvenlik** | Yok (Uçucu veri) | WAL (Write-Ahead Logging) | Bellekteki veri elektrik kesintisinde kaybolur, DB ise log tutarak kalıcılık sağlar. |

---

# Video [Linki](https://youtu.be/7bdKTGqPug4) 


---

# Açıklama (Ort. 600 kelime)

Bu çalışma kapsamında, dünyanın en gelişmiş açık kaynak ilişkisel veri tabanı yönetim sistemlerinden biri olan PostgreSQL'in (v18-dev) mimarisi; sistem programlama ve veri yapıları perspektifinden derinlemesine analiz edilmiştir. Modern veri tabanlarının devasa veri yığınlarını milisaniyeler içerisinde nasıl yönettiği, belleği nasıl optimize ettiği ve olası sistem çökmelerine karşı veri bütünlüğünü nasıl koruduğu sorularına, doğrudan kaynak kodları (C dili) üzerinden yanıtlar aranmıştır. Disk üzerindeki fiziksel adreslemeden başlayarak, tampon bellek (Buffer Pool) yönetim algoritmalarına, B+ Tree tabanlı indeksleme mekanizmalarından WAL güvenlik protokolüne kadar uzanan bu süreç, teorik bilgisayar bilimleri kavramlarının endüstriyel standartlardaki pratik karşılıklarını ortaya koymaktadır.

PostgreSQL'in veriyi diskte nasıl adreslediğini (block_id + offset) ve bu adresleme için itemptr.h dosyasındaki ItemPointerData yapısını nasıl kullandığını teknik olarak inceledik. Özellikle md.c dosyasındaki mdreadv fonksiyonu üzerinden, veri tabanının FileReadV sistem çağrısıyla diskin belirli bir konumuna (seekpos) giderek nasıl Rastgele Erişim yaptığını ve seekpos += nbytes satırı ile işletim sistemi seviyesindeki eksik okumaları (Partial Read) nasıl yönettiğini kod seviyesinde kanıtladık.

PostgreSQL'in sayfa bazlı çalıştığını PageHeaderData yapısındaki pd_linp dizisinden anlıyoruz. Bu dizi, tek bir sayfa içinde çok sayıda satırın adresini tutar. Biz diskten bu yapıyı bir bütün olarak okuduğumuzda, aslında içindeki tüm satırları ve sayfanın boş alan bilgilerini (pd_lower, pd_upper) tek seferde belleğe almış oluyoruz.

LRU / CLOCK Algoritmaları ve I/O Minimizasyonu (Bellek Yönetimi) PostgreSQL'in bellek yönetimi mimarisi, diskten okunan sayfaların RAM üzerinde verimli bir şekilde saklanmasını sağlayan Buffer Pool mekanizmasına dayanır ve bu süreç freelist.c dosyasındaki Clock Sweep (CLOCK) algoritmasıyla yönetilir. Sistem, her sayfanın kullanım puanını (usage_count) denetleyerek verinin popülerliğini saptar; eğer bir sayfa yakın zamanda kullanılmışsa puanı düşürülüp sayfaya bir "şans" daha verilerek RAM'de tutulması sağlanır ki bu durum, verinin diskten tekrar okunmasını engelleyerek Disk I/O operasyonlarını doğrudan minimize eden en kritik performans hamlesidir. Son aşamada ise puanı sıfıra düşen ve artık "soğuk" olarak nitelendirilen sayfalar "kurban" (victim) seçilerek bellekten tahliye edilir; böylece sınırlı RAM kapasitesiyle devasa disk verilerinin en verimli şekilde kopyalanması ve yönetilmesi (Caching & Replacement) garanti altına alınmış olur.

PostgreSQL’in indeksleme performansı, milyonlarca kayıt arasından hedeflenen verinin fiziksel adresine minimum disk okuması (I/O) ile ulaşmayı sağlayan B+ Tree yapısına dayanır ve bu süreç kod seviyesinde nbtsearch.c dosyasındaki fonksiyonlarla yönetilir. Arama işlemi başladığında _bt_search() fonksiyonu ağacın kökünden (root) yapraklara (leaf) doğru hiyerarşik bir iniş gerçekleştirerek $O(\log n)$ karmaşıklığıyla hedef bloğu saptar; ilgili blok belleğe alındıktan sonra ise _bt_binsrch() fonksiyonu devreye girerek sayfa içindeki sıralı kayıtlar üzerinde Binary Search (ikili arama) algoritmasını işleterek CPU maliyetini minimize eder.  Bu yapının en büyük avantajı olan sıralı tarama (range scan) yeteneği ise nbtree.h dosyasındaki btpo_next ve btpo_prev işaretçileriyle sağlanır; bu yatay bağlantılar sayesinde sistem, belirli aralıktaki verileri okurken her seferinde ağacın tepesine çıkmak yerine disk üzerindeki kardeş sayfalar arasında doğrudan geçiş yaparak sorgu hızını maksimize eder.

PostgreSQL’in veri güvenliğini sağlayan en temel kuralı olan WAL (Write-Ahead Logging) ilkesi, bufmgr.c dosyası içerisindeki FlushBuffer fonksiyonu ile kod seviyesinde garanti altına alınmıştır. Bu mekanizmaya göre, bellekteki (Shared Buffers) kirli bir sayfanın ana veri dosyasına (smgrwrite) kalıcı olarak yazılabilmesi için, o sayfadaki değişikliği tanımlayan günlük kaydının (LSN) mutlaka daha önce diske işlenmiş olması gerekir; bu durum kodda yer alan XLogFlush(recptr) çağrısının gerçek veri yazımından önce çalışmasıyla ispatlanmaktadır. Bu sıralama sayesinde, sistem veri dosyalarını güncellemeden önce hızlı olan sıralı disk erişimiyle (Sequential I/O) log kaydını diske "not eder" ve olası bir çökme anında henüz veri dosyasına yansıtılmamış tüm işlemler bu loglar sayesinde %100 tutarlılıkla geri yüklenebilir. Sonuç olarak WAL ilkesi, veri tabanının hem rastgele disk erişimi maliyetini düşürerek performans kazanmasını hem de çökme sonrası kurtarma (crash recovery) kabiliyeti kazanmasını sağlayarak veri bütünlüğünü korur.


## VT Üzerinde Gösterilen Kaynak Kodları

Disk Erişimi: [Link1](https://github.com/postgres/postgres/blob/master/src/include/storage/itemptr.h) [Link2](https://github.com/postgres/postgres/blob/master/src/backend/storage/smgr/md.c) \
VT için Page (Sayfa) Anlamı: [Link](https://github.com/postgres/postgres/blob/master/src/include/storage/bufpage.h) \
Buffer Pool: [Link](https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/freelist.c) \
Veri Yapıları Perspektifi (B+ Tree): [Link1](https://github.com/postgres/postgres/blob/master/src/include/access/nbtree.h)  [Link2](https://github.com/postgres/postgres/blob/master/src/backend/access/nbtree/nbtsearch.c) \
DB diske yazarken (WAL): [Link](https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/bufmgr.c)

... \
...
