# Ödev 3.2 — Cevaplar

SQL dosyaları: `sql-mastery/queries/` altında, her sorunun numarasıyla.

---

## 01 — Hesabı kapalı kullanıcılar kimler, kaç tane?

**SQL:** `sql-mastery/queries/q01_pasif_kullanicilar.sql`

```sql
SELECT count(*) AS pasif_kullanici_sayisi
FROM users
WHERE is_active = false;

SELECT id, email, country, created_at
FROM users
WHERE is_active = false
ORDER BY created_at DESC
LIMIT 10;
```

**Sonuç:**

```
 pasif_kullanici_sayisi
------------------------
                    993
(1 row)

  id   |        email        | country |       created_at
-------+---------------------+---------+------------------------
  3491 | user3491@ornek.com  | DE      | 2026-03-30 12:28:28+00
   867 | user867@ornek.com   | TR      | 2026-03-27 21:31:45+00
 14388 | user14388@ornek.com | TR      | 2026-03-27 12:56:38+00
  2383 | user2383@ornek.com  | TR      | 2026-03-26 17:42:52+00
 13203 | user13203@ornek.com | GB      | 2026-03-26 08:52:39+00
 12217 | user12217@ornek.com | TR      | 2026-03-25 15:47:42+00
 18944 | user18944@ornek.com | TR      | 2026-03-25 14:35:43+00
  6050 | user6050@ornek.com  | NL      | 2026-03-25 13:10:35+00
    93 | user93@ornek.com    | TR      | 2026-03-23 07:21:40+00
 19478 | user19478@ornek.com | TR      | 2026-03-23 07:14:52+00
(10 rows)
```

**İş yorumu:**

20.000 kayıtlı kullanıcının 993'ü, yani %4,97'si kapalı durumda. Tek başına bu oran bir alarm değil; kıyaslanacak bir hedef ya da geçmiş dönem olmadan "yüksek mi düşük mü" sorusunun cevabı yok. Asıl sınır, sorgunun cevaplayamadığı yerde: şemada hesabın **ne zaman** kapandığını tutan bir kolon bulunmuyor. `created_at` açılış tarihidir, dolayısıyla yukarıdaki liste "en son kapanan hesaplar" değil, "en son açılmış olan kapalı hesaplar"dır. Bunun pratik sonucu şu: kapanma oranının zaman içinde artıp artmadığını ya da bir kampanya sonrası sıçrama olup olmadığını bu şemayla ölçemeyiz. Terk (churn) analizi yapılacaksa şemaya `users.deactivated_at` eklenmesi gerekir; aksi hâlde elimizde yalnızca bugünün anlık fotoğrafı var, bir eğilim yok.

**İyileştirme notu:** `users.deactivated_at timestamptz NULL` kolonu — kapanma tarihini tutar ve churn analizini mümkün kılar. Kararı Ödev 3.4'e (analitik katman) bırakıyoruz; orada SCD2 ile birlikte değerlendirilecek.

---

## 02 — Ülkesi bilinmeyen kullanıcı sayısı kaç?

**SQL:** `sql-mastery/queries/q02_ulkesi_bilinmeyen.sql`

**Sonuç:**

```
 yanlis_yontem          <- WHERE country = NULL
---------------
             0

 ulkesi_bilinmeyen      <- WHERE country IS NULL
-------------------
               965

 toplam_kullanici | ulkesi_dolu | ulkesi_bos
------------------+-------------+------------
            20000 |       19035 |        965

 bos_yuzde
-----------
      4.83

    ulke    | kullanici
------------+-----------
 TR         |     13423
 DE         |      1869
 US         |       972
 bilinmiyor |       965
 NL         |       935
 FR         |       932
 GB         |       904
```

**İş yorumu:**

