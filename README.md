# Hydra — Kapsamlı Parametre ve Kullanım Rehberi

## 1. Hydra Nedir?

Hydra (THC-Hydra), çok sayıda protokolü destekleyen, paralel çalışabilen bir **brute-force / credential testing** aracıdır. SSH, FTP, HTTP(S) form/basic auth, SMB, RDP, MySQL, PostgreSQL, SMTP, VNC, Telnet gibi onlarca servise karşı kullanıcı adı/parola denemesi yapar.

Temel yapı:

```
hydra -l KULLANICI -P PAROLA_LISTESI HEDEF SERVIS
```

Örneğin:

```
hydra -l admin -P rockyou.txt ssh://lab.local
```

Burada:

```
hydra
│
├── -l admin           → sabit kullanıcı adı
├── -P rockyou.txt     → denenecek parola listesi
└── ssh://lab.local    → hedef + servis
```

nuclei önceden yazılmış YAML template'leri çalıştırırken, hydra tek bir işi (kimlik doğrulama denemesi) binlerce kombinasyonla tekrarlar — sqlmap'in payload denemesine, nuclei'nin ise template eşleştirmesine benzer şekilde, hydra'nın mantığı "doğru kullanıcı/parola kombinasyonunu paralel ve hızlı şekilde tara"dır.

## 2. Hydra'nın Mantığını Anlamak

```
Kullanıcı listesi x Parola listesi
     |
     v
Her kombinasyon için bir login denemesi oluşturulur
     |
     v
Deneme, belirtilen servise (SSH, FTP, HTTP...) gönderilir
     |
     v
Servisin cevabı başarı/başarısızlık kriterine göre değerlendirilir
     (HTTP için "fail string", SSH/FTP için protokolün kendi yanıt kodu...)
     |
     v
Başarılı ise → bulgu raporlanır
Başarısız ise → sıradaki kombinasyona geçilir
```

Bu nedenle hydra'yı öğrenmek, aslında doğru listeleri (kullanıcı/parola), doğru servise, doğru paralellik ve doğru başarı/başarısızlık kriteriyle uygulamayı öğrenmek demektir — nuclei'deki "hangi template/tag/severity" mantığının yerini burada "hangi liste/servis/eşleştirme kriteri" mantığı alır.

## 3. Kurulum

### Kali Linux üzerinde (genelde hazır gelir)

```
sudo apt install hydra
```

### Kaynaktan derleme

```
git clone https://github.com/vanhauser-thc/thc-hydra
cd thc-hydra
./configure
make && sudo make install
```

### GTK arayüzü (opsiyonel)

```
hydra-gtk
```

## 4. Temel Parametreler — Kullanıcı Adı Belirleme

### `-l`

**Kısa özet:** Tek, sabit bir kullanıcı adı belirtir.

```
hydra -l admin -P pass.txt ssh://lab.local
```

### `-L`

**Kısa özet:** Kullanıcı adlarını içeren bir dosya (satır satır) belirtir.

```
hydra -L kullanicilar.txt -P pass.txt ssh://lab.local
```

**Günlük kullanım:** Recon'da bulunan olası kullanıcı adlarını (isim.soyisim, e-posta öncesi kısım vb.) tek seferde denemek için kullanılır.

## 5. Parola Belirleme

### `-p`

**Kısa özet:** Tek, sabit bir parola belirtir.

```
hydra -l admin -p Summer2024! ssh://lab.local
```

### `-P`

**Kısa özet:** Parolaları içeren bir dosya (satır satır) belirtir.

```
hydra -l admin -P rockyou.txt ssh://lab.local
```

### `-C`

**Kısa özet:** `kullanici:parola` formatında kombinasyon dosyası kullanır (`-l`/`-L` ve `-p`/`-P` ile birlikte kullanılmaz).

```
hydra -C kombinasyonlar.txt ssh://lab.local
```

**Dikkat:** Sızmış/leak olmuş gerçek kombinasyon listeleri (örn. credential stuffing listeleri) kullanmak, yetkisiz hesaplara erişim denemesi anlamına gelebilir; yalnızca izinli ortamlarda ve kendi test verinle kullan.

## 6. Hedef ve Servis Belirleme

### Hedef URL formatı

```
hydra [seçenekler] servis://hedef[:port]
```

Örnek:

```
hydra -l admin -P pass.txt ftp://lab.local
hydra -l admin -P pass.txt ssh://lab.local:2222
```

### `-M`

