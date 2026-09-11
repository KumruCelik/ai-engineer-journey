# Ödev 3.2 — 50 iş sorusu

SQL dosyaları: `sql-mastery/queries/qNN_kisa_ad.sql`
Her soru için gereken: Türkçe soru cümlesi, SQL, sonuç çıktısı, 1 paragraf iş yorumu.

Durum: [ ] açık · [x] kapalı

---

## A. Seviye 1 — tek tablo, filtre ve hesap

- [x] 01 — Hesabı kapalı (`is_active = false`) kullanıcılar kimler?
- [x] 02 — Ülkesi bilinmeyen kullanıcı sayısı kaç? (NULL sayma tuzağı)
- [x] 03 — Liste fiyatı 500'ün üzerindeki ürünler, pahalıdan ucuza.
- [x] 04 — Her ürünün marjı ve marj yüzdesi nedir?
- [x] 05 — Stoğu tükenmiş ama hâlâ aktif görünen ürünler hangileri?
- [x] 06 — Son 30 günde açılan hesap sayısı kaç?
- [x] 07 — Siparişler durumlarına göre nasıl dağılıyor?
- [x] 08 — Ürünleri fiyat bandına göre sınıflandır (ucuz / orta / pahalı).
- [x] 09 — Haftanın hangi gününde en çok sipariş veriliyor?
- [x] 10 — Geçerlilik süresi dolmuş kupon kodları hangileri?

## B. Seviye 2 — birleştirme ve gruplama

- [x] 11 — Her kullanıcının sipariş sayısı ve toplam harcaması.
- [x] 12 — Hiç sipariş vermemiş kullanıcılar kimler?
- [x] 13 — Kategori bazında toplam ciro.
- [x] 14 — Adet bazında en çok satan 10 ürün.
- [x] 15 — Ciro bazında en çok satan 10 ürün. 14 ile farkı ne anlatıyor?
- [x] 16 — Sipariş başına ortalama sepet tutarı.
- [x] 17 — Sipariş başına ortalama kalem sayısı.
- [x] 18 — Ödemesi başarısız olan sipariş oranı.
- [x] 19 — Funnel: created → paid → shipped → delivered dönüşüm oranları. (zorunlu)
- [x] 20 — İptal edilen siparişlerdeki ürünler: sepete girip satılmayanlar. (zorunlu)
- [x] 21 — Kupon kullanımının marj üzerindeki etkisi. (zorunlu)
- [x] 22 — Ürün başına ortalama puan ve yorum sayısı. (çok yorumlu kullanıcı tuzağı)
- [x] 23 — Hiç yorum almamış ürünler.
- [x] 24 — Ülke bazında sipariş sayısı ve ciro.
- [x] 25 — Kargo firması bazında ortalama teslim süresi.

## C. Seviye 3 — CTE ve window function

- [ ] 26 — Kohort retention tablosu: yeni kullanıcıların N. ay dönüş oranı. (zorunlu)
- [ ] 27 — RFM segmentasyonu (NTILE ile). (zorunlu)
- [ ] 28 — Ürün başına 7 günlük hareketli ortalama satış. (zorunlu)
- [ ] 29 — Ardışık günlerde alışveriş yapan kullanıcı serileri (gaps-and-islands). (zorunlu)
- [ ] 30 — Her kategoride ilk 3 ürün (ROW_NUMBER + PARTITION BY). (zorunlu)
- [ ] 31 — Ay bazında büyüme oranı (LAG). (zorunlu)
- [ ] 32 — İlk sipariş ile ikinci sipariş arasındaki medyan süre. (zorunlu)
- [ ] 33 — Aynı kullanıcının aynı ürünü tekrar alma oranı. (zorunlu)
- [ ] 34 — Zaman içinde fiyat değişimi: SCD2 tablosundan geçerli fiyatı bulma. (zorunlu, Ödev 3.4'ten sonra)
- [ ] 35 — Kullanıcı-ürün başına en son yorum (DISTINCT ON).
- [ ] 36 — Kümülatif ciro seyri (SUM OVER).
- [ ] 37 — Kategori ağacında bir kökün altındaki tüm ürünler (recursive CTE).
- [ ] 38 — Her kullanıcının ilk siparişinde aldığı ürünler.
- [ ] 39 — Kullanıcı başına sipariş aralıklarının ortalaması.
- [ ] 40 — En sık birlikte satın alınan ürün çiftleri (self join).
- [ ] 41 — Her ayın en iyi 3 müşterisi (RANK).
- [ ] 42 — Kullanıcı yaşam boyu değeri ve ilk 90 günün payı.
- [ ] 43 — Stok defterinden gün gün stok seyri (kümülatif toplam).
- [ ] 44 — Ciroda ilk %20 müşterinin payı (Pareto).
- [ ] 45 — Terk oranı: daha önce sipariş vermiş ama son 90 gündür vermeyen kullanıcılar.

## D. Veri kalitesi

- [ ] 46 — `products.stock_cached`, defter toplamına eşit mi?
- [ ] 47 — Ödeme toplamı sipariş toplamına eşit mi?
- [ ] 48 — `order_items.unit_price` ile `products.list_price`'ın saptığı satırlar. (sapma beklenen — neden?)
- [ ] 49 — Durumu `shipped` olup kargo kaydı bulunmayan siparişler.
- [ ] 50 — `inv_order_link` kuralına aykırı satır var mı? Kısıt olmasaydı bunu nasıl arardık?
