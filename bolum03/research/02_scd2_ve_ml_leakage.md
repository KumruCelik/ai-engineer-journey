# SCD Type 2 Nedir ve Neden ML Özelliği Üretirken Kritiktir?

## Problem: boyutlar değişir

Bir müşteri taşınır, evlenip soyadını değiştirir, üyelik seviyesi yükselir. İşlemsel veritabanı bu değişikliği `UPDATE` ile uygular ve eski değer kaybolur. Bu, işlemsel sistem için doğru davranıştır — kargo bugünkü adrese gitmelidir.

Analitik taraf için ise yıkıcıdır. "Geçen yılın Almanya cirosu" sorusu, siparişlerin **o tarihte** hangi ülkeye ait olduğunu gerektirir. Müşteri bu yıl Türkiye'ye taşındıysa ve yalnızca güncel değer saklanıyorsa, geçen yılın Almanya cirosu bugün değişir. Rapor, üretildiği güne göre farklı sonuç verir.

Yavaş değişen boyut (Slowly Changing Dimension) yaklaşımları bu soruna üç cevap verir. **Type 1** eski değeri siler, geçmişi feda eder. **Type 3** yalnızca bir önceki değeri ayrı bir kolonda tutar, sınırlı bir hafıza sağlar. **Type 2** her değişiklikte yeni bir satır açar ve eskisini kapatır; geçmişin tamamı korunur.

## SCD2 mekaniği

Boyut tablosuna üç kolon eklenir: `valid_from`, `valid_to`, `is_current`. Satırın kimliği artık doğal anahtar (`user_id`) değil, yapay bir anahtardır (`customer_sk`), çünkü aynı müşterinin birden çok satırı olacaktır.

Yükleme iki adımlıdır: değişmiş güncel satırlar kapatılır (`valid_to = now()`, `is_current = false`), ardından güncel satırı olmayan her müşteri için yeni sürüm açılır. Kendi uygulamamda bu mantığı `star/load_price_scd2.sql` ve `load_star.sql` içinde yazdım ve tekrar tekrar çalıştırılabilir (idempotent) olduğunu doğruladım.

Kritik kural — bir müşterinin aynı anda yalnızca bir güncel sürümü olabilir — veritabanı seviyesinde garanti altına alınabilir:

```sql
CREATE UNIQUE INDEX ON star.dim_customer (user_id) WHERE is_current;
```

Kısmi benzersiz index, kuralı belgeye değil şemaya yazar. Belgedeki kural unutulur; şemadaki kural unutulamaz.

Olgu tablosu, boyuta doğal anahtarla değil, **olay anında geçerli olan sürümün** yapay anahtarıyla bağlanır:

```sql
JOIN star.dim_customer d
  ON d.user_id = o.user_id
 AND o.ordered_at >= d.valid_from
 AND o.ordered_at <  COALESCE(d.valid_to, 'infinity')
```

Dönem sınırları sol kapalı–sağ açık seçilir; iki sürümün sınır anında çakışması böyle önlenir.

## Ölçüm: aynı sipariş, iki farklı cevap

Kendi veritabanımda test ettim. Dört siparişi bulunan bir kullanıcının ülkesini OLTP'de `TR`'den `DE`'ye değiştirip yükleyiciyi tekrar çalıştırdım. Boyut tablosunda eski sürüm kapandı, yeni sürüm açıldı. Ardından aynı dört siparişi iki kaynaktan sorguladım:

```
 siparis_tarihi | star (siparis anindaki ulke) | OLTP (bugunku ulke)
 2025-06-30     | TR                           | DE
 2025-09-21     | TR                           | DE
 2026-01-21     | TR                           | DE
 2026-06-05     | TR                           | DE
```

Star katmanı doğru cevabı, OLTP bugünün cevabını veriyor. Aradaki fark tam olarak sızıntının kendisidir.

## ML açısından neden kritik

Makine öğrenmesinde model, geçmiş olaylardan öğrenir ve gelecekteki olayları tahmin eder. Eğitim verisi hazırlanırken her örnek için "o anda ne biliniyordu" sorusu cevaplanmak zorundadır. Buna **zaman noktası doğruluğu** (point-in-time correctness) denir.

`users.country` gibi güncel bir değer geçmiş tarihli bir örneğe eklendiğinde, modele o tarihte var olmayan bir bilgi verilmiş olur. Buna **veri sızıntısı (leakage)** denir ve etkisi sinsidir: model eğitim ve doğrulama kümelerinde yüksek başarı gösterir, çünkü sızan bilgi her iki kümede de mevcuttur. Üretimde ise aynı bilgi yoktur — gelecek henüz yaşanmamıştır — ve model çöker. Ekip, doğrulama başarısına güvendiği için sorunun nerede olduğunu aylarca bulamaz.

Sızıntı en tehlikeli hâlini hedefle ilişkili alanlarda alır. "Bu müşteri terk edecek mi" modelinde `is_active` kolonunun güncel değeri kullanılırsa, model aslında cevabı okur. Doğruluk %99 çıkar ve model hiçbir şey öğrenmemiştir.

SCD2 bu hatayı yapısal olarak imkânsız kılar: boyuttan bir değer okumak için tarih vermek zorunludur, ve verilen tarihte geçerli olan sürüm döner. Özellik mağazalarının (feature store) "time travel" özelliği tam olarak bu mekanizmanın paketlenmiş hâlidir.

## Maliyeti

SCD2 bedava değildir. Boyut tablosu her değişiklikte büyür; sık değişen kolonlar ana boyutu şişirir. Çözüm **mini-boyut** kalıbıdır: hızlı değişen birkaç kolonu ayrı bir SCD2 tablosuna almak. Ödev 3.4'te fiyat için bunu uyguladım — `dim_product` SCD1 kaldı, fiyat ve maliyet `dim_product_price` tablosunda sürümlendi.

İkinci maliyet sorgu karmaşıklığıdır: her birleştirmeye tarih aralığı koşulu eklenir. Üçüncüsü, yanlış kurulan bir SCD2'nin sessizce satır çoğaltmasıdır — sürüm dönemleri çakışırsa aynı olgu birden çok boyut satırıyla eşleşir ve tüm toplamlar şişer. Bu yüzden çakışma kontrolü düzenli çalıştırılmalıdır; kendi uygulamamda dört tutarlılık kontrolü yazdım ve dördü de temiz çıktı.
