# Python Tuzak Koleksiyonu

Format: kod → tahminim → gerçek → neden → kural.
Hedef: 40 madde.

---

## 1. Mutable default argüman

```python
def ekle(x, liste=[]):
    liste.append(x)
    return liste
```

**Gerçek:** Ardışık çağrılar `[1]`, `[1,2]`, `[1,2,3]` döndürüyor — liste sıfırlanmıyor.
Üçünü tek `print`'te çağırırsam üçü de `[1,2,3]` görünüyor, çünkü **hepsi aynı nesne**.

**Neden:** Varsayılan değer fonksiyon **tanımlandığında bir kez** oluşturulur, her çağrıda değil.

**Kural:** Varsayılan olarak `None` yaz, gövdede oluştur:
```python
def ekle(x, liste=None):
    if liste is None:
        liste = []
```

---

## 2. Liste çarpımı iç nesneyi kopyalamaz

```python
a = [[0] * 3] * 3
a[0][0] = 1  # → [[1,0,0], [1,0,0], [1,0,0]]
```

**Neden:** `* 3` aynı iç listeye **üç referans** üretiyor, üç ayrı liste değil.

**Kural:** `[[0]*3 for _ in range(3)]` — comprehension her turda yeni liste kurar.

---

## 3. `is` vs `==` ve sayı önbelleği

```python
x = 256
y = 256  # x is y → True
p = 257
q = 257  # dosyada True, REPL'de False
```

**Neden:** CPython −5..256 arası tam sayıları önbellekte tutar. 257 önbellek dışında; ama
dosya olarak çalıştırıldığında derleyici aynı kod nesnesindeki eşit sabitleri tekilleştirir.
REPL'de her satır ayrı kod nesnesi olduğu için tekilleştirme olmaz.

**Kural:** Sayı ve string'de **asla `is` kullanma**. `is` sadece `None`, `True`, `False`
ve "gerçekten aynı nesne mi" sorusu için.

---

## 4. `copy` yüzeysel, `deepcopy` derin

```python
s = copy.copy(orj)  # iç listeler PAYLAŞILIR
d = copy.deepcopy(orj)  # iç listeler de kopyalanır
```

**Kural:** İç içe yapı varsa `deepcopy`. Ama pahalıdır — büyük veride önce
"gerçekten kopya mı lazım?" diye sor.

---

## 5. Tuple'ın değişmezliği yüzeyseldir

```python
t = (1, [2, 3])
t[1].append(4)  # → (1, [2, 3, 4])
```

**Neden:** Tuple **kendi elemanlarına olan referansları** dondurur, o nesnelerin içeriğini değil.

**Kural:** Tuple'ı hashlenebilir sanma — içinde liste varsa `dict` anahtarı olamaz.

---

## 6. Parametreye yeniden atama dışarıyı etkilemez

```python
def temizle(liste):
    liste = []  # sadece yerel adı yeniden bağlar
    # liste.clear() olsaydı dışarıyı DEĞİŞTİRİRDİ
```

**Kural:** Python'da "referansla mı değerle mi" sorusunun cevabı burada:
nesneyi **değiştirmek** dışarıya yansır, adı **yeniden bağlamak** yansımaz.

---

## 7. Modül adı stdlib'i gölgeliyor

`collections.py` veya `typing.py` adında bir dosya, stdlib modülünü gölgeler.
`from collections import Counter` senin dosyanı bulmaya çalışır; mypy de bozulur.

**Kural:** Modül adı koymadan önce `python3 -c "import <ad>"` ile stdlib'de var mı bak.

---

## 8. Anahtar kelime modül adı olamaz

`async.py` dosyası oluşturulabilir ama `import py_core.async` **sözdizimi hatası** verir.

**Kural:** `async`, `class`, `import`, `lambda`, `from`, `is`, `not` — hiçbiri modül/değişken adı olamaz.
Liste: `python3 -c "import keyword; print(keyword.kwlist)"`

---

---

## 9. Yanlış koleksiyon seçimi kodu yüzlerce kat yavaşlatıyor

