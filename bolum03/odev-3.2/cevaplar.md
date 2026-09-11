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

---

## 11 — Her kullanıcının sipariş sayısı ve toplam harcaması

**SQL:** `sql-mastery/queries/q11_kullanici_siparis_harcama.sql`

**Sonuç:**

```
 join_sonucu_satir | gercek_siparis_sayisi
-------------------+-----------------------
            168920 |                100000

  id   | yanlis_siparis_sayisi | dogru_siparis_sayisi | toplam_harcama | ortalama_sepet
-------+-----------------------+----------------------+----------------+----------------
  6754 |                  2750 |                 1613 |      367074.02 |         227.57
  3511 |                  1780 |                 1025 |      247024.69 |         241.00
 13040 |                  1194 |                  730 |      162050.24 |         221.99
  3614 |                  1039 |                  617 |      132945.23 |         215.47
 16900 |                   903 |                  544 |      121361.25 |         223.09
```

**İş yorumu:**

Harcama, kullanıcılar arasında aşırı derecede yoğunlaşmış: en aktif kullanıcı tek başına 1.613 sipariş vermiş, yani toplam 100.000 siparişin %1,6'sı. Gerçek bir perakende operasyonunda tek bir bireysel hesabın bu paya ulaşması olağandışıdır ve böyle bir satır normalde kurumsal hesap, bot trafiği veya veri hatası şüphesiyle incelenir. Buradaki kaynak veri kurgusudur: kullanıcı seçimi güç yasasıyla yapılıyor ve üs değeri (0,7) gerçekçi olmayacak kadar yoğunlaştırıcı.

İkinci ve analitik açıdan daha kısıtlayıcı bulgu ortalama sepet tutarında: ilk on beş kullanıcının hepsi 215–272 TL bandında. Gerçek bir müşteri tabanında bu ölçü geniş yayılır — sık ve düşük tutarlı alışveriş yapan müşteri ile seyrek ve yüksek tutarlı alışveriş yapan müşteri birbirinden ayrışır. Bu veride sepet içeriği kullanıcıdan bağımsız seçildiği için parasal değer, sipariş sayısının yaklaşık sabit bir katına eşit hâle geliyor. Sonucu: RFM segmentasyonunda (27. soru) Monetary ve Frequency boyutları aynı bilgiyi taşıyacak ve üç boyutlu segmentasyon pratikte tek boyuta çökecektir. Sorgu doğru çalışacak, ancak ürettiği segmentler ayırt edici olmayacaktır.

**Metodolojik not — fan-out:** `orders` ile `order_items` birleştirildiğinde bir sipariş satırı, o siparişin her kalemiyle ayrı ayrı eşleşir; join sonucu 168.920 satırdır ve bu sipariş sayısı değil kalem sayısıdır. Bu durumda `count(*)` ve `count(o.id)` sipariş sayısını 1,69 kat şişirir (örnek: 2.750 yerine doğru değer 1.613). Doğru sayım `count(DISTINCT o.id)` ile yapılır. Şişmiş sayı makul göründüğü için hata rapor okunurken fark edilmez; bu yüzden her join'den önce beklenen satır sayısı yazılmalıdır.

Fan-out her ölçüyü bozmaz: `sum(oi.quantity * oi.unit_price)` doğrudur, çünkü çoğaltılan taraf `orders`'tır, kalem satırları sonuçta birer kez bulunur. Kural: join'de "çok" tarafındaki tablonun ölçüleri güvenli, "bir" tarafındaki tablonun ölçüleri şişer. `orders` tablosunda toplam tutar kolonu tutmama kararı (DESIGN.md) bu nedenle isabetlidir; böyle bir kolon olsaydı kalemlerle birleştiren her sorgu sessizce yanlış ciro üretirdi.

---

## 12 — Hiç sipariş vermemiş kullanıcılar kimler?

**SQL:** `sql-mastery/queries/q12_siparis_vermeyen_kullanicilar.sql`

**Sonuç:** 20.000 kullanıcıdan **1.715'i** (%8,6) hiç sipariş vermemiş. Üç farklı yazım (LEFT JOIN + IS NULL, NOT EXISTS, NOT IN) aynı sonucu verdi.

```
 siparis_vermeyen
------------------
             1715

-- ornek kullanicilar (en yeni kayitlar)
 18721 | FR | 2026-03-30
  2241 | FR | 2026-03-29
  3970 | TR | 2026-03-29
```

**NOT IN tuzağı — canlı kanıt** (nullable kolon üzerinde):

```
 not_exists_ile      not_in_ile
----------------    ------------
          12047               0

 toplam_hareket | order_id_dolu | order_id_null
----------------+---------------+---------------
         164012 |        151820 |         12192
```

**İş yorumu:**

Kayıtlı kullanıcıların %8,6'sı hiç sipariş vermemiş. Bu oran tek başına bir dönüşüm sorunu göstergesi değildir, çünkü örneklem incelendiğinde sipariş vermeyen kullanıcıların en son kaydolanlar olduğu görülüyor (2026-03-26 ile 03-30 arası). Yeni kaydolmuş bir kullanıcının henüz sipariş vermemiş olması beklenen durumdur. "Hiç sipariş vermemiş" ölçüsü bu hâliyle iki farklı olguyu bir arada sayıyor: henüz dönüşmemiş yeni kullanıcı ile hiç dönüşmemiş eski kullanıcı. Kayıt-sipariş dönüşümünü ölçmek için kullanıcılar kayıt tarihinden bu yana geçen süreye (tenure) göre ayrılmalı; örneğin "kaydolduktan sonra 30 gün geçmiş ve hâlâ sipariş vermemiş kullanıcılar" anlamlı bir dönüşüm kaybı ölçüsüdür.

