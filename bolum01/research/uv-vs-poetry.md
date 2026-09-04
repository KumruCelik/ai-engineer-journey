# uv vs poetry vs pip-tools vs conda — hangisini ne zaman?

Hafta 1'de `uv`'yi kullanmaya başladım ama "neden bu" sorusuna cevabım yoktu.
Kendi makinemde ölçtüm.

## Ölçüm

Aynı üç paketi (`pandas`, `numpy`, `scikit-learn`) her araçla kurdum. Cache
durumunun sonucu tamamen değiştirdiğini fark ettiğim için hem soğuk hem sıcak
cache ile ölçtüm — sadece birini ölçseydim yanıltıcı bir tablo çıkardı.

Ortam: WSL2 Ubuntu, Python 3.12.3.

| Araç | Soğuk cache | Sıcak cache |
|---|---|---|
| uv | 24,0 sn | **0,14 sn** |
| pip + venv | 54,7 sn | 24,9 sn |
| poetry | 54,2 sn | — |

Soğuk kurulumda `uv`, `pip`'in yaklaşık iki katı hızlı. Ama asıl fark sıcak
cache'te: **178 kat**.

## Bu farkın nereden geldiği

`time` çıktısındaki `user` satırı beni asıl sonuca götürdü.

Sıcak cache'li `pip` kurulumunda `real` 24,9 sn, `user` 19,4 sn. Yani geçen
sürenin neredeyse tamamı **CPU'da** geçmiş, ağda değil. Paketler zaten diskte
duruyordu ama `pip` her sanal ortama tekrar açıp kuruyor.

`uv`'de ise sıcak kurulum 0,14 sn ve `user` sadece 0,05 sn. `uv` paketleri
merkezi bir cache'te tutuyor ve sanal ortama kopyalamak yerine bağlantı
veriyor. İş yapmıyor, işaret ediyor.

Bağımlılık çözümlemesinde de fark var: `uv` soğukta 1,13 sn, sıcakta 15 ms
çözdü. `poetry` 2,5 sn. Küçük görünüyor ama bu üç paket için; onlarca
bağımlılığı olan gerçek bir projede bu süre büyüyor.

## Peki hız her şey mi

Değil. Bu araçların asıl işi hızlı kurmak değil, **aynı ortamı tekrar
üretebilmek**.

`dev-setup` projemde `pyproject.toml` 544 bayt, `uv.lock` 160 KB. Ben dört
paket istedim, onlar kendi bağımlılıklarını getirdi. Lock dosyası bu ağacın
tamamını tam sürüm ve hash ile sabitliyor. `uv sync` çalıştıran herkes —
CI sunucusu dahil — bit bit aynı ortamı alıyor.

`pip install -r requirements.txt` bunu garanti etmiyor. Dosyada `pandas>=2.0`
yazıyorsa herkes kurduğu günün en yeni sürümünü çeker. "Bende çalışıyordu"
cümlesinin kaynağı bu.

## Kendi kararım

| Durum | Seçim |
|---|---|
| Yeni Python projesi | **uv** |
| Kütüphane yayınlayacaksam | poetry (paketleme akışı daha olgun) |
| Eski, sadece `requirements.txt` olan repo | pip-tools ile lock ekle |
| CUDA, MKL gibi Python dışı bağımlılıklar | conda |

`uv`'yi seçiyorum çünkü `pip` + `venv` + lock işini tek araçta, çok daha hızlı
yapıyor. Sıcak cache'teki 0,14 saniye özellikle önemli: her yeni denemede
ortam kurmaktan kaçınmıyorum, çünkü maliyeti sıfıra yakın.

`conda`'yı hafife almıyorum. Python dışı ikili bağımlılıkları (CUDA sürücüleri,
derlenmiş matematik kütüphaneleri) `pip` dünyası temiz çözemiyor. GPU ile
çalışmaya başladığımda muhtemelen ona döneceğim.

## Ne ölçmedim / sınırlar

- **conda'yı ölçmedim.** Kurulu değil ve 500 MB+ indirme gerektiriyor. Bu
  yüzden onunla ilgili yazdıklarım ölçüme değil dokümantasyona dayanıyor.
- **Tek makine, tek ölçüm.** Her ölçümü bir kez yaptım. Ağ dalgalanması
  sonucu etkilemiş olabilir; sağlıklısı 3-5 tekrarın medyanını almaktı.
- **poetry'yi sıcak cache ile ölçmedim.** İlk kurulumdan sonra tekrarlamadım.
- **Sadece üç paket.** Gerçek bir projede 40-50 bağımlılık olur; çözümleme
  süresindeki fark orada daha belirgin çıkabilir.
- **Sadece kurulum hızına baktım.** Monorepo desteği, özel paket sunucuları,
  yayınlama akışı gibi konulara girmedim — bunlar takım ortamında seçimi
  değiştirebilir.
