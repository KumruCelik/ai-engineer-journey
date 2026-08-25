# Hafta 1 — Kontrol Soruları

## 1. `pip install` ile `uv add` arasında bağımlılık çözümlemesi farkı nedir?

`pip install` kurulum anında çözüm yapar ve sonucu hiçbir yere kaydetmez.
`requirements.txt`'te `pandas>=2.0` yazıyorsa, kuran herkes o günün en yeni
sürümünü alır. Aynı dosyadan iki kişi farklı ortam elde edebilir.

`uv add` iki dosyaya birden yazar: `pyproject.toml`'a benim istediğim paketi
(gevşek aralıkla), `uv.lock`'a ise çözülmüş ağacın tamamını tam sürüm ve
hash'lerle. `dev-setup` projemde `pyproject.toml` 544 bayt, `uv.lock` 160 KB.
Ben dört paket istedim ama bağımlılık ağacı çok daha geniş; lock dosyası o
ağacın tamamını sabitliyor.

Sonuç olarak asıl fark hız değil **tekrar üretilebilirlik**: `uv sync`
çalıştıran herkes — ben, ekip arkadaşım, CI sunucusu — bit bit aynı ortamı
alıyor. Hız da var (ölçtüm: soğuk cache'te 24 sn vs 55 sn) ama o ikincil.

`pip` tarafında bunu istiyorsan `pip-tools` ile ayrıca lock üretmen gerekir;
`uv`'de varsayılan davranış bu.

## 2. `__pycache__` neden `.gitignore`'da olmalı?

Çünkü türetilmiş bir çıktı. Python `.py` dosyasını çalıştırırken bytecode'a
derleyip `.pyc` olarak önbelleğe alıyor. Kaynak kod değil, kaynağın ürünü.
Genel kural: **kaynaktan otomatik üretilebilen hiçbir şey versiyonlanmaz.**
`.venv/`, `node_modules/`, `dist/` da aynı kategoride.

Boyut en önemsiz sebep. Asıl problemler şunlar:

`.pyc` dosyaları Python sürümüne ve platforma özgü. Ben WSL'de Python 3.12
ile üretiyorum, başkası macOS'ta 3.11 ile. Commit'lersek her push'ta ikimizin
dosyaları birbirini ezer ve sürekli anlamsız çakışma çıkar.

Daha kötüsü: bayat bir `.pyc`, silinmiş bir `.py` dosyasının yerine geçip
"olmayan modül nasıl import edildi" tarzı saatler süren hatalara yol açabilir.

## 3. Docker'da `COPY pyproject.toml` ile `COPY app/` sırasını neden ayırıyoruz?

Docker her komutu bir **katman** yapıp önbelleğe alıyor. Bir katmanın girdisi
değişirse o katman ve **ondan sonraki bütün katmanlar** yeniden çalışıyor.
Cache'i geçersiz kılan şey, o satırda kopyalanan dosyaların içeriği.

Buradan çıkan kural: **değişim sıklığına göre sırala, seyrek değişen üstte.**

Benim `Dockerfile`'ımda önce `pyproject.toml` + `uv.lock` kopyalanıyor, sonra
`uv sync` çalışıyor, en son kod geliyor. Kodum günde onlarca kez değişiyor,
bağımlılıklarım ayda bir. Bu sırayla kod değiştiğinde `uv sync` katmanı
cache'ten geliyor ve build saniyeler sürüyor.

Tek `COPY . .` yazsaydım her kod değişikliği bütün bağımlılıkları yeniden
kurdururdu. `python:3.12-slim` üstüne pandas + numpy kurmak dakikalar demek.

## 4. `git rebase` ile `git merge` farkı — ekip çalışması bağlamında

`merge` yeni bir commit oluşturur ve **iki ebeveyni** olur. Geçmişte gerçekten
ne olduğunu korur: iki dal ayrılmış, paralel ilerlemiş, sonra birleşmiş.
Senaryo 4'te ürettiğim grafik tam olarak bu:

```
*   7d9498d Merge branch 'feature/renk'
|\
| * bf80a94 feat: renk mavi
* | 9251d11 feat: renk kirmizi
|/
```

`rebase` ise commit'leri yeni bir taban üzerine **yeniden oynatır**. Geçmiş
düz bir çizgi olur, ama commit'ler artık aynı commit değildir. Senaryo 2'de
`--amend` yaptığımda hash `3d4a1da`'dan `da034a1`'e döndü — çünkü commit'in
kimliği içeriğine ve ebeveynine bağlı, ebeveyn değişince hash de değişiyor.
`rebase` de aynı mekanizmayla çalışıyor.

Ekipteki tehlike burada. Bir dalı push ettiysem, arkadaşım onu çekmiştir.
Sonra o dalı rebase edersem, aynı işi içeren ama farklı hash'li commit'ler
üretmiş olurum. Arkadaşımın kopyasındaki eski commit'lerle benimkiler artık
farklı nesneler; `git pull` ikisini birleştirmeye çalışır ve aynı değişiklik
iki kez görünür. Bu yüzden altın kural: **paylaşılan dalı rebase etme.**

Pratikte benim kuralım: kendi dalımdayken, henüz push etmemişken `rebase -i`
ile geçmişimi temizlerim. `main`'e katarken `merge` (veya GitHub'ın squash
merge'ü) kullanırım. Squash merge zaten ikisinin karışımı: dalın commit'leri
tek commit'e inip `main`'e ekleniyor.

## Bilmediklerim / emin olmadıklarım

- `merge` ile `squash merge` arasındaki farkın uzun vadede geçmiş üzerindeki
  etkisini tam kavramadım, Hafta 2'de bakacağım.
- `pip-tools`'u hiç kullanmadım, sadece okudum.
