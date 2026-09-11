# Ödev 3.3 — Performans Laboratuvarı

Ortam: PostgreSQL 16.13 (Docker), WSL2. Ölçümler `EXPLAIN (ANALYZE, BUFFERS)` ile alınmıştır.
Not: Süre makine yüküne göre değişir; buffer sayısı deterministiktir ve iş miktarının daha dürüst ölçüsüdür.

---

## P1 — `stock_cached` doğrulaması

**Sorgu:** `products.stock_cached` değeri `inventory_movements` toplamıyla uyuşmayan ürünler.
**Dosya:** `sql-mastery/perf/p1_stok_dogrulama.sql`

### Yavaş sürüm — bağıntılı alt sorgu
Seq Scan on products p (actual time=12245.542..12245.544 rows=0 loops=1)
Filter: (stock_cached <> COALESCE((SubPlan 1), '0'::bigint))
Rows Removed by Filter: 2000
Buffers: shared hit=3104072
SubPlan 1
-> Aggregate (actual time=6.065..6.065 rows=1 loops=2000)
-> Seq Scan on inventory_movements m (actual time=0.454..6.053 rows=82 loops=2000)
Filter: (product_id = p.id)
Rows Removed by Filter: 163930
JIT: Total 105.797 ms
Execution Time: 12272.654 ms

### Hızlı sürüm — defter bir kez gruplanıp LEFT JOIN
Hash Left Join (actual time=54.112..54.116 rows=0 loops=1)
Buffers: shared hit=1624
-> Seq Scan on products p (rows=2000 loops=1)
-> Hash -> HashAggregate (rows=2000 loops=1)
-> Seq Scan on inventory_movements (rows=164012 loops=1, actual 13.121 ms)
Execution Time: 54.252 ms

### Karşılaştırma

| Ölçü | Yavaş | Hızlı | Kazanç |
|---|---|---|---|
| Yürütme süresi | 12.272,65 ms | 54,25 ms | **226×** |
| Buffer (shared hit) | 3.104.072 | 1.624 | **1.911×** |
| `inventory_movements` taraması | 2.000 kez | 1 kez | — |
| JIT ek yükü | 105,80 ms | yok | — |

### Analiz

Yavaş sürümün süresinin tamamına yakını tek bir düğümden geliyor: `Aggregate ... loops=2000` düğümü her çalıştırmada 6,065 ms sürüyor, toplam 12.130 ms. İç `Seq Scan` her seferinde 164.012 satırın tamamını okuyup 163.930'unu eliyor; toplamda 328 milyon satır incelemesi yapılıyor.

Buffer sayısı bu israfı daha net gösteriyor: 3,1 milyon sayfa erişimi, 8 KB'lik sayfa boyutuyla yaklaşık 24 GB veri hareketi anlamına geliyor. Hızlı sürümde aynı iş 1.624 sayfayla tamamlanıyor.

Planlayıcı, alt sorgu içeren filtrenin seçiciliğini tahmin edemiyor: 1.990 satır bekliyor, gerçek sonuç 0 satır. Alt sorgulu filtrelerde planlayıcının kör kalması yaygındır ve kötü plan seçimlerinin yaygın sebeplerindendir. Ayrıca tahmini maliyetin yüksekliği JIT derlemesini tetikliyor ve zaten yavaş olan sorguya 105 ms saf ek yük biniyor.

**Kazanç index ile değil, sorgunun yeniden yazılmasıyla elde edildi.** "Her ürün için defteri tara" yaklaşımı yerine "defteri bir kez özetle, sonra eşleştir" yaklaşımı kullanıldı. Algoritmayı düzeltmek, aynı algoritmayı indeksle hızlandırmaktan daha büyük kazanç sağlar; indeks etkisi P2'de ayrıca ölçülmüştür.

---

## P2 — Index etkisi: aynı sorgu, dört sürüm

**Dosya:** `sql-mastery/perf/p2_index_etkisi.sql`
**Eklenen index:** `CREATE INDEX idx_inv_product ON inventory_movements (product_id);`

### Sonuçlar