Kendi makinemde ölçtüm (`timeit`, N = 100.000 eleman, 1000 tekrar):

| İşlem | Süre (sn) | Karşılaştırma |
|---|---|---|
| `lst.append(x)` | 0,000030 | — |
| `lst.insert(0, x)` | 0,022330 | append'e göre **744× yavaş** |
| `lst.pop()` | 0,000025 | — |
| `lst.pop(0)` | 0,035702 | pop()'a göre **1.428× yavaş** |
| `dq.appendleft(x)` | 0,000020 | `list.insert(0)`'a göre **1.117× hızlı** |
| `dq.popleft()` | 0,000076 | `list.pop(0)`'a göre **470× hızlı** |
| `x in lst` | 0,651437 | — |
| `x in st` | 0,000027 | listeye göre **24.127× hızlı** |
| `d[k]` | 0,000062 | — |

**Neden:**

Liste bellekte ardışık bir blok. Sona eklemek boş yere yazmak demek, sabit süre.
Başa eklemek ise geri kalan 100.000 elemanın hepsini bir kaydırmak demek, doğrusal süre.

`deque` çift yönlü bağlı bir yapı, iki ucunda da sabit maliyet. Bedeli ortadan
indeksleme: `lst[5000]` listede sabit, deque'te doğrusal.

`set` ve `dict` hash tablosu. Aranan değerin hash'i hesaplanıp doğrudan yerine
gidiliyor, kaç eleman olduğu fark etmiyor. Liste ise tek tek karşılaştırıyor.

**Pratikte ne demek:**

Tek arama 0,65 milisaniye, küçük görünüyor. Ama 100.000 elemanlı bir listede
100.000 arama yaparsam yaklaşık 65 saniye sürer. Aynı işi `set` ile yaparsam
yaklaşık 3 milisaniye. Tek satırlık bir değişiklik (`lst = set(lst)`), bir dakika
bekleyen script ile anında biten script arasındaki fark.

**Kural — erişim desenine göre seç:**

| Erişim deseni | Yapı |
|---|---|
| Sona ekle/çıkar, indeksle eriş | `list` |
| İki uçtan da ekle/çıkar, kayan pencere | `deque` |
| "Var mı?" sorusu, tekilleştirme | `set` |
| Anahtar → değer eşleme, sayma | `dict` / `Counter` |

**Not:** Bu fark küçük veride görünmüyor. 100 elemanlı listede `list` ile `set`
arasında hiçbir şey hissetmiyorum. Yani ölçek, kararı belirliyor — Hafta 1'de
28 MB'lık CSV'de öğrendiğim şeyin aynısı.


---

## 11. Docstring fonksiyonun içinde olmalı

```python
 """Açıklama."""          # ← YANLIŞ: def'in üstünde
def f(a: int) -> int:
    return a
```

**Gerçek:** `IndentationError: unindent does not match any outer indentation level`

**Neden:** Docstring, fonksiyon gövdesinin **ilk ifadesidir**. `def` satırının üstüne
konursa Python onu bağımsız bir string ifadesi sanıyor ve girinti hesabı bozuluyor.

**Kural:** `def` satırının altında, `return` ile aynı hizada (4 boşluk). Sınıflarda da
`class` satırının altında.

---

## 12. `return` yanlışlıkla döngünün içinde

```python
for k in kelimeler:
    sayac[k] = sayac.get(k, 0) + 1
    return sayac  # ← bir girinti fazla
```

**Gerçek:** Fonksiyon ilk kelimeden sonra dönüyor, `{"a": 1}` veriyor.

**Nasıl yakalandı:** mypy "Missing return statement" dedi. Mantığı: liste boşsa döngü
hiç çalışmaz, `return`'e ulaşılmaz, fonksiyon örtük `None` döner — ama imza
`dict[str, int]` diyor. Tip denetleyicisi bir **girinti hatasını** ortaya çıkardı.

**Kural:** Testler yeşilken bile mypy'yi çalıştır; ikisi farklı hata sınıflarını yakalıyor.

---

