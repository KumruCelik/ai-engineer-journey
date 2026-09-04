# Hafta 2 — Retro

## Bu hafta yeni ne yapabiliyorum?

- Generator ile sabit bellekte akış işleyen fonksiyonlar yazabiliyorum
  (`parcala_lazy`, `pencerele` — girdi liste değil akış olabiliyor)
- Argümanlı ve argümansız decorator yazabiliyorum, `functools.wraps`'in
  neyi koruduğunu ölçtüm
- `contextmanager` ile kaynak yönetimi yapabiliyorum; temizlik işini
  `finally`'ye koyduğum için hata durumunda da çalışıyor
- `Protocol` ile kalıtım gerektirmeyen arayüz tanımlayabiliyorum
- `asyncio` ile eşzamanlı istek yönetebiliyorum: `gather`, `Semaphore`,
  `wait_for`, üstel geri çekilmeli retry
- mypy'yi 418 satırda temiz geçirebiliyorum
- Koleksiyon seçimini big-O ile gerekçelendirebiliyorum ve bunu ölçtüm

## Neyi hâlâ AI olmadan yapamıyorum?

- Test dosyalarını sıfırdan tasarlamak — testleri hep bana verildi, ben
  gövdeleri yazdım. Şartnameyi kendim kurmayı denemedim.
- pytest `fixture`, `conftest.py`, `monkeypatch` — bu hafta hiç kullanmadım
- `hypothesis` ile property-based test — hiç denemedim
- Karmaşık tip imzalarını (generic, Protocol) ilk denemede doğru yazmak
- Cuma AI-free günüydü ama OOP kategorisinde yardım aldım

## Hangi hatam bana en çok şey öğretti?

`kelime_say`'de `return`'ü döngünün içinde bırakmam. Testler o an yeşildi
(hatayı sonradan, import düzenlerken yaptım), ruff da bir şey demedi.
Yakalayan mypy oldu: "Missing return statement". Mantığı şuydu — liste boşsa
döngü hiç çalışmaz, `return`'e ulaşılmaz, fonksiyon örtük `None` döner ama
imza `dict[str, int]` diyor. Yani bir **tip denetleyicisi**, bir **girinti
hatasını** ortaya çıkardı.

Çıkardığım ders: test, lint ve tip denetimi farklı hata sınıflarını yakalıyor.
Üçünü birden çalıştırmak üç ayrı ağ germek demek.

## Portföye ne eklendi?

- py-core — https://github.com/KumruCelik/py-core
  80 egzersiz, 90 test, %100 kapsam, mypy + ruff temiz
- python-gotchas.md — 20 madde, hepsi gerçekten yaşadığım şeyler

## Rubrik (0-5)

| Konu | Not | Gerekçe |
|---|---|---|
| Python idiyomları | 3 | 80 fonksiyon yazdım, mekanizmalarını açıklayabiliyorum; bazı imzaları ilk denemede doğru kuramadım |
| Generator / lazy | 3 | Sabit bellek mantığını anlıyorum, `tekrarsiz_lazy`'nin neden sabit bellekli olmadığını fark ettim |
| Decorator / context manager | 3 | İkisini de yazabiliyorum; argümanlı decorator'ın üç katmanını hâlâ düşünerek kuruyorum |
| Typing / mypy | 3 | 418 satır temiz geçti; `Protocol` ve generic sözdizimini yardımla öğrendim |
| asyncio | 2 | Çalışan kod yazdım ama tasarım kararlarını (ne zaman async, ne zaman değil) henüz kendim veremiyorum |
| Test yazma | 2 | Testleri ben tasarlamadım, gövdeleri yazdım. fixture/mock hiç kullanmadım. |
| Araç zinciri (ruff/uv/mypy) | 4 | Dört farklı araçta gerekçeli istisna tanımladım, kuralları kapatmadım |

## Hafta 3'e taşınanlar

- pytest fixture, conftest.py, monkeypatch (kontrol sorusu 5 bunu bekliyor)
- hypothesis ile property-based test
- Kendi test şartnamemi yazmak
- Kontrol soruları 3 ve 4
- İki araştırma yazısı (GIL, veri kodu testi)
