# GIL nedir, ne zaman engel olur, ne zaman olmaz?

*Ödev 2.5.1 — kendi ölçümlerimle. ~850 kelime.*

## Mekanizma

**GIL** (Global Interpreter Lock), CPython yorumlayıcısındaki tek bir kilittir.
Kuralı basit: aynı anda yalnızca **bir** iş parçacığı Python bayt kodu
çalıştırabilir. Dört thread açsan da, on iki çekirdeğin de olsa, Python kodu
sırayla çalışır.

Sebebi bellek yönetimidir. Python'da her nesne bir **referans sayacı** taşır —
"bu nesneye kaç yerden işaret ediliyor?" Sayaç sıfıra inince nesne silinir.
`a = b` gibi sıradan bir atama bile bu sayacı değiştirir. İki iş parçacığı
sayacı aynı anda güncellerse ya sayaç olması gerekenden yüksek kalır (bellek
sızar) ya da erken sıfırlanır (kullanımdaki nesne silinir, program çöker).

Her nesneye ayrı bir kilit koymak çözüm olurdu ama maliyeti çok yüksek: her
atamada kilit alıp bırakmak, tek iş parçacıklı programları da yavaşlatırdı.
CPython bunun yerine tek bir küresel kilit seçti. Bu, tek iş parçacıklı kodu
hızlı tutar ve C uzantısı yazmayı kolaylaştırır — Python'un bugünkü kütüphane
ekosisteminin varlık sebeplerinden biri budur.

Kritik ayrıntı: **kilit her zaman tutulmaz.** Yorumlayıcı bir iş parçacığını
yaklaşık her 5 milisaniyede bir duraklatır (`sys.setswitchinterval`), ve daha
önemlisi, **I/O beklerken kilidi bırakır**. `socket.recv`, `file.read`,
`time.sleep` — hepsi kilidi bırakıp bekler.

## Ölçüm

Aynı işi üç şekilde çalıştırdım: sıralı, dört iş parçacığı, dört süreç. İki iş
türü için ayrı ayrı. CPU işi hiç beklemiyor (`sum(i*i for i in range(5e6))`),
I/O işi hiç hesap yapmıyor (`time.sleep(0.5)`). Her ölçümde dört iş var, yani
ideal hızlanma dört kat.

| İş türü | Sıralı | 4 iş parçacığı | 4 süreç |
| --- | --- | --- | --- |
| CPU-yoğun | 1.08 sn | **1.04 sn** (1.0×) | 0.43 sn (2.5×) |
| I/O-yoğun | 2.02 sn | **0.51 sn** (4.0×) | 0.52 sn (3.9×) |

*(`scripts/gil.py`, Python 3.12.3, 12 mantıksal çekirdek, WSL2.)*

Aynı `ThreadPoolExecutor`, aynı işçi sayısı, aynı makine. Bir satırda hiçbir
şey kazandırdı, diğerinde tam dört kat.

CPU satırında iş parçacıkları **hiç yardım etmedi** çünkü dördü de aynı kilidi
paylaştı; sırayla çalıştılar ve üstüne bir de kilit devri maliyeti ödediler.
Süreçler 2.5 kat kazandırdı — 4 kat değil, çünkü süreç başlatmak ve sonuçları
serileştirmek zaman alıyor; 1 saniyelik bir işte bu sabit maliyet göze
batıyor.

I/O satırında iş parçacıkları tam dört kat kazandırdı. Dört thread'in dördü de
`sleep` içindeyken kilit serbest; hiçbiri diğerini engellemiyor.

## Gerçek iki iş

Bu ayrımı iki gerçek ödevde de ölçtüm. Ayırt edici gösterge `user / real`
oranı: harcanan CPU zamanının geçen süreye bölümü.

**1 GB log dosyasında gruplama** (`perf-lab/LOG.md`): `real 60 sn, user 60 sn`
→ oran **1.00**. İşlemci sürekli çalışıyor. Burada iş parçacığı işe yaramaz;
`multiprocessing` 4 çekirdekte 2.0 kat, 12 çekirdekte 3.7 kat kazandırdı.

**API'den 1000 kayıt çekme** (`perf-lab/ASYNC.md`): `real 94 sn, user 0.8 sn`
→ oran **0.009**. İşlemci ömrünün %99'unda boşta; program ağdan cevap
bekliyor. Burada `asyncio` tek iş parçacığıyla 9.8 kat kazandırdı.

Aynı dil, aynı makine, iki iş — ve zıt çözümler. "Python yavaş" cümlesi bu
tabloyu açıklamıyor; "hangi kaynak darboğaz?" sorusu açıklıyor.

## Karar kuralı

1. **Önce ölç.** `time` komutunun `user / real` oranına bak.
2. **Oran ~1 ise CPU-yoğun.** İş parçacığı işe yaramaz. `multiprocessing` veya
   `concurrent.futures.ProcessPoolExecutor` kullan. Bedeli: süreçler bellek
   paylaşmaz, veri serileştirilerek gider gelir. Küçük işlerde bu maliyet
   kazancı yer.
3. **Oran ~0 ise I/O-yoğun.** GIL engel değil. Az sayıda eşzamanlı iş için
   iş parçacığı yeterli; binlerce eşzamanlı bağlantı için `asyncio` daha
   ucuz — her iş parçacığı megabaytlarca yığın (stack) ister, coroutine
   kilobayt.
4. **Üçüncü ve çoğu zaman en iyi yol: GIL'i hiç dövmemek.** `numpy`, `polars`,
   `pandas` gibi kütüphaneler ağır işi C veya Rust'ta yapar ve o sırada GIL'i
   **bırakır**. Ölçümümde `polars`, saf Python'dan 41 kat hızlıydı ve bunu
   satır başına 9.5 kat az hesapla yaptı — yani asıl kazanç paralellikten değil,
   Python nesnesi üretmemekten geliyordu.

Veri mühendisliğinde 4. maddenin cevabı 2. maddeden daha sık doğrudur.
"Paralelleştireyim" refleksinden önce "bu işi zaten yapan, GIL'i bırakan bir
kütüphane var mı?" diye sormak gerekiyor.

## Değişen tablo

Python 3.13 ile GIL'siz bir derleme (**free-threaded build**, PEP 703) deneysel
olarak geldi. Referans sayaçları thread-güvenli hâle getirilerek kilit
kaldırılıyor. Ama bedava değil: tek iş parçacıklı performansta ölçülebilir bir
kayıp var ve C uzantılarının uyarlanması gerekiyor. Yaygınlaşması yıllar
alacak.

O zamana kadar — ve muhtemelen sonrasında da — pratik cevap aynı: **darboğazın
nerede olduğunu ölç, sonra aracını seç.** GIL bir engel değil, bir kısıt;
hangi işte devreye girdiğini bilen için sorun çıkarmıyor.

## Kaynaklar ve ölçüm kodu

- Ölçüm betiği: `perf-lab/scripts/gil.py`
- CPU-yoğun ölçüm: `perf-lab/LOG.md`
- I/O-yoğun ölçüm: `perf-lab/ASYNC.md`
- PEP 703 — Making the Global Interpreter Lock Optional in CPython