Stok hareketi olmayan sipariş sayısı 12.047 olarak bulundu ve bu değer bağımsız olarak doğrulanabiliyor: 07. soruda `created` durumundaki 6.977 ve `cancelled` durumundaki 5.070 sipariş, toplam 12.047 eder. Bu iki grup için satış hareketi üretilmiyor. Çapraz doğrulamanın tutması, sorgunun doğruluğuna dair güçlü bir kanıttır.

**Metodolojik not — anti-join ve `NOT IN`:**

Bir tabloda karşılığı olmayan satırları bulmanın üç yolu vardır: `LEFT JOIN` + `IS NULL`, `NOT EXISTS`, `NOT IN`. İlk ikisi her zaman güvenlidir, üçüncüsü değildir.

`inventory_movements.order_id` kolonunda 12.192 `NULL` bulunuyor (alım ve düzeltme hareketlerinde sipariş yoktur). Bu kolon üzerinden yazılan `NOT IN` sorgusu, doğru cevap 12.047 iken **0 döndürdü ve hata vermedi**. Sebebi üç değerli mantıktır: `x NOT IN (a, b, NULL)` ifadesi `x <> a AND x <> b AND x <> NULL` anlamına gelir; son karşılaştırma `NULL` döndüğü için bileşik ifade hiçbir zaman `TRUE` olamaz ve `WHERE` hiçbir satırı geçirmez. Kural: alt sorguyla olumsuzlama yapılacaksa `NOT EXISTS` kullanılmalı; `NOT IN` yalnızca alt sorgudaki kolonun `NOT NULL` olduğu şemadan doğrulandığında kullanılmalıdır.

**Metodolojik not — `LEFT JOIN` sonrası filtre:** `LEFT JOIN` sonrasında `WHERE sag_tablo.kolon IS NOT NULL` yazmak, eşleşmeyen satırları eleyeceği için sorguyu sessizce `INNER JOIN`'e çevirir; `LEFT` yazmanın anlamı kalmaz. Anti-join kalıbında kontrol edilecek kolon, sağ tablonun `NOT NULL` bir kolonu (tercihen birincil anahtarı) olmalıdır; nullable bir kolon seçilirse gerçek eşleşmeler de elenir.

---

## 13 — Kategori bazında toplam ciro

**SQL:** `sql-mastery/queries/q13_kategori_ciro.sql`

**Kapsam kararı:** Ciro = `paid`, `shipped`, `delivered` durumundaki siparişler. `created` ve `cancelled` hariç (para girmedi), `returned` hariç (net ciro ölçülüyor).
**Kısıt:** Ürünler yalnızca yaprak kategorilere bağlı; kök kategori toplamları ağacı yukarı toplamayı gerektirir (37. soru).

**Sonuç:** 32 yaprak kategori, toplam ciro 19.724.818,08 TL.

```
 kategori_id |       kategori        | siparis_sayisi | satilan_adet |    ciro    | brut_marj
-------------+-----------------------+----------------+--------------+------------+------------
          30 | Kitap Premium         |           8989 |        12515 | 2934301.03 | 1263361.69
           8 | Kirtasiye Aksesuar    |          26672 |        38049 | 1729130.71 |  776262.59
          28 | Kitap Aksesuar        |           9283 |        13166 | 1169534.99 |  387332.48
          14 | Mutfak Yeni Sezon     |           7726 |        10802 |  992925.51 |  347237.49
          29 | Kitap Yeni Sezon      |           3957 |         5573 |  769824.97 |  238107.13
          ...
          19 | Giyim Yeni Sezon      |            937 |         1304 |  237204.70 |  108550.39
(32 satir)
```

**İş yorumu:**

Ciro kategoriler arasında dengesiz dağılıyor: ilk kategori (2,93 milyon TL) sonuncunun (237 bin TL) on iki katı ve tek başına toplam cironun %15'ini oluşturuyor. Ancak bu sıralamadan kategori bazlı bir iş sonucu çıkarılamaz, çünkü veri üreticisi ürünleri kategorilere tamamen rastgele dağıtıyor; bir kategorinin lider olması, o kategoriye tesadüfen pahalı ve popüler ürünlerin düşmesinden kaynaklanıyor. Sorgu doğrudur, kategori isimleriyle kurulacak yorum değildir.

Kategorilerden bağımsız olarak geçerli kalan bulgu, iki farklı ciro modelinin varlığı: `Kitap Premium` 12.515 adet satışla 2,93 milyon TL ciro üretiyor (adet başına ~234 TL), `Kirtasiye Aksesuar` ise 38.049 adetle 1,73 milyon TL (adet başına ~45 TL). Birincisi düşük hacim-yüksek birim fiyat, ikincisi yüksek hacim-düşük birim fiyat modelidir. Brüt marj oranları da ayrışıyor: `Bahce Aksesuar` %52,4 marjla çalışırken `Elektronik Premium` %29,1'de kalıyor. Gerçek bir katalogda bu ayrışma stok, fiyatlama ve kampanya kararlarının temeli olurdu.

**Metodolojik not — toplanabilirlik:** `siparis_sayisi` kolonundaki değerler toplandığında 142.950 çıkıyor, oysa kapsamdaki gerçek sipariş sayısı 85.974'tür. Bir sipariş birden çok kategoriden ürün içerebildiği için her kategoride ayrı sayılıyor. `count(DISTINCT ...)` gruplar arasında toplanabilir bir ölçü değildir; `sum(ciro)` ve `sum(adet)` toplanabilirdir, oranlar ve ortalamalar değildir. Ara toplam satırı içeren her raporda ölçünün toplanabilirliği önce doğrulanmalıdır.

---

## 14 ve 15 — Adet bazında ve ciro bazında en çok satan 10 ürün

**SQL:** `sql-mastery/queries/q14_q15_en_cok_satan_urunler.sql`
**Kapsam:** `paid`, `shipped`, `delivered` siparişler.

**Sonuç — adet bazında ilk 10:**