| Sürüm | Yürütme süresi | Buffer | Referansa göre |
|---|---|---|---|
| Kötü sorgu, index yok | 12.647,47 ms | 3.104.072 | 1× |
| Kötü sorgu + index | 121,66 ms | 93.219 | 104× |
| Yeniden yazım, index yok | 54,25 ms | 1.624 | 233× |
| Yeniden yazım + index | 29,77 ms | 1.624 | plan değişmedi |

### Analiz

**Index, kötü sorguyu 104 kat hızlandırdı ancak yeniden yazımın yerini tutmadı.** Index eklendikten sonra alt sorgudaki tarama `Seq Scan`'den `Bitmap Index Scan`'e döndü ve her çalıştırma 6,27 ms'den 0,038 ms'ye indi. Buna rağmen toplam süre 121,66 ms'de kaldı; yeniden yazılmış sürüm index olmadan bile 54,25 ms ile daha hızlıdır. Kötü bir algoritmayı hızlandırmak, doğru algoritmayı kullanmanın yerine geçmiyor.

**Index'in gizli maliyeti heap erişimidir.** Plan şunu gösteriyor:

```
->  Bitmap Heap Scan on inventory_movements m  (actual time=0.008..0.033 rows=82 loops=2000)
      Heap Blocks: exact=89072
      ->  Bitmap Index Scan on idx_inv_product  (actual time=0.004..0.004 rows=82 loops=2000)
            Buffers: shared hit=3930 read=145
```

Index'in kendisini okumak 3.930 buffer tuttu; satırların kendisine erişmek 89.072 heap bloğu gerektirdi. Bir ürünün hareketleri tabloda ardışık değil dağınık yerleştiği için her satır ayrı bir blok erişimi doğuruyor. Index okunacak satır sayısını azaltır, satırların diskteki dağınıklığını azaltmaz.

**Index'in maliyeti:** Tablo 12 MB, index 1.168 kB (%10 ek yer). Ayrıca her `INSERT`, `UPDATE` ve `DELETE` işleminde index de güncellenir ve `VACUUM` yükü artar. Index eklemek yalnızca okuma hızını değil, yazma hızını ve bakım maliyetini de etkiler.

---

## Index'in işe yaramadığı durum #1 — tablonun tamamı taranıyor

Yeniden yazılmış sorgu, index eklendikten sonra da index kullanmadı:

```
->  HashAggregate  (rows=2000 loops=1)
      ->  Seq Scan on inventory_movements  (rows=164012 loops=1)
            Buffers: shared hit=1552
```

Buffer sayısı index öncesiyle birebir aynı (1.624). 54 ms'den 29 ms'ye düşüş index'ten değil, ön belleğin ısınmasından kaynaklanıyor.

**Neden:** Sorgu `GROUP BY product_id` ile tablonun tamamını özetliyor. Tüm satırlar zaten okunacaksa index üzerinden dolaşmak düz taramadan pahalıdır; her satır için önce index yaprağına, sonra heap bloğuna gitmek gerekir. Planlayıcı maliyet hesabını yapıp index'i reddetti ve doğru kararı verdi.

**Genel kural:** Index, tablonun küçük bir kısmını seçen sorgularda işe yarar. Seçicilik düştükçe (okunacak satır oranı arttıkça) index'in avantajı kaybolur ve belirli bir eşikten sonra düz tarama daha hızlı olur.

---

## P3 — `refresh_stock.sql` UPDATE'i

**Dosya:** `sql-mastery/perf/p3_refresh_stock.sql`
**Not:** `EXPLAIN ANALYZE` UPDATE'i gerçekten çalıştırır; ölçümler `BEGIN; ... ROLLBACK;` içinde alınmıştır.

### Sonuçlar

| Sürüm | Yürütme süresi | Buffer | Kirletilen sayfa | Güncellenen satır |
|---|---|---|---|---|
| Orijinal (index yok, `make seed` içinde ölçüldü) | 13.220 ms | — | — | 2.000 |
| Aynı sorgu + `idx_inv_product` | 214,18 ms | 113.005 | 78 | 2.000 |
| `UPDATE ... FROM`, bağıntılı alt sorgu korunmuş | 208,23 ms | 115.230 | 26 | 2.000 |
| Defter bir kez gruplanmış + değişmeyen satırlar atlanmış | **31,18 ms** | **1.700** | **0** | **0** |