Kullanıcıların %4,83'ünün (965 kişi) ülkesi kayıtlı değil. Bu oranın kendisinden daha önemlisi, bu kitlenin raporlarda nasıl davrandığı: `bilinmiyor` grubu 965 kişiyle Hollanda, Fransa ve Birleşik Krallık kullanıcılarının her birinden daha kalabalık. Ülke kırılımlı bir rapor `WHERE country IS NOT NULL` ile ya da sessizce bir `GROUP BY country` ile hazırlandığında bu 965 kişi tabloda hiç görünmez ve toplamlar, ana tablodaki kullanıcı sayısıyla tutmaz. Bu tür raporlarda iki seçenek var: ya `COALESCE(country, 'bilinmiyor')` ile grubu açıkça göstermek, ya da hariç tutup bunu raporun başında yazmak. Kabul edilemez olan üçüncü yol, eksik ülkeleri baskın değere (burada TR) doldurmak olurdu — bu, veriyi temizlemek değil uydurmaktır ve ülke bazlı her kararı bozar.

Metodolojik not: `WHERE country = NULL` sorgusu hata vermeden 0 döndürdü. `NULL` ile yapılan eşitlik karşılaştırmasının sonucu `TRUE` değil `NULL`'dur, `WHERE` ise yalnızca `TRUE` olan satırları geçirir. Bu yüzden eksik veri sorgulanırken tek doğru araç `IS NULL` / `IS NOT NULL`'dır. Aynı kural `count(*)` ile `count(kolon)` arasındaki farkı da açıklar: `count(kolon)` `NULL` satırları saymaz, ikisinin farkı doğrudan eksik veri sayısını verir.

---

## 03 — Liste fiyatı 500'ün üzerindeki ürünler, pahalıdan ucuza

**SQL:** `sql-mastery/queries/q03_pahali_urunler.sql`

**Sonuç (özet):** 2000 üründen 61'i (%3,05) 500 TL'nin üzerinde. En pahalı ürün 1.675,69 TL.

```
  id  |    sku    | list_price | unit_cost | stock_cached
------+-----------+------------+-----------+--------------
  273 | SKU-00273 |    1675.69 |   1006.72 |         1358
 1040 | SKU-01040 |    1397.37 |    953.94 |         1477
  170 | SKU-00170 |    1286.03 |    877.73 |         1025
  547 | SKU-00547 |    1285.69 |    948.72 |         1077
 1056 | SKU-01056 |    1253.00 |    662.77 |         1433
 1328 | SKU-01328 |     824.75 |    948.46 |         1454   <- maliyet > fiyat
```

**İş yorumu:**

Katalogda 500 TL üstü ürün sayısı 61 ve bu, fiyat dağılımının beklenen sonucu: fiyatlar lognormal dağılıyor, medyan ~100 TL, dolayısıyla 500 TL eşiğini aşma olasılığı ~%2,9. Gözlenen %3,05 bu beklentiyle uyumlu — yani katalogda fiyatlama anomalisi yok, uzun kuyruk normal davranıyor. Buna karşılık listede `SKU-01328` göze çarpıyor: 824,75 TL liste fiyatına karşı 948,46 TL birim maliyet, yani her satışta zarar. Katalogda bu durumda 10 ürün var. Zararına satış bilinçli bir kampanya kararı olabilir (stok eritme, müşteri kazanma), ama bilinçsizse doğrudan marj kaybıdır; bu yüzden şemada `unit_cost <= list_price` kısıtı bilerek konulmadı — meşru bir iş kararını veritabanı reddetmemeli, ama rapor bunu görünür kılmalı.

**Metodolojik not:** `ORDER BY list_price DESC, id` yazımındaki ikinci kolon deterministik sıralama içindir. Fiyatı birebir eşit iki ürün olduğunda sıra rastgele kalmaz; sayfalamalı raporlarda aynı kaydın iki sayfada birden çıkmasını engeller. Ayrıca kategoriye göre sıralanmış listede `LIMIT` kullanmak, sondaki kategorileri görünmez kılar — "her kategoriden ilk N" sorusu ayrı bir araç (window function) gerektirir.

---

## 04 — Her ürünün marjı ve marj yüzdesi nedir?

**SQL:** `sql-mastery/queries/q04_urun_marji.sql`

