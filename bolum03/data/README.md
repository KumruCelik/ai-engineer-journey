# Bölüm 3 — Veri kaynakları

Kural gereği veri dosyaları commit'lenmez; onları üreten komutlar commit'lenir.
Bu klasör, Bölüm 3'te kullanılan iki veri setinin nasıl elde edildiğini kaydeder.

## 1. Sentetik e-ticaret verisi (Ödev 3.1–3.4)

**Üretici:** `sql-mastery/scripts/generate.py`
**Çıktı:** `sql-mastery/data/*.csv` (gitignore'da)
**Yükleme:** `sql-mastery/seed/load.sql` (`\copy` ile)

Sıfırdan kurup doldurmak için:

```bash
cd ~/projects/sql-mastery && make seed
```

Üretilen hacim: 685.966 satır — 20.000 kullanıcı, 2.000 ürün, 100.000 sipariş,
168.920 sipariş kalemi, 164.012 stok hareketi, 108.094 ödeme, 77.999 kargo,
26.810 yorum, 21.008 sipariş-kupon bağı, 50 kupon, 40 kategori.

Üreticinin rastgelelik tohumu sabittir (`random.seed(42)`), dolayısıyla aynı
komut her çalıştırmada aynı veriyi üretir.

Bilinen üretici kusurları ve düzeltme notları `notes/odev-3.2/cevaplar.md`
içinde ilgili soruların altında kayıtlıdır.

## 2. NYC TLC sarı taksi verisi (Ödev 3.5)

**Kaynak:** NYC Taxi & Limousine Commission, açık veri
**Dosya:** 2024 Ocak, sarı taksi yolculukları
**Boyut:** 47,6 MB (Parquet) · 2.964.624 satır · 19 kolon

```bash
mkdir -p ~/projects/sql-mastery/data
curl -L -o ~/projects/sql-mastery/data/yellow_2024_01.parquet \
  https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet
```

Veri kalitesi bulguları `notes/odev-3.5/duckdb_analizi.md` içindedir.
