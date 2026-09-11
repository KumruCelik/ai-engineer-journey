# Bölüm 3 — Kontrol Soruları

Cevaplar bu bölümde yapılan ölçümlere dayanmaktadır; ilgili soru numaraları ve dosyalar belirtilmiştir.

---

## 1. `LEFT JOIN` sonrası `WHERE b.col IS NOT NULL` yazmak ne yapar? `INNER JOIN`'den farkı nedir?

**Hiçbir farkı kalmaz — sorguyu sessizce `INNER JOIN`'e çevirir.**

`LEFT JOIN`, sağ tarafta eşleşme bulunamayan sol satırları da sonuçta tutar ve sağ tablonun kolonlarını `NULL` yapar. `WHERE b.col IS NOT NULL` koşulu tam olarak bu satırları eler. Geriye yalnızca eşleşenler kalır, yani `INNER JOIN`'in sonucu. `LEFT` yazmanın tüm anlamı kaybolur; sorgu çalışır, hata vermez, yanlış da değildir — sadece yazdığın şey düşündüğün şey değildir.

Bunun tersi, anti-join kalıbıdır ve bilerek kullanılır:

```sql
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL          -- eslesme BULUNAMAYANLAR
```

Burada kontrol edilen kolon sağ tablonun `NOT NULL` bir kolonu (tercihen birincil anahtarı) olmalıdır. Nullable bir kolon seçilirse gerçek eşleşmeler de elenir ve sonuç sessizce bozulur.

*Ölçüm: Soru 12 — 20.000 kullanıcıdan 1.715'i hiç sipariş vermemiş.*

---

## 2. `NOT IN` bir alt sorguda `NULL` varsa ne olur? Neden?

**Sorgu hiçbir satır döndürmez ve hata vermez.**

`x NOT IN (a, b, c)` ifadesi mantıksal olarak `x <> a AND x <> b AND x <> c` demektir. Listedeki değerlerden biri `NULL` ise o karşılaştırmanın sonucu `TRUE` veya `FALSE` değil `NULL` olur (üç değerli mantık). `TRUE AND NULL` sonucu `NULL`'dur, ve `WHERE` yalnızca `TRUE` olan satırları geçirir. Dolayısıyla bileşik ifade hiçbir zaman `TRUE` olamaz.

Kendi verimde ölçtüm: `inventory_movements.order_id` kolonunda 12.192 `NULL` bulunuyor (alım ve düzeltme hareketlerinde sipariş yoktur). Aynı soru iki yazımla soruldu:

```
 not_exists_ile | not_in_ile
          12047 |          0
```

Doğru cevap 12.047, `NOT IN` sıfır döndürdü ve hiçbir uyarı vermedi.

**Kural:** Alt sorguyla olumsuzlama yapılacaksa `NOT EXISTS` kullanılmalıdır. `NOT IN` yalnızca alt sorgudaki kolonun `NOT NULL` olduğu şemadan doğrulandığında güvenlidir.

*Ölçüm: Soru 12 — `queries/q12_siparis_vermeyen_kullanicilar.sql`*

---

## 3. `COUNT(*)` ile `COUNT(column)` farkı hangi durumda tehlikeli sonuç verir?

`COUNT(*)` satırları sayar, içeriğe bakmaz. `COUNT(kolon)` o kolonu `NULL` **olmayan** satırları sayar. İkisinin farkı, o kolondaki `NULL` sayısına eşittir:

```
 toplam_kullanici | ulkesi_dolu | ulkesi_bos
            20000 |       19035 |        965
```

**Tehlikeli olduğu durum: elle ortalama hesaplarken.** `sum(x) / count(*)` ile `avg(x)` farklı sonuç verir. `avg` `NULL` satırları paydaya katmaz, elle bölme katar. Kolonun %5'i boşsa elle hesaplanan ortalama sistematik olarak düşük çıkar ve fark hiçbir yerde uyarı üretmez.

İkinci tehlikeli durum join sonrasıdır: `COUNT(o.id)` gibi bir sayım, fan-out yüzünden çoğalmış satırları da sayar ve sipariş sayısı yerine kalem sayısını verir. Doğru sayım `COUNT(DISTINCT o.id)`'dir.

*Ölçüm: Soru 02 ve Soru 11.*

---

## 4. Bir join sonucunda satır sayım beklenenden 3 kat fazla — hangi üç şeyi kontrol ederim?

**1) Fan-out: "bir" tarafın "çok" tarafıyla birleşmesi.** Bir siparişin birden çok kalemi varsa, `orders` satırı kalem sayısı kadar tekrarlanır. Kendi verimde `orders` 100.000 satır, `order_items` 168.920 satır; join sonucu 168.920 satır — yani sipariş sayısı değil kalem sayısı. Kontrol: `count(*)` ile `count(DISTINCT üst_tablo.id)` karşılaştırılır, oran fan-out katsayısıdır (bende 1,69).