**Sonuç:** Katalog ortalaması %42,64. En yüksek marj %64,96; zararına satılan 10 ürün var ve hepsinin marjı tam olarak −%15,00.

```
 ortalama_marj_yuzde | en_dusuk_marj | en_yuksek_marj
---------------------+---------------+----------------
               42.64 |       -123.71 |         668.97

  id  |    sku    | list_price | unit_cost |  marj   | marj_yuzde
------+-----------+------------+-----------+---------+------------
 1328 | SKU-01328 |     824.75 |    948.46 | -123.71 |     -15.00
  572 | SKU-00572 |     714.34 |    821.49 | -107.15 |     -15.00
  115 | SKU-00115 |     299.86 |    344.84 |  -44.98 |     -15.00
  ...   (10 urunun tamami -15.00)
```

**İş yorumu:**

Katalog genelinde ortalama marj %42,64 ve bu, maliyet kurgusunun beklenen sonucu: birim maliyet liste fiyatının %35–%80'i arasında dağıldığı için marj oranı %20–%65 bandında, ortalaması %42,5 olmalıydı. Gözlenen değer bu beklentiyle örtüşüyor, yani fiyatlama tarafında sistematik bir sapma yok.

Asıl bulgu zarar eden ürünlerde: 10 ürünün marj yüzdesi virgülden sonraki iki hane dahil tam olarak −15,00. Gerçek bir katalogda on farklı ürünün marjının birebir aynı çıkması pratikte imkânsızdır; böyle bir düzgünlük neredeyse her zaman üç şeyden birini gösterir — bir varsayılan değerin tabloya yazılması, bir dönüşüm adımının sabit uygulanması, ya da verinin sentetik olması. Burada üçüncüsü geçerli: sentetik veri üreticisi bu ürünlerin maliyetini `list_price * 1.15` olarak sabitliyor. Bu, üreticinin gerçekçilik kusurudur ve düzeltilmesi gerekir; gerçek bir katalogda zarar oranları ürün ürün değişir. Analiz açısından çıkarılacak ders ise metodolojik: fazla düzgün olan veri, doğru veri değil şüpheli veridir.

İkinci not, sıralama ölçüsü üzerine: lira cinsinden en yüksek marj 668,97 TL'yken yüzde cinsinden en yüksek marj %64,96'dır ve bu ikisi farklı ürünlere aittir. Marj yüzdesine göre sıralanan bir liste ucuz ürünleri, lira marjına göre sıralanan liste pahalı ürünleri öne çıkarır. Hangi listenin doğru olduğu soruya bağlıdır — kârlılık kararları genelde lira marjına, fiyatlama kararları yüzdeye bakar.

**İyileştirme notu:** `scripts/generate.py` içindeki zarar anomalisi sabit çarpan yerine rastgele bir aralık kullanmalı (örn. `list_price * random.uniform(1.02, 1.35)`).

---

## 05 — Stoğu tükenmiş ama hâlâ aktif görünen ürünler hangileri?

**SQL:** `sql-mastery/queries/q05_stogu_biten_aktif_urunler.sql`

**Sonuç:** 2000 üründen 16'sının stoğu negatif, 1984'ü pozitif, **tam olarak sıfır olan hiç yok**. Negatif stoklu 16 üründen 12'si hâlâ aktif.

```
 pozitif_stok | sifir_stok | negatif_stok
--------------+------------+--------------
         1984 |          0 |           16

  id  |    sku    | list_price | stock_cached | is_active
------+-----------+------------+--------------+-----------
  169 | SKU-00169 |      34.19 |       -29919 | t
  118 | SKU-00118 |      29.57 |       -13427 | t
  186 | SKU-00186 |     290.86 |        -8762 | t
 1573 | SKU-01573 |      53.65 |        -6608 | t
  720 | SKU-00720 |      20.68 |        -4225 | t
```

**İş yorumu:**

