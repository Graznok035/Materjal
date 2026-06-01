# Linux pilet 1 ülesannete lahenduskäik

See juhend keskendub ainult **Linux pilet 1 eriosa ülesannetele**.

Siin ei ole lahti kirjutatud üldist Linuxi baasosa:

- Ansible paigaldus UbuntuServerisse
- Ansible inventory loomine
- `hkhk` kasutaja loomine
- sudo grupp
- SSH võtmega ligipääs
- SSH parooliga sisselogimise keelamine

Need kuuluvad Linuxi üldise baasosa alla.

---

## Pilet 1 põhiteemad

Linux pilet 1 keskendub järgmistele teemadele:

| Teema | Mida tuleb teha |
|---|---|
| DebianPilet1 ligipääs | Taastada ligipääs DebianPilet1 serverile või teenusele |
| WordPress | Taastada WordPressi sisuhaldussüsteem |
| Apache | Taastada veebiserveri töö |
| MySQL/MariaDB | Taastada andmebaasi töö ja paroolid |
| Debian upgrade | Uuendada Debian 11.9 versioonile Debian 13.5 |
| MySQL upgrade | Uuendada vana MySQL/MariaDB versioon |
| SSL | Seadistada veebilehele SSL sertifikaat |
| UFW | Lubada ainult vajalikud pordid |
| Vaultwarden | Paigaldada paroolihalduse keskkond UbuntuServerisse |
| DNS | Luua paroolihaldusele FQDN DNS kirje |
| Dokumentatsioon | Kirjeldada kogu tööprotsess |

---

# 1. Algkontroll DebianPilet1 serveris

## Eesmärk

Kõigepealt tuleb aru saada, mis serveris juba olemas on:

- mis Debian versioon on peal
- kas Apache töötab
- kas andmebaas töötab
- kus WordPress asub
- mis pordid on avatud
- kas DNS/nimelahendus töötab

## Käsud

```bash
hostname
hostname -f
ip a
ip route
cat /etc/os-release
cat /etc/debian_version
```

Kontrolli teenuseid:

```bash
systemctl status apache2
systemctl status mysql
systemctl status mariadb
```

Kui ei tea, kas andmebaas on MySQL või MariaDB:

```bash
mysql --version
mariadb --version
```

Kontrolli veebikausta:

```bash
ls -lah /var/www/
ls -lah /var/www/html/
```

Otsi WordPressi konfiguratsiooni:

```bash
find /var/www -name wp-config.php
```

Kontrolli porte:

```bash
ss -tulpen
```

Kontrolli tulemüüri:

```bash
sudo ufw status verbose
```

---

# 2. Taasta ligipääs DebianPilet1 serverile

## Eesmärk

Kui serverisse saab sisse, aga kasutajal pole õigusi, tuleb kontrollida kasutajaid ja sudo õiguseid.

## Kontrolli olemasolevaid kasutajaid

```bash
cat /etc/passwd | grep home
```

või:

```bash
ls /home
```

Kontrolli, kes on sudo grupis:

```bash
getent group sudo
```

## Lisa kasutaja sudo gruppi

Näide kasutajaga `hkhk`:

```bash
sudo usermod -aG sudo hkhk
```

Kontroll:

```bash
groups hkhk
```

Kui kasutaja peab uuesti sisse logima:

```bash
exit
```

Pärast uuesti sisselogimist:

```bash
sudo whoami
```

Oodatud väljund:

```text
root
```

## Vajadusel muuda kasutaja parool

```bash
sudo passwd hkhk
```

---

# 3. Kontrolli WordPressi andmebaasi seadeid

## Eesmärk

WordPressi ligipääsu taastamiseks tuleb kõigepealt leida andmebaasi nimi, kasutaja ja parool.

Need asuvad tavaliselt failis:

```text
wp-config.php
```

## Leia `wp-config.php`

```bash
sudo find /var/www -name wp-config.php
```

Näide:

```bash
sudo nano /var/www/html/wp-config.php
```

