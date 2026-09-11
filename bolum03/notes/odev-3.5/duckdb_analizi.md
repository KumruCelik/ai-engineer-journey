# Ödev 3.5 — DuckDB ile dosya analitiği

**Veri:** NYC TLC sarı taksi, 2024 Ocak. `yellow_tripdata_2024-01.parquet`
**Boyut:** 47,6 MB (Parquet) · 2.964.624 satır · 19 kolon
**Araç:** DuckDB CLI — sunucu kurulumu yok, veri yüklemesi yok, dosya doğrudan sorgulanıyor.

## Neden Parquet 47 MB?

Ödev metninde veri ~3 GB olarak geçiyor; bu ham/CSV boyutudur. Parquet kolon bazlı ve sıkıştırılmış bir biçimdir: aynı kolondaki değerler yan yana durduğu için sıkıştırma oranı satır bazlı biçimlerden kat kat yüksektir (tekrarlı `payment_type`, `VendorID` gibi kolonlar neredeyse bedavaya gelir). Yaklaşık 60 kat küçülme elde ediliyor.

## 10 sorgunun sonuçları

**Saat bazında talep.** Zirve 18:00 (%7,18), dip 04:00 (%0,56) — 12,8 kat fark. Talep akşam saatlerinde yoğunlaşıyor; filo ve vardiya planlaması bu eğriye göre kurulur.

**Mesafe ekonomisi.** Mil başına maliyet mesafeyle keskin düşüyor:

| Bant | Yolculuk | Ort. ücret | Mil başına |
|---|---|---|---|
| 0–1 mil | 677.513 | 7,42 | **36,12** |
| 1–3 mil | 1.447.292 | 12,71 | 11,98 |
| 3–10 mil | 549.991 | 26,59 | 7,40 |
| 10+ mil | 229.457 | 62,77 | **5,30** |

Sabit taban ücret kısa yolculuklarda toplam tutara hâkim oluyor. Yolculukların yarısı 1–3 mil bandında.

**Havalimanı imzası.** En yoğun alış noktası 132 (JFK) ortalama 15,49 mil, 138 (LaGuardia) 9,59 mil; kalan sekiz nokta 1,7–2,9 mil bandında (Manhattan içi). Konum kimliğini bilmeden bile ortalama mesafe havalimanlarını ele veriyor.

**Bahşiş.** Kartlı ödemelerde bahşiş oranı medyanı %26,25 (p25 %20,6 · p75 %31,1 · p95 %41,2). Nakit ödemelerde (`payment_type = 2`) ortalama bahşiş 0,00 — nakit bahşiş sisteme hiç girmiyor. Bahşiş analizinde nakit yolculukların dahil edilmesi ortalamayı yapay olarak düşürür.

## Veri kalitesi bulguları

| Kontrol | Sonuç | Oran |
|---|---|---|
| Negatif toplam tutar | 35.504 | %1,20 |
| Sıfır mesafe | 60.371 | %2,04 |
| Varış kalkıştan önce | 56 | %0,002 |
| Ay dışında tarih | 18 | — |
| `passenger_count` boş | 140.162 | %4,73 |

**En eski yolculuk 2002-12-31.** Ocak 2024 dosyasında yirmi yıl öncesine ait kayıtlar var. Tarih filtresi uygulanmadan yapılan her zaman serisi analizi bu satırlar yüzünden bozulur.

**Sıfır mesafeyle 5.000 dolarlık yolculuklar.** En pahalı on yolculuğun beşinde `trip_distance = 0` ve tutar 1.000–5.000 dolar arasında. Bunlar ölçüm hatası veya hatalı giriştir; ortalama tutar hesabına dahil edilmeleri sonucu belirgin şekilde kaydırır.

**Çapraz eşleşme:** `payment_type = 0` olan kayıt sayısı 140.162, `passenger_count IS NULL` olan kayıt sayısı da 140.162. Aynı kayıtlardır — belirli bir veri sağlayıcısının sistemi bu iki alanı hiç doldurmuyor. Eksik veri rastgele değil, kaynağa bağlı ve sistematiktir; "eksikleri ortalamayla doldurma" gibi bir müdahale bu durumda tüm bir sağlayıcının davranışını uydurmak anlamına gelir.

## DuckDB vs pandas

Aynı üç toplama işlemi, aynı dosya üzerinde:

| İşlem | DuckDB | pandas | Oran |
|---|---|---|---|
| Dosyayı belleğe yükleme | **gerekmiyor** | 717,3 ms | — |
| 1. Günlük yolculuk + ort. tutar | 81,8 ms | 1.726,1 ms | **21,1×** |
| 2. Saat bazında dağılım | 51,1 ms | 292,5 ms | 5,7× |
| 3. Ödeme tipine göre bahşiş | 16,0 ms | 60,5 ms | 3,8× |
| **Toplam (yükleme dahil)** | **148,9 ms** | **2.796,4 ms** | **18,8×** |
| **Tepe bellek** | **171,7 MB** | **1.336,9 MB** | **7,8×** |
| DataFrame'in bellekteki boyutu | — | 398,6 MB | |

Yalnızca iki kolon okunduğunda: DuckDB 16,8 ms, pandas (`columns=` ile) 92,4 ms — 5,5×.

### Farkın kaynağı

**pandas önce tüm dosyayı belleğe açar.** 47,6 MB'lik Parquet dosyası bellekte 398,6 MB'lik bir DataFrame'e dönüşüyor — 8,4 kat şişme. Sıkıştırma açılıyor, kolonlar Python/NumPy nesnelerine dönüştürülüyor, `object` tipindeki metin kolonları ek yük getiriyor. Süreç tepe belleği 1,3 GB'a çıkıyor.

**DuckDB dosyayı hiç yüklemez.** Sorguyu analiz edip yalnızca gereken kolonları, yalnızca gereken satır gruplarını Parquet'ten okur (projection ve predicate pushdown). Bellekte tutulan şey dosya değil, sorgunun ara sonucudur.

İkinci fark **vektörleştirilmiş yürütme**: DuckDB veriyi satır satır değil, binlik bloklar hâlinde işler ve birden çok çekirdeği kullanır. `Run Time` çıktılarında `user` süresinin `real` süreden büyük olması bunun kanıtıdır (ör. 0,097 real / 0,250 user) — iş paralel yürütülmüş.

pandas'ın avantajı, veri bir kez belleğe alındıktan sonra üzerinde serbestçe Python kodu çalıştırılabilmesidir. Ancak tek seferlik toplama işleri için bu avantaj bedelini karşılamıyor: dosya belleğe sığmayacak kadar büyüdüğünde pandas tamamen durur, DuckDB çalışmaya devam eder.

## Çıkarım

Tek seferlik analitik sorgular için veriyi bir sunucuya yüklemek ya da belleğe açmak gereksiz bir adımdır. Dosya olduğu yerde sorgulanabildiğinde hem süre hem bellek bir büyüklük mertebesi azalıyor. Veri mühendisliğinde "şu dosyaya bir bakayım" ihtiyacının varsayılan aracı budur.