Negatif stok fiziksel olarak imkânsızdır: var olmayan mal satılamaz. Buna rağmen 16 ürünün defter bakiyesi eksidedir ve en uç örnekte açık 29.919 adettir. Bu, satışların engellenmediği ve stok kontrolünün sipariş anında yapılmadığı anlamına gelir; 12 ürünün hâlâ `is_active = true` olması, bu ürünlerin satışa açık kalmaya devam ettiğini gösterir. Gerçek bir işletmede bunun karşılığı, teslim edilemeyecek siparişlerin alınmasıdır.

Bulgunun kaynağı veri tarafında: sentetik üretici her ürüne, talebinden bağımsız olarak 4–8 kez ve 50–500 adet arası alım yazıyor. Satışlar ise güç yasasıyla dağılıyor, yani en popüler birkaç ürün toplam talebin büyük kısmını soğuruyor. Alım hacmi popülerlikle ölçeklenmediği için yalnızca en çok satan ürünler eksiye düşüyor — 1984 ürünün stoğuna ise neredeyse hiç dokunulmuyor. Bu keskin ayrışmanın kendisi de bir bulgu: talep, ürünlerin küçük bir azınlığında yoğunlaşıyor.

**Metodolojik not:** Soru `stock_cached = 0` diye yazılsaydı sonuç sıfır satır olurdu ve "stoğu biten ürün yok" sonucuna varılırdı. Hiçbir ürünün stoğu tam olarak sıfır değil. Eşik koşullarında "tam olarak eşit" yerine "bu tarafta kalan" (`<= 0`) yazmak, sessiz yanlış sonucu engelliyor.

**Tasarım notu:** K-003'te `stock_cached` üzerine `CHECK (stock_cached >= 0)` konulmamıştı; gerekçe, türetilmiş kolonda negatif değerin hata değil alarm olduğuydu. Kısıt konulmuş olsaydı `make refresh-stock` başarısız olur, stok hiç güncellenemez ve bu bulgu görünmez kalırdı.

**İyileştirme notu:** `scripts/generate.py` içinde alım miktarı ürün popülerliğiyle ölçeklenmeli; aksi hâlde stok defteri gerçekçi olmuyor.

---

## 06 — Son 30 günde açılan hesap sayısı kaç?

**SQL:** `sql-mastery/queries/q06_son_30_gun_hesaplar.sql`

**Sonuç:**

```
 bugune_gore            <- WHERE created_at >= now() - interval '30 days'
-------------
           0

       ilk_kayit        |       son_kayit        |             simdi
------------------------+------------------------+-------------------------------
 2024-01-01 00:05:29+00 | 2026-03-30 21:11:31+00 | 2026-09-11 09:30:48.967777+00

 son_7_gun | son_30_gun | son_90_gun     <- referans: max(created_at)
-----------+------------+------------
       156 |        717 |       2152
```

**İş yorumu:**

Veri setinin son kullanıcı kaydı 2026-03-30, sorgunun çalıştırıldığı tarih ise 2026-09-11. Aradaki 165 günlük boşluk nedeniyle `now()` referanslı "son 30 gün" filtresi hiçbir satır döndürmüyor. Sorgu hatalı değil; göreli tarih filtrelerinin referans noktası hatalı. Statik veri setlerinde ve test ortamlarında doğru referans, sistemin bugünü değil verinin kendi son tarihidir.

Veri setinin kendi son tarihine göre bakıldığında son 30 günde 717, son 7 günde 156, son 90 günde 2152 hesap açılmış. Bu değerler beklenen aralıkta: 20.000 kullanıcı 820 günlük pencereye düzgün dağıtıldığı için günlük ortalama ~24 hesaptır; 30 gün için beklenen 732, 90 gün için 2195. Yani kullanıcı kazanımında ne hızlanma ne yavaşlama var — kayıt akışı sabit. Bunun kendisi bir bulgu: sipariş hacminde mevsimsellik varken (kasım-aralık zirvesi) kullanıcı kazanımında hiç yoksa, kampanya dönemlerindeki satış artışı yeni müşteriden değil mevcut müşterinin daha çok alışverişinden geliyor demektir.

