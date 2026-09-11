# Ödev 3.4 — Analitik katman (star schema) ve SCD2

## Şema

Analitik tablolar `star` şeması altında; OLTP tabloları `public` altında kalıyor.

| Tablo | Grain — bir satır neyi temsil ediyor | Tip |
|---|---|---|
| `star.dim_date` | Bir takvim günü | statik boyut |
| `star.dim_customer` | **Bir müşterinin bir geçerlilik dönemi** | **SCD2** |
| `star.dim_product` | Bir ürünün güncel hâli | SCD1 (üzerine yazma) |
| `star.fct_orders` | Bir sipariş | olgu |
| `star.fct_order_items` | Bir sipariş kalemi | olgu |

Satır sayıları: 911 / 20.001 / 2.000 / 100.000 / 168.920.
Doğrulama: `sum(fct_orders.brut_tutar)` = 19.724.818,08 TL — OLTP'den hesaplanan ciroyla birebir aynı.

## Tasarım kararları

**`dim_product` SCD2 değil SCD1.** Tarihsel fiyat zaten `order_items` tablosunda sipariş anında dondurulmuş durumda (K-002), dolayısıyla ürün boyutunda sürüm tutmaya gerek yok. Olgu tablosunda snapshot varsa boyutta SCD2 gereksizdir.

**`dim_customer` SCD2.** Müşterinin ülkesi, adı veya aktiflik durumu değiştiğinde eski sürüm kapatılır, yeni sürüm açılır. "Bu sipariş verildiğinde müşteri hangi ülkedeydi" sorusu ancak böyle cevaplanır.

**İlk sürümün `valid_from` değeri `-infinity`.** Kullanıcının kayıt saatinden birkaç saat önce görünen bir sipariş bile ilk sürüme düşer; hiçbir sipariş boşta kalmaz. Doğrulandı: `orders` ile `fct_orders` satır sayıları eşit, kayıp sipariş yok.

**Veritabanı seviyesinde SCD2 garantisi:** `CREATE UNIQUE INDEX ... ON star.dim_customer (user_id) WHERE is_current` — kısmi benzersiz index, bir müşterinin birden çok güncel sürümü olmasını imkânsız kılıyor. Ayrıca iki `CHECK` kısıtı dönem tutarlılığını (`valid_to > valid_from`) ve `is_current` ile `valid_to` uyumunu zorunlu kılıyor.

## Idempotent yükleme

`star/load_star.sql` tekrar tekrar çalıştırılabilir:

- `dim_date` → `ON CONFLICT DO NOTHING`
- `dim_product` → `ON CONFLICT DO UPDATE` (SCD1 upsert)
- `dim_customer` → iki adımlı SCD2 birleştirme: (a) OLTP'de değişmiş güncel satırları kapat, (b) güncel satırı olmayan her kullanıcı için yeni sürüm aç
- `fct_orders`, `fct_order_items` → `ON CONFLICT DO UPDATE`

**Test:** Yükleyici arka arkaya iki kez çalıştırıldı, satır sayıları değişmedi (20.000 / 100.000 / 168.920).

## SCD2 testi

Test kullanıcısı (id 7, 4 siparişi var) OLTP'de `TR`'den `DE`'ye taşındı ve yükleyici tekrar çalıştırıldı.

```
 customer_sk | user_id | country |          valid_from           |           valid_to            | is_current
-------------+---------+---------+-------------------------------+-------------------------------+------------
           7 |       7 | TR      | -infinity                     | 2026-09-11 14:04:01.091241+00 | f
       20001 |       7 | DE      | 2026-09-11 14:04:01.091241+00 |                               | t
```

Eski sürüm kapandı, yeni sürüm açıldı, `dim_customer` 20.000'den 20.001 satıra çıktı.

### Tarihsel doğruluk ve leakage

Aynı dört sipariş, iki farklı kaynaktan:

```
 star (dogru)                OLTP (sizintili)
 siparis_anindaki_ulke       oltp_bugunku_ulke
 TR                          DE
 TR                          DE
 TR                          DE
 TR                          DE
```

Star katmanı siparişin verildiği andaki ülkeyi veriyor; OLTP yalnızca bugünkü hâli tuttuğu için geçmişe bugünün bilgisini yansıtıyor.