## 13. Boş kap yaratıyorsan tipini yaz

```python
sonuc = {}  # mypy: Need type annotation
sonuc: dict[str, int] = {}  # ✓
gorulen: set[int] = set()  # ✓
pencere: deque[int] = deque()  # ✓
```

**Neden:** Boş kaptan tip çıkarılamaz. İlk atamayı görmeden mypy'nin hiçbir ipucu yok.

**Kural:** Bu hafta dört kez karşıma çıktı. Boş `{}`, `[]`, `set()`, `deque()` yazarken
tip açıklamasını refleks haline getir.

---

## 14. `__eq__` tanımlayınca sınıf hashlenemez oluyor

```python
class Vektor:
    def __eq__(self, other: object) -> bool: ...

    # __hash__ otomatik olarak None yapılır
```

**Gerçek:** `{Vektor(1,2)}` → `TypeError: unhashable type`

**Neden:** Eşitlik tanımı değiştiyse hash tanımı da değişmeli; Python tutarsızlığı
engellemek için `__hash__`'i düşürüyor.

**Kural:** Değer nesnesi yazıyorsan `__hash__`'i de yaz (`hash((self.x, self.y))`),
ya da `@dataclass(frozen=True)` kullan — o ikisini birlikte üretir.

---

## 15. `d.get(k)` ile `d[k]` tip genişliği açısından farklı

```python
max(d, key=d.get)  # mypy hatası
max(d, key=lambda k: d[k])  # ✓
```

**Neden:** `d.get` eksik anahtarda `None` döndürebildiği için tipi
`Callable[[str], int | None]`. `None` sıralanamaz. `d[k]` ise ya `int` döndürür ya
`KeyError` fırlatır — tip dar kalır.

**Kural:** "Güvenli" görünen `get`, dönüş tipini genişletiyor ve bu genişlik aşağı
akıştaki her kullanımı etkiliyor. Güvenlik ücretsiz değil.

---

## 16. `typing.Callable` ve arkadaşları eskidi

```python
from typing import Callable, Iterable, Iterator  # eski
from collections.abc import Callable, Iterable, Iterator  # ✓
```

**Neden:** Python 3.9'dan beri bu soyut tipler `collections.abc` altında.
`typing` sürümleri geriye uyumluluk için duruyor.

**Kural:** ruff'ın `UP` (pyupgrade) kuralları bunu otomatik yakalıyor. Aynı grup
`Generic[T]` yerine `class Kutu[T]` ve `TypeVar` yerine `def f[T](...)` sözdizimini
de öneriyor (Python 3.12+).

---

## 17. `match` bütün yolları kapatmıyorsa fonksiyon sessizce `None` dönüyor

```python
def renk_kodu(renk: Renk) -> str:
    match renk:
        case "kirmizi":
            return "#FF0000"
        case "yesil":
            return "#00FF00"
        case "mavi":
            return "#0000FF"
    # case _ yok → örtük None
```

**Gerçek:** mypy "Missing return statement"; çalışma zamanında beklenen `KeyError`
yerine `None`.

**Kural:** İki seçenek — ya sözlük araması kullan (`d[k]` zaten `KeyError` fırlatır),
ya `case _: assert_never(renk)` ekle. İkincisi Literal'a yeni değer eklendiğinde
mypy'nin **derleme zamanında** uyarmasını sağlıyor.

---

## 18. Python Türkçe büyük/küçük harf kurallarını bilmiyor

```python
"İ".lower()  # → 'i̇'  (i + birleşen nokta: İKİ karakter)
"I".lower()  # → 'i'   (Türkçede 'ı' olmalıydı)
"ı".upper()  # → 'I'   (Türkçede 'I' doğru ama İ/I ayrımı kayıp)
```

**Neden:** `str.lower()` Unicode'un dil-bağımsız kurallarını uyguluyor; Türkçenin
noktalı/noktasız i ayrımı özel bir durum.