**Metodolojik not:** Göreli tarih filtresi yazarken referansın `now()` mı yoksa verinin `max()` değeri mi olacağı açıkça kararlaştırılmalı ve raporda belirtilmelidir. Üretim raporlarında `now()` doğrudur; geçmiş veri analizinde ve test ortamlarında verinin kendi ufku doğrudur.

---

## 07 — Siparişler durumlarına göre nasıl dağılıyor?

**SQL:** `sql-mastery/queries/q07_siparis_durum_dagilimi.sql`

**Sonuç:**

```
  status   | siparis_sayisi | yuzde
-----------+----------------+-------
 delivered |          63110 | 63.11
 shipped   |          12933 | 12.93
 paid      |           9931 |  9.93
 created   |           6977 |  6.98
 cancelled |           5070 |  5.07
 returned  |           1979 |  1.98

-- durum + ulke kirilimi (ilk satirlar)
 delivered | TR               |          41854
 delivered | DE               |           6018
 delivered |                  |           3281   <- shipping_country NULL
```

**İş yorumu:**

Siparişlerin %63,11'i teslim edilmiş, %12,93'ü yolda, %9,93'ü ödenmiş ama henüz kargolanmamış, %6,98'i ise oluşturulup ödenmemiş durumda. İptal oranı %5,07, iade oranı %1,98. Sağlıklı bir okuma için bu altı durum bir hunidir: siparişlerin yaklaşık %7'si ödeme adımını hiç geçmiyor, %5'i iptal ediliyor, kalan kısmın büyük bölümü teslimata ulaşıyor. Ödenmiş ama kargolanmamış %9,93'lük dilim operasyonel bir birikimdir; bu oranın zaman içinde artması depo tarafında darboğaz anlamına gelir. Dönüşüm oranlarının adım adım hesabı 19. soruda yapılacak.

Ülke kırılımında dikkat çeken bulgu, `shipping_country` değeri `NULL` olan 3.281 teslim edilmiş siparişin ayrı bir grup olarak görünmesi — teslim edilen siparişlerin %5,2'si. Bu grup raporda etiketsiz, boş bir hücre olarak çıkıyor ve okuyan kişi tarafından biçimlendirme hatası sanılmaya açık. Ülke kırılımlı her raporda `COALESCE(shipping_country, 'bilinmiyor')` kullanılmalı.

**Metodolojik not:** `GROUP BY` ile `WHERE` farklı katmanlarda çalışır: `WHERE` gruplama öncesinde satırları, `HAVING` gruplama sonrasında grupları eler. Ayrıca `GROUP BY` ve `DISTINCT`, eşitlik değil "ayırt edilemezlik" ölçütü kullandığı için tüm `NULL` değerleri tek bir grupta toplar — `WHERE country = NULL` hiçbir satır bulmazken `GROUP BY country` `NULL` grubunu görünür kılar. Aynı veride iki farklı `NULL` kuralı geçerlidir.

---

## 08 — Ürünleri fiyat bandına göre sınıflandır

**SQL:** `sql-mastery/queries/q08_fiyat_bantlari.sql`

Bantlar: `ucuz` < 50 TL, `orta` 50–250 TL, `pahali` > 250 TL.

**Sonuç:**

```
 fiyat_bandi | urun_sayisi | yuzde | en_dusuk | en_yuksek | ortalama | ort_marj_yuzde
-------------+-------------+-------+----------+-----------+----------+----------------
 ucuz        |         439 | 21.95 |     9.90 |     49.96 |    33.29 |          42.40
 orta        |        1283 | 64.15 |    50.03 |    249.04 |   118.44 |          43.02
 pahali      |         278 | 13.90 |   250.39 |   1675.69 |   420.25 |          41.22
```

**İş yorumu:**

Katalog orta segmentte yoğunlaşıyor: ürünlerin %64'ü 50–250 TL bandında, %22'si bu bandın altında, %14'ü üstünde. Dağılım lognormal fiyat kurgusunun beklenen sonucu (50 TL altı için ~%21, 250 TL üstü için ~%14) ve sistematik bir sapma göstermiyor.

