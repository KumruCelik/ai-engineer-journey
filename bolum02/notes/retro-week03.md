# Hafta 3 Retro — mini-etl

## Ne yapıldı

Sıfırdan bir akış tabanlı ETL kütüphanesi: `mini-etl`.

Sıra **dikey dilim** yöntemiyle kuruldu — katman katman değil, her aşamada
çalışan bir şey olacak şekilde:

1. **Aşama 1:** `CsvSource` → `esle`/`filtrele` → `CsvSink` → `Rapor`, uçtan uca
2. **Aşama 2:** hata yalıtımı + dead letter dosyası
3. **Aşama 4:** `argparse` tabanlı CLI
4. **Aşama 5:** bellek ölçümü + README

Aşama 3 (ek format ve kaynaklar) gerekçesiyle kesildi — DESIGN.md, Karar 6.

## Sayılar

| | |
| --- | --- |
| Test | 33 |
| Modül | 5 (`record`, `source`, `transform`, `sink`, `pipeline`) + `cli` |
| Çekirdek bağımlılık | 0 (yalnızca stdlib) |
| Tepe bellek, 200.000 satır | 0.23 MB (naif yöntem: 108 MB) |

## Bu hafta yapılan hatalar ve çıkarılan dersler

### Dil ve sözdizimi

| Hata | Ders |
| --- | --- |
| `ilk.ad` — sözlükte nokta ile erişim | `Record = dict` kararının somut bedeli. DESIGN.md bunu öngörmüştü. |
| `iter[])` — köşeli parantez ve fazla parantez | `iter` bir fonksiyon; `SyntaxError` dosyanın hiç okunmadığı anlamına gelir, `collected 0 items` bunun işareti. |
| `donusum== lambda ...` | Fonksiyon çağrısında `=` isimlendirir, `==` karşılaştırır. Parantez içinde `==` görürsen şüphelen. |
| `isistance` | Yazım hatası; `NameError` en ucuz hata türü. |
| `NameError: 'ilk'`, `NameError: 'yol'` (iki kez) | Her fonksiyon kapalı bir oda. Test kopyaladıktan sonra aşağıdan yukarı oku. |

### Test yazmayla ilgili

| Hata | Ders |
| --- | --- |
| Üç fonksiyon `test_` ile başlamıyordu | pytest onları sessizce atladı. `collected N items` sayısını her zaman kontrol et. |
| "Boş akış" testi dolu akış veriyordu | Testin adı ile yaptığı iş uyuşmalı. |
| Beklenen değer aynı mantıkla hesaplanmıştı | Kod ile kodu test etmek. Beklenen değeri **elle** yaz. |
| `assert cikti != girdi` | `!=` "bir şey değişti" der, "doğru şey değişti" demez. |
| Filtre sınırı verinin dışında (`>= 18`, yaşlar 22/26/31) | Test ayırt edici veri ister; hiçbir kaydı elemeyen filtre testi hiçbir şey ispatlamaz. |

### Araçlar

| Hata | Ders |
| --- | --- |
| `F811` — `Transform` iki kez tanımlanmış | Python bunu hata saymaz, ikinci tanım birinciyi sessizce ezer. Linter, dilin yakalamadığını yakaladı. |
| `F841` — kullanılmayan değişken | Genelde bir yazım hatasının işareti. |
| Bash: `feat(core)!:` → `unrecognized history modifier` | `!` çift tırnak içinde bile özeldir. Metin **tek tırnak** ister. |
| `ruff format` markdown içindeki kod bloklarını da biçimlendiriyor | `make lint` yerelde CI ile birebir aynı olmalı. |
| `"elif".upper()` → `ELIF` (iki kez) | Hafta 2'de yazdığım 18 numaralı gotcha, bu hafta iki kez yaşandı. |

## Öngörülen vs gerçekleşen

- **Öngörülen:** `Transform.__call__` imzasının `(akis, rapor)` olması gerekeceği
  (Karar 4). Aşama 2'de aynen gerçekleşti; 5 test kırıldı ve tam olarak
  dokunulan yerde kırıldı.
- **Öngörülemeyen:** `Transform`a `ad` alanı gerektiği. Bir kaydın hangi adımda
  reddedildiğini bilmeden hata ayıklamak mümkün değil.