Vaata sealt read:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wordpressuser' );
define( 'DB_PASSWORD', 'parool' );
define( 'DB_HOST', 'localhost' );
```

Kiirkontroll käsuga:

```bash
sudo grep DB_ /var/www/html/wp-config.php
```

---

# 4. Taasta andmebaasi ligipääs

## Eesmärk

Kui WordPress ei saa andmebaasiga ühendust, tuleb kontrollida, kas andmebaas töötab ja kas kasutaja/parool on õiged.

## Kontrolli andmebaasi teenust

MariaDB puhul:

```bash
sudo systemctl status mariadb
```

MySQL puhul:

```bash
sudo systemctl status mysql
```

Kui teenus ei tööta:

```bash
sudo systemctl restart mariadb
```

või:

```bash
sudo systemctl restart mysql
```

Luba teenus käivitumisel:

```bash
sudo systemctl enable mariadb
```

või:

```bash
sudo systemctl enable mysql
```

## Logi andmebaasi sisse root kasutajana

```bash
sudo mysql
```

või:

```bash
sudo mariadb
```

Kontrolli andmebaase:

```sql
SHOW DATABASES;
```

Kontrolli kasutajaid:

```sql
SELECT user, host FROM mysql.user;
```

Kui WordPressi kasutaja parool on vaja taastada:

```sql
ALTER USER 'wordpressuser'@'localhost' IDENTIFIED BY 'UusTugevParool123!';
FLUSH PRIVILEGES;
EXIT;
```

Kui `ALTER USER` ei tööta, proovi:

```sql
SET PASSWORD FOR 'wordpressuser'@'localhost' = PASSWORD('UusTugevParool123!');
FLUSH PRIVILEGES;
EXIT;
```

Seejärel muuda sama parool ka WordPressi konfiguratsioonis:

```bash
sudo nano /var/www/html/wp-config.php
```

Muuda rida:

```php
define( 'DB_PASSWORD', 'UusTugevParool123!' );
```

Testi andmebaasi kasutajaga:

```bash
mysql -u wordpressuser -p wordpress
```

Kui saad sisse, on andmebaasi kasutaja korras.

---

# 5. Taasta WordPressi admin ligipääs

## Variant A: WP-CLI olemasolul

Kontrolli, kas `wp` käsk on olemas:

```bash
wp --info
```

Kui on olemas, vaata kasutajaid:

```bash
cd /var/www/html
sudo -u www-data wp user list
```

Muuda admin parool:

```bash
cd /var/www/html
sudo -u www-data wp user update admin --user_pass='UusAdminParool123!'
```

Kui admin kasutajanimi pole `admin`, kasuta õiget kasutajanime.

---

## Variant B: MySQL/MariaDB kaudu

Logi andmebaasi:

```bash
sudo mysql
```

Vali WordPressi andmebaas:

```sql
USE wordpress;
```

Vaata kasutajaid:

```sql
SELECT ID, user_login, user_email FROM wp_users;
```

Muuda admin parool:

```sql
UPDATE wp_users
SET user_pass = MD5('UusAdminParool123!')
WHERE user_login = 'admin';
```

Välju:

```sql
EXIT;
```

Seejärel proovi WordPressi admin lehte:

```text
http://debian-serveri-ip/wp-admin
```

või FQDN kaudu:

```text
http://veeb.sinuNimi.local/wp-admin
```

---

# 6. Taasta Apache veebiserveri töö

## Eesmärk

WordPress peab brauseris avanema.

## Paigalda vajalikud paketid

```bash
sudo apt update
sudo apt install apache2 php php-mysql mariadb-server -y
```

Vajadusel lisa levinud PHP moodulid:

```bash
sudo apt install php-cli php-curl php-gd php-mbstring php-xml php-zip -y
```

## Kontrolli Apache staatust

```bash
sudo systemctl status apache2
```

Kui vaja, käivita uuesti:

```bash
sudo systemctl restart apache2
```

Luba käivitumisel:

```bash
sudo systemctl enable apache2
```

## Kontrolli Apache konfiguratsiooni

```bash
sudo apache2ctl configtest
```

Oodatud väljund:

```text
Syntax OK
```

## Kontrolli veebilehte lokaalselt

```bash
curl -I http://localhost
```

Kui WordPress on `/var/www/html` all, kontrolli õiguseid:

```bash
sudo chown -R www-data:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

Taaskäivita Apache:

```bash
sudo systemctl restart apache2
```

---

# 7. Uuenda Debian 11.9 versioonile Debian 13.5

## Oluline