Asıl bulgu marj sütununda: üç bandın ortalama marjı sırasıyla %42,40, %43,02 ve %41,22 — yani fiyat segmenti ile kârlılık arasında pratikte hiç ilişki yok. Gerçek kataloglarda segment başına marj politikası farklılaşır; giriş seviyesi ürünlerde marj düşük tutulup hacimle kazanılır, premium segmentte marj yüksektir. Buradaki düzlük, birim maliyetin fiyattan bağımsız rastgele bir oran olarak üretilmesinden kaynaklanıyor ve üreticinin gerçekçilik kusurudur. Segment bazlı fiyatlama analizinin bu veri üzerinde anlamlı sonuç vermeyeceği not edilmelidir.

İkinci bulgu en düşük fiyatta: yedi ürünün liste fiyatı tam olarak 9,90 TL. Üretici fiyatı `min(max(lognormal, 9.9), 25000)` ile kırpıyor, dolayısıyla dağılımın alt kuyruğu tek bir değere yığılıyor. Bu, kırpılma (clipping/censoring) etkisidir ve gerçek verilerde de sık görülür — sensör alt sınırı, formda zorunlu minimum tutar, API üst limiti. Kırpılmış veriyle hesaplanan ortalama ve standart sapma, gerçek dağılımı değil kırpılmış hâlini ölçer. Bir değerde yığılma görüldüğünde ilk soru, o değerin bir sınır olup olmadığı olmalıdır.

**Metodolojik not:** `CASE WHEN` dallarında sıra belirleyicidir; ilk uyan koşul kazanır, bu yüzden dar aralık geniş aralıktan önce yazılmalıdır. `ELSE` yazılmazsa eşleşmeyen satırlar `NULL` etiketli bir gruba düşer. Kategorik bantlar `ORDER BY` ile alfabetik sıralanırsa anlamsız bir sıra çıkar (`orta, pahali, ucuz`); bandın temsil ettiği sayıya göre sıralamak gerekir (`ORDER BY min(list_price)`).

---

## 09 — Haftanın hangi gününde en çok sipariş veriliyor?

**SQL:** `sql-mastery/queries/q09_haftanin_gunu.sql`
Oturum saat dilimi: `Etc/UTC`.

**Sonuç:**

```
 gun_no |  gun_adi  | siparis | yuzde
--------+-----------+---------+-------
      0 | Sunday    |   16941 | 16.94
      6 | Saturday  |   16647 | 16.65
      1 | Monday    |   13678 | 13.68
      2 | Tuesday   |   13351 | 13.35
      3 | Wednesday |   13288 | 13.29
      4 | Thursday  |   13108 | 13.11
      5 | Friday    |   12987 | 12.99

-- aylik (ilk 12)
 2025-12-01 | 10907      2026-04-01 | 7965      2026-01-01 | 3892
 2025-11-01 |  9627      2026-03-01 | 7524      2025-09-01 | 3565
 2026-06-01 |  9446      2025-10-01 | 4064      2025-07-01 | 3516
 2026-05-01 |  8464      2026-02-01 | 3994      2025-08-01 | 3317
```

**İş yorumu:**

Sipariş hacmi hafta sonlarında belirgin şekilde yükseliyor: Pazar %16,94 ve Cumartesi %16,65 ile hafta içi günlerin (%13,0–13,7) yaklaşık 1,27 katı. Operasyonel karşılığı, depo ve müşteri hizmetleri kapasitesinin hafta sonuna göre planlanması gerektiğidir; hafta içine göre kurulmuş bir vardiya düzeni talebin en yoğun olduğu iki günde yetersiz kalır.