**Toplam hızlanma: 424×** (13.220 ms → 31,18 ms).

### Analiz

**Taban çizgisi düzeltmesi:** İlk ölçümdeki 13,2 saniye index bulunmayan bir veritabanında alınmıştı. P2'de eklenen index sonrasında aynı sorgu 214 ms'ye indi. Performans karşılaştırmalarında taban çizgisi, karşılaştırılacak sürümle aynı koşullarda alınmalıdır; aksi hâlde index'in kazancı sorgu yeniden yazımının hanesine yazılır.

**`UPDATE ... FROM` yazımı tek başına kazanç sağlamadı.** İkinci sürüm 208,23 ms ile birinciyle aynı bandda kaldı, çünkü bağıntılı alt sorgu `FROM` içinde korunmuştu ve planda `loops=2000` değişmeden durdu. Sorgunun biçimini değiştirmek optimizasyon değildir; yapılan iş miktarını değiştirmek optimizasyondur.

**Asıl kazanç iki değişiklikten geliyor:**

1. Defter `GROUP BY product_id` ile bir kez özetleniyor — `loops=2000` ortadan kalkıyor.
2. `AND p.stock_cached IS DISTINCT FROM COALESCE(d.bakiye, 0)` koşuluyla değeri zaten doğru olan satırlar güncellenmiyor. Plan `Rows Removed by Filter: 2000` gösteriyor, yani hiçbir satır yazılmadı.

İkinci nokta yazma maliyeti açısından belirleyici. Postgres MVCC kullandığı için bir satırı aynı değerle güncellemek bile yeni bir satır sürümü yazar; eski sürüm ölü kalır ve `VACUUM` yükü doğurur. Plandaki `dirtied` sayacı bunu gösteriyor: birinci sürüm 78 sayfa kirletti, son sürüm sıfır.

**JIT ek yükü:** İlk iki sürümde tahmini maliyet JIT eşiğinin üzerinde kaldığı için Postgres sorguyu makine koduna derledi (90,19 ms ve 86,31 ms). Bu, ilk iki sürümün toplam süresinin yaklaşık %40'ıdır. Son sürümde tahmini maliyet eşiğin altında kaldığı için JIT devreye girmedi.

---

## P4 — Yorumsuz ürünler (q23)

**Dosya:** `sql-mastery/perf/p4_yorumsuz_urunler.sql`
**Eklenen indexler:** `order_items(product_id)`, `order_items(order_id)`, `reviews(product_id)`
**Tasarım:** 2×2 — (orijinal / yeniden yazılmış) × (index öncesi / sonrası)

| Sürüm | Yürütme süresi | Buffer | JIT ek yükü |
|---|---|---|---|
| Orijinal, index yok | 1.925,86 ms | 406.667 | 229,59 ms |
| Yeniden yazılmış, index yok | 65,48 ms | 2.780 | yok |
| **Orijinal, index var** | **30,69 ms** | 15.169 | 17,04 ms |
| Yeniden yazılmış, index var | 70,06 ms | 2.560 | yok |

**En iyi / en kötü: 62,7× hızlanma** (1.925,86 ms → 30,69 ms).

### Analiz

**Index yokken yeniden yazım 29 kat kazandırdı.** Orijinal sorguda `SELECT` listesindeki bağıntılı alt sorgu her aday ürün için `order_items` tablosunu baştan sona tarıyordu (`Seq Scan ... Rows Removed by Filter: 168911, loops=246`), toplam 406.667 buffer erişimi. Yeniden yazılmış sürüm aynı bilgiyi tek bir `HashAggregate` ile üretip 2.780 buffer'da tamamladı.

**Index'ler eklendikten sonra sıralama tersine döndü.** Orijinal sorgu 30,69 ms'ye inerek yeniden yazılmış sürümden (70,06 ms) 2,3 kat hızlı hâle geldi.