Debiani uuendust ei ole mõistlik teha otse 11 → 13 ühe hüppega.

Õige loogika:

```text
Debian 11 bullseye → Debian 12 bookworm → Debian 13 trixie
```

Enne uuendamist tee Proxmoxis snapshot või varukoopia.

## Kontroll enne uuendamist

```bash
cat /etc/os-release
cat /etc/debian_version
df -h
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove --purge -y
```

Tee sources.list varukoopia:

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup
```

---

## Uuendus Debian 11 → Debian 12

Asenda `bullseye` sõnaga `bookworm`:

```bash
sudo sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list
```

Kontrolli faili:

```bash
cat /etc/apt/sources.list
```

Uuenda paketiloend:

```bash
sudo apt update
```

Tee minimaalne upgrade:

```bash
sudo apt upgrade --without-new-pkgs -y
```

Tee täielik upgrade:

```bash
sudo apt full-upgrade -y
```

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
cat /etc/os-release
cat /etc/debian_version
```

---

## Uuendus Debian 12 → Debian 13

Asenda `bookworm` sõnaga `trixie`:

```bash
sudo sed -i 's/bookworm/trixie/g' /etc/apt/sources.list
```

Kontrolli faili:

```bash
cat /etc/apt/sources.list
```

Uuenda paketiloend:

```bash
sudo apt update
```

Tee minimaalne upgrade:

```bash
sudo apt upgrade --without-new-pkgs -y
```

Tee täielik upgrade:

```bash
sudo apt full-upgrade -y
```

Puhasta:

```bash
sudo apt autoremove --purge -y
```

Restart:

```bash
sudo reboot
```

Kontrolli lõplikku versiooni:

```bash
cat /etc/os-release
cat /etc/debian_version
```

Oodatud tulemus:

```text
Debian GNU/Linux 13
13.5
```

---

# 8. Uuenda MySQL/MariaDB versioon

## Eesmärk

Pärast süsteemi uuendust tuleb kontrollida, et andmebaas on uuendatud ja töötab.

## Kontrolli versiooni

```bash
mysql --version
```

või:

```bash
mariadb --version
```

## Uuenda paketid

```bash
sudo apt update
sudo apt install mariadb-server mariadb-client -y
```

## Käivita andmebaasi upgrade tööriist

Uuematel MariaDB versioonidel:

```bash
sudo mariadb-upgrade
```

Vanematel süsteemidel:

```bash
sudo mysql_upgrade
```

Taaskäivita teenus:

```bash
sudo systemctl restart mariadb
```

Kontrolli:

```bash
sudo systemctl status mariadb
```

Testi sisselogimist:

```bash
sudo mariadb
```

ja:

```sql
SHOW DATABASES;
EXIT;
```

---

# 9. Seadista Apache SSL sertifikaat

## Eesmärk

Veebileht peab avanema HTTPS kaudu.

Kui eksamil pole vaja ametlikku Let’s Encrypt sertifikaati, sobib sisemine/self-signed sertifikaat.

## Luba SSL moodul

```bash
sudo a2enmod ssl
sudo a2enmod rewrite
sudo systemctl restart apache2
```

## Loo sertifikaat

Asenda FQDN enda veebilehe nimega.

Näide:

```text
veeb.sinuNimi.local
```

Loo sertifikaadi kaust:

```bash
sudo mkdir -p /etc/ssl/localcerts
```

Loo self-signed sertifikaat:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/localcerts/veeb.key \
-out /etc/ssl/localcerts/veeb.crt
```

Common Name küsimuse juures sisesta:

```text
veeb.sinuNimi.local
```

## Loo Apache HTTPS konfiguratsioon

```bash
sudo nano /etc/apache2/sites-available/wordpress-ssl.conf
```

Lisa:

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

Luba sait:

```bash
sudo a2ensite wordpress-ssl.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Kontrolli:

```bash
curl -k -I https://veeb.sinuNimi.local
```

Kui DNS veel ei tööta, testi IP kaudu:

```bash
curl -k -I https://SERVERI_IP
```

---

# 10. Seadista UFW tulemüür DebianPilet1 serveris

## Eesmärk

Lubatud peavad olema ainult vajalikud pordid.

WordPressi/Apache serveris on tavaliselt vaja:

| Port | Teenus |
|---|---|
| 22/tcp | SSH |
| 80/tcp | HTTP |
| 443/tcp | HTTPS |

## Seadistus

```bash
sudo apt install ufw -y
```

Vaikimisi keela sissetulev liiklus:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Luba vajalikud pordid:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Lülita UFW sisse:

```bash
sudo ufw enable
```

Kontroll:

```bash
sudo ufw status numbered
```

---

# 11. Paigalda Vaultwarden UbuntuServerisse

## Eesmärk

UbuntuServerisse tuleb paigaldada paroolihalduse keskkond.

Soovitatav lihtne lahendus on kasutada Dockerit ja Vaultwarden konteinerit.

## Paigalda Docker

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin -y
```

Luba Docker käivitumisel:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Kontroll:

```bash
docker --version
sudo docker ps
```

## Loo Vaultwardeni kaust

```bash
sudo mkdir -p /opt/vaultwarden
cd /opt/vaultwarden
```

## Loo docker-compose fail

```bash
sudo nano docker-compose.yml
```

Sisu:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    ports:
      - "127.0.0.1:8080:80"
    volumes:
      - ./vw-data:/data
```

Käivita konteiner:

```bash
sudo docker compose up -d
```

Kontroll:

```bash
sudo docker ps
```

Kontroll lokaalselt:

```bash
curl -I http://127.0.0.1:8080
```

---

# 12. Seadista Vaultwardenile Apache reverse proxy UbuntuServeris

## Eesmärk

Vaultwarden võiks avaneda FQDN kaudu HTTPS-iga, näiteks:

```text
paroolihaldus.sinuNimi.local
```

Paigalda Apache:

```bash
sudo apt install apache2 -y
```

Luba vajalikud moodulid:

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod ssl
sudo systemctl restart apache2
```

Loo sertifikaat:

```bash
sudo mkdir -p /etc/ssl/localcerts
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/localcerts/paroolihaldus.key \
-out /etc/ssl/localcerts/paroolihaldus.crt
```

Common Name:

```text
paroolihaldus.sinuNimi.local
```

Loo Apache konfiguratsioon:

```bash
sudo nano /etc/apache2/sites-available/vaultwarden.conf
```

Sisu:

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

Luba sait:

```bash
sudo a2ensite vaultwarden.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Test:

```bash
curl -k -I https://paroolihaldus.sinuNimi.local
```

---

# 13. Seadista UFW UbuntuServeris

## Eesmärk

UbuntuServeris, kus töötab Vaultwarden, peavad olema lubatud ainult vajalikud pordid.

Kui Vaultwarden on Apache reverse proxy taga, siis väljast on vaja ainult:

| Port | Teenus |
|---|---|
| 22/tcp | SSH |
| 80/tcp | HTTP, kui vaja |
| 443/tcp | HTTPS |

Kuna konteiner on seotud aadressile `127.0.0.1:8080`, ei pea porti 8080 võrku avama.

## Käsud

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status numbered
```

---

# 14. Lisa DNS kirjed

## Eesmärk

Teenused peavad avanema FQDN nimega.

Näited:

```text
veeb.sinuNimi.local
paroolihaldus.sinuNimi.local
```

## Kui DNS on Windows Serveris

DNS Manageris lisa A-kirjed:

| Nimi | IP |
|---|---|
| `veeb` | DebianPilet1 IP |
| `paroolihaldus` | UbuntuServer IP |

Näide:

```text
veeb.sinuNimi.local -> DebianPilet1 IP
paroolihaldus.sinuNimi.local -> UbuntuServer IP
```

## Kui testid ajutiselt `/etc/hosts` failiga

Linux kliendis:

```bash
sudo nano /etc/hosts
```

Lisa:

```text
DEBIAN_IP veeb.sinuNimi.local
UBUNTU_IP paroolihaldus.sinuNimi.local
```

Windows kliendis:

```text
C:\Windows\System32\drivers\etc\hosts
```

Lisa administraatori õigustes:

```text
DEBIAN_IP veeb.sinuNimi.local
UBUNTU_IP paroolihaldus.sinuNimi.local
```

## Kontroll

Linuxis:

```bash
ping veeb.sinuNimi.local
ping paroolihaldus.sinuNimi.local
```

Windowsis:

```cmd
nslookup veeb.sinuNimi.local
nslookup paroolihaldus.sinuNimi.local
```

---

# 15. Lõputestid

## DebianPilet1 veebiserver

```bash
systemctl status apache2
systemctl status mariadb
curl -I http://localhost
curl -k -I https://veeb.sinuNimi.local
```

Brauseris:

```text
https://veeb.sinuNimi.local
```

WordPress admin:

```text
https://veeb.sinuNimi.local/wp-admin
```

---

## Debian versioon

```bash
cat /etc/os-release
cat /etc/debian_version
```

Oodatud:

```text
Debian 13
13.5
```

---

## Andmebaas

```bash
mysql --version
sudo mariadb -e "SHOW DATABASES;"
```

---

## Tulemüür

```bash
sudo ufw status verbose
```

Lubatud peaksid olema ainult vajalikud pordid:

```text
22/tcp
80/tcp
443/tcp
```

---

## Vaultwarden

UbuntuServeris:

```bash
sudo docker ps
curl -I http://127.0.0.1:8080
curl -k -I https://paroolihaldus.sinuNimi.local
```

Brauseris:

```text
https://paroolihaldus.sinuNimi.local
```

---

# 16. Dokumentatsiooni näidis

Dokumentatsiooni jaoks kirjuta umbes nii:

```markdown
## Linux pilet 1 dokumentatsioon