```
  id  |    sku    | list_price | satilan_adet |    ciro
------+-----------+------------+--------------+------------
  169 | SKU-00169 |      34.19 |        31430 |  993796.20
  118 | SKU-00118 |      29.57 |        15467 |  423435.15
  186 | SKU-00186 |     290.86 |        10010 | 2693709.88
 1573 | SKU-01573 |      53.65 |         7586 |  376481.38
  720 | SKU-00720 |      20.68 |         5969 |  114126.80
```

**Sonuç — ciro bazında ilk 10:**

```
  id  |    sku    | list_price | satilan_adet |    ciro
------+-----------+------------+--------------+------------
  186 | SKU-00186 |     290.86 |        10010 | 2693709.88
  169 | SKU-00169 |      34.19 |        31430 |  993796.20
 1088 | SKU-01088 |     576.19 |          931 |  496179.87
  118 | SKU-00118 |      29.57 |        15467 |  423435.15
  470 | SKU-00470 |     279.59 |         1509 |  390285.35
```

**İş yorumu:**

İki listede yalnızca 5 ürün ortak; ölçü değiştiğinde sıralamanın yarısı değişiyor. Üç tipik profil ayrışıyor: SKU-00169 adette birinci ama 34 TL birim fiyatıyla ciroda ikinci (yüksek hacim, düşük değer); SKU-00186 hem yüksek hacimli hem 290 TL birim fiyatlı olduğu için ciroda birinci; SKU-01088 yalnızca 931 adet satmasına rağmen 576 TL birim fiyatıyla ciro listesine giriyor ve adet listesinde hiç görünmüyor (düşük hacim, yüksek değer).

Pratik sonucu, "en çok satan ürün" ifadesinin tek başına anlamsız olmasıdır. Stok ve depo planlaması adet ölçüsüne, kârlılık ve kampanya kararları ciro ölçüsüne bakar; raf/vitrin kararları ikisini birden gerektirir. Bir raporda "en çok satan" başlığı kullanılıyorsa ölçünün hangisi olduğu açıkça yazılmalıdır.

Ciro yoğunlaşması dikkat çekici düzeyde: SKU-00186 tek başına 2,69 milyon TL ile toplam 19,72 milyonluk cironun %13,7'sini üretiyor. Bu, tek ürüne bağımlılık anlamına gelir ve gerçek bir işletmede tedarik riski olarak izlenir. Buradaki kaynak veri kurgusudur — ürün popülerliği güç yasasıyla dağıtılıyor ve üs değeri gerçekçi olmayacak kadar yoğunlaştırıcı.

**Çapraz doğrulama:** Adet listesindeki ürün kimlikleri (169, 118, 186, 1573, 720, 1604, 674, 775, 617, 462), 05. soruda stoğu negatife düşen ürünlerin listesiyle örtüşüyor. Beklenen sonuç budur: stoğu eksiye düşen ürünler alımdan fazla satanlardır. İki bağımsız sorgunun aynı ürün kümesine işaret etmesi, her ikisinin de doğru yazıldığına dair kanıttır.

---

## 16 ve 17 — Ortalama sepet tutarı ve sipariş başına ortalama kalem sayısı

**SQL:** `sql-mastery/queries/q16_q17_ortalama_sepet_ve_kalem.sql`
**Kapsam:** `paid`, `shipped`, `delivered` siparişler (85.974 sipariş).

**Sonuç:**

```
 siparis_sayisi | ortalama_sepet | en_kucuk_sepet | en_buyuk_sepet
----------------+----------------+----------------+----------------
          85974 |         229.43 |           8.46 |        4307.92

 kalem_ortalamasi_YANLIS
-------------------------
                  135.83

 ortalama_kalem | en_az | en_cok
----------------+-------+--------
          1.689 |     1 |      5

 kalem_sayisi | siparis_sayisi | yuzde
--------------+----------------+-------
            1 |          39779 | 46.27
            2 |          34440 | 40.06
            3 |          10506 | 12.22
            4 |           1200 |  1.40
            5 |             49 |  0.06
```

**İş yorumu:**

Ortalama sepet tutarı 229,43 TL, sepetler 8,46 TL ile 4.307,92 TL arasında değişiyor. Sipariş başına ortalama kalem sayısı 1,689; siparişlerin %46'sı tek kalemlik, %40'ı iki kalemlik. Üç ve üzeri kalem içeren sipariş oranı %13,7'de kalıyor. Gerçek bir operasyonda bu tablo, çapraz satış (cross-sell) potansiyelinin kullanılmadığını gösterirdi: müşterilerin neredeyse yarısı tek ürün alıp çıkıyor ve sepet büyütmeye yönelik öneri mekanizması ya yok ya etkisiz. Sepet başına kalem sayısını 1,69'dan 2,0'a çıkarmak, birim fiyat sabitken ciroyu yaklaşık %18 artırırdı.

**Bulgu — veri üreticisinde hata:** Gözlenen kalem sayısı dağılımı, üreticiye tanımlanan ağırlıklarla (1:%45, 2:%28, 3:%15, 4:%8, 5:%4; beklenen ortalama 1,98) uyuşmuyor. Sapma sistematik olduğu için veri değil kod incelendi ve hata bulundu: `while len(secilen) < random.choice(ITEM_COUNTS)` satırında hedef kalem sayısı döngünün her turunda yeniden çekiliyor, bir kez belirlenip sabitlenmiyor. Bu varsayımla hesaplanan dağılım (0,450 / 0,4015 / 0,1307 / 0,0171 / 0,0007) gözlenen dağılımla (0,4627 / 0,4006 / 0,1222 / 0,0140 / 0,0006) örtüşüyor ve hatayı doğruluyor.

Düzeltme, hedef kalem sayısının döngü öncesinde bir kez belirlenmesidir. Düzeltme bu aşamada uygulanmadı: mevcut cevapların tamamı bu veri setine ait sayılar içeriyor ve verinin değişmesi hepsini geçersiz kılardı. Veri kendi içinde tutarlı olduğundan analizlerin doğruluğu etkilenmiyor; düzeltme, diğer üretici iyileştirmeleriyle birlikte bölüm sonunda uygulanıp tüm sorgular yeniden çalıştırılacaktır.

