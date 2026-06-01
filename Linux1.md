# Linux pilet 1 ülesannete lahenduskäik

See juhend keskendub ainult **Linux pilet 1 eriosa ülesannetele**.

Siin ei ole lahti kirjutatud üldist Linuxi baasosa:

- Ansible paigaldus UbuntuServerisse
- Ansible inventory loomine
- `hkhk` kasutaja loomine
- sudo gruppi lisamine
- SSH võtmega ligipääs
- SSH parooliga sisselogimise keelamine

---

## Pilet 1 põhiteemad

| Teema | Mida tuleb teha |
|---|---|
| DebianPilet1 | Taastada ligipääs ja teenused |
| WordPress | Taastada WordPressi sisuhaldussüsteem |
| Apache | Taastada veebiserveri töö |
| MySQL/MariaDB | Taastada andmebaasi töö |
| Debian upgrade | Uuendada Debian 11.9 versioonile Debian 13.5 |
| SSL | Seadistada veebilehele HTTPS |
| UFW | Lubada ainult vajalikud pordid |
| Vaultwarden | Paigaldada paroolihaldur UbuntuServerisse |
| DNS | Luua FQDN kirjed teenustele |
| Dokumentatsioon | Dokumenteerida kogu protsess |

---

# 1. Algkontroll DebianPilet1 serveris

## Eesmärk

Kõigepealt tuleb aru saada:

- mis serveriga on tegu
- mis IP-aadress serveril on
- mis Debian versioon on paigaldatud
- kas Apache töötab
- kas andmebaas töötab
- kus asub WordPress
- mis pordid on avatud

---

## 1.1 Kontrolli hostname’i

```bash
hostname
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse serveri nimi, näiteks `debianpilet1` | Kui nimi on vale või segane, dokumenteeri praegune nimi. Vajadusel muuda hostname hiljem käsuga `sudo hostnamectl set-hostname debianpilet1` |

Kontrolli FQDN-i:

```bash
hostname -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse täielik nimi, näiteks `debianpilet1.sinuNimi.local` | Kui tuleb error või ainult hostname, siis FQDN pole õigesti seadistatud. Kontrolli `/etc/hosts` ja DNS kirjeid |

---

## 1.2 Kontrolli IP-aadressi

```bash
ip a
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed võrguliidesel IP-aadressi, näiteks `10.x.x.x/24` | Kui IP puudub, kontrolli võrguühendust, DHCP-d või staatilist IP seadistust |

Kontrolli gateway’d:

```bash
ip route
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `default via ...` rida | Kui default route puudub, ei pruugi server saada internetti ega teistesse võrkudesse |

---

## 1.3 Kontrolli Debiani versiooni

```bash
cat /etc/os-release
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed Debiani versiooni infot, alguses tõenäoliselt Debian 11 | Kui fail puudub või süsteem pole Debian, dokumenteeri tulemus ja kontrolli `cat /etc/debian_version` |

```bash
cat /etc/debian_version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Alguses võib olla näiteks `11.9` | Kui versioon on juba uuem, dokumenteeri see. Kui versioon on vanem/katki, tee enne uuendamist paketisüsteemi kontroll |

---

# 2. Kontrolli Apache, MySQL/MariaDB ja WordPressi olemasolu

## 2.1 Kontrolli Apache teenust

```bash
systemctl status apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui `inactive`, käivita `sudo systemctl start apache2`. Kui `failed`, kontrolli `sudo apache2ctl configtest` ja logisid |

Kui Apache ei tööta, proovi käivitada:

```bash
sudo systemctl start apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna viga | Kui tuleb viga, käivita `sudo apache2ctl configtest` |

Kontrolli Apache konfiguratsiooni:

```bash
sudo apache2ctl configtest
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Syntax OK` | Kui näitab veaga faili ja rea numbrit, ava see fail `sudo nano FAILINIMI` ja paranda süntaks |

---

## 2.2 Kontrolli MySQL/MariaDB teenust

Proovi MariaDB staatust:

```bash
systemctl status mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui teenust pole, proovi `systemctl status mysql`. Kui teenus on `failed`, vaata logi `sudo journalctl -xeu mariadb` |

Proovi MySQL staatust:

```bash
systemctl status mysql
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` või info, et teenust pole | Kui ei ole MySQL, kasutatakse tõenäoliselt MariaDB-d |

Kontrolli versiooni:

```bash
mysql --version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse MySQL või MariaDB versioon | Kui `command not found`, paigalda andmebaasiserver hiljem käsuga `sudo apt install mariadb-server -y` |

---

## 2.3 Kontrolli veebikausta

```bash
ls -lah /var/www/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed näiteks `html` kausta | Kui `/var/www` puudub, ei pruugi Apache/WordPress paigaldatud olla |

```bash
ls -lah /var/www/html/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed WordPressi faile, näiteks `wp-config.php`, `wp-content`, `wp-admin` | Kui näed ainult `index.html`, siis WordPress võib olla muus kaustas või pole paigaldatud |

Otsi WordPressi konfiguratsioonifaili:

```bash
sudo find /var/www -name wp-config.php
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Leitakse fail, näiteks `/var/www/html/wp-config.php` | Kui ei leia, siis WordPress pole selles asukohas või fail on kustutatud. Otsi laiemalt: `sudo find / -name wp-config.php 2>/dev/null` |

---

# 3. Kontrolli WordPressi andmebaasi seadeid

## 3.1 Ava WordPressi konfiguratsioon

Kui `wp-config.php` asub `/var/www/html` all:

```bash
sudo grep DB_ /var/www/html/wp-config.php
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST` väärtuseid | Kui fail puudub, leia õige asukoht `sudo find / -name wp-config.php 2>/dev/null` |

Näide, mida otsid:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wordpressuser' );
define( 'DB_PASSWORD', 'parool' );
define( 'DB_HOST', 'localhost' );
```

Ava fail vajadusel muutmiseks:

```bash
sudo nano /var/www/html/wp-config.php
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail avaneb tekstiredaktoris | Kui tuleb `No such file`, on fail teises asukohas |

---

# 4. Taasta andmebaasi ligipääs

## 4.1 Logi andmebaasi root õigustes

```bash
sudo mysql
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Avaneb MySQL/MariaDB käsurida `MariaDB [(none)]>` või `mysql>` | Kui tuleb ligipääsu viga, proovi `sudo mariadb`. Kui teenus ei tööta, käivita `sudo systemctl restart mariadb` |

Alternatiiv:

```bash
sudo mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Avaneb MariaDB käsurida | Kui käsk puudub, kontrolli andmebaasi paigaldust |

---

## 4.2 Kontrolli andmebaase

Andmebaasi sees:

```sql
SHOW DATABASES;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed WordPressi andmebaasi, näiteks `wordpress` | Kui WordPressi andmebaasi pole, kontrolli `wp-config.php` faili `DB_NAME` väärtust |

Vali WordPressi andmebaas:

```sql
USE wordpress;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Database changed` | Kui tuleb `Unknown database`, on andmebaasi nimi vale või andmebaas puudub |

Kontrolli tabelid:

```sql
SHOW TABLES;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed tabeleid nagu `wp_users`, `wp_posts`, `wp_options` | Kui tabeleid pole, võib andmebaas tühi või vale olla |

---

## 4.3 Kontrolli andmebaasi kasutajaid

```sql
SELECT user, host FROM mysql.user;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed WordPressi kasutajat, näiteks `wordpressuser` | Kui kasutajat pole, loo kasutaja või paranda `wp-config.php` vastavalt olemasolevale kasutajale |

---

## 4.4 Muuda WordPressi andmebaasi kasutaja parool

Näide:

```sql
ALTER USER 'wordpressuser'@'localhost' IDENTIFIED BY 'UusTugevParool123!';
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk läheb läbi ilma errorita | Kui tuleb viga, kontrolli kasutajanime ja hosti käsuga `SELECT user, host FROM mysql.user;` |

Rakenda õigused:

```sql
FLUSH PRIVILEGES;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Query OK` | Kui tuleb viga, kontrolli, kas oled andmebaasis root õigustes |

Välju:

```sql
EXIT;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Jõuad tagasi Linuxi käsureale | Kui ei välju, kasuta `\q` |

Muuda sama parool WordPressi konfiguratsioonis:

```bash
sudo nano /var/www/html/wp-config.php
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Saad muuta `DB_PASSWORD` väärtuse samaks | Kui fail on teises asukohas, kasuta eelnevalt leitud `wp-config.php` teed |

---

## 4.5 Testi andmebaasi kasutajat

```bash
mysql -u wordpressuser -p wordpress
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Pärast parooli sisestamist saad andmebaasi sisse | Kui tuleb `Access denied`, on parool, kasutaja või host vale. Kontrolli uuesti `wp-config.php` ja `mysql.user` tabelit |

Välju:

```sql
EXIT;
```

---

# 5. Taasta WordPressi admin ligipääs

## Variant A: WP-CLI abil

Kontrolli, kas WP-CLI on olemas:

```bash
wp --info
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse WP-CLI info | Kui `command not found`, kasuta MySQL/MariaDB varianti |

Mine WordPressi kausta:

```bash
cd /var/www/html
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Oled WordPressi kaustas | Kui kaust pole õige, mine sinna, kus asub `wp-config.php` |

Kuva WordPressi kasutajad:

```bash
sudo -u www-data wp user list
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed WordPressi kasutajaid | Kui tuleb andmebaasi error, kontrolli `wp-config.php` ja andmebaasi ühendust |

Muuda admin parool:

```bash
sudo -u www-data wp user update admin --user_pass='UusAdminParool123!'
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kasutaja parool muudetakse | Kui kasutajat `admin` pole, vaata õige kasutajanimi käsuga `wp user list` |

---

## Variant B: MySQL/MariaDB kaudu

Logi andmebaasi:

```bash
sudo mysql
```

Vali andmebaas:

```sql
USE wordpress;
```

Kuva kasutajad:

```sql
SELECT ID, user_login, user_email FROM wp_users;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed WordPressi kasutajanimesid | Kui tabelit `wp_users` pole, võib tabeliprefix olla teine. Kontrolli `wp-config.php` failist `$table_prefix` väärtust |

Muuda admin parool:

```sql
UPDATE wp_users
SET user_pass = MD5('UusAdminParool123!')
WHERE user_login = 'admin';
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Query OK`, vähemalt 1 rida muudetud | Kui 0 rida muutus, pole kasutajanimi `admin`. Kasuta eelnevas käsus nähtud kasutajanime |

Välju:

```sql
EXIT;
```

---

# 6. Taasta Apache veebiserveri töö

## 6.1 Paigalda vajalikud paketid

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse ilma errorita | Kui tuleb repo error, kontrolli `/etc/apt/sources.list` faili |

```bash
sudo apt install apache2 php php-mysql mariadb-server -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketid paigaldatakse või on juba olemas | Kui paketid ei leidu, on repo vale või internet/DNS ei tööta |

Lisa PHP moodulid:

```bash
sudo apt install php-cli php-curl php-gd php-mbstring php-xml php-zip -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Moodulid paigaldatakse | Kui mõni pakett puudub, jätka olemasolevatega ja dokumenteeri puuduv pakett |

---

## 6.2 Käivita teenused

```bash
sudo systemctl enable apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache lubatakse käivitumisel | Kui teenust pole, paigalda `apache2` |

```bash
sudo systemctl restart apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk lõpeb veata | Kui tuleb error, käivita `sudo apache2ctl configtest` |

```bash
sudo systemctl enable mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| MariaDB lubatakse käivitumisel | Kui teenust pole, kontrolli kas kasutusel on `mysql` |

```bash
sudo systemctl restart mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| MariaDB käivitub | Kui ei käivitu, vaata `sudo journalctl -xeu mariadb` |

---

## 6.3 Kontrolli veebilehte

```bash
curl -I http://localhost
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Vastus `HTTP/1.1 200 OK` või `301/302` | Kui tuleb `Connection refused`, Apache ei kuula. Kontrolli `systemctl status apache2` |

Kontrolli, kas port 80 kuulab:

```bash
sudo ss -tulpen | grep :80
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `apache2` protsessi pordil 80 | Kui midagi ei näe, Apache ei kuula porti 80 või teenus ei tööta |

---

## 6.4 Paranda WordPressi failiõigused

```bash
sudo chown -R www-data:www-data /var/www/html
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna viga | Kui tuleb `No such file`, on WordPress teises kaustas |

```bash
sudo find /var/www/html -type d -exec chmod 755 {} \;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaustade õigused seatakse | Kui tuleb veateade, kontrolli kausta olemasolu |

```bash
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failide õigused seatakse | Kui tuleb veateade, kontrolli kausta olemasolu |

Taaskäivita Apache:

```bash
sudo systemctl restart apache2
```

---

# 7. Uuenda Debian 11.9 versioonile Debian 13.5

## Oluline

Ära tee uuendust otse 11 → 13.

Turvalisem järjekord:

```text
Debian 11 → Debian 12 → Debian 13
```

Enne uuendamist tee Proxmoxis snapshot või varukoopia.

---

## 7.1 Kontroll enne uuendamist

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/` partitsioonil on piisavalt vaba ruumi | Kui ruum on täis, tee `sudo apt autoremove --purge -y` ja puhasta logisid |

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui repo error, kontrolli `/etc/apt/sources.list` |

```bash
sudo apt full-upgrade -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Praeguse versiooni paketid uuendatakse | Kui tuleb katkine pakett, käivita `sudo apt --fix-broken install` |

```bash
sudo apt autoremove --purge -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Vanad paketid eemaldatakse | Kui midagi ei eemaldata, on ka okei |

Tee sources.list varukoopia:

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Varukoopia luuakse | Kui fail puudub, kontrolli `/etc/apt/sources.list.d/` kausta |

---

## 7.2 Uuendus Debian 11 → Debian 12

Asenda `bullseye` sõnaga `bookworm`:

```bash
sudo sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna väljundit | Kui failis pole `bullseye`, ava fail `cat /etc/apt/sources.list` ja vaata, mis release seal on |

Kontrolli faili:

```bash
cat /etc/apt/sources.list
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failis on `bookworm` | Kui on ikka `bullseye`, muuda käsitsi `sudo nano /etc/apt/sources.list` |

Uuenda paketiloend:

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend tuleb Debian 12 repost | Kui tuleb GPG/repo error, kontrolli sources.list ridu |

Tee minimaalne upgrade:

```bash
sudo apt upgrade --without-new-pkgs -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Osa pakette uuendatakse | Kui tuleb error, käivita `sudo apt --fix-broken install` |

Tee täielik upgrade:

```bash
sudo apt full-upgrade -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Süsteem uuendatakse Debian 12 peale | Kui küsib config failide kohta, vali üldiselt `keep the local version`, kui oled ebakindel |

Puhasta:

```bash
sudo apt autoremove --purge -y
```

Restart:

```bash
sudo reboot
```

Pärast restarti kontrolli:

```bash
cat /etc/debian_version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse Debian 12 versioon | Kui näitab ikka 11, siis upgrade ei lõppenud. Käivita uuesti `sudo apt full-upgrade -y` |

---

## 7.3 Uuendus Debian 12 → Debian 13

