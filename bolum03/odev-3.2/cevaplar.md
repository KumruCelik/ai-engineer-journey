# Ödev 3.2 — Cevaplar

SQL dosyaları: `sql-mastery/queries/` altında, her sorunun numarasıyla.

---

## 01 — Hesabı kapalı kullanıcılar kimler, kaç tane?

**SQL:** `sql-mastery/queries/q01_pasif_kullanicilar.sql`

```sql
SELECT count(*) AS pasif_kullanici_sayisi
FROM users
WHERE is_active = false;

SELECT id, email, country, created_at
FROM users
WHERE is_active = false
ORDER BY created_at DESC
LIMIT 10;
```

**Sonuç:**

```
 pasif_kullanici_sayisi
------------------------
                    993
(1 row)

  id   |        email        | country |       created_at
-------+---------------------+---------+------------------------
  3491 | user3491@ornek.com  | DE      | 2026-03-30 12:28:28+00
   867 | user867@ornek.com   | TR      | 2026-03-27 21:31:45+00
 14388 | user14388@ornek.com | TR      | 2026-03-27 12:56:38+00
  2383 | user2383@ornek.com  | TR      | 2026-03-26 17:42:52+00
 13203 | user13203@ornek.com | GB      | 2026-03-26 08:52:39+00
 12217 | user12217@ornek.com | TR      | 2026-03-25 15:47:42+00
 18944 | user18944@ornek.com | TR      | 2026-03-25 14:35:43+00
  6050 | user6050@ornek.com  | NL      | 2026-03-25 13:10:35+00
    93 | user93@ornek.com    | TR      | 2026-03-23 07:21:40+00
 19478 | user19478@ornek.com | TR      | 2026-03-23 07:14:52+00
(10 rows)
```

**İş yorumu:**

20.000 kayıtlı kullanıcının 993'ü, yani %4,97'si kapalı durumda. Tek başına bu oran bir alarm değil; kıyaslanacak bir hedef ya da geçmiş dönem olmadan "yüksek mi düşük mü" sorusunun cevabı yok. Asıl sınır, sorgunun cevaplayamadığı yerde: şemada hesabın **ne zaman** kapandığını tutan bir kolon bulunmuyor. `created_at` açılış tarihidir, dolayısıyla yukarıdaki liste "en son kapanan hesaplar" değil, "en son açılmış olan kapalı hesaplar"dır. Bunun pratik sonucu şu: kapanma oranının zaman içinde artıp artmadığını ya da bir kampanya sonrası sıçrama olup olmadığını bu şemayla ölçemeyiz. Terk (churn) analizi yapılacaksa şemaya `users.deactivated_at` eklenmesi gerekir; aksi hâlde elimizde yalnızca bugünün anlık fotoğrafı var, bir eğilim yok.

**İyileştirme notu:** `users.deactivated_at timestamptz NULL` kolonu — kapanma tarihini tutar ve churn analizini mümkün kılar. Kararı Ödev 3.4'e (analitik katman) bırakıyoruz; orada SCD2 ile birlikte değerlendirilecek.

---

## 02 — Ülkesi bilinmeyen kullanıcı sayısı kaç?

**SQL:** `sql-mastery/queries/q02_ulkesi_bilinmeyen.sql`

**Sonuç:**

```
 yanlis_yontem          <- WHERE country = NULL
---------------
             0

 ulkesi_bilinmeyen      <- WHERE country IS NULL
-------------------
               965

 toplam_kullanici | ulkesi_dolu | ulkesi_bos
------------------+-------------+------------
            20000 |       19035 |        965

 bos_yuzde
-----------
      4.83

    ulke    | kullanici
------------+-----------
 TR         |     13423
 DE         |      1869
 US         |       972
 bilinmiyor |       965
 NL         |       935
 FR         |       932
 GB         |       904
```

**İş yorumu:**

Kullanıcıların %4,83'ünün (965 kişi) ülkesi kayıtlı değil. Bu oranın kendisinden daha önemlisi, bu kitlenin raporlarda nasıl davrandığı: `bilinmiyor` grubu 965 kişiyle Hollanda, Fransa ve Birleşik Krallık kullanıcılarının her birinden daha kalabalık. Ülke kırılımlı bir rapor `WHERE country IS NOT NULL` ile ya da sessizce bir `GROUP BY country` ile hazırlandığında bu 965 kişi tabloda hiç görünmez ve toplamlar, ana tablodaki kullanıcı sayısıyla tutmaz. Bu tür raporlarda iki seçenek var: ya `COALESCE(country, 'bilinmiyor')` ile grubu açıkça göstermek, ya da hariç tutup bunu raporun başında yazmak. Kabul edilemez olan üçüncü yol, eksik ülkeleri baskın değere (burada TR) doldurmak olurdu — bu, veriyi temizlemek değil uydurmaktır ve ülke bazlı her kararı bozar.

Metodolojik not: `WHERE country = NULL` sorgusu hata vermeden 0 döndürdü. `NULL` ile yapılan eşitlik karşılaştırmasının sonucu `TRUE` değil `NULL`'dur, `WHERE` ise yalnızca `TRUE` olan satırları geçirir. Bu yüzden eksik veri sorgulanırken tek doğru araç `IS NULL` / `IS NOT NULL`'dır. Aynı kural `count(*)` ile `count(kolon)` arasındaki farkı da açıklar: `count(kolon)` `NULL` satırları saymaz, ikisinin farkı doğrudan eksik veri sayısını verir.
