# Veri işleme kodu nasıl test edilir?

*Ödev 2.5.2 — `mini-etl` yazarken kendi yaptığım hatalarla. ~1050 kelime.*

## Neden ayrı bir konu

Sıradan bir fonksiyonda test yazmak kolaydır: girdi belli, çıktı belli,
`assert topla(2, 3) == 5`. Veri işleme kodunda üç şey bunu bozar.

**Girdi kontrol edilemiyor.** Kodu sen yazıyorsun ama veriyi başkası
üretiyor — genellikle yıllar önce yazılmış, artık kimsenin bakmadığı bir
sistem. Sütun adı değişir, tarih biçimi kayar, boş alan `""` yerine `NULL`
yazan bir metin olur.

**Doğruluk belirsiz.** "Bu çıktı doğru mu?" sorusunun cevabı çoğu zaman
tek bir sayı değil, bir dizi özelliktir: satır sayısı korunmuş mu, hiçbir
kayıt sessizce düşmüş mü, tipler beklenen mi.

**Hatalar sessiz.** Bir web uygulamasında hata 500 döndürür ve birisi arar.
Veri boru hattında hata çoğu zaman **çalışmaya devam eder** ve yanlış sonuç
üretir. Hafta 1'de incelediğim suç verisinde `arrest` sütunu bazı satırlarda
kaymıştı; program hiç şikâyet etmemişti.

Bu yüzden veri kodunda test, "kod çalışıyor mu" sorusundan çok "veri
bozulmadan geçti mi" sorusunu sorar.

## Beş desen

### 1. Şema testi

Verinin **şeklini** doğrular: hangi sütunlar var, tipleri ne, hangileri boş
olabilir.

```python
def test_kaynak_beklenen_sutunlari_veriyor(ornek_csv):
    ilk = next(CsvSource(ornek_csv).oku())
    assert set(ilk) == {"id", "ad", "yas"}
```

Ucuz ama yüksek getirili: yukarı akıştaki bir sütun adı değişikliği, boru
hattının derinlerinde anlaşılmaz bir `KeyError`'a dönüşmeden önce burada
yakalanır. Şema testi olmayan bir boru hattında bu hata üretimde bulunur.

### 2. Değişmez (invariant) testi

Girdiye bakmadan **her zaman** doğru olması gereken bir ilişki yazılır.
`mini-etl`'de en değerlisi bir korunum yasasıydı:

```python
assert rapor.okunan == rapor.yazilan + rapor.reddedilen
```

Okunan her kayıt ya çıktıya gitmiş ya reddedilmiş olmalı; arada kaybolan
olmamalı. Bu iddia hiçbir örneğe bağlı değil, dolayısıyla kod değişse bile
geçerliliğini koruyor. Bir `continue` yanlış yere konursa anında kırılır.

### 3. Gidiş-dönüş (round-trip) testi

Yaz, geri oku, karşılaştır:

```python
CsvSink(yol).yaz(iter(kayitlar))
assert list(CsvSource(yol).oku()) == kayitlar
```

Tek testte **iki modülün birbiriyle uyumlu** olduğu kanıtlanıyor. Ayrı ayrı
doğru ama birbirine uymayan bir okuyucu/yazıcı çifti mümkündür — bu test onu
yakalar. Serileştirme yapan her yerde (CSV, JSON, Parquet, veritabanı) ilk
yazılacak testtir.

### 4. Özellik tabanlı (property-based) test

Örnekleri sen seçmezsin; verinin **şeklini** tarif edersin, kütüphane yüzlerce
örnek üretir — özellikle can sıkıcı olanları: boş dizeler, virgüller, tırnak
işaretleri, alfabe dışı karakterler.

```python
@given(kayitlar=kayit_listesi())
def test_yaz_oku_gidis_donus(kayitlar):
    ...
    assert geri == kayitlar
```

`hypothesis` bir hata bulduğunda durmaz; girdiyi kırpıp kırpıp **hâlâ hata
veren en küçük örneği** bulur (shrinking). 40 karakterlik karmaşık bir dize
yerine sana `""` gösterir.

Yan faydası: stratejiyi yazarken etki alanını açıkça tanımlamak zorunda
kalırsın. `mini-etl`'de "değerlerimde kontrol karakteri olmaz" varsayımı, ancak
strateji yazarken görünür hâle geldi.

### 5. Altın dosya ve regresyon testi

**Altın dosya (golden file):** beklenen çıktı bir dosyada tutulur, test
üretilen çıktıyı onunla karşılaştırır. Karmaşık çıktılarda (rapor, dönüştürülmüş
veri seti) elle `assert` yazmaktan pratiktir.

Tehlikesi: çıktı değişince "herhalde doğrudur" deyip altın dosyayı güncellemek.
O anda test, kodu değil kendini onaylar. Kural: **altın dosya güncellemesi ayrı
bir commit olmalı ve diff'i gözden geçirilmeli.**

**Regresyon testi:** üretimde bulunan her hata için önce onu yakalayan bir test
yazılır (kırmızı), sonra düzeltilir (yeşil). Aynı hata iki kez üretime çıkmaz.

## Testin kendisi nasıl yanlış olur