Asenda `bookworm` sõnaga `trixie`:

```bash
sudo sed -i 's/bookworm/trixie/g' /etc/apt/sources.list
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna väljundit | Kui failis pole `bookworm`, kontrolli `cat /etc/apt/sources.list` |

Kontrolli faili:

```bash
cat /etc/apt/sources.list
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failis on `trixie` | Kui on ikka `bookworm`, muuda käsitsi `sudo nano /etc/apt/sources.list` |

Uuenda paketiloend:

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend tuleb Debian 13 repost | Kui tuleb repo error, kontrolli sources.list ridu |

Tee minimaalne upgrade:

```bash
sudo apt upgrade --without-new-pkgs -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Esmased paketid uuendatakse | Kui tuleb sõltuvuste viga, käivita `sudo apt --fix-broken install` |

Tee täielik upgrade:

```bash
sudo apt full-upgrade -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Süsteem uuendatakse Debian 13 peale | Kui katkeb, tee `sudo dpkg --configure -a`, siis korda `sudo apt full-upgrade -y` |

Puhasta:

```bash
sudo apt autoremove --purge -y
```

Restart:

```bash
sudo reboot
```

Pärast restarti:

```bash
cat /etc/debian_version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse Debian 13.x, ülesande järgi eesmärk 13.5 | Kui pole 13, uuendus ei lõppenud või repo jäi valeks |

```bash
cat /etc/os-release
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `VERSION_ID="13"` või sarnane | Kui näitab 12 või 11, kontrolli sources.list ja korda upgrade |

---

# 8. Uuenda MySQL/MariaDB

## 8.1 Kontrolli versiooni

```bash
mysql --version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse MySQL/MariaDB versioon | Kui `command not found`, paigalda `sudo apt install mariadb-server mariadb-client -y` |

```bash
mariadb --version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse MariaDB versioon | Kui käsku pole, võib kasutada `mysql --version` |

---

## 8.2 Paigalda/uuenda MariaDB

```bash
sudo apt install mariadb-server mariadb-client -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| MariaDB paigaldatakse või uuendatakse | Kui tuleb repo error, kontrolli apt allikaid |

Käivita upgrade tööriist:

```bash
sudo mariadb-upgrade
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Andmebaasi süsteemitabelid uuendatakse | Kui käsku pole, proovi `sudo mysql_upgrade` |

Alternatiiv:

```bash
sudo mysql_upgrade
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Upgrade lõpeb edukalt või ütleb, et pole vajalik | Kui tuleb ühenduse viga, kontrolli `sudo systemctl status mariadb` |

Taaskäivita andmebaas:

```bash
sudo systemctl restart mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Teenus käivitub | Kui `failed`, vaata `sudo journalctl -xeu mariadb` |

Kontroll:

```bash
sudo systemctl status mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui mitte, kontrolli logisid ja konfiguratsiooni |

---

# 9. Seadista Apache SSL sertifikaat

## 9.1 Luba vajalikud moodulid

```bash
sudo a2enmod ssl
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Moodul lubatakse või öeldakse, et juba lubatud | Kui käsk puudub, pole Apache õigesti paigaldatud |

```bash
sudo a2enmod rewrite
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Moodul lubatakse või on juba lubatud | Kui tuleb error, kontrolli Apache paigaldust |

Taaskäivita Apache:

```bash
sudo systemctl restart apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache käivitub | Kui failed, käivita `sudo apache2ctl configtest` |

---

## 9.2 Loo sertifikaadi kaust

```bash
sudo mkdir -p /etc/ssl/localcerts
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse või oli juba olemas | Kui permission denied, kasuta `sudo` |

---

## 9.3 Loo self-signed sertifikaat

Asenda FQDN enda nimega, näiteks:

```text
veeb.sinuNimi.local
```

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/localcerts/veeb.key \
-out /etc/ssl/localcerts/veeb.crt
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Luuakse `veeb.key` ja `veeb.crt` | Kui openssl puudub, paigalda `sudo apt install openssl -y` |

Kontrolli faile:

```bash
ls -lah /etc/ssl/localcerts/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `veeb.key` ja `veeb.crt` | Kui faile pole, korda openssl käsku |

---

## 9.4 Loo Apache HTTPS konfiguratsioon

```bash
sudo nano /etc/apache2/sites-available/wordpress-ssl.conf
```

Lisa sisu:

```apache
<VirtualHost *:443>
    ServerName veeb.sinuNimi.local
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/ssl/localcerts/veeb.crt
    SSLCertificateKeyFile /etc/ssl/localcerts/veeb.key

    <Directory /var/www/html>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/wordpress_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/wordpress_ssl_access.log combined
</VirtualHost>
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei saa salvestada, kontrolli, et kasutasid `sudo nano` |

Luba sait:

```bash
sudo a2ensite wordpress-ssl.conf
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Sait lubatakse | Kui failinimi vale, kontrolli `ls /etc/apache2/sites-available/` |

Kontrolli Apache konfiguratsiooni:

```bash
sudo apache2ctl configtest
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Syntax OK` | Kui näitab viga, paranda viidatud fail ja rida |

Laadi Apache uuesti:

```bash
sudo systemctl reload apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache laeb seadistuse uuesti | Kui tuleb error, tee `sudo systemctl restart apache2` ja vaata logi |

Testi HTTPS:

```bash
curl -k -I https://veeb.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `HTTP/1.1 200 OK`, `301` või `302` | Kui nimi ei lahendu, lisa DNS kirje. Kui connection refused, kontrolli porti 443 ja Apache staatust |

---

# 10. Seadista UFW tulemüür DebianPilet1 serveris

## 10.1 Paigalda UFW

```bash
sudo apt install ufw -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW paigaldatakse või on juba olemas | Kui paketti ei leita, kontrolli apt repo ja internetti |

---

## 10.2 Määra vaikereeglid

```bash
sudo ufw default deny incoming
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Sissetulev liiklus keelatakse vaikimisi | Kui UFW käsku pole, paigalda UFW |

```bash
sudo ufw default allow outgoing
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Väljaminev liiklus lubatakse | Kui error, kontrolli UFW paigaldust |

---

## 10.3 Luba vajalikud pordid

```bash
sudo ufw allow 22/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SSH port lubatakse | Kui kasutad teist SSH porti, luba õige port enne UFW sisselülitamist |

```bash
sudo ufw allow 80/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP port lubatakse | Kui pole vaja HTTP-d, võib hiljem eemaldada |

```bash
sudo ufw allow 443/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTPS port lubatakse | Kui ei tööta, kontrolli UFW staatust |

---

## 10.4 Lülita UFW sisse

```bash
sudo ufw enable
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW aktiveerub | Kui oled SSH-ga sees ja port 22 pole lubatud, võid ühenduse kaotada. Kontrolli enne `sudo ufw status` |

Kontrolli reegleid:

```bash
sudo ufw status numbered
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed lubatud porte 22, 80 ja 443 | Kui mõni puudub, lisa vastav `sudo ufw allow PORT/tcp` |

---

# 11. Paigalda Vaultwarden UbuntuServerisse

## 11.1 Paigalda Docker

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui DNS/repo error, kontrolli võrku ja DNS-i |

```bash
sudo apt install docker.io docker-compose-plugin -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Docker ja Compose plugin paigaldatakse | Kui pakette ei leita, kontrolli Ubuntu repo seadistust |

Luba Docker:

```bash
sudo systemctl enable docker
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Docker lubatakse käivitumisel | Kui teenust pole, kontrolli Docker paigaldust |

Käivita Docker:

```bash
sudo systemctl start docker
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Docker käivitub | Kui failed, vaata `sudo journalctl -xeu docker` |

Kontrolli Dockerit:

```bash
docker --version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse Dockeri versioon | Kui `command not found`, paigaldus ei õnnestunud |

```bash
sudo docker ps
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse konteinerite tabel, isegi kui tühi | Kui daemon error, Docker ei tööta |

---

## 11.2 Loo Vaultwardeni kaust

```bash
sudo mkdir -p /opt/vaultwarden
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse | Kui permission denied, kasuta `sudo` |

```bash
cd /opt/vaultwarden
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Oled `/opt/vaultwarden` kaustas | Kui kausta pole, loo see eelmise käsuga |

---

## 11.3 Loo docker-compose fail

```bash
sudo nano docker-compose.yml
```

Lisa:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    ports:
      - "8080:80"
    volumes:
      - ./vw-data:/data
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei saa salvestada, kontrolli, et oled õiges kaustas ja kasutasid sudo |

Käivita Vaultwarden:

```bash
sudo docker compose up -d
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Konteiner luuakse ja käivitub | Kui image download ei õnnestu, kontrolli internetti/DNS-i |

Kontrolli konteinerit:

```bash
sudo docker ps
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed konteinerit `vaultwarden` staatusega `Up` | Kui konteiner puudub või exited, vaata `sudo docker logs vaultwarden` |

Testi lokaalselt:

```bash
curl -I http://127.0.0.1:8080
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP vastus Vaultwardenilt | Kui connection refused, konteiner ei tööta või port pole seotud |

---

# 12. Seadista Vaultwardenile Apache reverse proxy

## 12.1 Paigalda Apache

```bash
sudo apt install apache2 -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache paigaldatakse | Kui paketti ei leita, kontrolli apt repo |

Luba vajalikud moodulid:

```bash
sudo a2enmod proxy
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Proxy moodul lubatakse | Kui Apache puudub, paigalda apache2 |

```bash
sudo a2enmod proxy_http
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP proxy moodul lubatakse | Kui error, kontrolli Apache mooduleid |

```bash
sudo a2enmod headers
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Headers moodul lubatakse | Kui error, kontrolli Apache paigaldust |

```bash
sudo a2enmod ssl
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SSL moodul lubatakse | Kui error, paigalda/taasta Apache |

Taaskäivita Apache:

```bash
sudo systemctl restart apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache käivitub | Kui failed, kontrolli `sudo apache2ctl configtest` |

---

## 12.2 Loo Vaultwardeni sertifikaat