**Metodolojik not — türetilmiş tablo:** "Sipariş başına ortalama" iki aşamalı bir hesaptır: önce sipariş bazında toplama, sonra bu toplamların ortalaması. Doğrudan `avg(quantity * unit_price)` yazmak sipariş değil kalem ortalamasını verir ve sonucu %41 düşük gösterir (135,83'e karşı 229,43). İki sonuç arasındaki ilişki `135,83 × 1,689 = 229,4` olarak doğrulanıyor; bu çarpımın tutması iki sorgunun da doğru yazıldığının kanıtıdır.

**Metodolojik not — ortalamaların ortalaması:** Alt gruplarda hesaplanmış ortalamaların tekrar ortalanması, grup büyüklükleri farklı olduğunda yanlış sonuç verir. Genel ortalama her zaman ham veriden hesaplanmalıdır. Bu, 13. soruda kaydedilen toplanabilirlik kuralının aynısıdır.

---

## 18 — Ödemesi başarısız olan siparişlerin oranı

**SQL:** `sql-mastery/queries/q18_basarisiz_odeme_orani.sql`

**Sonuç:**

```
 status  | odeme_sayisi | yuzde | toplam_tutar
---------+--------------+-------+--------------
 success |       100000 | 92.51 |  21787312.34
 failed  |         8094 |  7.49 |   1753608.86

 toplam_siparis | basarisiz_denemeli | yuzde
----------------+--------------------+-------
         100000 |               8094 |  8.09

 siparis_durumu | siparis | basarili_odemesi_olan | basarisiz_denemesi_olan | iade_kaydi_olan
----------------+---------+-----------------------+-------------------------+-----------------
 delivered      |   63110 |                 63110 |                    5128 |               0
 shipped        |   12933 |                 12933 |                    1043 |               0
 paid           |    9931 |                  9931 |                     777 |               0
 created        |    6977 |                  6977 |                     590 |               0
 cancelled      |    5070 |                  5070 |                     406 |               0
 returned       |    1979 |                  1979 |                     150 |               0

 basarili_odemesi_olmayan_siparis
----------------------------------
                                0
```

**İş yorumu:**

Siparişlerin %8,09'unda en az bir başarısız ödeme denemesi var, ancak hepsi sonunda başarıyla ödenmiş: başarılı ödemesi olmayan tek bir sipariş yok. Bu, ödeme altyapısının analizini bu veri üzerinde anlamsız kılıyor, çünkü başarısızlık hiçbir zaman siparişi engellemiyor.

Üç tutarsızlık tespit edildi ve üçü de aynı kök nedenden geliyor — üretici her siparişe, durumuna bakmaksızın bir başarılı ödeme yazıyor:

1. **İade ödemesi hiç yok.** 1.979 sipariş `returned` durumunda, buna karşılık `refunded` statüsünde tek bir ödeme kaydı bulunmuyor. Mal geri gelmiş, para geri gitmemiş. Şema bu durumu destekliyor (`payments_amount_sign` kısıtı iade için negatif tutar zorunlu kılıyor); eksik olan veridir.
2. **`created` durumundaki 6.977 siparişin tamamında başarılı ödeme var.** `created`, ödeme yapılmamış siparişi ifade eder; durum ile ödeme kaydı çelişiyor.
3. **`cancelled` durumundaki 5.070 siparişin tamamında başarılı ödeme, hiçbirinde iade var.** İptal edilen siparişlerden tahsilat yapılmış ve iade edilmemiş.

Ayrıca ödeme başarısızlığı oranı her sipariş durumunda ~%8 civarında sabit; gerçek bir sistemde başarısız ödeme ile iptal arasında güçlü bir ilişki beklenirdi.

**Metodolojik not:** `payments` tablosundan tutar toplanırken `status` filtresi zorunludur. Başarısız denemelerin toplamı 1.753.608 TL'dir ve bunlar gerçekleşmemiş işlemlerdir; filtresiz `sum(amount)` ciroyu yaklaşık %8 fazla gösterir.

**Çapraz doğrulama:** Başarılı ödemelerin toplamı 21.787.312 TL, sipariş başına 217,90 TL. 16. soruda hesaplanan ortalama sepet 229,43 TL idi. Aradaki %5'lik fark kupon indirimleriyle açıklanıyor (siparişlerin %18'i kupon kullanıyor). İki bağımsız hesabın bu farkla örtüşmesi beklenen sonuçtur.

**İyileştirme notları (`scripts/generate.py`):**
- `returned` siparişler için negatif tutarlı `refunded` ödeme kaydı üretilmeli.
- `created` siparişler için başarılı ödeme üretilmemeli.
- `cancelled` siparişler ya ödeme almamalı ya da iade kaydı içermeli.
- Ödeme başarısızlığı ile sipariş durumu ilişkilendirilmeli.

---

## 19 — Funnel: created → paid → shipped → delivered dönüşüm oranları

**SQL:** `sql-mastery/queries/q19_funnel.sql`

**Varsayımlar:** `orders.status` yalnızca güncel durumu tutar, geçmiş tutulmaz. Aşamalar kümülatif sayılmıştır: teslim edilmiş bir sipariş ödeme ve kargo aşamalarından geçmiş kabul edilir. `returned` siparişler teslim aşamasına dahildir. `cancelled` siparişlerin hangi aşamada iptal edildiği bilinmediği için funnel dışında bırakılmıştır.

**Sonuç:**

```
 asama1_olusturuldu | asama2_odendi | asama3_kargolandi | asama4_teslim
--------------------+---------------+-------------------+---------------
             100000 |         87953 |             78022 |         65089

 olusturuldu_to_odendi | odendi_to_kargolandi | kargolandi_to_teslim | uctan_uca
-----------------------+----------------------+----------------------+-----------
                 87.95 |                88.71 |                83.42 |     65.09

 gecis                      | dusen_siparis
----------------------------+---------------
 olusturuldu -> odendi      |          6977
 odendi -> kargolandi       |          9931
 kargolandi -> teslim       |         12933
 iptal (asamasi bilinmiyor) |          5070
```

