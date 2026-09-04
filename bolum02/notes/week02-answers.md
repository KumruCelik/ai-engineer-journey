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