### Eesmärk

Eesmärk oli taastada DebianPilet1 serveris töötav WordPressi veebileht, uuendada Debian operatsioonisüsteem versioonile 13.5, uuendada andmebaasiteenus, seadistada SSL, piirata tulemüüriga lubatud pordid ning paigaldada UbuntuServerisse Vaultwarden paroolihaldus.

### Kasutatud masinad

| Masin | Roll |
|---|---|
| DebianPilet1 | Apache, WordPress, MariaDB/MySQL |
| UbuntuServer | Vaultwarden paroolihaldus |
| Windows/DNS server | DNS kirjete haldus |

### Tehtud seadistused

- Kontrollisin DebianPilet1 serveri versiooni ja teenuste olekut.
- Taastasin WordPressi andmebaasi kasutaja ligipääsu.
- Taastasin WordPress admin kasutaja parooli.
- Kontrollisin ja parandasin Apache veebiserveri konfiguratsiooni.
- Uuendasin Debian 11 süsteemi järjest Debian 12 ja Debian 13 peale.
- Uuendasin MariaDB/MySQL teenuse.
- Seadistasin Apache SSL sertifikaadi.
- Lubasin UFW tulemüüris ainult vajalikud pordid.
- Paigaldasin UbuntuServerisse Dockeriga Vaultwardeni.
- Seadistasin Vaultwardenile Apache reverse proxy ja HTTPS ligipääsu.
- Lisasin DNS kirjed veebilehele ja paroolihaldusele.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `cat /etc/debian_version` | Debian 13.5 |
| `systemctl status apache2` | Apache töötab |
| `systemctl status mariadb` | MariaDB töötab |
| `curl -k -I https://veeb.sinuNimi.local` | HTTPS vastab |
| `sudo ufw status` | Lubatud ainult vajalikud pordid |
| `sudo docker ps` | Vaultwarden konteiner töötab |
| `https://paroolihaldus.sinuNimi.local` | Vaultwarden avaneb |

### Kokkuvõte