Bu fark, makine öğrenmesinde **veri sızıntısı (leakage)** olarak adlandırılır. Geçmiş tarihli bir özellik üretilirken `users.country` kullanılırsa, modele o tarihte var olmayan bir bilgi verilmiş olur. Model eğitim ve doğrulama aşamalarında yüksek başarı gösterir, üretimde çöker — çünkü üretimde gelecekteki değer bilinmez. SCD2, geçmişi olduğu gibi saklayarak bu hatayı yapısal olarak imkânsız kılar.

### Tutarlılık kontrolleri

| Kontrol | Beklenen | Sonuç |
|---|---|---|
| Her kullanıcının tam olarak bir güncel sürümü | 0 ihlal | 0 |
| Kapalı sürümlerin `valid_to` değeri dolu | 0 ihlal | 0 |
| Sürüm dönemleri çakışmıyor | 0 ihlal | 0 |
| Sürüm bulamamış sipariş | 0 | 0 |

## 10 iş sorusu — OLTP vs STAR

| # | Soru | OLTP (ms) | STAR (ms) | Kazanç |
|---|---|---|---|---|
| 1 | Aylık ciro | 169,93 | 33,00 | 5,1× |
| 2 | Kategori bazında ciro | 91,78 | 29,16 | 3,1× |
| 3 | Ciro bazında ilk 10 ürün | 74,11 | 32,06 | 2,3× |
| 4 | Müşteri başına harcama (ilk 10) | 104,69 | 41,26 | 2,5× |
| 5 | Sipariş durumu dağılımı | 15,15 | 20,05 | **0,76×** |
| 6 | Hafta günü bazında sipariş | 33,80 | 19,48 | 1,7× |
| 7 | Ortalama sepet tutarı | 74,44 | 0,54 | **139×** |
| 8 | Kupon indiriminin marj etkisi | 77,61 | 0,53 | **146×** |
| 9 | Ülke bazında ciro | 59,47 | 26,58 | 2,2× |
| 10 | Kök kategori bazında ciro | 68,57 | 23,79 | 2,9× |
| | **Toplam** | **769,55** | **226,45** | **3,4×** |

### Süre analizi

**7 ve 8. sorularda 140 kat kazanç**, ölçünün sipariş taneciğinde önceden hesaplanmış olmasından geliyor. `star.fct_orders.brut_tutar` ve `indirim_tutar` yükleme sırasında bir kez hesaplanıp saklandığı için sorgu tek bir tablo taraması ile tamamlanıyor; OLTP'de aynı sonuç için kalemleri sipariş bazında toplamak ve kupon tablosunu ayrıca birleştirmek gerekiyor. Olgu tablosunun varlık sebebi budur: sorulacak taneciğe önceden indirgemek.

**5. soruda star daha yavaş.** `fct_orders` 12 kolonlu, `orders` 5 kolonlu. Yalnızca `status` kolonuna bakan bir sorgu satır başına daha fazla bayt okumak zorunda kalıyor. Denormalizasyonun bedeli budur: geniş satırlar, dar sorguları yavaşlatır. Satır bazlı depolamada bir kolonu okumak için satırın tamamını okumak gerekir; kolon bazlı depolama (Ödev 3.5) tam olarak bu sorunu çözer.

### Okunabilirlik analizi

Süreden bağımsız olarak sorgu karmaşıklığı belirgin şekilde azalıyor:

| Soru | OLTP yapısı | STAR yapısı |
|---|---|---|
| 8 (kupon marj etkisi) | 2 CTE + 2 join + `COALESCE` | tek tablo, iki `sum()` |
| 10 (kök kategori) | `WITH RECURSIVE` + 4 tablolu join | tek join, `dim_product.kok_kategori` |
| 7 (ortalama sepet) | iç içe iki seviye gruplama | tek `avg()` |
| 2 (kategori cirosu) | 4 tablolu join | 2 tablolu join |

Kategori hiyerarşisi yükleme sırasında `dim_product.kok_kategori` kolonuna düzleştirildiği için, analiz yapan kişinin recursive CTE yazmasına gerek kalmıyor. Analitik katmanın asıl değeri sürede değil, **karmaşıklığın sorgudan yüklemeye taşınmasındadır**: karmaşık mantık bir kez yazılır, her sorguda tekrar yazılmaz ve her analistin aynı hatayı tekrar yapma riski ortadan kalkar.
