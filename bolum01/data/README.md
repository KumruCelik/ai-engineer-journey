# Veri Seti — Chicago Suç Kayıtları

## Kaynak
Chicago şehri açık veri portalı (Socrata SODA API), `ijzp-q8t2` veri seti.
https://data.cityofchicago.org/resource/ijzp-q8t2.csv

## İndirme komutu

    curl -o suclar.csv "https://data.cityofchicago.org/resource/ijzp-q8t2.csv?\$limit=100000"

`$limit=100000` ile 100.000 kayıt çekildi. Veri setinin tamamı ~8 milyon kayıt;
bu ödev için gereksiz büyük.

Not: `\$` kaçış karakteri şart — kaçırılmazsa bash `$limit`'i değişken sanıp siler.

## Dosya özellikleri

| Özellik | Değer |
|---|---|
| Boyut | 28 MB |
| Kolon sayısı | 22 |
| Kayıt sayısı | 100.000 |
| **Satır sayısı** | **299.889** |

Satır ve kayıt sayısının farklı olması tesadüf değil: `location` kolonu tırnak
içinde satır sonu barındırıyor, yani bir kayıt 3 satıra yayılabiliyor.
Koordinatı olmayan 56 kayıt tek satır tutuyor. Doğrulama:

    1 (başlık) + 3 × 99.944 + 1 × 56 = 299.889

## Kolonlar

    1  id                 7  description            13 ward            19 updated_on
    2  case_number        8  location_description   14 community_area  20 latitude
    3  date               9  arrest                 15 fbi_code        21 longitude
    4  block              10 domestic               16 x_coordinate    22 location
    5  iucr               11 beat                   17 y_coordinate
    6  primary_type       12 district               18 year

## Bu veri seti neden seçildi

Ödevin amacı, terminal araçlarının büyük veride neden vazgeçilmez olduğunu
görmekti. 1000 satırlık bir dosyada `grep | sort | uniq -c` ile Excel arasında
pratik bir fark yok. Fark, dosya Excel'i kilitleyecek boyuta geldiğinde
ortaya çıkıyor: 28 MB terminal araçları için hiçbir şey, Excel için ağır.

Ayrıca veri gerçekten kirli — eksik koordinatlar, alan içi virgüller,
tırnak içi satır sonları, gece yarısına yazılmış sentinel tarihler. Temiz bir
örnek veri setinde bunların hiçbiri öğrenilemezdi.

## Veri neden repoda yok

`.gitignore` içinde `*.csv` ve `data/` satırları var. İndirilebilir olan şey
versiyonlanmaz; repoda veri yerine **veriyi üreten komut** durur. Bu dosyayı
okuyan herkes (6 ay sonraki ben dahil) yukarıdaki `curl` komutuyla aynı veriyi
elde edebilir.