Sebebi seçicilik ve `LIMIT`. Koşulu sağlayan ürün sayısı 246'dır. Index'li orijinal sorgu `Nested Loop Semi Join` ile yalnızca bu ürünlere index üzerinden erişiyor ve 15.169 buffer kullanıyor. Yeniden yazılmış sürüm ise `teslim` CTE'sini kurabilmek için teslim edilmiş 106.541 sipariş kaleminin tamamını toplamak zorunda; sonuçta yalnızca 10 satır döndürecek olsa bile bu işi yapıyor.

**Çıkarım:** "Bağıntılı alt sorgu kötüdür, CTE'ye çevrilmelidir" şeklinde genel bir kural yoktur. Sonuç kümesi küçük ve seçici olduğunda, uygun index varsa satır satır index araması tam toplamadan hızlıdır; index yoksa tam toplama kazanır. Doğru yaklaşım veriye, seçiciliğe ve mevcut index'lere bağlıdır ve ölçülerek belirlenir.

**Index maliyeti:**

| Nesne | Boyut |
|---|---|
| `order_items` (tablo) | 13 MB |
| `idx_order_items_order` | 3.280 kB |
| `idx_order_items_product` | 1.208 kB |
| `reviews` (tablo) | 2.000 kB |
| `idx_inv_product` | 1.168 kB |
| `idx_reviews_product` | 240 kB |

`order_items` tablosuna eklenen iki index toplam 4.488 kB, yani tablo boyutunun %34'ü kadar ek yer kaplıyor.

---

## P5 — Gereksiz join, disk taşması, JIT ve work_mem (q47)

**Dosyalar:** `sql-mastery/perf/p5_gereksiz_join.sql`, `perf/p5b_jit_ve_workmem.sql`
**Eklenen indexler:** `payments(order_id)`, `order_coupons(order_id)`

