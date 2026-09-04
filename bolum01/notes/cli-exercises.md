# Ödev 1.2 — CLI Veri İşleme

Veri: bkz. `../data/README.md`
Kural: Python yok, sadece terminal araçları.

---

## Görev 1 — Satır sayısı

**Komut:**

```
wc suclar.csv
```

**Çıktı:**

```
299889  1015913  28961166  suclar.csv
satır   kelime   bayt
```

Dosyada 299.889 satır var. Ama ben veriyi indirirken 100.000 kayıt istemiştim.
Aradaki fark dikkatimi çekti, o yüzden gerçek kayıt sayısını ayrıca saydım:

```
grep -c -E '^"[0-9]+","' suclar.csv    → 100000
grep -c -E ',,,$' suclar.csv           → 56
```

**Doğrulama:**

Dosyaya baktığımda bazı kayıtların 3 satıra yayıldığını gördüm. Koordinatı
olmayanlar tek satır tutuyor. Buradan iki denklem kurdum:

```
299889 = 1 (başlık) + 3T + 1S
100000 = T + S
→ T = 99.944,  S = 56
```

Önce kâğıtta S = 56 çıkardım, sonra komutla ölçtüm: yine 56. Tahminim tuttu,
yani kayıtların yapısını doğru anlamışım.

**Ne öğrendim:**

`wc -l` sadece satır sonu karakterlerini sayıyor. Bu dosyada `location` alanı
tırnak içinde satır sonu barındırdığı için bir kayıt üç satıra yayılabiliyor,
ve `wc -l` üç kat fazla sayıyor. Yani hızlı ama bu tür dosyalarda güvenilmez.

---

## Görev 2 — Frekans tablosu

**3. kolon (date):**

```
grep -E '^"[0-9]+","' suclar.csv | cut -d',' -f3 | sort | uniq -c | sort -rn | head -20
```

```
51 "2026-04-01T00:00:00.000"
47 "2026-06-01T00:00:00.000"
38 "2026-05-01T00:00:00.000"
37 "2026-07-21T00:00:00.000"
37 "2026-07-01T00:00:00.000"
35 "2026-07-24T00:00:00.000"
34 "2026-05-26T00:00:00.000"
33 "2026-05-19T00:00:00.000"
31 "2026-07-31T00:00:00.000"
31 "2026-07-22T00:00:00.000"
...
```

**Bu çıktı işe yaramadı.** İki sebebi var.

Birincisi, 100.000 kayıt içinde en sık geçen tarih sadece 51 kez geçiyor.
Yani neredeyse her kaydın kendi tarihi var. Böyle bir kolondan anlamlı bir
frekans tablosu çıkmıyor. Tarih zaten bir kategori değil; işe yaraması için
yıl, ay, gün gibi parçalara ayırmak gerekiyor.

İkincisi ve daha ilginci: listedeki 20 tarihin hepsi gece yarısı
(`T00:00:00.000`). Chicago'da suçlar gece 00:00'da işlenmiyor tabii ki.
Anlaşılan saati bilinmeyen kayıtlar 00:00 olarak yazılmış. Aynı şekilde ayın
1'i listenin tepesinde — "nisan içinde bir ara oldu" denen kayıtlar 1 Nisan'a
yazılmış. Bunlar gerçek zaman değil, boş kalmasın diye konmuş değerler.

**6. kolon (primary_type):**

```
grep -E '^"[0-9]+","' suclar.csv | cut -d',' -f6 | tr -d '"' | sort | uniq -c | sort -rn | head -20
```

```
21062 THEFT
18787 BATTERY
11079 CRIMINAL DAMAGE
 9119 ASSAULT
 8025 MOTOR VEHICLE THEFT
 6605 BURGLARY
 6583 OTHER OFFENSE
 5209 DECEPTIVE PRACTICE
 2677 NARCOTICS
 2354 CRIMINAL TRESPASS
 2344 WEAPONS VIOLATION
 2038 ROBBERY
  714 CRIMINAL SEXUAL ASSAULT
  644 OFFENSE INVOLVING CHILDREN
  614 SEX OFFENSE
  600 PUBLIC PEACE VIOLATION
  441 INTERFERENCE WITH PUBLIC OFFICER
  282 STALKING
  195 HOMICIDE
  154 ARSON
```

