# Ödev 1.3 — Git Senaryoları

Beş senaryoyu `git-lab` adında bir test reposunda bilfiil yaşadım.
Amaç: Git'i "add-commit-push" seviyesinden çıkarmak.

---

## Zihinsel model

Senaryolara başlamadan önce şunu oturttum:

- **Commit** = o andaki dosya durumunun fotoğrafı + bir önceki commit'e bir ok
- **Dal** = bir commit'e yapıştırılmış etiket, başka bir şey değil
- **HEAD** = şu an nerede olduğumu gösteren işaretçi

Bunu anlayınca komutlar ezber olmaktan çıktı. "Commit'i başka dala taşımak"
aslında "kopyala, sonra eski etiketi geri al" demekmiş.

---

## Senaryo 1 — Yanlış dala commit attım

Kazayı üretmek için `feature/login` dalını açtım ama üzerine geçmeden `main`'de
commit attım.

```
git checkout feature/login
git cherry-pick f2d2813
git checkout main
git reset --hard HEAD~1
```

**Sonuç:**

```
* 1059e92 (feature/login) feat: login ozelligi
* 046431b (HEAD -> main) feat: ucuncu satir
```

**Dikkatimi çeken:** Commit'in hash'i `f2d2813` iken `1059e92` oldu.
`cherry-pick` taşımıyor, **kopyalıyor**. Hash değişti çünkü commit'in kimliği
içeriğine ve bir önceki commit'e bağlı — ebeveyn değişince hash de değişiyor.
Orijinali `reset` ile ben sildim.

---

## Senaryo 2 — Son 3 commit'i tek commit'e indirdim

3 tane `wip` commit'i attım, sonra:

```
git config --global core.editor nano
git rebase -i HEAD~3
```

Editörde ilk satırı `pick` bırakıp diğer ikisini `squash` yaptım.

### Burada hata yaptım

Editöre örnek metindeki sahte hash'leri (`aaa1111`, `bbb2222`) olduğu gibi
yazdım. Git var olmayan commit'leri aradı ve rebase yarıda kaldı.

Ne olduğunu `git status` söyledi:

```
interactive rebase in progress; onto 046431b
```

`git rebase --abort` ile başa döndüm, tekrar denedim. Bu sefer **sadece
satırların ilk kelimesini** değiştirdim, hash'lere hiç dokunmadım.

**Sonuç:** 3 commit tek commit oldu, `notlar.txt` içindeki 3 satır da yerinde
kaldı. Squash commit'leri birleştiriyor, içeriği silmiyor.

Commit mesajını sonradan düzelttim:

```
git commit --amend -m "feat: notlar dosyasi eklendi"
```

`--amend` de hash'i değiştiriyor (`3d4a1da` → `da034a1`). Bu yüzden push
edilmiş bir commit'i amend'lememek gerekiyor.

**Öğrendiğim:** Kafam karıştığında ilk komut `git status`. Git ne durumda
olduğumu ve ne yapabileceğimi zaten söylüyor, tahmin etmeye gerek yok.

---

## Senaryo 3 — `.env`'i geçmişten temizledim

Önce sırrın gerçekten geçmişte durduğunu kanıtladım:

    git log -p --all -- .env
    → DB_PASSWORD=<sahte-test-sifresi>

Dosyayı silip yeni commit atmak yetmiyor; sır eski commit'in içinde kalıyor.

Temizlemek için:

```
sudo apt install -y git-filter-repo
git filter-repo --path .env --invert-paths --force
```

Sonra tekrar kontrol ettim, `git log -p --all -- .env` hiçbir şey döndürmedi.
Commit de tamamen kaybolmuş, çünkü içi boşalınca `filter-repo` onu düşürmüş.

**En önemli nokta:** Geçmişi temizlemek ikinci adım. Eğer repo push edilmişse
sır zaten GitHub'ın sunucularında, başkalarının klonlarında, CI loglarında
olabilir. **Birinci adım her zaman şifreyi iptal edip yenisini üretmek.**

---

## Senaryo 4 — Merge conflict çözdüm

