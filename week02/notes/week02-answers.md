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