**İş yorumu:**

Siparişlerin %87,95'i ödeme aşamasına, %78,02'si kargo aşamasına, %65,09'u teslimata ulaşıyor. En büyük kayıp kargo-teslimat geçişinde: 12.933 sipariş yolda kalmış durumda. Gerçek bir operasyonda bu, teslim edilemeyen gönderiler anlamına gelir ve kargo firması bazında incelenmesi gereken bir operasyonel alarmdır (25. soruda kargo firması kırılımına bakılacak).

Ancak bu oranlardan davranışsal bir sonuç çıkarılamaz: veri üreticisi sipariş durumunu müşteri davranışına göre değil sabit ağırlıklara göre atıyor, dolayısıyla aşamalar arası kayıp doğrudan bu ağırlıkların kendisidir. Yöntem doğru, iş sonucu anlamsızdır.

İki yapısal kısıt kaydedilmelidir:

**Funnel çok geç başlıyor.** İlk aşama "sipariş oluşturuldu"dur; gerçek bir e-ticaret funnel'ı ziyaret, ürün görüntüleme, sepete ekleme ve ödeme başlatma adımlarını da içerir. Şemada oturum ve sepet verisi bulunmadığı için dönüşümün asıl kaybedildiği adımlar ölçülemiyor. Uçtan uca %65,09 oranı, gerçek bir funnel için olağanüstü yüksek görünür; çünkü ölçülen yalnızca son adımdır.

**Durum geçmişi tutulmuyor.** `orders.status` tek bir güncel değer taşır, dolayısıyla "bu sipariş ne zaman ödendi, ne zaman kargolandı" sorusu cevaplanamıyor ve iptallerin hangi aşamada gerçekleştiği bilinemiyor. Funnel bu nedenle ölçüm değil varsayım üzerine kurulu. Çözüm bir olay tablosudur: `order_status_history(order_id, status, changed_at)`. Her durum değişikliğinin bir satır olarak tutulması hem funnel'ı ölçüme dönüştürür hem de aşamalar arası geçiş sürelerini hesaplanabilir kılar.

**İyileştirme notu:** Şemaya `order_status_history` tablosu eklenmeli; Ödev 3.4'te SCD2 ile birlikte değerlendirilecek.

---

## 20 — Sepete girip satın alınmayan ürünler

**SQL:** `sql-mastery/queries/q20_terk_edilen_urunler.sql`

**Kapsam beyanı:** Şemada sepet tablosu bulunmuyor; sepete eklenip siparişe dönüşmeyen ürünler ölçülemiyor. En yakın karşılık olarak `created` ve `cancelled` durumundaki siparişlerdeki ürünler alınmıştır. Bu, gerçek sepet terkinden dar bir kümedir.

**Sonuç:**

```
 terk_edilen_adet | terk_edilen_tutar | siparis_sayisi | etkilenen_urun
------------------+-------------------+----------------+----------------
            28169 |        2795633.42 |          12047 |           1620

  id  |    sku    | list_price | toplam_adet | satilan | terk_edilen | terk_orani
------+-----------+------------+-------------+---------+-------------+------------
 1025 | SKU-01025 |     221.28 |         235 |     184 |          49 |      20.85
 1572 | SKU-01572 |      81.70 |         166 |     128 |          34 |      20.48
   42 | SKU-00042 |     109.95 |         109 |      84 |          21 |      19.27
 1488 | SKU-01488 |     111.35 |         529 |     420 |          98 |      18.53
```

**İş yorumu:**

Satışa dönüşmeyen siparişlerin toplam değeri 2.795.633 TL, 28.169 adet ürün ve 1.620 farklı SKU'yu kapsıyor. Genel terk oranı %12,05'tir (12.047 sipariş / 100.000). Bu tutar, gerçekleşen 19,7 milyon TL'lik cironun %14'üne karşılık geliyor ve kurtarılabilir gelirin büyüklüğünü gösteriyor — gerçek bir operasyonda ödeme hatırlatma ve terk edilmiş sepet e-postası gibi müdahalelerin hedefi bu tutardır.

**Metodolojik not — orana göre sıralamanın yanıltıcılığı:** Ürün bazında terk oranı sıralandığında ilk on beş ürün %17,3–%20,9 bandında çıkıyor, yani genel ortalamanın (%12,05) belirgin şekilde üzerinde. Ancak bu ürünlerin hiçbiri gerçekten farklı değildir: veri üreticisi sipariş durumunu ürüne bakmadan atadığı için her ürünün gerçek terk oranı %12'dir ve gözlenen farkların tamamı örnekleme dalgalanmasıdır. 100–500 adet hacimli bir üründe %12'lik oranın standart sapması %1,5–3 aralığındadır; 2.000 ürün arasından en yükseği seçildiğinde doğal olarak dalgalanmanın uç noktası seçilmiş olur.

Bu, oran bazlı sıralamaların genel sorunudur: sıralamanın tepesi, gerçekten farklı olanları değil en oynak olanları toplar. Aynı mekanizma "en çok gelişme gösteren okul" listelerinin küçük okullarla dolmasının da sebebidir. Minimum hacim eşiği (burada `HAVING sum(quantity) >= 100`) sorunu hafifletir ancak ortadan kaldırmaz; tam çözüm istatistikseldir (güven aralığı veya Bayesçi düzeltme). Oran sıralayan her raporda sorulması gereken soru şudur: gözlenen fark, bu hacimde beklenen dalgalanmadan büyük mü?

Eşik konulmadığında oluşabilecek küçük payda sorunu bu veri setinde gerçekleşmedi; terk oranı %100 olan ürün bulunmuyor.