`main`'de `renk.txt` içine KIRMIZI, `feature/renk`'te MAVI yazdım. Merge
edince Git karar veremedi:

```
CONFLICT (add/add): Merge conflict in renk.txt
```

Dosyanın içi şöyleydi:

```
<<<<<<< HEAD
KIRMIZI
=======
MAVI
>>>>>>> feature/renk
```

Üstteki blok bulunduğum dalın, alttaki getirilen dalın versiyonu.

### Burada da hata yaptım

İşaretleri silerken sadece `<<<<<<<` ve `>>>>>>>` sembollerini sildim, ama
`HEAD` ve `feature/renk` kelimeleri dosyada kaldı. Çatışma artığını
commit'lemiş oldum.

İşaretler aslında **tam satır**: `<<<<<<< HEAD` satırının tamamı silinmeliydi.

İkinci bir commit ile düzelttim. Bundan sonra `.pre-commit-config.yaml`'a
`check-merge-conflict` hook'unu ekledim — aynı hatayı bir daha yapmayayım diye.
Disiplini insana değil araca yaptırmak daha güvenli.

**Merge sonrası grafik:**

```
*   7d9498d Merge branch 'feature/renk'
|\
| * bf80a94 feat: renk mavi
* | 9251d11 feat: renk kirmizi
|/
* da034a1 feat: notlar dosyasi eklendi
```

Merge commit'in **iki ebeveyni** var. Rebase olsaydı bu Y şekli düz bir çizgi
olurdu — ikisi arasındaki fark tam olarak bu.

---

## Senaryo 5 — `git bisect` ile hatayı buldum

10 commit'lik bir geçmiş oluşturdum, 5. commit'e gizli bir bug koydum.
`hesap.sh` 4 yazması gerekirken 5 yazıyordu.

Elle tek tek denemek yerine otomatik aramayı kullandım:

```
git bisect start HEAD <892017a>
git bisect run bash -c '[ "$(./hesap.sh)" = "4" ]'
```

```
5a62ee6 is the first bad commit
chore: degisiklik 5
```

**3 testte buldu.** `bisect` ikili arama yapıyor, her adımda kalan aralığı
yarıya bölüyor. 10 commit için 3 test, 1000 commit için 10 test yeterli olurdu.

Sonunda `git bisect reset` ile normale döndüm.

**Bağlantı:** `bisect run`'a verdiğim komut burada küçük bir kabuk kontrolüydü.
Gerçek projede orada `pytest` çalışır. Yani testim varsa Git hatayı benim
yerime buluyor — Ödev 1.1'de test yazmanın karşılığı bu.

---

## Bonus: reflog

Senaryolar sırasında `git reflog`'a baktım ve o güne kadar yaptığım her
hareketi orada gördüm:

```
046431b HEAD@{0}: rebase (start): checkout HEAD~3
c94ce82 HEAD@{1}: commit: wip: son
046431b HEAD@{4}: reset: moving to HEAD~1
1059e92 HEAD@{6}: cherry-pick: feat: login ozelligi
f2d2813 HEAD@{8}: commit: feat: login ozelligi
```

`reset --hard` ile "sildiğim" `f2d2813` hâlâ duruyordu. İstesem geri
dönebilirdim.

**Sonuç:** Commit'lenmiş hiçbir şey gerçekten kaybolmuyor. Bu yüzden
`reset --hard`'dan korkmama gerek yok. Ama `git add` yapmadığım çalışma
reflog'a girmiyor — o gerçekten gidiyor.

---

## Özet

| Senaryo | Aklımda kalan |
|---|---|
| Yanlış dala commit | `cherry-pick` kopyalar, `reset` etiketi taşır |
| Squash | Editörde sadece ilk kelimeyi değiştir; `--abort` kaçış tuşu |
| `.env` temizliği | Önce şifreyi iptal et, sonra geçmişi temizle |
| Merge conflict | İşaret satırlarının tamamı silinir; hook ile otomatikleştir |
| `bisect` | Test varsa Git hatayı senin yerine bulur |

En çok işime yarayan iki komut: `git status` (neredeyim) ve `git reflog`
(nereden geldim).