**Ne öğrendim:**

Anlamlı bir frekans tablosu, az sayıda farklı değeri olan bir kolonda çıkıyor.
Burada 20 suç tipi tüm kayıtların %99,5'ini kapsıyor. İlk 4 tip tek başına
%60, ilk 8 tip %86. Geri kalanı çok küçük parçalara dağılmış.

En dikkat çeken şey aradaki uçurum: THEFT 21.062, HOMICIDE 195. Yaklaşık 137
kat fark. Şöyle düşündüm: "bu kayıt cinayet mi?" diye soran bir model her
seferinde "hayır" dese %99,8 doğru çıkardı. Hiçbir şey öğrenmeden. Demek ki
böyle dengesiz verilerde doğruluk oranına bakmak yanıltıcı.

Bir de yöntem olarak şunu fark ettim: ne gece yarısı meselesini ne de bu
dengesizliği ilk üç satıra bakarak görebilirdim. İkisi de ancak tüm dağılımı
sayınca ortaya çıktı.

---

## Görev 3 — Koşullu süzme

**Komut:**

```
grep HOMICIDE suclar.csv > homicide.csv
```

**Kontrol:**

```
wc -l homicide.csv                       → 195
grep -c -E '^"[0-9]+","' homicide.csv    → 195
```

Görev 2'de de HOMICIDE 195 çıkmıştı. Üç sayı da aynı, "tamam" diyecektim.

**Ama dosya bozuk çıktı.**

```
head -1 homicide.csv | tail -c 60

,"2026-08-21T15:49:54.000","41.765078582","-87.609205731","
```

Satır açık bir tırnakla bitiyor. Sebebini anladım: `grep` "HOMICIDE" kelimesini
arıyor, o kelime kaydın sadece ilk satırında geçiyor. Kaydın devam satırlarında
o kelime olmadığı için onlar atılmış. Yani her kayıttan sadece üçte biri kalmış,
kapanış tırnağı da onlarla birlikte gitmiş.

Bu dosyayı bir CSV okuyucuya versem, tırnağı hâlâ açık gördüğü için bir sonraki
kaydı o alanın içine katardı.

**Ne öğrendim:**

İki sayının uyuşması doğru oldukları anlamına gelmiyor. `wc -l` ve `grep -c`
aynı cevabı verdi, çünkü ikisi de aynı yanlış varsayımdan yola çıkıyordu:
"bir kayıt bir satırdır". Aynı varsayımı paylaşan iki ölçüm birbirini
doğrulamıyor, sadece tekrarlıyor. Hatayı ancak dosyanın kendisine bakınca
gördüm.

---

## Görev 4 — join

**Komutlar:**

```
grep -E '^"[0-9]+","' suclar.csv | cut -d',' -f1,6 | LC_ALL=C sort -t',' -k1,1 > a.csv
grep -E '^"[0-9]+","' suclar.csv | cut -d',' -f1,9 | LC_ALL=C sort -t',' -k1,1 > b.csv
LC_ALL=C join -t',' -1 1 -2 1 a.csv b.csv > birlesik.csv
```

`join` iki dosyayı baştan sona tek seferde okuduğu için ikisinin de anahtara
göre sıralı olması gerekiyor. `LC_ALL=C` yazmam da gerekti; yoksa `sort` ile
`join` farklı sıralama kuralları kullanıp hata veriyor.

Bir de fark ettim ki `join` eşleşmeyen anahtarları sonuçtan atıyor. Sonradan
öğrendim, SQL'de buna INNER JOIN deniyormuş.

**Doğrulama:**

`arrest` kolonunda sadece true/false olmalı. Kontrol ettim:

```
cut -d',' -f2 b.csv | sort | uniq -c | sort -rn | head -20
```