- **Ölçülen:** Akış iddiası. Veri 10 kat arttığında akışın belleği hiç
  değişmedi. İddia artık ölçüme dayanıyor.

## Neyi farklı yapardım

_(kendi cevaplarını yaz)_

1. `Record = dict[str, Any]` kararı bu hafta sana neye mal oldu, ne kazandırdı?
   Baştan alsan aynı kararı verir miydin?

Bedeli somut oldu: testleri yazarken `ilk.ad` yazıp `AttributeError` aldım, ve
alan adını yanlış yazdığımda mypy beni uyarmadı — hata çalışma anında `KeyError`
olarak çıktı. Kazancı da somut: `sec`, `yeniden_adlandir` ve `tip_cevir` üçü de
onar satır, çünkü hepsi sözlük işlemi; `dataclass` olsaydı çalışma anında sınıf
üretmem gerekirdi.

Kütüphane için aynı kararı yine verirdim, çünkü şemayı ben değil veri
belirliyor. Ama şemanın bilindiği bir işte — tek bir tablodan okuyan bir boru
hattında — `TypedDict` seçerdim: aynı sözlük davranışını korur, üstüne alan
adlarını mypy'ye denetletir.

2. Testi önce yazmak mı, kodu önce yazmak mı daha çok işine yaradı? Hangi
   durumda hangisi?

İkisi farklı işe yaradı. Kodu önce yazdığım Aşama 1'de beş test tasarım hatası
yaptım ve hiçbiri kırmızı vermedi — testin kendisi yanlışsa yeşil renk bir şey
ifade etmiyor.

Testlerin zaten var olduğu Aşama 2'de ise `Transform.__call__` imzasını
değiştirdim: tam beş test kırıldı ve tam dokunduğum yerde kırıldı. Elle arama
yapmadım, test çıktısı bana eksiksiz bir yapılacaklar listesi verdi.


3. `>>` operatörünü aşırı yüklemek (`filtrele(...) >> esle(...)`) okunabilirliği
   artırdı mı, yoksa öğrenmesi gereken yeni bir kural mı ekledi?

Okunabilirliği artırdı: `filtrele(...) >> esle(...)` verinin aktığı yönde,
soldan sağa okunuyor; alternatifi olan iç içe çağrı tersten okunuyor. Bu hafta
`>>` yüzünden bir tane bile hata almadım.

Ama bedava değil. `>>` Python'da normalde bit kaydırmadır — okuyucuya öğrenmesi
gereken yeni bir kural ekliyorum. Ayrıca `rapor`u zincir boyunca taşımak için
`__rshift__` içine ayrı bir iç fonksiyon yazmam gerekti; `lambda` yetmedi.


4. Aşama 3'ü kesme kararını bir kod incelemesinde savunman istense ne derdin?

Kapsamı kestim çünkü sınırlı vakitte genişlik yerine derinlik daha değerliydi; gerekçeyi kod yazmadan önce belgeye yazdım, ve kesmenin borç 
yaratmadığının kanıtı bir hafta sonra üç sınıfı çekirdeğe dokunmadan eklemem oldu.



## Sonraki haftaya devredenler"
desen = r"## Neyi farklı yapardım\n[\s\S]*?
assert re.search(desen, metin), "bolum bulunamadi"
metin = re.sub(desen, yeni, metin, count=1)

metin = metin.rstrip() + """

### Durum — Hafta 4 sonu

Yukarıdaki liste Hafta 3 bittiğinde neyin eksik olduğunun kaydıdır; olduğu gibi
duruyor. Maddelerin tamamı Hafta 4'te tamamlandı:

| Madde | Nerede |
| --- | --- |
| Tuzak koleksiyonu 40 maddeye | `bolum02/notes/python-gotchas.md` |
| Kontrol soruları 3 ve 4 | `bolum02/notes/week02-answers.md` |
| GIL araştırması | `bolum02/research/gil.md` |
| Veri kodu testi araştırması | `bolum02/research/veri-kodu-testi.md` |
| `dev-setup` kusur geri bildirimi | PR ile birleştirildi |
| `tests/test_main.py` ölü kodu | silindi |
| Ödev 2.2 kalan kalemleri | `mini-etl` — `SqliteSink`, `StdoutSink`, `HttpSource`, logging, YAML |
| Ödev 2.3 ve 2.4 | `perf-lab` — `LOG.md`, `ASYNC.md` |