`mini-etl`'i yazarken beş test tasarım hatası yaptım. **Hiçbiri kırmızı
vermedi** — bu yüzden en pahalı grup onlardı.

**Test hiç çalışmıyordu.** Üç fonksiyonun adı `test_` ile başlamıyordu; pytest
onları sessizce atladı. Tek belirti `collected 1 item` satırıydı. *Her koşuda
toplanan test sayısını oku.*

**Beklenen değeri kodun mantığıyla hesaplamak.**

```python
assert list(cikti) == [k for k in girdi if int(k["yas"]) >= 18]
```

Sağ taraf, test edilen filtrenin elle tekrarı. Filtrede `>` yerine `<` yazılsa
iki taraf da aynı şekilde yanlış olur ve test geçer. *Beklenen değeri elle yaz;
veriyi biliyorsan sonucu da bilirsin.*

**Ayırt etmeyen veri.** Yaşlar `22, 26, 31`; filtre `>= 18`. Üçü de geçiyor,
yani filtre tamamen silinse çıktı aynı olurdu. *Sınır, verinin içinden geçmeli:*
`> 23` yapınca `okunan=3, yazilan=2` oldu ve iki sayının farklı olması testin
gerçekten ölçtüğünün kanıtı hâline geldi.

**Zayıf iddia.** `assert cikti != girdi` — "bir şey değişti" der, "doğru şey
değişti" demez. Dönüşüm bütün alanları silseydi de geçerdi.

**Adı ile işi uyuşmayan test.** `test_bos_kaynak_sifir_rapor_veriyor` üç
satırlık bir dosya kullanıyordu. Altı ay sonra okuyan, olmayan bir garantiye
güvenir.

Ortak sorusu şu: **bu testi geçirebilecek yanlış bir kod yazabilir miyim?**
Cevap evetse test yeterince ayırt edici değildir.

## Kapsam ne söyler, ne söylemez

`mini-etl`'de %100 satır kapsamına ulaştım. Aynı gün Dockerfile'ın **yanlış
modülü** çalıştırdığını buldum: imaj kütüphaneyi değil şablondan kalan örnek
kodu koşturuyordu. Testler yeşil, lint temiz, kapsam tam.

Kapsam tek bir şey söyler: *bu satır hiç çalışmadı*. Bu, gerçek bir sinyaldir —
`mini-etl`'de iki gerçek boşluk ortaya çıkardı (CLI'da iki bayrağın
zincirlenmesi, JSONL'de boş satır atlama). Ama "bu satır doğru davranıyor"
demez ve testlerin dışında kalan hiçbir şeyi (yapılandırma, dağıtım, şema)
kapsamaz.

Kapsamı bir hedef değil, bir **soru listesi** olarak kullanmak gerekiyor:
"burası neden hiç çalışmamış?"

## Test verisi nereden gelir

Üç kaynak var ve üçünün de sorunu var.

**Üretim verisinin kopyası** en gerçekçisidir ama kişisel veri taşır. KVKK ve
GDPR kapsamında bir test ortamına isim, adres veya kimlik numarası kopyalamak
ihlaldir. Maskeleme de tam çözüm değil: doğum tarihi, posta kodu ve cinsiyet
üçlüsü çoğu nüfusta tek bir kişiyi işaret eder — yeniden kimliklendirme
mümkündür. Ayrıca üretim verisi değişir; dünkü testin bugün geçmesi garanti
değildir.

**Elle yazılmış küçük örnekler** tekrarlanabilir ve okunabilirdir; testin ne
iddia ettiği bakınca anlaşılır. Ama yalnızca senin düşünebildiğin durumları
kapsar.

**Sentetik üretim** (`hypothesis` veya kendi üretecin) senin düşünmediklerini
de dener, ama gerçek verinin tuhaflıklarını — bozuk kodlama, kaymış sütun,
tarih yerine "N/A" — içermez.

Pratik karar: değişmezler için sentetik üretim, davranış için elle yazılmış
küçük örnekler, ve üretimde bulunan her bozuk satırın **temizlenmiş** hâli için
birer regresyon testi.

## Pratik kontrol listesi

Bir veri dönüşümü için test yazarken:

1. **Şema** — beklenen sütunlar ve tipler
2. **Boş girdi** — sıfır satırda patlamıyor mu
3. **Bozuk satır** — akış duruyor mu, yoksa kayıt reddedilip devam mı ediyor
4. **Korunum** — okunan = yazılan + elenen + reddedilen
5. **Gidiş-dönüş** — yazdığını geri okuyabiliyor musun
6. **Ayırt edici veri** — testin ölçtüğü davranışı gerçekten tetikliyor mu
7. **Sınır değerleri** — tam eşitlik, tek satır, tek sütun, en büyük değer
8. **Özellik testi** — değişmezleri yüzlerce rastgele girdide sınama

İlk altısı örnek tabanlı testlerle, sonuncusu `hypothesis` ile yazılır.
Sekizinin de ortak amacı aynı: **sessiz bozulmayı gürültülü hataya çevirmek.**

## Kaynaklar

- Kod ve testler: `mini-etl` (`tests/`, `DESIGN.md`)
- Hata defteri: Hafta 3 Saha Defteri, Bölüm 12
- `hypothesis` belgeleri — strateji yazımı ve shrinking