```
83579 "false"
14367 "true"
 1604  FEET
  220  BIKE WITH VIN"
   44  UBER
   39 "COMMERCIAL / BUSINESS OFFICE"
   38  MOTOR HOME"
   25  BIKE NO VIN"
   18 "STREET"
   ...
```

83.579 + 14.367 = 97.946 doğru. Geriye 2.054 bozuk kayıt kalıyor, yani %2.

**Sebep:**

7. kolonda (`description`) bazı değerlerin içinde virgül var:
`"AGGRAVATED - HANDS, FISTS, FEET, NO / MINOR INJURY"`. `cut -d','` tırnakları
tanımadığı için bu virgülleri de ayıraç sayıyor. O kayıtlarda kolonlar kayıyor
ve 9. kolondan `arrest` yerine alakasız bir şey geliyor.

A dosyası bu yüzden sağlam kaldı: 6. kolon, virgüllü kolondan önce geliyor.

**Neden ciddi:**

Bozulan kayıtlar rastgele değil. Çöp değerlere bakınca nereden geldikleri
belli: "FEET" BATTERY'den, "BIKE WITH VIN" MOTOR VEHICLE THEFT'ten geliyor.
Yani hata belli suç tiplerinde toplanmış, THEFT'e neredeyse hiç dokunmamış.

Bu veriden tutuklama oranı hesaplasam 14.367 / 97.946 = %14,67 bulurdum. Ama
hesaba katmadığım 2.054 kaydın çoğu BATTERY. Onların tutuklama oranı farklıysa
bulduğum sayı yanlış olur — üstelik hangi yönde yanlış olduğunu da bilemem.

En rahatsız edici tarafı: `cut` hata vermedi, uyarı da vermedi. Sessizce yanlış
cevap verdi.

---

## Beş bulgu

1. **Satır ile kayıt aynı şey değil.** CSV alanları tırnak içinde satır sonu
   barındırabiliyor. `wc -l` 299.889 dedi, gerçek kayıt sayısı 100.000'di.

2. **Tarih kolonu kullanılabilir değildi.** Neredeyse her kaydın kendi tarihi
   vardı, üstelik listenin tepesindeki değerlerin hepsi "bilinmiyor" yerine
   konmuş gece yarısı damgalarıydı.

3. **Veri çok dengesiz.** THEFT 21.062, HOMICIDE 195. Böyle bir veride doğruluk
   oranı hiçbir şey ifade etmiyor.

4. **`grep` çok satırlı kaydı ortadan kesiyor.** Sonuç geçersiz bir CSV oluyor,
   ama sayılar uyuştuğu için sağlam görünüyor.

5. **`cut` CSV bilmiyor, sadece virgülden bölüyor.** Alan içi virgüllerde veri
   sessizce bozuluyor — burada kayıtların %2'si.

---

## Terminal araçlarının sınırı

`grep`, `cut`, `sort`, `uniq` satır satır çalışıyor. Dosyanın formatını
bilmiyorlar, sadece karakterlere bakıyorlar. Bu onları çok hızlı yapıyor;
28 MB'lık dosyada hepsi saniyeler içinde cevap verdi.

Ama aynı sebepten, tırnak ve alan kuralları olan dosyalarda güvenilmiyorlar.
Bu ödevde yaşadığım üç hatanın da kökeni aynıydı: araç tırnağı tanımıyor.

Kendi kuralım:

| Durum | Ne kullanırım |
|---|---|
| Hızlı bakma, sayma, log arama | `grep`, `cut`, `sort`, `uniq`, `wc` |
| İçinde virgül/tırnak olabilecek CSV | `duckdb`, `csvkit`, `miller` |
| Asıl analiz ve dönüştürme | pandas |

Ama en çok şunu öğrendim: hangi aracı seçtiğimden çok, çıktıya inanmadan önce
başka bir yoldan kontrol etmem önemli. Bu ödevdeki dört hatayı da komut yazarak
değil, çıkan sonucu sorgulayarak buldum.
