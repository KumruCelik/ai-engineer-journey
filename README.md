# ai-engineer-journey

40 haftalık "Yapay Zekâ & Veri Odaklı Yazılım Mühendisi" programının öğrenme
kaydı. Her bölüm bir klasör: notlar, araştırma yazıları, kontrol soruları ve
retro.

Kod çıktıları ayrı repolarda tutuluyor; bu repo yazılı kısmın evi.

## Yapı

```
weekNN/
├── notes/      haftalık notlar, kontrol soruları, retro
├── research/   araştırma yazıları
└── data/       veri kaynakları (dosyalar gitignore'da, sadece README)
```

## İlerleme

| Hafta | Bölüm | Çıktı | Durum |
|---|---|---|---|
| 1 | Ortam, araçlar, mühendislik hijyeni | [dev-setup](https://github.com/KumruCelik/dev-setup) · [fastapi-docker](https://github.com/KumruCelik/fastapi-docker) | ✅ |
| 2–4 | Python & yazılım mühendisliği disiplini | [py-core](https://github.com/KumruCelik/py-core),  [mini-etl](https://github.com/KumruCelik/mini-etl), [perf-lab](https://github.com/KumruCelik/perf-lab) | ✅  |
| 4–6 | SQL & veri modelleme | sql-mastery | ⬜ |
| 6–9 | Lineer cebir | linalg-from-scratch | ⬜ |
| 9–11 | Kalkülüs & optimizasyon | microautograd | ⬜ |
| 11–15 | Olasılık & istatistik | stat-lab, ab-test-kit | ⬜ |
| 12–16 | Ürün, metrik, para | 3 iş dokümanı | ⬜ |
| 15–17 | Veri analizi & EDA | eda-playbook | ⬜ |
| 17–22 | Klasik makine öğrenmesi | ml-from-scratch, ml-competition | ⬜ |
| 22–27 | Derin öğrenme | dl-lab, mini-gpt | ⬜ |
| 27–32 | NLP & LLM mühendisliği | rag-production | ⬜ |
| 30–36 | Veri mühendisliği & MLOps | ml-platform | ⬜ |
| 34–36 | ML sistem tasarımı | 8 tasarım dokümanı | ⬜ |
| 36–40 | Piyasaya hazırlık | CV, blog, soru bankası | ⬜ |

## Bölüm 1 çıktıları

- [CLI veri işleme notları](week01/notes/cli-exercises.md) — 100k kayıtlık
  Chicago suç verisinde terminal araçlarının gücü ve sınırları
- [Git senaryoları](week01/notes/git-scenarios.md) — 5 kurtarma senaryosu
- [uv vs poetry vs pip-tools vs conda](week01/research/uv-vs-poetry.md) —
  kendi makinemde ölçülmüş karşılaştırma
- [Kontrol soruları](bolum01/notes/week01-answers.md)
- [Retro](week01/notes/retro-week01.md)

## Bölüm 2 çıktıları

**Araştırma yazıları**

- [GIL nedir, ne zaman engel olur?](bolum02/research/gil.md) — kendi
  ölçümlerimle: aynı iş, iş parçacığı ile 1.0×, süreç ile 2.5×; I/O işinde
  iş parçacığı 4.0×
- [Veri işleme kodu nasıl test edilir?](bolum02/research/veri-kodu-testi.md) —
  beş test deseni ve `mini-etl`'de yaptığım beş test tasarım hatası

**Notlar**

- [Python tuzak koleksiyonu](bolum02/notes/python-gotchas.md) — 40 madde,
  her biri kod → gerçek → neden → kural biçiminde
- [Kontrol soruları](bolum02/notes/week02-answers.md) — `Protocol` ve `ABC`
  farkı, 100 GB CSV'yi 8 GB RAM'de işlemenin üç stratejisi
- [Hafta 2 retro](bolum02/notes/retro-week02.md) ·
  [Hafta 3 retro](bolum02/notes/retro-week03.md)

**Ölçüm raporları** *(perf-lab reposunda)*

- [1 GB log dosyasında dört toplama yöntemi](https://github.com/KumruCelik/perf-lab/blob/main/LOG.md)
  — naif döngü, generator, `multiprocessing`, `polars`; süre ve bellek tablosu
- [Senkron ve asenkron HTTP istemci karşılaştırması](https://github.com/KumruCelik/perf-lab/blob/main/ASYNC.md)
  — `asyncio.Semaphore` ile eşzamanlılık kontrolü ve rate limit davranışı

**Tasarım belgesi** *(mini-etl reposunda)*

- [DESIGN.md](https://github.com/KumruCelik/mini-etl/blob/main/DESIGN.md) —
  sekiz tasarım kararı, her biri reddedilen alternatifi ve gerekçesiyle;
  uygulama sonrası sapmalar ve bilinen sınırlar

## Çalışma yöntemi

Her konuda aynı döngü: **tahmin et → ölç → tahmin tutmadıysa kaydet.**

Tahmin tuttuğunda hiçbir şey öğrenilmiyor; tutmadığında tuzak koleksiyonuna
bir madde ekleniyor. Bu repodaki 40 maddenin tamamı böyle birikti.