| Sürüm | Yürütme süresi | Not |
|---|---|---|
| Orijinal (gereksiz `orders` join'i) | 765,96 ms | HashAggregate diske taştı (Batches: 21) |
| Gereksiz join kaldırıldı | 569,81 ms | −%26 |
| + `payments(order_id)` index'i | 510,95 ms | GroupAggregate'e geçti, taşma bitti |
| + JIT kapalı | **248,77 ms** | JIT saf ek yükmüş |
| + work_mem 64MB | 374,62 ms | **daha yavaş** |

**Toplam: 765,96 → 248,77 ms = 3,1×**

### Bulgu 1 — Gereksiz join

`siparis_tutari` CTE'si `orders` tablosuyla birleştiriliyordu ancak `orders`'tan yalnızca `o.id` kullanılıyordu ve bu değer zaten `oi.order_id` içinde mevcut. Yabancı anahtar kısıtı her `order_items` satırının bir `orders` satırına karşılık geldiğini garanti ettiğinden join ne satır eliyor ne kolon ekliyordu.

Postgres bu join'i `LEFT JOIN` yazıldığında kendisi kaldırıyor, `INNER JOIN` yazıldığında kaldırmıyor:

```
INNER JOIN:  Merge Join ... Index Only Scan using orders_pkey        → 138,23 ms
LEFT JOIN:   GroupAggregate ← Index Scan on order_items (orders yok) →  99,16 ms
```

Kaldırma koşulu, sağ tarafta benzersiz anahtar bulunması ve sağ tablodan hiçbir kolon kullanılmamasıdır. `INNER JOIN` prensipte satır eleyebileceği için planlayıcı bu çıkarımı yapmaz. Gereksiz `INNER JOIN`'ler elle temizlenmelidir.

### Bulgu 2 — Hash toplama diske taştı

`payments` düğümünde plan şunu gösteriyordu:

```
Planned Partitions: 4  Batches: 21  Memory Usage: 8249kB  Disk Usage: 3568kB
temp read=398 written=751
```

Hash tablosu `work_mem`'e (4MB) sığmadı, 21 parçaya bölünüp geçici dosyalara yazıldı. `Batches: 1` dışındaki her değer bellek yetersizliğinin işaretidir ve planlarda kontrol edilmelidir.

### Bulgu 3 — Index'in ikinci faydası: sıralı erişim

`payments(order_id)` index'i okunan satır sayısını azaltmadı (tablonun tamamı yine okundu), ancak satırları `order_id` sırasına dizili sunduğu için planlayıcı `HashAggregate` yerine `GroupAggregate` seçebildi. Sıralı gruplamada bellekte tek bir grup tutulur; taşma ortadan kalktı ve düğüm süresi 143,94 ms'den 63,40 ms'ye indi.

Index yalnızca filtreleme için değil, `GROUP BY`, `ORDER BY` ve merge join için gereken sıralamayı sağlamak için de kullanılır.

### Bulgu 4 — Kötü tahmin → şişmiş maliyet → gereksiz JIT

Planlayıcı `Hash Left Join` için 3.420.283.467 satır tahmin etti; gerçek sonuç 100.000 satır. Her CTE'de `order_id`'nin benzersiz olduğunu göremediği için birleştirmenin satır çoğaltacağını varsayıyor. Tahmini maliyet 51 milyona çıkınca JIT eşiği aşıldı ve sorgu makine koduna derlendi: 157–327 ms saf ek yük, toplam sürenin %31–43'ü.

`SET jit = off` ile süre 510,95 ms'den 248,77 ms'ye indi. Kısa süren analitik sorgularda kötü satır tahmini nedeniyle tetiklenen JIT, yaygın bir gizli maliyettir.

### Bulgu 5 — Daha çok bellek her zaman daha hızlı değildir

`work_mem` 4MB'den 64MB'ye çıkarıldığında süre 248,77 ms'den 374,62 ms'ye **yükseldi**. Sebep, planlayıcının artan bellekle birlikte `GroupAggregate` (index üzerinden akışkan) yerine `HashAggregate` (Seq Scan + 43 MB hash tablosu) seçmesidir. Kaynak artırmak planı değiştirir; değişen plan daha iyi olmayabilir. Ayar değişiklikleri her zaman ölçümle doğrulanmalıdır.

---

## Index'in işe yaramadığı durumlar (P6)

**Dosya:** `sql-mastery/perf/p6_index_ise_yaramaz.sql`

| Durum | Sorgu | Plan | Sonuç |
|---|---|---|---|
| Referans | `WHERE sku = 'SKU-00169'` | `Index Scan using products_sku_uq` | 0,049 ms |
| **A — kolona fonksiyon** | `WHERE lower(sku) = 'sku-00169'` | `Seq Scan`, 1.999 satır elendi | 1,239 ms |
| A — çözüm | aynı sorgu + ifade index'i | `Index Scan using idx_products_sku_lower` | 0,035 ms |
| Referans | `WHERE id = 169` | `Index Scan using products_pkey` | 0,032 ms |
| **B — kolona tip dönüşümü** | `WHERE id::text = '169'` | `Seq Scan` | 0,337 ms |
| **D — baştan joker** | `WHERE sku LIKE '%00169'` | `Seq Scan` | 0,248 ms |
| D — önek araması | `WHERE sku LIKE 'SKU-001%'` | `Seq Scan` | 0,238 ms |
| **E — OR, iki farklı kolon** | `sku = ... OR name = ...` | `Seq Scan` | 0,301 ms |
| C — düşük seçicilik | `WHERE is_active = true` (%92) | `Index Only Scan`, Heap Fetches: 0 | 0,189 ms |

### Neden

**A — Kolona uygulanan fonksiyon index'i körleştirir.** Index `sku` değerlerini saklar, `lower(sku)` değerlerini değil; Postgres ikisinin ilişkisini bilmediği için index'i kullanamaz. Çözüm ifade index'idir: `CREATE INDEX ... ON products (lower(sku))`. Aynı sorgu 1,239 ms'den 0,035 ms'ye indi (35×).

**B — Tip dönüşümü de bir fonksiyondur.** `id::text` yazıldığında index'teki `bigint` değerler kullanılamaz. Uygulama kodunda parametrenin yanlış tiple gönderilmesi bu durumu sessizce yaratır; filtre çalışır ama tablo baştan sona taranır.

**D — Baştan joker `LIKE '%...'` index'le aranamaz.** B-tree index önekten sona doğru sıralıdır; sonu bilinen ama başı bilinmeyen bir desen için sıralama işe yaramaz. Tam metin araması için `pg_trgm` veya `tsvector` gerekir.

Önek araması (`LIKE 'SKU-001%'`) teorik olarak index'le yapılabilir, ancak varsayılan collation'da B-tree sıralaması `LIKE` karşılaştırmasıyla uyuşmadığı için kullanılmaz. Bunun için index `text_pattern_ops` operatör sınıfıyla oluşturulmalıdır.

**E — `OR` ile iki farklı kolon.** Her koşul ayrı bir index gerektirir; yalnızca `sku` indexli olduğu için planlayıcı `name` koşulunu index'le karşılayamaz ve tabloyu taramak zorunda kalır. İki kolon da indexli olsaydı `BitmapOr` ile iki index birleştirilebilirdi.

**C — Düşük seçicilikte beklentinin düzeltilmesi.** Satırların %92'sini seçen koşulda index'in kullanılmaması beklenirdi; ancak sorgu `count(*)` olduğu ve gereken tüm bilgi index içinde bulunduğu için Postgres `Index Only Scan` seçti ve heap'e hiç gitmedi (`Heap Fetches: 0`).

Düzeltilmiş kural: düşük seçicilik index'in avantajını ortadan kaldırır **yalnızca tabloya (heap) gitmek gerektiğinde**. Sorgunun ihtiyaç duyduğu tüm kolonlar index içinde varsa (kapsayıcı index), seçicilik belirleyici olmaktan çıkar.

---

## Ödev 3.3 — Özet

| # | Sorgu | Önce | Sonra | Kazanç | Asıl düzeltme |
|---|---|---|---|---|---|
| P1 | `stock_cached` doğrulaması | 12.272 ms | 54 ms | **226×** | Bağıntılı alt sorgu → tek geçişli gruplama |
| P3 | `refresh_stock` UPDATE | 13.220 ms | 26 ms | **508×** | Tek geçiş + değişmeyen satırları atlama |
| P4 | Yorumsuz ürünler (q23) | 1.926 ms | 31 ms | **63×** | FK index'leri (seçici sorgu) |
| P5 | Ödeme doğrulaması (q47) | 766 ms | 249 ms | **3,1×** | Gereksiz join + index + JIT kapatma |
| A | `lower(sku)` araması | 1,24 ms | 0,035 ms | **35×** | İfade index'i |

### Çıkarılan kurallar

1. `loops=N` gördüğünde süreyi N ile çarp; sorgunun asıl maliyeti oradadır.
2. `Rows Removed by Filter` yüksekse, okunmaması gereken satırlar okunuyordur.
3. Buffer sayısı süreden daha güvenilir bir iş ölçüsüdür; makine yüküne göre değişmez.
4. `Batches > 1` bellek taşmasının işaretidir.
5. Algoritmayı düzeltmek, kötü algoritmayı indexlemekten genellikle daha çok kazandırır — ama sonuç kümesi küçük ve seçiciyse tersi doğru olabilir (P4).
6. Index yalnızca filtrelemeye değil, sıralama gerektiren işlemlere de (GROUP BY, merge join) hizmet eder.
7. Kötü satır tahmini şişmiş maliyet doğurur; şişmiş maliyet gereksiz JIT derlemesi tetikler.
8. Kaynak artırmak (work_mem) planı değiştirir; değişen plan daha yavaş olabilir.
9. Index'in maliyeti yer (tablo boyutunun %10–34'ü), yazma yavaşlaması ve VACUUM yüküdür.

### D ekleme — önek araması ve collation

Veritabanı collation'ı `en_US.utf8`. Bu collation'da B-tree index'in sıralaması `LIKE` önek karşılaştırmasıyla uyuşmadığı için `WHERE sku LIKE 'SKU-001%'` sorgusu normal index'i kullanamıyordu. `text_pattern_ops` operatör sınıfıyla açılan index sorunu çözdü:

```
Bitmap Index Scan on idx_products_sku_pattern
  Index Cond: ((sku ~>=~ 'SKU-001'::text) AND (sku ~<~ 'SKU-002'::text))
Execution Time: 0.135 ms   (onceki: 0.238 ms, Seq Scan)
```

Önek araması yapılacak metin kolonlarında, veritabanı collation'ı `C` değilse index `text_pattern_ops` ile oluşturulmalıdır.