**Kural:** Türkçe metinde büyük/küçük harf normalleştirmesi yaparken sonucu doğrula.
Karşılaştırma için elle eşleme tablosu ya da `casefold()` + özel durum gerekebilir.
Bu hafta ünlü sayma egzersizinde sonucu bozmadı ama sessiz hata kaynağı.

---

## 19. `groupby` grubu tembel ve tek kullanımlık

```python
for k, g in groupby(items, key=anahtar):
    yield k, list(g)  # list() ŞART
```

**Neden:** `groupby` grup nesnesini tembel döndürüyor ve bir sonraki gruba
geçildiğinde önceki grup **geçersizleşiyor**. Sabitlemezsen sessizce boş gruplar alırsın.

**Kural:** Tembelliğin bedeli var: tembel bir sonucu saklıyorsan önce sabitle.
Aynı mantık generator'lar için de geçerli (madde 10).

---

## 20. `str.isdigit()` negatif sayıyı yakalamıyor

```python
"-5".isdigit()  # False
```

**Neden:** `isdigit` yalnızca rakam karakterlerine bakıyor; eksi işareti rakam değil.

**Gerçek:** Metinden tam sayı ayıklarken negatifler sessizce düşüyor. Çözüm regex:
`re.findall(r"-?\d+", s)` — `-?` "eksi işareti olabilir de olmayabilir de" demek.

**Kural:** Karakter sınıfı kontrolleri (`isdigit`, `isalpha`, `isalnum`) tek karakter
mantığıyla çalışıyor; sayı ayrıştırma için yetersizler.

## 20. Aynı isimde ikinci tanım sessizce ezer

```python
class Transform:      # satır 9
    ...
class Transform:      # satır 20 — hata değil!
    ...
```

**Gerçek:** Python şikâyet etmiyor. İkinci tanım birincinin yerine geçiyor; satır 9–19 arası **ölü kod** oluyor.
**Neden:** `class` ve `def` birer atama işlemidir. İkinci atama ismi yeniden bağlar, tıpkı `x = 1; x = 2` gibi.
**Kural:** Bir dosyada bir ismin tek tanımı olmalı. `ruff` bunu `F811` ile yakalar — dilin yakalamadığını linter yakalıyor.

---

## 21. pytest, `test_` ile başlamayan fonksiyonu sessizce atlar

```python
def bos_akis_sifir_donduruyor():   # çalışmaz
    ...
```

**Gerçek:** `collected 1 item` — dört test yazılmıştı, üçü hiç çalışmadı. Uyarı yok.
**Neden:** pytest testleri isimden keşfeder; varsayılan desen `test_*`.
**Kural:** Her koşuda `collected N items` satırını oku. Yazdığın sayıyla eşleşmiyorsa dur.

---

## 22. `yield`i `try` bloğunun içinde bırakmak

```python
for kayit in akis:
    try:
        yield f(kayit)          # yanlış
    except Exception:
        ...
```

**Gerçek:** `yield` satırında akış duraklar ve kontrol tüketiciye geçer. Tüketici tarafındaki olaylar da bu `except`'e düşer.
**Neden:** Generator `yield`de askıya alınır; o sırada dışarıdan gelen istisnalar `yield` noktasından fırlar.
**Kural:** `try`ı olabildiğince dar tut — sadece patlayabilecek satır içeride olsun:
```python
try:
    sonuc = f(kayit)
except Exception:
    ...
    continue
yield sonuc
```

---

## 23. Bash'te `!` çift tırnak içinde korunmaz

```bash
git commit -m "feat(core)!: hata yalitimi"
# -bash:  : unrecognized history modifier
```

**Gerçek:** Komut parçalandı, commit atılmadı. Sonraki `git push` "Everything up-to-date" dedi ve hatayı gizledi.
**Neden:** `!` geçmiş genişletme karakteridir (`!!`, `!$`, `!:`). Çift tırnak `$`, `` ` ``, `\` ve `!` **dışındaki** her şeyi korur.
**Kural:** `!` içeren metin **tek tırnak** ister. Tek tırnak içinde hiçbir karakter özel değildir.

---

## 24. `dataclass`ta değişebilir varsayılan yazılamaz

```python
@dataclass
class Rapor:
    hatalar: list = []     # ValueError