```bash
sudo mkdir -p /etc/ssl/localcerts
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust on olemas | Kui permission denied, kasuta sudo |

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/localcerts/paroolihaldus.key \
-out /etc/ssl/localcerts/paroolihaldus.crt
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Tekivad sertifikaadi ja võtme failid | Kui openssl puudub, paigalda `sudo apt install openssl -y` |

---

## 12.3 Loo Apache reverse proxy konfiguratsioon

```bash
sudo nano /etc/apache2/sites-available/vaultwarden.conf
```

Lisa:

```apache
<VirtualHost *:443>
    ServerName paroolihaldus.sinuNimi.local

    SSLEngine on
    SSLCertificateFile /etc/ssl/localcerts/paroolihaldus.crt
    SSLCertificateKeyFile /etc/ssl/localcerts/paroolihaldus.key

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/

    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"

    ErrorLog ${APACHE_LOG_DIR}/vaultwarden_error.log
    CustomLog ${APACHE_LOG_DIR}/vaultwarden_access.log combined
</VirtualHost>
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei saa salvestada, kontrolli õiguseid |

Luba sait:

```bash
sudo a2ensite vaultwarden.conf
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Sait lubatakse | Kui failinimi vale, kontrolli `ls /etc/apache2/sites-available/` |

Kontrolli konfiguratsiooni:

```bash
sudo apache2ctl configtest
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Syntax OK` | Kui näitab viga, paranda viidatud rida |

Laadi Apache uuesti:

```bash
sudo systemctl reload apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache laeb seadistuse | Kui error, tee `sudo systemctl status apache2` |

Testi Vaultwardenit:

```bash
curl -k -I https://paroolihaldus.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP vastus Vaultwardenilt | Kui nimi ei lahendu, lisa DNS kirje. Kui 502, kontrolli Docker konteinerit |

---

# 13. Seadista UFW UbuntuServeris

```bash
sudo apt install ufw -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW paigaldatakse | Kui repo error, kontrolli apt |

```bash
sudo ufw default deny incoming
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Sissetulev liiklus keelatakse vaikimisi | Kui käsk puudub, paigalda UFW |

```bash
sudo ufw default allow outgoing
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Väljaminev liiklus lubatakse | Kui error, kontrolli UFW paigaldust |

```bash
sudo ufw allow 22/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SSH lubatakse | Kui SSH port on muu, luba õige port |

```bash
sudo ufw allow 80/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP lubatakse | Kui HTTP pole vajalik, võib selle hiljem eemaldada |

```bash
sudo ufw allow 443/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTPS lubatakse | Kui ei lisandu, kontrolli `sudo ufw status numbered` |

```bash
sudo ufw enable
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW aktiveerub | Kui oled SSH-ga sees, veendu enne, et 22/tcp on lubatud |

```bash
sudo ufw status numbered
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed lubatud porte 22, 80, 443 | Kui port puudub, lisa see `sudo ufw allow PORT/tcp` |

---

# 14. Lisa DNS kirjed

## Kui DNS on Windows Serveris

Lisa DNS Manageris A-kirjed:

| Nimi | IP |
|---|---|
| `veeb` | DebianPilet1 IP |
| `paroolihaldus` | UbuntuServer IP |

Näited:

```text
veeb.sinuNimi.local
paroolihaldus.sinuNimi.local
```

---

## Kui testid ajutiselt Linuxi /etc/hosts failiga

```bash
sudo nano /etc/hosts
```

Lisa:

```text
DEBIAN_IP veeb.sinuNimi.local
UBUNTU_IP paroolihaldus.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Nimed hakkavad samas masinas lahenduma | Kui ei lahendu, kontrolli kirjavigu ja IP-aadresse |

Kontroll:

```bash
ping veeb.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Nimi lahendub DebianPilet1 IP-ks | Kui `Name or service not known`, DNS või hosts kirje puudub |

```bash
ping paroolihaldus.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Nimi lahendub UbuntuServeri IP-ks | Kui ei lahendu, kontrolli DNS/hosts kirjet |

---

# 15. Lõputestid

## DebianPilet1

```bash
systemctl status apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui failed, kontrolli `apache2ctl configtest` ja logisid |

```bash
systemctl status mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui failed, kontrolli `journalctl -xeu mariadb` |

```bash
curl -I http://localhost
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP vastus | Kui connection refused, Apache ei tööta |

```bash
curl -k -I https://veeb.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTPS vastus | Kui DNS error, kontrolli DNS. Kui SSL error, kontrolli Apache SSL conf |

```bash
cat /etc/debian_version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Debian 13.x, ülesande järgi eesmärk 13.5 | Kui näitab 11/12, upgrade jäi pooleli |

```bash
sudo ufw status verbose
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Lubatud ainult vajalikud pordid | Kui liiga palju porte avatud, eemalda üleliigsed `sudo ufw delete allow PORT/tcp` |

---

## UbuntuServer / Vaultwarden

```bash
sudo docker ps
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `vaultwarden` konteiner on `Up` | Kui `Exited`, vaata `sudo docker logs vaultwarden` |

```bash
curl -I http://127.0.0.1:8080
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Vaultwarden vastab lokaalselt | Kui connection refused, konteiner ei tööta või port vale |