**Çapraz doğrulama:** Kapsamdaki sipariş sayısı 12.047, 12. soruda stok hareketi bulunmayan sipariş sayısıyla aynı. İki bağımsız sorgu aynı kümeyi işaret ediyor.

---

## 21 — Kupon kullanımının marj etkisi

**SQL:** `sql-mastery/queries/q21_kupon_marj_etkisi.sql`
**Kapsam:** `paid`, `shipped`, `delivered` siparişler (85.974 sipariş, bunların 15.503'ü kuponlu — %18,03).

**Sonuç:**

```
   grup   | siparis_sayisi | ort_sepet | ort_brut_marj | ort_indirim | ort_net_marj | brut_marj_yuzde | net_marj_yuzde
----------+----------------+-----------+---------------+-------------+--------------+-----------------+----------------
 kuponlu  |          15503 |    228.64 |         87.36 |       65.51 |        21.84 |           38.21 |           9.55
 kuponsuz |          70471 |    229.60 |         87.71 |        0.00 |        87.71 |           38.20 |          38.20

 toplam_brut_marj | toplam_indirim | toplam_net_marj | indirimin_marja_orani
------------------+----------------+-----------------+-----------------------
       7535002.18 |     1015655.59 |      6519346.59 |                 13.48
```

**İş yorumu:**

Kuponlu ve kuponsuz siparişlerin ortalama sepet tutarları pratikte aynı (228,64 TL ve 229,60 TL), brüt marj oranları da aynı (%38,21 ve %38,20). Buna karşılık kuponlu siparişlerde ortalama 65,51 TL indirim uygulanıyor ve net marj 87,71 TL'den 21,84 TL'ye, yani %75 oranında düşüyor. Net marj oranı %38,20'den %9,55'e geriliyor. Toplamda kuponlar brüt marjın %13,48'ini (1.015.656 TL) tüketiyor.

Bu veri setinde kuponlar siparişlere rastgele atandığı için karşılaştırma nedensel olarak yorumlanabilir: kuponlu grup deney, kuponsuz grup kontrol niteliğindedir ve atama rastgeledir. Sepet tutarlarının eşit çıkması, kuponun harcama davranışını değiştirmediğini gösteriyor. Kupon, yalnızca zaten gerçekleşecek satışlardan marj eksiltiyor.

Gerçek veride aynı karşılaştırma bu şekilde yorumlanamaz. Kuponu kimin kullandığı rastgele değildir; fiyata duyarlı ve satın almaya zaten yakın müşteriler kupon arar. Bu nedenle "kupon kullananların sepeti daha büyük" gözlemi kuponun sepeti büyüttüğünü kanıtlamaz, yalnızca kupon kullananların farklı bir müşteri grubu olduğunu gösterir — buna seçilim yanlılığı denir. Kampanya değerlendirmesinde doğru soru "kupon kullananlar daha mı çok harcadı" değil, "kupon olmasaydı bu satış yine gerçekleşir miydi" sorusudur (artımsallık). Bu soru ancak rastgele atamalı bir test (holdout grubu) ile cevaplanabilir.

**Metodolojik not — çifte fan-out:** `orders`, `order_items` ve `order_coupons` doğrudan birleştirildiğinde sonuç 30.469 satır, doğru kalem sayısı ise 26.212'dir; şişme 1,16 kattır çünkü kuponlu siparişlerin bir kısmı iki kupon içerir. Aynı üst tablonun iki farklı alt tablosu doğrudan birleştirildiğinde her biri diğerinin satır sayısı kadar çoğalır ve iki ölçü iki farklı katsayıyla bozulur; oranlar bile tutmaz. Doğru yöntem, her alt tabloyu kendi taneciğinde ayrı ayrı toplayıp sonucu sipariş kimliği üzerinden birleştirmektir (CTE ile).

---

## 22 — Ürün başına ortalama puan ve yorum sayısı

**SQL:** `sql-mastery/queries/q22_urun_puan_ortalamasi.sql`
**Bağlam:** K-005 gereği aynı kullanıcı aynı ürüne birden çok yorum yazabilir.

**Sonuç:**

```
 toplam_yorum | benzersiz_kullanici_urun_cifti | fazladan_yorum
--------------+--------------------------------+----------------
        26810 |                          23972 |           2838

-- naif (butun yorumlar)          -- tekillestirilmis (kullanici-urun basina son yorum)
  id  | yorum_sayisi | ort_puan     id  | yorum_sayisi | ort_puan
------+--------------+----------  ------+--------------+----------
  169 |         4225 |    4.009     169 |         2979 |    4.012
  118 |         2111 |    4.056     118 |         1671 |    4.067
  186 |         1298 |    4.062     186 |         1078 |    4.079

 urun_sayisi | ortalamasi_degisen | en_buyuk_sapma | ortalama_sapma
-------------+--------------------+----------------+----------------
        1751 |                157 |          0.600 |         0.0068
```

**İş yorumu:**

Toplam 26.810 yorumun 2.838'i (%10,6) aynı kullanıcı-ürün çiftinin tekrarlanmış yorumudur. Tekilleştirmenin etkisi iki ölçüde çok farklı:

**Yorum sayısı ciddi biçimde şişiyor.** En çok yorum alan üründe naif sayım 4.225, tekilleştirilmiş sayım 2.979 — %29 fark. Ürün sayfasında gösterilen değerlendirme sayısı tüketici için güvenilirlik göstergesidir; %29 abartılı bir sayı yanıltıcıdır.

**Ortalama puan ise pratikte değişmiyor.** 1.751 üründen yalnızca 157'sinin ortalaması değişiyor ve ortalama sapma 0,0068. Bunun nedeni tekrarlı yorumların rastgele puan almasıdır; ortalamayı sistematik bir yöne çekmiyorlar. Gerçek veride bu varsayım geçerli olmazdı: bir kullanıcı aynı ürüne ikinci kez yorum yazıyorsa genellikle görüşü değiştiği içindir, dolayısıyla tekrarlı yorumlar yönlüdür ve ortalamayı da kaydırır.

**Etki az yorumlu ürünlerde yoğunlaşıyor.** Ortalama sapma ihmal edilebilir olsa da en büyük sapma 0,600 puandır. Az yorum alan bir üründe tek bir tekrarlı yorum ortalamayı belirgin şekilde oynatır ve ürün puanının satın alma kararında en etkili olduğu yer tam olarak yeni, az yorumlu ürünlerdir. Bu nedenle ürün puanı gösteren her ekranda tekilleştirme uygulanmalı; kullanıcı-ürün çifti başına yalnızca en güncel yorum sayılmalıdır.

**Doğrulama:** Ortalama puanların 4,0 civarında toplanması beklenen sonuçtur; üreticideki puan ağırlıkları (5:%45, 4:%30, 3:%13, 2:%7, 1:%5) 4,03 ortalama verir.

**Metodolojik not — "en sonuncuyu al" kalıbı:** Kullanıcı-ürün çifti başına en güncel yorumu seçmek için önce her çift için `max(created_at)` hesaplayan bir CTE oluşturulup `reviews` tablosu bu CTE ile tarih üzerinden birleştirildi. Window function'lardan önceki standart çözüm budur. Aynı sonuç `DISTINCT ON` veya `ROW_NUMBER()` ile tek adımda elde edilebilir (35. soruda karşılaştırılacak).

**Yan bulgu:** Yorum alan ürün sayısı 1.751'dir; katalogdaki 2.000 üründen 249'u hiç yorum almamıştır (23. soru).

---

## 23 — Hiç yorum almamış ürünler

**SQL:** `sql-mastery/queries/q23_yorumsuz_urunler.sql`

**Sonuç:** 2.000 üründen 249'u (%12,45) hiç yorum almamış.

```
                     grup                     | urun_sayisi
----------------------------------------------+-------------
 1 - hic siparise girmemis                    |           0
 2 - siparise girmis ama hic teslim edilmemis |           3
 3 - teslim edilmis ama yorum almamis         |         246

  id  |    sku    | list_price | is_active | teslim_edilen_adet
------+-----------+------------+-----------+--------------------
 1967 | SKU-01967 |      61.52 | f         |                 28
  204 | SKU-00204 |     157.94 | t         |                 20
 1453 | SKU-01453 |     151.73 | f         |                 20
```

**İş yorumu:**

Yorumsuz ürünlerin tamamına yakını (246/249) aslında satılmış ve teslim edilmiş ürünlerdir; hiç siparişe girmemiş tek bir ürün yoktur. Yani sorun ürünün satılmaması değil, satılan üründen geri bildirim toplanamamasıdır. Yorumsuz ürünlerin tamamı düşük hacimli: en çoğu 28 adet teslim edilmiş. Teslim edilen her kalem için yorum bırakma olasılığı sabit olduğundan, az satan ürünün hiç yorum almama olasılığı yüksektir.

Bunun iş karşılığı soğuk başlangıç (cold start) döngüsüdür: yorumu olmayan ürün tüketici tarafından daha riskli görülür ve daha az satar, az satan ürün daha az yorum alır. Döngü kendiliğinden kırılmaz. Teslimat sonrası yorum isteme kampanyaları bu 246 ürünü öncelikli hedef almalıdır; katalogun %12,3'ünün sosyal kanıt olmadan satışa sunulması, dönüşüm kaybı anlamına gelir.

İkinci grup olan "siparişe girmiş ama hiç teslim edilmemiş" 3 ürün ayrıca incelenmeye değerdir: bu ürünler sipariş ediliyor ancak hiçbiri teslimata ulaşmıyor. Gerçek bir operasyonda stok, tedarik veya kargo tarafında bir sorun işaretidir.

**Performans notu:** (c) sorgusu 1,755 saniye sürdü; bu ana kadarki sorguların 20–40 katı. Sebep, `SELECT` listesindeki bağıntılı alt sorgunun her ürün için `order_items` ve `orders` tablolarını yeniden taraması ve yabancı anahtar kolonlarında index bulunmamasıdır. Ödev 3.3 için aday sorgu olarak kaydedildi.

---

## Ödev 3.3 için biriken yavaş sorgu adayları

| # | Sorgu | Süre | Şüphelenilen sebep |
|---|---|---|---|
| 1 | `seed/refresh_stock.sql` — `UPDATE products SET stock_cached = (bagintili alt sorgu)` | 13,2 sn | Her ürün için 164 bin satırlık defterin yeniden taranması |
| 2 | `q23` (c) — `SELECT` içinde bağıntılı alt sorgu | 1,76 sn | Ürün başına `order_items` + `orders` taraması, FK index'i yok |
| 3 | `q21` (b) — CTE + LEFT JOIN, 86 bin sipariş | 0,45 sn | `order_items.order_id` ve `order_coupons.order_id` index'siz |
| 4 | `checks.sql` (1) — `stock_cached` doğrulaması | 11,1 sn | 1 ile aynı kalıp |

Ortak nokta: yabancı anahtar kolonlarında index bulunmuyor (bilinçli karar — 002_products.sql notu).

---

## 24 — Ülke bazında sipariş sayısı ve ciro

**SQL:** `sql-mastery/queries/q24_ulke_bazinda_ciro.sql`
**Kapsam:** `paid`, `shipped`, `delivered` siparişler.

**Sonuç:**

```
    ulke    | siparis |  adet  |    ciro     | ort_sepet | ciro_payi
------------+---------+--------+-------------+-----------+-----------
 TR         |   56986 | 132481 | 13041878.86 |    228.86 |     66.12
 DE         |    8262 |  19375 |  1916286.25 |    231.94 |      9.72
 bilinmiyor |    4423 |  10189 |  1006331.55 |    227.52 |      5.10
 US         |    4111 |   9725 |   946203.24 |    230.16 |      4.80
 FR         |    4087 |   9695 |   941693.05 |    230.41 |      4.77
 GB         |    4120 |   9462 |   938335.44 |    227.75 |      4.76
 NL         |    3985 |   9307 |   934089.69 |    234.40 |      4.74

 toplam_siparis | ayni  | farkli | farkli_yuzde
----------------+-------+--------+--------------
         100000 | 45210 |  54790 |        54.79
```

**İş yorumu:**

Ciro büyük ölçüde tek pazara bağlı: Türkiye toplam cironun %66,12'sini üretiyor, ikinci sıradaki Almanya %9,72'de kalıyor. Gönderi ülkesi bilinmeyen siparişler %5,10'luk payla üçüncü sırada ve dört gerçek ülkeden daha büyük bir dilim oluşturuyor; `COALESCE` ile etiketlenmeseydi bu kitle raporda hiç görünmeyecek, ülke toplamları genel ciroya eşit çıkmayacaktı.

Ortalama sepet tutarları ülkeler arasında pratikte aynı (227,52 – 234,40 TL). Gerçek bir çok ülkeli operasyonda sepet tutarı, ürün karması ve marj ülkeye göre belirgin şekilde ayrışır; buradaki düzlük veri üreticisinin ülkeyi tamamen rastgele ataması sonucudur ve ülke bazlı fiyatlama/kampanya analizinin bu veri üzerinde anlamlı olmayacağını gösterir.

**Veri kalitesi bulgusu:** Siparişlerin %54,79'unda gönderi ülkesi, kullanıcının kayıtlı ülkesinden farklı. Gerçek bir sistemde bu oran %5-10 bandında beklenir (hediye gönderimi, iş adresi, taşınma). %55'lik bir fark, iki alanın birbirinden bağımsız üretildiğini gösteriyor ve ülke bazlı hiçbir analizin hangi alana dayandırılacağı sorusunu belirsiz bırakıyor.

**Metodolojik not — `IS DISTINCT FROM`:** İki kolonun farklı olup olmadığı sorgulanırken her iki taraf da `NULL` olabiliyorsa `<>` operatörü kullanılamaz; `NULL` içeren karşılaştırma `TRUE` dönmez ve satır sessizce elenir. `IS DISTINCT FROM` iki `NULL` değeri eşit sayar, `NULL` ile dolu değeri farklı sayar. Bu, `GROUP BY` ve `DISTINCT`'in `NULL`'lara uyguladığı "ayırt edilemezlik" mantığının operatör karşılığıdır.

---

## 25 — Kargo firması bazında ortalama teslim süresi

**SQL:** `sql-mastery/queries/q25_kargo_teslim_suresi.sql`

**Sonuç:**

```
 kargo_firmasi | gonderi | teslim_edilen | teslim_edilmeyen | teslim_edilmeme_yuzde | ort_teslim_gun
---------------+---------+---------------+------------------+-----------------------+----------------
 Yurtici       |   15781 |         13124 |             2657 |                 16.84 |           4.00
 MNG           |   15562 |         13092 |             2470 |                 15.87 |           4.00
 PTT           |   15422 |         12827 |             2595 |                 16.83 |           4.01
 Aras          |   15612 |         12969 |             2643 |                 16.93 |           4.01
 UPS           |   15622 |         13059 |             2563 |                 16.41 |           4.01

 toplam_gonderi | ortalamaya_giren | ortalamanin_disinda_kalan
----------------+------------------+---------------------------
          77999 |            65071 |                     12928
```

**İş yorumu:**

Beş kargo firmasının ortalama teslim süresi 4,00–4,01 gün, teslim edilememe oranı %15,87–%16,93 aralığında. Firmalar arasında ayırt edici bir fark yok; veri üreticisi kargo firmasını rastgele atadığı ve teslim süresini 1–7 gün arasında düzgün dağıttığı için bu beklenen sonuçtur. Firma seçimi kararı bu veri üzerinde alınamaz.

Teslim süresi dağılımı da gerçekçi değil: minimum 1, maksimum 7 gün ve arada düzgün dağılım var. Gerçek teslimat süreleri sağa çarpık dağılır — çoğu gönderi 2-4 günde varır, küçük bir kuyruk 15-30 günü bulur. Bu kuyruk, müşteri şikâyetlerinin ve operasyonel maliyetin büyük kısmını üretir ve burada tamamen yok.

**Metodolojik not — hayatta kalma yanlılığı:** Ortalama teslim süresi yalnızca teslim edilmiş gönderiler üzerinden hesaplanabilir, çünkü teslim edilmemiş gönderinin `delivered_at` değeri `NULL`'dur ve `avg()` `NULL` satırları sessizce atlar. 77.999 gönderinin 12.928'i (%16,6) ortalamanın dışında kaldı ve bunlar tam olarak hiç varmayan gönderilerdir. Uç durumda, paketleri hiç teslim etmeyen bir kargo firması "ortalama teslim süresi" tablosunda mükemmel görünürdü; tek bir satırı ortalamaya girmeyeceği için.

Bu nedenle teslim performansı tek bir ortalamayla raporlanamaz. Doğru rapor iki sayıyı birlikte verir: teslim edilenlerin ortalama süresi **ve** teslim edilememe oranı. Bu sorguda ikisi bilerek yan yana konulmuştur.

**Çapraz doğrulama:** `shipments` tablosunda 12.928 `in_transit`, 63.092 `delivered`, 1.979 `returned` kaydı var. `orders` tablosunda karşılıkları sırasıyla 12.933, 63.110 ve 1.979. Farklar 5 + 18 + 0 = 23 ve bu, `seed/checks.sql` ikinci kontrolünün bulduğu "kargo kaydı olmayan sipariş" sayısıyla birebir aynı. Üreticiye bilerek yerleştirilen anomali, bağımsız bir sorguda aynı sayıyla doğrulanmış oldu.