```

**Gerçek:** Python bunu çalışma anında reddediyor: "mutable default ... use default_factory".
**Neden:** 1 numaralı tuzağın aynısı — varsayılan bir kez oluşur, bütün örnekler aynı listeyi paylaşırdı.
**Kural:** `field(default_factory=list)`. Dil, bilinen bir tuzağa karşı burada kendini koruyor.

---

## 25. `Protocol` varsayılan olarak `isinstance` ile kullanılamaz

```python
class Source(Protocol):
    def oku(self): ...

isinstance(kaynak, Source)
# TypeError: ... only be used with @runtime_checkable protocols
```

**Gerçek:** Çalışma zamanı kontrolü hata veriyor.
**Neden:** `Protocol` öncelikle **statik** denetim içindir; mypy okur, yorumlayıcı bilmez.
**Kural:** Çalışma zamanında kontrol edeceksen `@runtime_checkable` ekle. Ama dekoratör yalnızca **metot adlarına** bakar, imzalara değil — asıl garanti hâlâ mypy'den gelir.

---

## 26. CSV'de tip yoktur

```python
esle(lambda k: {**k, "yas": int(k["yas"]) + 1})
assert [k["yas"] for k in cikti] == [23, 27, 32]     # kırmızı
```

**Gerçek:** Geri okunan değerler `"23"`, `"27"`, `"32"` — dize.
**Neden:** CSV düz metindir; `csv.DictReader` her alanı `str` olarak verir, yazarken de `str()` uygulanır.
**Kural:** CSV'den gelen her şey dizedir. Testte beklenen değerleri tırnak içinde yaz; sayısal iş yapacaksan sınırda açıkça `int()`/`float()` çevir.

---

## 27. `iter` bir fonksiyondur

```python
CsvSink(yol).yaz(iter[])      # SyntaxError: unmatched ')'
CsvSink(yol).yaz(iter([]))    # doğru
```

**Gerçek:** `SyntaxError` — dosya **hiç okunmadı**, `collected 0 items`.
**Neden:** Parantez fonksiyon çağırır; köşeli parantez ya liste yapar ya eleman alır.
**Kural:** `SyntaxError` ile başarısız test farklı şeylerdir. `collected 0 items` görünce mantığa değil yazıma bak.

---

## 28. Fonksiyon çağrısında `==` yazmak

```python
Pipeline(kaynak=..., hedef=..., donusum== lambda x: x)
# SyntaxError: positional argument follows keyword argument
```

**Gerçek:** Python `donusum == lambda x: x` ifadesini bir **değer** olarak okudu; isimli argümanlardan sonra isimsiz argüman geldi.
**Neden:** Çağrıda `=` parametreyi isimlendirir, `==` karşılaştırma yapar.
**Kural:** Parantez içinde `==` görürsen neredeyse her zaman yanlıştır.

---

## 29. `if x:` ile `if x is not None:` aynı şey değil

```python
if self.donusum:            # boş liste, 0, "" de atlanır
if self.donusum is not None:  # sadece None atlanır
```

**Gerçek:** "Yok mu?" sorusu ile "boş mu?" sorusu farklı cevaplar verir.
**Neden:** `if x:` nesnenin doğruluk değerine bakar; `0`, `""`, `[]`, `{}` yanlış sayılır.
**Kural:** Varlık kontrolü kimlik sorusudur, `is None` / `is not None` ile sorulur.

---

## 30. `time.time()` süre ölçümünde geriye gidebilir

```python
baslangic = time.time()
...
sure = time.time() - baslangic     # negatif olabilir
```

**Gerçek:** Sistem saati NTP ile senkronize olursa veya kullanıcı değiştirirse fark negatif çıkabilir.
**Neden:** `time.time()` duvar saatidir; `time.perf_counter()` yalnızca ileri giden bir sayaçtır.
**Kural:** Süre ölçümünde her zaman `perf_counter`. Tek başına anlamsızdır, iki ölçümün **farkı** anlamlıdır.

---