**2) Çifte fan-out: aynı üst tablonun iki farklı alt tablosunun birlikte birleştirilmesi.** `orders` hem `order_items` hem `order_coupons` ile birleştirilirse, kalemler kupon sayısı kadar, kuponlar kalem sayısı kadar çoğalır ve iki ölçü iki farklı katsayıyla bozulur. Kendi ölçümümde 26.212 olması gereken satır sayısı 30.469 çıktı. Çözüm: her alt tabloyu kendi taneciğinde ayrı ayrı toplayıp sipariş kimliği üzerinden birleştirmek.

**3) Eksik veya yanlış join koşulu.** `ON` yan tümcesi unutulursa kartezyen çarpım oluşur; bileşik anahtarın yalnızca bir kolonu yazılırsa kısmi çarpım oluşur. Planı `EXPLAIN` ile kontrol etmek gerekir: `Nested Loop` düğümünde `Join Filter` yoksa veya `rows` tahmini tablo satır sayılarının çarpımına yakınsa koşul eksiktir.

**Ek kontrol:** Boyut tablosu SCD2 ise, sürüm dönemlerinin çakışıp çakışmadığı kontrol edilmelidir. Çakışan dönemler aynı olgunun birden çok boyut satırıyla eşleşmesine ve tüm toplamların şişmesine yol açar.

*Ölçüm: Soru 11 ve Soru 21.*

---

## 5. Window function ile `GROUP BY` arasındaki temel fark nedir?

**`GROUP BY` satırları birleştirip yok eder; window function satırları korur ve yanlarına hesaplanmış bir kolon ekler.**

`GROUP BY category_id` ile 2.000 ürün 32 satıra iner ve ürün bilgisi kaybolur. `avg(list_price) OVER (PARTITION BY category_id)` ile 2.000 satırın hepsi durur, her birinin yanında kendi kategorisinin ortalaması yazar. "Bu ürün kendi kategorisinin ortalamasının ne kadar üstünde" gibi sorular yalnızca ikinci yolla cevaplanabilir.

İki pratik sonuç:

- **İşleme sırası:** Window function'lar `WHERE` ve `HAVING`'den sonra hesaplanır, dolayısıyla bu yan tümcelerde kullanılamazlar. "Her grupta ilk N" sorguları bu yüzden her zaman iki katmanlıdır: pencere bir CTE'de hesaplanır, filtre dışarıda uygulanır. Güncel sıra: `FROM → WHERE → GROUP BY → HAVING → WINDOW → SELECT → DISTINCT → ORDER BY → LIMIT`.
- **Çerçeve:** `OVER` içinde `ORDER BY` bulunduğunda varsayılan çerçeve "pencerenin başından mevcut satıra kadar"dır. Bu yüzden `SUM(x) OVER (ORDER BY ay)` toplam değil kümülatif toplam üretir. `OVER ()` genel toplamı, `OVER (PARTITION BY yil)` yıl toplamını verir — aynı fonksiyon, üç farklı sonuç.

*Ölçüm: Soru 30 ve Soru 36.*

---

## 6. Bir tabloya index eklemenin maliyeti nedir?

Faydası bilinir: seçici sorgularda okunacak satır sayısını azaltır, ayrıca sıralı erişim sağlayarak `GROUP BY` ve merge join'i hızlandırır. Maliyeti dört başlıktadır.

**1) Disk alanı.** Kendi veritabanımda `order_items` tablosu 13 MB; eklenen iki index 3.280 kB ve 1.208 kB, toplam tablo boyutunun %34'ü. `inventory_movements` için index, tablonun %10'u kadar yer kapladı.

**2) Yazma yavaşlaması.** Her `INSERT`, `UPDATE` ve `DELETE` işleminde index de güncellenir. Bir tabloda beş index varsa, tek bir satır eklemek altı yapıya yazmak demektir. Toplu yükleme yapılan tablolarda index'leri yükleme öncesi kaldırıp sonra yeniden oluşturmak yaygın bir uygulamadır.

**3) Bakım yükü.** PostgreSQL'de güncellenen satırın yeni bir sürümü yazılır; eski sürümler `VACUUM` ile temizlenir ve bu iş index'leri de kapsar. Index sayısı arttıkça `VACUUM` süresi ve şişme (bloat) riski artar.

**4) Planlayıcı maliyeti ve yanlış seçim riski.** Her index, planlayıcının değerlendirmesi gereken bir seçenek daha demektir. Ayrıca index her zaman kazandırmaz — kendi ölçümlerimde dört durumda hiç kullanılmadı ya da fayda etmedi: kolona fonksiyon uygulandığında (`lower(sku)`), kolona tip dönüşümü uygulandığında (`id::text`), baştan joker içeren `LIKE '%...'` desenlerinde, ve `OR` ile birbirinden bağımsız iki kolon sorgulandığında. Ayrıca tablonun tamamını tarayan bir toplama sorgusunda planlayıcı index'i bilerek reddetti ve doğru kararı verdi.

**Dengeleyici ölçüm:** Aynı sorguda index 104 kat kazandırdı (12.647 ms → 121,7 ms), ancak sorgunun yeniden yazılması index olmadan 233 kat kazandırdı (12.647 ms → 54,3 ms). Index eklemek ilk refleks olmamalıdır; önce yapılan iş miktarı sorgulanmalıdır.

*Ölçüm: `notes/odev-3.3/olcumler.md` — P2 ve P6.*
