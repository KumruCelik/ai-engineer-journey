# Hafta 2 — Kontrol Soruları

## 1. `list` yerine `deque` kullanmam gereken bir senaryo, big-O gerekçesiyle

**Senaryo:** Bir API servisinde son 1000 isteğin yanıt süresini tutuyorum ve
her yeni istekte ortalamayı güncelliyorum. Yeni ölçüm sona ekleniyor, en eski
ölçüm baştan çıkıyor. Buna kayan pencere (sliding window) deniyor.

**`list` ile:**

```python
pencere.append(yeni_sure)
if len(pencere) > 1000:
    pencere.pop(0)      # ← sorun burada
```

`append` sabit sürede çalışıyor ama `pop(0)` doğrusal: listenin ilk elemanını
çıkarınca kalan 999 elemanın hepsi bir sola kayıyor. Yani her istekte 999
kaydırma yapılıyor.

**`deque` ile:**

```python
from collections import deque
pencere = deque(maxlen=1000)
pencere.append(yeni_sure)   # 1000'i aşınca en eskiyi kendisi atıyor
```

`deque` çift yönlü bağlı bir yapı olduğu için iki uçtan ekleme ve çıkarma sabit
sürede oluyor. Ayrıca `maxlen` sayesinde taşmayı elle yönetmem gerekmiyor.

**Ölçümüm:** 100.000 elemanlı yapıda `list.pop(0)` 0,035702 sn,
`deque.popleft()` 0,000076 sn — yaklaşık **470 kat** fark. Saniyede binlerce
istek alan bir serviste bu fark, gecikmenin kendisi olur.

**Ne zaman `deque` kullanmam:** Ortadan indeksleme yapıyorsam. `lst[5000]`
listede sabit sürede, deque'te doğrusal. Kuyruk davranışı yoksa `deque`
seçmenin bir faydası yok.



## 2. Bir decorator'da `functools.wraps` kullanmazsam ne kaybederim?

Decorator aslında fonksiyonu değiştirmiyor, **yerine başka bir fonksiyon koyuyor**.
`@sayac` yazmak `selam = sayac(selam)` demek; artık `selam` adı içerideki
`wrapper`'ı gösteriyor. `wraps` olmazsa orijinal fonksiyonun kimliği kayboluyor.

Kendi makinemde ölçtüm:

| | `__name__` | `__doc__` | imza |
|---|---|---|---|
| wraps'sız | `wrapper` | `None` | `(*args, **kwargs)` |
| wraps'lı | `selam_b` | korunmuş | `(ad: str) -> str` |

Somut kayıplar:

- `help(f)` işe yaramaz, docstring gitmiştir
- Hata izlerinde gerçek fonksiyon adı yerine `wrapper` görünür — hangi
  fonksiyonun patladığını bulmak zorlaşır
- `inspect.signature` ile imza okuyan araçlar çalışmaz. FastAPI rotaların
  parametrelerini, pytest fixture'ları, typer/click gibi CLI kütüphaneleri
  hep imzaya bakar; dekore edilmiş fonksiyon `(*args, **kwargs)` görünürse
  bu araçlar bozulur.
- IDE otomatik tamamlaması ve tip denetimi zayıflar

`wraps` bunları `__name__`, `__doc__`, `__module__`, `__qualname__`,
`__annotations__` ve `__wrapped__` alanlarını kopyalayarak çözüyor.
`__wrapped__` sayesinde orijinal fonksiyona da erişilebiliyor.

Kural: **her decorator'da `@wraps(f)` yaz.** Maliyeti bir satır, unutmanın
maliyeti hata ayıklarken kaybedilen saatler.

---

## 3. `Protocol` ile `ABC` arasındaki fark

**`ABC` isimsel (nominal) tipleme yapar:** bir sınıf, arayüzü sağladığını
**miras alarak** ilan eder. Kontrol çalışma anındadır — soyut metotlarını
doldurmayan bir sınıf örneklenemez, `TypeError` alırsın.

**`Protocol` yapısal (structural) tipleme yapar:** miras yoktur. Doğru
metotlara sahip her nesne arayüze uyar. Kontrol `mypy` tarafından, kodu
çalıştırmadan yapılır.

```python
class Source(Protocol):
    def oku(self) -> Iterator[Record]: ...

class CsvSource:                    # Source yazmiyor
    def oku(self) -> Iterator[Record]:
        ...

Pipeline(kaynak=CsvSource("g.csv"), ...)   # mypy memnun
```

`mini-etl`'de `Source` ve `Sink` için `Protocol` seçtim. Sebebi somut: bu bir
**kütüphane**. Kullanıcı kendi kaynağını yazarken benim modülümü import edip
sınıfımdan türemek zorunda kalmasın istedim. `oku()` metodu olan her nesne —
zaten var olan, benim varlığımdan habersiz bir sınıf bile — `Pipeline`'a
verilebiliyor.

`ABC` şu üç durumda daha doğru olurdu:

- **Ortak uygulama paylaşılacaksa.** `ABC` soyut olmayan metotlar da içerebilir;
  alt sınıflar onları miras alır. `Protocol` yalnızca şekil tarif eder.
- **Hiyerarşi senin kontrolündeyse.** Tüm alt sınıfları sen yazıyorsan mirasın
  maliyeti yok, kazancı var.
- **Çalışma anında sert garanti gerekiyorsa.** `ABC` eksik metotlu bir sınıfın
  **örneklenmesini** engeller.

`Protocol`ün bedeli de burada: varsayılan olarak `isinstance` ile
kullanılamaz, `@runtime_checkable` gerekir — ve o dekoratör bile yalnızca
**metot adlarına** bakar, imzalara değil. Yani `def oku(self, x, y, z)` yazan
bir sınıf `isinstance` kontrolünü geçer ama çağrıldığında patlar. Gerçek
garanti `mypy`'den gelir; `mypy` çalıştırmayan bir projede `Protocol`ün
koruması yoktur.

Özet: **`ABC` "sen benim çocuğumsun" der, `Protocol` "sen benim gibi
davranıyorsun" der.** Kütüphane sınırlarında ikincisi, uygulama içi
hiyerarşilerde birincisi.

---

## 4. 100 GB'lık CSV'yi 8 GB RAM'de nasıl işlerim?

Önce doğru soruyu sormak gerekiyor: **hangi işlem?** Cevap işleme göre
değişiyor, çünkü sığmayan şey girdi değil çoğu zaman **ara sonuç**.

| İşlem | Bellek davranışı |
| --- | --- |
| Filtreleme, eşleme | Sabit — akışla çözülür |
| Toplama (az sayıda grup) | Sabit — sayaç küçük kalır |
| Toplama (çok sayıda grup) | Grup sayısıyla büyür |
| Sıralama, birleştirme (join) | Veri boyutuyla büyür |

### Strateji 1 — Akış (streaming)

Dosyayı satır satır oku, işle, at. Aynı anda bellekte tek kayıt bulunur.

`mini-etl`'de bunu ölçtüm: 200.000 satırlık dosyada tepe bellek **0.23 MB**,
20.000 satırlıkta da **0.23 MB**. Veri 10 kat arttı, bellek değişmedi. Naif
yöntem (`readlines()`) aynı işte 108 MB kullanmıştı — `O(n)` ile `O(1)` farkı.

Uygun olduğu işler: filtreleme, eşleme, doğrulama, az gruplu toplama.
Sınırı: tek geçiş, rastgele erişim yok, sıralama yapılamaz.

### Strateji 2 — Parçalama (chunking) ve dış birleştirme

Dosyayı sabit boyutlu parçalara böl, her parçanın **kısmi sonucunu** hesapla,
sonra kısmi sonuçları birleştir. Klasik map-reduce.

`perf-lab`'de 1 GB'lık logu bayt aralıklarına bölüp dört ayrı süreçte saydım:
her süreç yalnızca 13 MB kullandı, sonuçlar `Counter.update` ile birleşti.
Duvar saati 60 saniyeden 30'a indi.

Ara sonuç da sığmıyorsa (çok sayıda grup, sıralama, join) parçalar diske yazılır
ve **dış sıralama/birleştirme** yapılır — veritabanlarının onlarca yıldır
kullandığı yöntem. `sort -S 1G` komutu bunu hazır yapar.

### Strateji 3 — Sütunlu format ve out-of-core motor

CSV'yi bir kez **Parquet**'e çevir, sonra `polars` veya `DuckDB` ile işle.
Üç kazanç:

- **Sütun budama:** 22 sütunun 3'ü lazımsa yalnızca o 3'ü okunur.
- **Sıkıştırma:** 100 GB CSV, Parquet'te tipik olarak 10-25 GB'a iner.
- **Yüklem itme (predicate pushdown):** filtre okuma sırasında uygulanır;
  elenen satırlar belleğe hiç girmez.

`polars`'ın `scan_*` API'si tembel çalışır ve akış (streaming) motoruyla
belleğe sığmayan veriyi işleyebilir. Ölçümümde `polars` 1 GB'lık CSV'yi 1.5
saniyede gruplayıp saydı — ama `collect()` ile 1849 MB kullandı. 100 GB için
akış modu şart; aksi hâlde en hızlı yöntem aynı zamanda belleği patlatan yöntem
olur.

### Dördüncü seçenek: Python'da hiç yapmamak

Tek seferlik bir iş için `sort`, `awk`, `join` gibi Unix araçları zaten dış
sıralama yapar ve akış hâlinde çalışır. Tekrarlayan bir iş için veriyi bir
veritabanına (DuckDB, ClickHouse) yükleyip SQL yazmak, elle boru hattı
yazmaktan hem hızlı hem sağlamdır.

**Karar sırası:** işlem akışla çözülüyorsa Strateji 1. Ara sonuç sığmıyorsa
Strateji 2. Aynı veriye defalarca farklı sorular sorulacaksa Strateji 3 —
dönüştürme maliyeti ilk sorguda amorti olur.