**Kısa özet:** Birden fazla hedefi içeren bir dosya belirtir (nuclei'deki `-l` hedef listesine benzer, hydra'da bu rol `-M`'dedir).

```
hydra -l admin -P pass.txt -M hedefler.txt ssh
```

### `-s`

**Kısa özet:** Varsayılan olmayan bir port belirtir.

```
hydra -l admin -P pass.txt -s 2222 ssh://lab.local
```

## 7. Desteklenen Servis Modülleri (sqlmap'teki DBMS, nuclei'deki protokol karşılığı)

En sık kullanılanlar:

```
ssh         → SSH kimlik doğrulama
ftp         → FTP kimlik doğrulama
smb         → Windows paylaşım/SMB
rdp         → Uzak masaüstü
telnet      → Telnet
mysql       → MySQL veritabanı
postgres    → PostgreSQL veritabanı
smtp        → E-posta sunucusu (auth)
vnc         → VNC uzak erişim
http-get / http-post-form  → Web formu / basic auth
https-get / https-post-form → TLS üzerinden web formu
```

Modül listesini görmek için:

```
hydra -U http-post-form
```

(herhangi bir modül adından sonra `-U` o modülün kendine özgü parametrelerini gösterir.)

## 8. HTTP Form Brute-Force (En Sık Karşılaşılan Senaryo)

### `http-post-form` / `http-get-form`

**Kısa özet:** Web giriş formlarına karşı brute-force yapmak için kullanılır; form alanlarını ve "başarısız giriş" ifadesini belirtmen gerekir.

**Söz dizimi:**

```
"YOL:FORM_ALANLARI:BASARISIZLIK_KOSULU"
```

**Örnek:**

```
hydra -l admin -P rockyou.txt lab.local http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:F=Invalid credentials"
```

**Mantık:**

```
/login.php                     → formun gönderildiği yol
user=^USER^&pass=^PASS^        → form alanları, ^USER^/^PASS^ yer tutucu
F=Invalid credentials          → bu metin cevapta varsa = başarısız
```

**Dikkat:** `F=` yerine `S=` ile "başarı koşulu" da tanımlanabilir (örn. `S=Dashboard` sayfasına yönlenmişse başarılı sayılır). Yanlış koşul seçimi, tüm denemelerin yanlış pozitif/negatif raporlanmasına yol açar.

## 9. Hız ve Eşzamanlılık Ayarları

### `-t`

**Kısa özet:** Paralel çalışacak görev (thread) sayısını belirler (varsayılan: 16).

```
hydra -l admin -P pass.txt -t 4 ssh://lab.local
```

**Neden önemli?** sqlmap'teki `--threads`, nuclei'deki `-c` (concurrency) ile aynı mantık: hedefi aşırı yüklememek veya hesap kilitleme (account lockout) mekanizmalarını tetiklememek için düşürülür.

### `-w`

**Kısa özet:** Her bağlantı denemesi için maksimum bekleme (timeout) süresi, saniye cinsinden.

```
hydra -l admin -P pass.txt -w 30 ssh://lab.local
```

### `-W`

**Kısa özet:** Denemeler arasına ek bekleme süresi ekler.

```
hydra -l admin -P pass.txt -W 2 ssh://lab.local
```

**Dikkat:** Birçok servis belirli sayıda başarısız denemeden sonra hesabı kilitler veya IP'yi engeller; `-t` düşürmek ve `-W` eklemek, sqlmap'teki `--delay` gibi bu riski azaltır ama tamamen ortadan kaldırmaz.

## 10. Çıktı Kontrolü

### `-o`

**Kısa özet:** Bulunan sonuçları bir dosyaya yazar.

```
hydra -l admin -P pass.txt -o sonuclar.txt ssh://lab.local
```

### `-b`

**Kısa özet:** Çıktı formatını belirler (`text`, `json`, `jsonv1`).

```
hydra -l admin -P pass.txt -o sonuclar.json -b json ssh://lab.local
```

### `-v` / `-V`

**Kısa özet:**
- `-v` → ayrıntılı çıktı
- `-V` → her denemeyi (kullanıcı/parola çifti dahil) ekrana basar

```
hydra -l admin -P pass.txt -V ssh://lab.local
```

**Dikkat:** `-V` çok büyük listelerde terminali anlamsız şekilde doldurabilir; genelde küçük test listeleriyle hata ayıklamak için kullanılır.

### `-f`

**Kısa özet:** İlk başarılı kullanıcı/parola bulunduğunda taramayı durdurur.

```
hydra -l admin -P pass.txt -f ssh://lab.local
```

**Günlük kullanım:** Tek bir geçerli kombinasyon yeterliyse (örn. zafiyet kanıtlama), taramayı gereksiz uzatmamak için kullanılır.

## 11. Diğer Önemli Bayraklar

### `-e`

**Kısa özet:** Ek kontrol seçenekleri ekler:
- `n` → boş parola dene
- `s` → kullanıcı adını parola olarak da dene
- `r` → ters çevrilmiş kullanıcı adını dene

```
hydra -l admin -P pass.txt -e nsr ssh://lab.local
```

### `-x`

**Kısa özet:** Belirli bir desene göre parola **üretir** (dosya yerine).

**Söz dizimi:** `-x MIN:MAX:KARAKTER_SETI`

```
hydra -l admin -x 4:6:aA1 ssh://lab.local
```

Bu örnek, 4-6 karakter uzunluğunda, küçük harf + büyük harf + rakam içeren tüm kombinasyonları dener.

**Dikkat:** Karakter seti büyüdükçe kombinasyon sayısı katlanarak artar; gerçek dünyada çoğunlukla hazır parola listeleri (`-P`) tercih edilir.

### `-R`

**Kısa özet:** Önceki yarıda kalmış bir taramayı kaldığı yerden devam ettirir.

```
hydra -R
```

### `-I`

**Kısa özet:** Önceki oturum dosyasını yok sayıp sıfırdan başlar.

## 12. Proxy Üzerinden Çalıştırma

**Kısa özet:** `HYDRA_PROXY` ortam değişkeni ile trafiği bir proxy'ye (ör. Burp Suite) yönlendirebilirsin.

```
export HYDRA_PROXY=http://127.0.0.1:8080
hydra -l admin -P pass.txt http-post-form "/login:user=^USER^&pass=^PASS^:F=Invalid"
```

sqlmap `--proxy` / nuclei `-proxy` ile birebir aynı amaç: trafiği incelemek/kaydetmek.

## 13. Gerçek Bir Tarama Akışı — Örnek Senaryo

```bash
# 1. Hedef servisi ve portu doğrula (nmap ile)
nmap -p 22,21,80 lab.local

# 2. Olası kullanıcı adlarını topla/oluştur
echo -e "admin\nroot\ntest" > kullanicilar.txt

# 3. Küçük, kontrollü bir ilk deneme (lockout riskini görmek için)
hydra -L kullanicilar.txt -P kucuk-liste.txt -t 4 -W 2 ssh://lab.local

# 4. Sonuç yoksa, daha büyük listeyle ve dikkatli hızla devam et
hydra -L kullanicilar.txt -P rockyou.txt -t 4 -W 2 -o sonuclar.txt ssh://lab.local

# 5. Web formu varsa ayrı bir deneme
hydra -L kullanicilar.txt -P rockyou.txt lab.local http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:F=Invalid credentials" -o web-sonuclar.txt
```

## 14. Mini Cheat Sheet

```bash
# Tek kullanıcı, parola listesi
hydra -l admin -P pass.txt ssh://lab.local

# Kullanıcı listesi + parola listesi
hydra -L users.txt -P pass.txt ftp://lab.local

# Kombinasyon dosyası (user:pass)
hydra -C combo.txt ssh://lab.local

# Port belirterek
hydra -l admin -P pass.txt -s 2222 ssh://lab.local

# HTTP form brute-force
hydra -l admin -P pass.txt lab.local http-post-form \
  "/login:user=^USER^&pass=^PASS^:F=Invalid"

# Thread/hız ayarı
hydra -l admin -P pass.txt -t 4 -W 2 ssh://lab.local

# İlk bulguda dur
hydra -l admin -P pass.txt -f ssh://lab.local

# Çıktıyı dosyaya/JSON'a yaz
hydra -l admin -P pass.txt -o sonuclar.txt ssh://lab.local
hydra -l admin -P pass.txt -o sonuclar.json -b json ssh://lab.local

# Pattern ile parola üretme
hydra -l admin -x 4:6:aA1 ssh://lab.local

# Yarıda kalan taramayı devam ettirme
hydra -R
```

## 15. Önemli Parametre Grupları

**Kullanıcı belirleme**
- `-l` / `-L`

**Parola belirleme**
- `-p` / `-P`
- `-C` (kombinasyon dosyası)
- `-x` (pattern üretimi)

**Hedef belirleme**
- `servis://hedef[:port]`
- `-s` (port)
- `-M` (hedef listesi)

**Hız / performans**
- `-t` (thread sayısı)
- `-w` / `-W` (timeout / bekleme)

**Çıktı**
- `-o`
- `-b` (format)
- `-v` / `-V`
- `-f` (ilk bulguda dur)

**Diğer**
- `-e` (ek kontroller: boş/aynı/ters parola)
- `-R` / `-I` (oturum devam/sıfırlama)
- `HYDRA_PROXY` (proxy)

## 16. Öğrenme Sırası

Bütün bayrakları aynı anda ezberlemeye çalışma.

**Önce:**
```
hydra -l KULLANICI -P LISTE servis://HEDEF
```

**Sonra:**
- `-L`, `-C`
- `-s` (port)

**Sonra:**
- `-t`, `-w`, `-W` (hız/lockout dengesi)

**Sonra:**
- `-o`, `-b`, `-v`, `-f`

**Sonra:**
- `http-post-form` / `http-get-form` söz dizimi

**Son olarak:**
- `-x` (pattern üretimi)
- `-e` (ek kontroller)
- `-R` / `-I` (oturum yönetimi)
- Proxy entegrasyonu

Bunlar en fazla arka plan bilgisi ve dikkat gerektirdiği için en sona bırakılmalıdır — özellikle `-x` ve agresif thread ayarları, hedef sistemde hesap kilitlenmesine veya hizmet aksamasına yol açabilir.

## 17. Hızlı Referans Tablosu

| Parametre | Görevi |
|---|---|
| `-l` | Tek kullanıcı adı |
| `-L` | Kullanıcı listesi dosyası |
| `-p` | Tek parola |
| `-P` | Parola listesi dosyası |
| `-C` | `user:pass` kombinasyon dosyası |
| `-x` | Pattern ile parola üretimi |
| `-s` | Port belirt |
| `-M` | Hedef listesi dosyası |
| `-t` | Paralel thread sayısı |
| `-w` | Bağlantı timeout süresi |
| `-W` | Denemeler arası bekleme |
| `-e` | Ek kontrol (ns r) |
| `-f` | İlk bulguda dur |
| `-o` | Sonucu dosyaya yaz |
| `-b` | Çıktı formatı (text/json) |
| `-v` / `-V` | Ayrıntılı çıktı |
| `-R` | Yarıda kalan taramayı sürdür |
| `-I` | Oturumu sıfırla |
| `HYDRA_PROXY` | Proxy üzerinden gönder |

## 18. Güvenli Çalışma Notu

Hydra tek başına yasa dışı bir araç değildir; birçok resmi sızma testi, bug bounty programı ve CTF eğitiminin standart parçasıdır.

Ancak brute-force denemeleri hedef sistemde **hesap kilitlenmesine, hizmet kesintisine veya yetkisiz erişime** doğrudan yol açabilir. Bu nedenle yalnızca:

- kendi kurduğun lab ortamları
- HTB Academy / HTB Labs gibi yetkili platformlar
- TryHackMe gibi izinli ortamlar
- açıkça yazılı izin (scope) verilen bug bounty / sızma testleri

üzerinde kullan. İzinsiz bir sisteme karşı hydra çalıştırmak, tıpkı sqlmap ve nuclei gibi, birçok ülkede olduğu kadar Türkiye'de de suç teşkil eder (TCK madde 243-244).

## 19. Son Kontrol Listesi

Bir hydra komutu gördüğünde/yazacağında şu soruları sor:

1. Kullanıcı adı sabit mi, liste mi, kombinasyon dosyası mı?
2. Parola sabit mi, liste mi, pattern (`-x`) ile mi üretiliyor?
3. Hedef servis/port doğru belirtilmiş mi?
4. HTTP form kullanılıyorsa, form alanları ve başarı/başarısızlık koşulu (`F=`/`S=`) doğru mu?
5. Thread sayısı (`-t`) ve bekleme (`-W`) hesap kilitlenmesi riskine göre ayarlanmış mı?
6. Sonuç nereye kaydediliyor (`-o` / `-b`)?
7. Proxy üzerinden mi gidiyor (Burp ile inceleme)?
8. Yarıda kalmış bir oturum mu devam ettiriliyor (`-R`)?
9. Bu hedef üzerinde test yapmaya yetkim var mı?

## 20. Sonuç

Hydra öğrenirken en önemli şey:

```
hydra -l admin -P rockyou.txt ssh://lab.local
```

komutunu ezberlemek değildir.

Asıl önemli olan, bunun arka planda:

1. Kullanıcı/parola kaynağını belirle
2. Doğru servis/portu hedefle
3. Hız ve lockout riskini dengele
4. (HTTP ise) doğru başarı/başarısızlık koşulunu tanımla
5. Sonucu kaydet ve raporla

adımlarının otomasyonu olduğunu bilmektir. nuclei binlerce kontrolü template kütüphanesiyle genişletirken, hydra tek bir işi (kimlik doğrulama denemesi) çok sayıda kombinasyonla ve paralellikle genişletir; bu mantık oturduğunda hydra da yalnızca bir "parola deneme aracı" olmaktan çıkar, modern kimlik doğrulama güvenliği testlerinin nasıl kurgulandığını anladığın bir öğrenme aracı haline gelir.
