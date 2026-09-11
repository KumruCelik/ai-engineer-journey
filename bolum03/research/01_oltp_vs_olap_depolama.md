# OLTP vs OLAP: Satır Bazlı ve Kolon Bazlı Depolama

## İki farklı iş yükü

Bir veritabanından iki tür iş istenir. Birincisi işlemseldir (OLTP): "şu siparişi kaydet", "bu kullanıcının adresini güncelle", "sipariş 4172'yi getir". Az sayıda satıra dokunur, ama satırın tamamıyla ilgilenir ve saniyede binlerce kez tekrarlanır. İkincisi analitiktir (OLAP): "geçen yılın aylık cirosu", "kategori bazında marj". Milyonlarca satıra dokunur, ama her satırın yalnızca birkaç kolonuna bakar ve sonuçta tek bir tablo üretir.

Bu iki ihtiyaç, verinin diske nasıl yerleştirileceği konusunda birbirine zıt cevaplar verir.

## Satır bazlı yerleşim

PostgreSQL gibi OLTP sistemleri veriyi **satır satır** saklar. Bir siparişin tüm kolonları diskte yan yanadır. "Sipariş 4172'yi getir" dendiğinde tek bir sayfa okunur ve satırın tamamı elde edilir — ideal davranış. Yeni satır eklemek de ucuzdur: sayfanın sonuna yazılır.

Bedeli analitik tarafta ortaya çıkar. `SELECT sum(brut_tutar)` sorgusu yalnızca bir kolona ihtiyaç duyar, ama o kolonun değerleri diskte 40–60 baytlık satırların içine dağılmıştır. Tek bir kolonu okumak için satırların tamamını okumak gerekir.

Bu etkiyi kendi şemamda ölçtüm. Ödev 3.4'te analitik katman kurulduktan sonra "sipariş durumu dağılımı" sorgusu OLTP tablosunda 15,1 ms, analitik tabloda 20,1 ms sürdü — analitik tablo **daha yavaş** çıktı. Sebebi, `fct_orders`'ın 12 kolonlu, `orders`'ın 5 kolonlu olmasıydı. Yalnızca `status` kolonunu sayan bir sorgu, satır bazlı yerleşimde geniş satırların bedelini ödüyor. Denormalizasyon analitik sorguların çoğunu hızlandırırken, dar sorguları yavaşlatıyor.

## Kolon bazlı yerleşim

Kolon bazlı biçimler (Parquet, ORC) aynı kolonun değerlerini yan yana saklar. `brut_tutar` kolonu diskte kesintisiz bir blok hâlinde durur; o kolonu okumak için başka hiçbir şey okunmaz. Buna **projection pushdown** denir: sorgu hangi kolonları istiyorsa yalnızca onlar diskten gelir.

İkinci kazanç sıkıştırmadır. Aynı kolondaki değerler aynı tipte ve birbirine benzerdir — tekrarlı `status` değerleri, dar aralıktaki tarihler, benzer büyüklükte tutarlar. Bu benzerlik sıkıştırma oranını satır bazlı yerleşime göre kat kat artırır.

Üçüncü kazanç yürütme biçimidir. Kolon bazlı motorlar veriyi satır satır değil, binlik bloklar hâlinde işler (vektörleştirilmiş yürütme) ve bu blokları birden çok çekirdeğe dağıtır.

## Ölçümler

Aynı veriyi (168.920 satırlık `star.fct_order_items`) iki biçimde tutup aynı üç sorguyu çalıştırdım.

**Depolama:**

| Biçim | Boyut |
|---|---|
| PostgreSQL tablosu (index'ler dahil) | 77 MB |
| CSV | 12 MB |
| Parquet | **5,5 MB** |

Parquet, Postgres'in yaklaşık on dörtte biri. Karşılaştırma tam adil değil: 77 MB, tablonun yanında altı index ve MVCC için satır başına tutulan görünürlük bilgisini de içeriyor. Yine de büyüklük mertebesi farkı gerçektir.

**Sorgu süresi:**

| Sorgu | PostgreSQL | DuckDB/Parquet | Oran |
|---|---|---|---|
| `sum(brut_tutar)` | 16,73 ms | 4 ms | 4,2× |
| `GROUP BY status` | 22,93 ms | 6 ms | 3,8× |
| `GROUP BY product_sk`, ilk 10 | 36,23 ms | 3 ms | 12,1× |
| **Toplam** | **75,89 ms** | **13 ms** | **5,8×** |

Fark, kolon sayısı arttıkça ve okunan kolon oranı düştükçe büyüyor. Üçüncü sorgu iki kolona bakıyor ve en büyük farkı orada görüyoruz.

**Bellek:** NYC taksi verisinde (2,96 milyon satır, 47,6 MB Parquet) aynı üç toplama işlemini DuckDB ve pandas ile çalıştırdım. DuckDB toplam 148,9 ms ve 171,7 MB tepe bellek kullandı; pandas 2.796,4 ms ve 1.336,9 MB. pandas dosyayı önce belleğe açtığı için 47,6 MB'lik dosya 398,6 MB'lik bir DataFrame'e dönüştü — 8,4 kat şişme. DuckDB dosyayı hiç yüklemedi.

## Ne zaman hangisi

Satır bazlı yerleşim, tek satır okuma ve sık yazma gerektiren işlemsel sistemler için doğrudur; MVCC, kilitleme ve indexleme bu yerleşimin üzerine kurulur. Kolon bazlı yerleşim, çok satır tarayıp az kolon okuyan analitik sorgular için doğrudur; buna karşılık tek satır güncellemek pahalıdır, çünkü satırın parçaları farklı bloklara dağılmıştır.

Pratikte ikisi birlikte kullanılır: işlemsel sistem satır bazlı kalır, veriler düzenli aralıklarla kolon bazlı bir analitik katmana kopyalanır. Bu bölümde kurduğum yapı da budur — OLTP şeması PostgreSQL'de, analitik katman `star` şemasında, dosya analitiği ise Parquet üzerinde DuckDB ile.

## Sınırlar

Ölçümler tek makinede, küçük hacimde (168 bin ve 2,96 milyon satır) ve sıcak önbellekle alınmıştır. Veri belleğe sığdığı için disk okuma maliyeti ölçüme neredeyse hiç girmedi; asıl fark disk darboğazının devreye girdiği hacimlerde çok daha büyür. Ayrıca PostgreSQL genel amaçlı bir işlemsel sistem, DuckDB ise yalnızca analitik için tasarlanmış bir motordur; karşılaştırma "hangisi daha iyi" sorusunun değil, "hangi yerleşim hangi işe uygun" sorusunun cevabıdır.