Linux pilet 1 lahenduse tulemusena töötab DebianPilet1 serveris WordPressi veebileht HTTPS kaudu ning UbuntuServeris töötab Vaultwarden paroolihaldus. Teenustele on loodud FQDN nimed ja tulemüüris on lubatud ainult vajalikud pordid.
```

---

# Troubleshooting

## 1. WordPress näitab "Error establishing a database connection"

Kontrolli `wp-config.php` faili:

```bash
sudo grep DB_ /var/www/html/wp-config.php
```

Kontrolli, kas andmebaas töötab:

```bash
sudo systemctl status mariadb
```

Testi WordPressi andmebaasi kasutajaga:

```bash
mysql -u wordpressuser -p wordpress
```

Kui sisse ei saa, muuda parool andmebaasis ja `wp-config.php` failis samaks.

---

## 2. Apache ei käivitu

Kontrolli konfiguratsiooni:

```bash
sudo apache2ctl configtest
```

Vaata logisid:

```bash
sudo journalctl -xeu apache2
sudo tail -n 50 /var/log/apache2/error.log
```

Tüüpilised põhjused:

| Probleem | Lahendus |
|---|---|
| Port 80/443 juba kasutusel | `sudo ss -tulpen` |
| Sertifikaadi failitee vale | kontrolli Apache conf failis `SSLCertificateFile` |
| Süntaksiviga conf failis | `apache2ctl configtest` näitab rea |

---

## 3. HTTPS ei avane

Kontrolli, kas SSL moodul on lubatud:

```bash
sudo apache2ctl -M | grep ssl
```

Kui ei ole:

```bash
sudo a2enmod ssl
sudo systemctl restart apache2
```

Kontrolli, kas port 443 kuulab:

```bash
sudo ss -tulpen | grep 443
```

Kontrolli UFW:

```bash
sudo ufw status
```

Kui 443 pole lubatud:

```bash
sudo ufw allow 443/tcp
```

---

## 4. DNS nimi ei lahendu

Kontroll:

```bash
nslookup veeb.sinuNimi.local
```

või Linuxis:

```bash
getent hosts veeb.sinuNimi.local
```

Kui vastust ei tule:

| Probleem | Lahendus |
|---|---|
| DNS kirje puudub | lisa A-kirje DNS serverisse |
| klient kasutab valet DNS-i | kontrolli `ipconfig /all` või `/etc/resolv.conf` |
| DNS vahemälu segab | Windowsis `ipconfig /flushdns` |

---

## 5. Debian upgrade läheb katki

Kontrolli katkiseid pakette:

```bash
sudo apt --fix-broken install
```

Jätka poolelijäänud seadistust:

```bash
sudo dpkg --configure -a
```

Uuenda uuesti:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Kui repo on vale, taasta varukoopia:

```bash
sudo cp /etc/apt/sources.list.backup /etc/apt/sources.list
sudo apt update
```

---

## 6. MariaDB/MySQL ei käivitu pärast upgrade’i

Vaata logi:

```bash
sudo journalctl -xeu mariadb
```

Kontrolli konfiguratsiooni:

```bash
sudo mysqld --verbose --help
```

Proovi upgrade tööriista:

```bash
sudo mariadb-upgrade
```

Taaskäivita:

```bash
sudo systemctl restart mariadb
```

---

## 7. Vaultwarden konteiner ei tööta

Kontrolli konteinerit:

```bash
sudo docker ps -a
```

Vaata logisid:

```bash
sudo docker logs vaultwarden
```

Taaskäivita:

```bash
cd /opt/vaultwarden
sudo docker compose down
sudo docker compose up -d
```

Kui port on hõivatud:

```bash
sudo ss -tulpen | grep 8080
```

---

## 8. Vaultwarden avaneb lokaalselt, aga mitte FQDN kaudu

Kontrolli Apache reverse proxy:

```bash
sudo apache2ctl configtest
sudo systemctl status apache2
```

Kontrolli DNS:

```bash
nslookup paroolihaldus.sinuNimi.local
```

Kontrolli HTTPS:

```bash
curl -k -I https://paroolihaldus.sinuNimi.local
```

Kontrolli, kas konteiner vastab:

```bash
curl -I http://127.0.0.1:8080
```

Kui lokaalne töötab, aga FQDN mitte, on viga tavaliselt:

- DNS kirjes
- Apache reverse proxy konfiguratsioonis
- UFW tulemüüris
- SSL konfiguratsioonis

---

# Kõige lühem spikker

```text
1. Kontrolli DebianPilet1 teenused
2. Leia wp-config.php
3. Taasta andmebaasi kasutaja/parool
4. Taasta WordPress admin parool
5. Pane Apache ja MariaDB tööle
6. Uuenda Debian 11 → 12 → 13
7. Uuenda MariaDB/MySQL
8. Tee Apache SSL
9. Luba UFW-s ainult 22, 80, 443
10. Paigalda UbuntuServerisse Docker
11. Paigalda Vaultwarden
12. Tee Vaultwardenile HTTPS reverse proxy
13. Lisa DNS kirjed
14. Testi veeb, WordPress, HTTPS, UFW ja Vaultwarden
15. Dokumenteeri
```