```bash
curl -k -I https://paroolihaldus.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Vaultwarden avaneb HTTPS kaudu | Kui 502, kontrolli Apache proxy ja Docker konteinerit. Kui DNS error, kontrolli DNS kirjet |

```bash
sudo ufw status verbose
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Lubatud 22, 80 ja 443 | Kui 8080 on avatud, pole see vajalik, sest Vaultwarden on seotud 127.0.0.1 külge |

---

# 16. Troubleshooting

## WordPress näitab “Error establishing a database connection”

Kontrolli:

```bash
sudo grep DB_ /var/www/html/wp-config.php
```

Kui andmed on valed, paranda:

```bash
sudo nano /var/www/html/wp-config.php
```

Kontrolli andmebaasi:

```bash
sudo systemctl status mariadb
```

Testi kasutajaga:

```bash
mysql -u wordpressuser -p wordpress
```

Kui tuleb `Access denied`, muuda andmebaasi parool ja pane sama parool `wp-config.php` faili.

---

## Apache ei käivitu

Kontrolli:

```bash
sudo apache2ctl configtest
```

Kui tulemus ei ole `Syntax OK`, paranda näidatud fail ja rida.

Vaata logi:

```bash
sudo journalctl -xeu apache2
```

või:

```bash
sudo tail -n 50 /var/log/apache2/error.log
```

---

## HTTPS ei tööta

Kontrolli SSL moodulit:

```bash
sudo apache2ctl -M | grep ssl
```

Kui väljund puudub:

```bash
sudo a2enmod ssl
sudo systemctl restart apache2
```

Kontrolli porti:

```bash
sudo ss -tulpen | grep :443
```

Kui 443 ei kuula, kontrolli Apache HTTPS konfiguratsiooni ja UFW reeglit.

---

## DNS nimi ei lahendu

Kontrolli:

```bash
getent hosts veeb.sinuNimi.local
```

Kui vastust pole:

- lisa DNS A-kirje
- kontrolli, et klient kasutab õiget DNS serverit
- ajutiselt lisa kirje `/etc/hosts` faili

---

## Debian upgrade jäi pooleli

Paranda katkised paketid:

```bash
sudo apt --fix-broken install
```

Lõpeta pooleliolev seadistus:

```bash
sudo dpkg --configure -a
```

Korda upgrade’i:

```bash
sudo apt update
sudo apt full-upgrade -y
```

---

## MariaDB ei käivitu

Vaata logi:

```bash
sudo journalctl -xeu mariadb
```

Proovi upgrade’i:

```bash
sudo mariadb-upgrade
```

Taaskäivita:

```bash
sudo systemctl restart mariadb
```

---

## Vaultwarden ei tööta

Kontrolli konteinereid:

```bash
sudo docker ps -a
```

Vaata logi:

```bash
sudo docker logs vaultwarden
```

Taaskäivita:

```bash
cd /opt/vaultwarden
sudo docker compose down
sudo docker compose up -d
```

---

# Kõige lühem spikker

```text
1. Kontrolli DebianPilet1 versioon, IP, hostname
2. Kontrolli Apache ja MariaDB staatust
3. Leia wp-config.php
4. Kontrolli DB_NAME, DB_USER, DB_PASSWORD
5. Taasta andmebaasi kasutaja parool
6. Taasta WordPress admin parool
7. Pane Apache ja MariaDB tööle
8. Uuenda Debian 11 → 12 → 13
9. Uuenda MariaDB/MySQL
10. Loo SSL sertifikaat
11. Seadista Apache HTTPS
12. Luba UFW-s 22, 80, 443
13. Paigalda UbuntuServerisse Docker
14. Paigalda Vaultwarden
15. Tee Apache reverse proxy
16. Lisa DNS kirjed
17. Testi HTTPS, WordPress, UFW ja Vaultwarden
18. Dokumenteeri
```


# Dockeri paigaldus ja kontroll Debian serveris

## Eesmärk

Serverisse paigaldati Docker, et vajadusel käivitada teenuseid konteineritena. Kontrolliti ka Docker Compose olemasolu.

Antud süsteemis kasutati Debian paketihaldust. Docker Compose paigaldus toimus paketina `docker-compose`, kuna paketti `docker-compose-plugin` ei olnud kasutatavatest repositooriumitest võimalik leida.

---

## 1. Pakettide nimekirja uuendamine

Enne Dockeri paigaldamist uuendati pakettide nimekiri.

```bash
sudo apt update
```

**Milleks?**  
See uuendab serveri infot selle kohta, millised paketid on repositooriumites saadaval.

---

## 2. Docker Engine ja Docker Compose paigaldamine

Paigaldati Docker Engine ning klassikaline Docker Compose pakett.

```bash
sudo apt install docker.io docker-compose -y
```

**Milleks?**

- `docker.io` paigaldab Docker Engine teenuse ehk Docker daemon'i.
- `docker-compose` paigaldab Compose tööriista, millega saab käivitada teenuseid `docker-compose.yml` faili põhjal.

---

## 3. Docker Compose plugini probleem

Prooviti paigaldada paketti:

```bash
sudo apt install docker.io docker-compose-plugin -y
```

Süsteem tagastas vea:

```text
Error: Unable to locate package docker-compose-plugin
```