Aylık tabloda kasım-aralık zirvesi görünüyor (2025 Aralık 10.907 sipariş ile en yüksek ay), ancak tablo naif okunduğunda yanıltıcıdır. 2024 Kasım ve Aralık, aynı mevsimsel etkiye sahip olmalarına rağmen ilk on ikide yer almıyor; buna karşılık mevsimsel etkisi olmayan 2026 Mart–Mayıs ayları, yine mevsimsel etkisi olmayan 2025 Ekim'in neredeyse iki katı hacimde. Fark mevsimsellikten değil, müşteri tabanının büyümesinden kaynaklanıyor: veri kurgusunda bir kullanıcı ancak kayıt tarihinden sonra sipariş verebiliyor, dolayısıyla dönemin başında sipariş verebilecek kullanıcı sayısı düşük, sonunda neredeyse tamamı uygun.

Bu, trend analizinin en yaygın hatasıdır: toplam hacim büyümesi, talep artışını değil taban büyümesini ölçebilir. Dönemler arası karşılaştırma yapılacaksa hacim, o dönemde aktif olan müşteri sayısına normalize edilmelidir (kişi başına sipariş). Kohort analizi (26. soru) bu yanılgıyı ortadan kaldırmak için vardır.

**Metodolojik not:** `timestamptz` bir kolondan `EXTRACT(DOW ...)` ile gün çıkarmak oturumun saat dilimine bağlıdır; UTC'de 23:00 olan bir kayıt İstanbul saatiyle ertesi güne düşer. Gün/ay kırılımlı her raporda saat dilimi açıkça belirtilmelidir. Bu sorgular `Etc/UTC` oturumunda çalıştırılmıştır.

---

## 10 — Geçerlilik süresi dolmuş kupon kodları hangileri?

**SQL:** `sql-mastery/queries/q10_suresi_dolmus_kuponlar.sql`
Referans tarih: verinin son sipariş tarihi, 2026-06-29.

**Sonuç:**

```
     durum     | kupon_sayisi
---------------+--------------
 suresi dolmus |           50

 en_kisa | en_uzun  |     ortalama
---------+----------+-------------------
 30 days | 174 days | 103 days 04:48:00
```

En geç biten kupon: `KUPON005`, 2026-04-03.

**İş yorumu:**

Elli kuponun tamamının süresi dolmuş; veri ufkunda yürürlükte tek bir kampanya yok. Kupon geçerlilik süreleri 30 ile 174 gün arasında, ortalama 103 gün. Gerçek bir e-ticaret operasyonunda her an aktif birkaç kampanya bulunur, dolayısıyla bu durum iş gerçekliğini değil veri üreticisinin kurgusunu yansıtıyor: `valid_from` zaman çizgisinin ilk 711 gününden seçiliyor ve üzerine en fazla 180 gün ekleniyor, böylece bütün bitiş tarihleri veri ufkunun gerisinde kalıyor.

Bunun analitik açıdan daha ciddi bir sonucu var: üretici kuponları siparişlere geçerlilik tarihine bakmadan uyguluyor. Veride, kuponun geçerlilik penceresi dışında kullanılmış siparişler bulunması bekleniyor. Bu tutarsızlık şema kısıtlarıyla yakalanamaz, çünkü `CHECK` tek bir satır üzerinde çalışır; "siparişin tarihi kuponun aralığında mı" sorusu `orders` ile `coupons` tablolarını birlikte ilgilendirir ve ancak bir `JOIN` ile sınanabilir. Veri kalitesi kontrolleri listesine eklenmiştir.

**Metodolojik not:** İki `timestamptz` değerinin farkı sayı değil `interval` tipindedir; ortalaması da `interval` olarak döner (`103 days 04:48:00`). Gün sayısı olarak tamsayı isteniyorsa `(valid_to::date - valid_from::date)` kullanılmalıdır.

**İyileştirme notları:**
- `scripts/generate.py`: kupon geçerlilik pencereleri veri ufkunu kapsayacak şekilde üretilmeli; en az birkaç kupon yürürlükte kalmalı.
- `scripts/generate.py`: kupon ancak geçerlilik penceresi içindeki siparişlere uygulanmalı.
- Yeni veri kalitesi kontrolü: geçerlilik aralığı dışında kullanılmış kuponlar (JOIN gerektirir, B grubunda yazılacak).