See tähendab, et pakett `docker-compose-plugin` ei olnud kasutatavates Debian repositooriumites selle nimega saadaval.

Lahendusena kasutati Debianis olemasolevat paketti:

```bash
sudo apt install docker-compose -y
```

Seega tuleb antud serveris kasutada Compose käske kujul:

```bash
docker-compose up -d
docker-compose down
docker-compose ps
```

mitte tingimata kujul:

```bash
docker compose up -d
```

---

## 4. Docker teenuse käivitamine

Pärast paigaldamist lubati Docker teenus automaatselt käivituma ning käivitati see kohe.

```bash
sudo systemctl enable --now docker
```

**Milleks?**  
See teeb kaks asja korraga:

- `enable` paneb Docker teenuse käivituma automaatselt pärast serveri restarti;
- `--now` käivitab teenuse kohe.

---

## 5. Docker teenuse oleku kontroll

Kontrolliti, kas Docker teenus töötab.

```bash
sudo systemctl status docker
```

Oodatav tulemus:

```text
active (running)
```

Kui teenus töötab, võib logides näha näiteks:

```text
Started docker.service - Docker Application Container Engine
```

See tähendab, et Docker daemon on käivitatud.

---

## 6. Docker käsu kontroll

Kontrolliti, kas Dockeriga saab ühenduse.

```bash
sudo docker ps
```

Kui Docker töötab, kuvatakse konteinerite nimekiri. Kui ühtegi konteinerit veel ei tööta, võib nimekiri olla tühi.

Oodatav väljund võib olla näiteks:

```text
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

Oluline on see, et ei tekiks enam viga:

```text
Cannot connect to the Docker daemon at unix:///var/run/docker.sock.
Is the docker daemon running?
```

---

## 7. Docker Compose kontroll

Kontrolliti klassikalise Docker Compose käsu olemasolu.

```bash
docker-compose version
```

Kui käsk kuvab versiooni, on Docker Compose olemas ja kasutatav.

Näide:

```text
docker-compose version 1.x.x
```

Võib proovida ka uuemat käsuvarianti:

```bash
docker compose version
```

Kui `docker compose version` ei tööta, aga `docker-compose version` töötab, kasutatakse selles serveris Compose käske sidekriipsuga kujul:

```bash
docker-compose up -d
```

---

## 8. Kasutaja lisamine Docker gruppi

Et kasutaja `hkh` saaks Dockerit kasutada ilma `sudo` käsuta, lisati kasutaja Docker gruppi.

```bash
sudo usermod -aG docker hkh
```

Muudatus rakendub pärast uuesti sisse logimist.

```bash
exit
ssh hkh@SERVERI_IP
```

Pärast uuesti sisselogimist kontrolliti:

```bash
docker ps
```

Kui õigused ei ole veel rakendunud või tekib veateade, saab eksami ajal kasutada Dockerit ka `sudo` abil:

```bash
sudo docker ps
```

---

## 9. Tekkinud probleem ja lahendus

### Probleem 1: Docker daemon ei töötanud

Alguses tekkis Docker käsu kasutamisel viga:

```text
Cannot connect to the Docker daemon at unix:///var/run/docker.sock.
Is the docker daemon running?
```

Samuti ei leitud teenust:

```text
Unit docker.service not found
```

See tähendas, et Docker käsurea tööriistad olid osaliselt olemas, kuid Docker Engine teenus ei olnud korrektselt paigaldatud või käivitatud.

### Lahendus

Paigaldati Docker Engine pakett:

```bash
sudo apt install docker.io docker-compose -y
```

Seejärel käivitati Docker teenus:

```bash
sudo systemctl enable --now docker
```

Kontrolliti teenuse olekut:

```bash
sudo systemctl status docker
```

Pärast seda töötas Docker daemon ning kontrollkäsk:

```bash
sudo docker ps
```

ei andnud enam Docker daemon ühenduse viga.

---

## 10. Kasutatud käsud kokkuvõtlikult

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable --now docker
sudo systemctl status docker
sudo docker ps
docker-compose version
sudo usermod -aG docker hkh
groups hkh
```

---

## 11. Kontrollpunktid

Dockeri seadistus loeti õnnestunuks, kui täidetud olid järgmised tingimused:

- `docker.service` on olekus `active (running)`;
- käsk `sudo docker ps` töötab ilma daemon veata;
- käsk `docker-compose version` kuvab Compose versiooni;
- vajadusel kuulub kasutaja `hkh` gruppi `docker`.

Kasutaja gruppide kontroll:

```bash
groups hkh
```

Oodatav tulemus sisaldab gruppi:

```text
docker
```

---

## 12. Märkus dokumentatsiooni jaoks

Selles Debian keskkonnas ei olnud pakett `docker-compose-plugin` saadaval. Seetõttu kasutati klassikalist Docker Compose paketti:

```bash
sudo apt install docker-compose -y
```

Antud lahenduses kasutatakse Compose käske kujul:

```bash
docker-compose up -d
```

Kui süsteemis on olemas uuem pluginipõhine Compose, võib kasutada ka kujul:

```bash
docker compose up -d
```

Antud serveris dokumenteeriti töötavaks lahenduseks `docker-compose`.


