# Materjal

# Linuxi eksamipiletite lahenduskäigud

See dokument sisaldab Linuxi näidispiletite 1 ja 2 loogilisi lahenduskäike.  
Juhendid on koostatud eksami konteksti arvestades: Ansible baasülesanne, SSH turvanõuded, tulemüür, DNS, dokumenteerimine ja kontrollkäsud.

---

# Linux. Pilet 1

## Märksõnad

- Veebiserveri haldus
- Andmebaasi haldus
- Linux single-user mode
- SSH serveri konfigureerimine
- Tulemüüri konfigureerimine
- Linuxi versiooniuuendus
- WordPressi uuendamine
- HTTPS seadistamine
- DNS kirjed
- Veebirakenduste paigaldamine
- Crontab
- Paroolihaldur

---

## Ülesande lühikokkuvõte

AS Vasemba kasutab WordPressi siseportaali. WordPress on aegunud, veebiserver kasutab HTTP-d ja Debian server on vana versiooniga. Lisaks on endine administraator muutnud ära Linuxi kasutaja, WordPressi peakasutaja ja andmebaasi peakasutaja paroolid.

Lahenduse eesmärk on:

1. Paigaldada UbuntuServerisse Ansible.
2. Seadistada Ansible ligipääs AlmaServerisse ja DebianServerisse.
3. Paigaldada Ansible abil:
   - AlmaServerisse veebiserver
   - DebianServerisse andmebaasiserver
4. Taastada ligipääs DebianPilet1 serverile.
5. Taastada WordPressi admin ligipääs.
6. Taastada vajadusel andmebaasi admin ligipääs.
7. Uuendada WordPress.
8. Uuendada Debian 11.9 versioonilt Debian 12.11 peale.
9. Muuta masina hostinimeks `vasemba.oige.local`.
10. Lisada DNS kirje `vasemba.oige.local`.
11. Genereerida SSL sertifikaat.
12. Viia veebileht HTTP pealt HTTPS peale.
13. Suunata HTTP päringud automaatselt HTTPS peale.
14. Seadistada tulemüür.
15. Paigaldada UbuntuServerisse paroolihaldur.
16. Lisada DNS kirje `paroolihaldur.oige.local`.
17. Salvestada uuendatud ligipääsud paroolihaldurisse.
18. Dokumenteerida kogu protsess.

---

## Olulised mõisted

### Ansible

Ansible on haldustööriist, millega saab ühest masinast hallata teisi Linuxi servereid SSH kaudu.

Näide:

```text
UbuntuServer → haldab → AlmaServer
UbuntuServer → haldab → DebianServer
```

---

### Inventory fail

Inventory failis on kirjas serverid, mida Ansible haldab.

Näide:

```ini
[webservers]
alma ansible_host=10.0.x.20

[dbservers]
debian ansible_host=10.0.x.30
```

---

### Playbook

Playbook on Ansible’i tööjuhend. Seal kirjeldatakse, mida serverites teha.

Näiteks:

```text
webservers grupis paigalda Apache
dbservers grupis paigalda MariaDB
```

---

### Single-user mode

Single-user mode ehk ühe kasutaja režiim on Linuxi päästerežiim.  
Seda saab kasutada siis, kui Linuxi kasutaja parool on kadunud või muudetud.

Selle pileti puhul kasutatakse seda DebianPilet1 kasutaja parooli taastamiseks.

---

### FQDN

FQDN tähendab täielikku domeeninime.

Näide:

```text
vasemba.oige.local
```

---

### DNS kirje

DNS kirje seob nime IP-aadressiga.

Näide:

```text
vasemba.oige.local → 10.0.x.40
```

---

### HTTPS ja SSL sertifikaat

HTTP on krüpteerimata veebiliiklus.  
HTTPS on krüpteeritud veebiliiklus.

HTTPS kasutamiseks on vaja SSL/TLS sertifikaati.

Eksami sisevõrgus võib kasutada self-signed sertifikaati.

---

### Crontab

Crontab võimaldab käske automaatselt ajastada.  
Näiteks saab sellega teha WordPressi andmebaasist igapäevase varukoopia.

---

## Näidis IP-plaan

`x` tuleb asendada enda Proxmoxi vmbr numbriga.

| Masin | Roll | IP |
|---|---|---|
| UbuntuServer | Ansible + Docker + paroolihaldur | `10.0.x.10` |
| AlmaServer | Webservers grupp | `10.0.x.20` |
| DebianServer | Dbservers grupp | `10.0.x.30` |
| DebianPilet1 | WordPress server | `10.0.x.40` |
| DNS server | AS Oige nimeserver | `10.0.x.5` |
| Klientmasin | Haldusmasin | `10.0.x.100` |

---

## Soovituslik tööjärjekord

1. Koosta IP-plaan.
2. Seadista masinate võrk.
3. Kontrolli ühendust pingiga.
4. Paigalda UbuntuServerisse Ansible.
5. Loo SSH võtmed.
6. Seadista SSH võtmetega ligipääs AlmaServerisse ja DebianServerisse.
7. Loo Ansible inventory fail.
8. Loo Ansible playbook.
9. Paigalda Ansible abil AlmaServerisse veebiserver ja DebianServerisse andmebaasiserver.
10. Taasta DebianPilet1 serveri kasutaja parool single-user mode abil.
11. Muuda DebianPilet1 hostname `vasemba.oige.local`.
12. Taasta WordPressi admin ligipääs.
13. Taasta vajadusel andmebaasi admin ligipääs.
14. Tee WordPressist varukoopia.
15. Uuenda WordPress.
16. Uuenda Debian 11.9 versioonilt Debian 12 peale.
17. Lisa DNS kirje `vasemba.oige.local`.
18. Genereeri SSL sertifikaat.
19. Seadista veebiserver HTTPS peale.
20. Seadista HTTP → HTTPS ümbersuunamine.
21. Seadista tulemüür.
22. Paigalda UbuntuServerisse paroolihaldur.
23. Lisa DNS kirje `paroolihaldur.oige.local`.
24. Salvesta uued ligipääsud paroolihaldurisse.
25. Kontrolli kogu lahendus üle.
26. Dokumenteeri kogu protsess.

---

## 1. Ansible paigaldamine UbuntuServerisse

UbuntuServeris:

```bash
sudo apt update
sudo apt install ansible openssh-client -y
```

Kontroll:

```bash
ansible --version
```

---

## 2. SSH võtmete loomine UbuntuServeris

```bash
ssh-keygen
```

Vajuta küsimuste peale `Enter`.

Kopeeri võti AlmaServerisse:

```bash
ssh-copy-id kasutaja@10.0.x.20
```

Kopeeri võti DebianServerisse:

```bash
ssh-copy-id kasutaja@10.0.x.30
```

Kontroll:

```bash
ssh kasutaja@10.0.x.20
exit
```

```bash
ssh kasutaja@10.0.x.30
exit
```

---

## 3. SSH turvaseadistus Linuxi serverites

Kõigis Linuxi serverites peab olema:

```text
root kasutaja SSH ligipääs keelatud
parooliga SSH login keelatud
võtmega SSH login lubatud
```

Ava SSH seadistus:

```bash
sudo nano /etc/ssh/sshd_config
```

Kontrolli või lisa read:

```conf
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Taaskäivita SSH.

Ubuntu/Debian:

```bash
sudo systemctl restart ssh
```

AlmaLinux:

```bash
sudo systemctl restart sshd
```

Kontrolli enne vana SSH ühenduse sulgemist, et võtmega sisselogimine töötab.

---

## 4. Ansible inventory faili loomine

UbuntuServeris:

```bash
mkdir ~/ansible
cd ~/ansible
nano inventory.ini
```

Sisu:

```ini
[webservers]
alma ansible_host=10.0.x.20 ansible_user=kasutaja

[dbservers]
debian ansible_host=10.0.x.30 ansible_user=kasutaja

[all:vars]
ansible_become=yes
ansible_become_method=sudo
```

Kontroll:

```bash
ansible all -i inventory.ini -m ping
```

Oodatav tulemus:

```text
SUCCESS
```

---

## 5. Ansible playbook veebiserveri ja andmebaasi jaoks

Valikud:

- veebiserver: Apache
- andmebaasiserver: MariaDB

Loo playbook:

```bash
nano install_services.yml
```

Sisu:

```yaml
---
- name: Paigalda veebiserver AlmaServerisse
  hosts: webservers
  become: yes
  tasks:
    - name: Paigalda Apache AlmaLinuxis
      ansible.builtin.dnf:
        name: httpd
        state: present

    - name: Käivita ja luba Apache
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: yes

- name: Paigalda andmebaasiserver DebianServerisse
  hosts: dbservers
  become: yes
  tasks:
    - name: Uuenda apt cache
      ansible.builtin.apt:
        update_cache: yes

    - name: Paigalda MariaDB
      ansible.builtin.apt:
        name: mariadb-server
        state: present

    - name: Käivita ja luba MariaDB
      ansible.builtin.service:
        name: mariadb
        state: started
        enabled: yes
```

Käivita playbook:

```bash
ansible-playbook -i inventory.ini install_services.yml
```

Kontroll:

```bash
ansible webservers -i inventory.ini -m shell -a "systemctl status httpd --no-pager"
```

```bash
ansible dbservers -i inventory.ini -m shell -a "systemctl status mariadb --no-pager"
```

---

## 6. DebianPilet1 ligipääsu taastamine single-user mode abil

Seda tehakse Proxmoxi konsoolist.

### Sammud

1. Ava Proxmoxis DebianPilet1 konsool.
2. Tee masinale restart.
3. GRUB menüüs vali Debian käivitusrida.
4. Vajuta `e`.
5. Leia rida, mis algab sõnaga `linux`.
6. Lisa rea lõppu:

```text
init=/bin/bash
```

7. Käivita muudetud kirje klahviga `Ctrl + X` või `F10`.

Kui saad shelli ette, tee failisüsteem kirjutatavaks:

```bash
mount -o remount,rw /
```

Muuda kasutaja parool:

```bash
passwd kasutaja
```

Taaskäivita masin:

```bash
reboot -f
```

---

## 7. DebianPilet1 hostname muutmine

```bash
sudo hostnamectl set-hostname vasemba.oige.local
```

Ava hosts fail:

```bash
sudo nano /etc/hosts
```

Lisa või muuda rida:

```text
10.0.x.40 vasemba.oige.local vasemba
```

Kontroll:

```bash
hostnamectl
hostname -f
```

Oodatav tulemus:

```text
vasemba.oige.local
```

---

## 8. WordPressi admin ligipääsu taastamine

Leia WordPressi konfiguratsioonifail:

```bash
sudo find /var/www -name wp-config.php
```

Ava fail:

```bash
sudo cat /var/www/html/wp-config.php
```

Otsi andmebaasi andmed:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wordpressuser' );
define( 'DB_PASSWORD', 'parool' );
```

Logi andmebaasi:

```bash
sudo mysql
```

Vali andmebaas:

```sql
SHOW DATABASES;
USE wordpress;
```

Vaata WordPressi kasutajaid:

```sql
SELECT ID, user_login, user_email FROM wp_users;
```

Muuda administraatori parool:

```sql
UPDATE wp_users SET user_pass = MD5('UusTugevParool123!') WHERE user_login = 'admin';
```

Välju:

```sql
EXIT;
```

Ava brauseris:

```text
http://10.0.x.40/wp-admin
```

või pärast DNS-i:

```text
https://vasemba.oige.local/wp-admin
```

---

## 9. Andmebaasi peakasutaja ligipääsu taastamine

Proovi esmalt:

```bash
sudo mysql
```

Kui töötab, saad admin õigustes sisse.

Kui ei tööta, peata MariaDB:

```bash
sudo systemctl stop mariadb
```

Käivita MariaDB ilma õiguste kontrollita:

```bash
sudo mysqld_safe --skip-grant-tables &
```

Logi sisse:

```bash
mysql
```

Muuda root parool:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'UusTugevDBParool123!';
FLUSH PRIVILEGES;
EXIT;
```

Käivita MariaDB uuesti:

```bash
sudo pkill mysqld
sudo systemctl start mariadb
```

Kontroll:

```bash
mysql -u root -p
```

---

## 10. WordPressi varukoopia

Failide backup:

```bash
sudo tar -czvf /root/wordpress-files-backup.tar.gz /var/www/html
```

Andmebaasi backup:

```bash
sudo mysqldump -u root -p wordpress > /root/wordpress-db-backup.sql
```

Kui andmebaasi nimi ei ole `wordpress`, kasuta seda nime, mis oli `wp-config.php` failis.

---

## 11. WordPressi uuendamine

Kõige lihtsam variant on WordPressi admin paneelis:

```text
Dashboard → Updates
```

Uuenda:

- WordPress core
- pluginad
- teemad

Kui olemas on WP-CLI:

```bash
wp core update --path=/var/www/html --allow-root
wp plugin update --all --path=/var/www/html --allow-root
wp theme update --all --path=/var/www/html --allow-root
```

Kontroll:

```bash
wp core version --path=/var/www/html --allow-root
```

---

## 12. Debian 11.9 uuendamine Debian 12 peale

Kontrolli praegust versiooni:

```bash
cat /etc/debian_version
lsb_release -a
```

Uuenda olemasolev süsteem:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt full-upgrade -y
sudo apt autoremove -y
```

Muuda sources list:

```bash
sudo nano /etc/apt/sources.list
```

Asenda `bullseye` sõnaga `bookworm`.

Näide:

```text
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

Uuenda paketiloend:

```bash
sudo apt update
```

Tee minimaalne upgrade:

```bash
sudo apt upgrade --without-new-pkgs -y
```

Tee full upgrade:

```bash
sudo apt full-upgrade -y
```

Puhasta:

```bash
sudo apt autoremove -y
sudo apt clean
```

Taaskäivita:

```bash
sudo reboot
```

Kontroll:

```bash
cat /etc/debian_version
lsb_release -a
```

---

## 13. DNS kirje `vasemba.oige.local`

DNS serveris lisa tsoonifaili:

```dns
vasemba    IN    A    10.0.x.40
```

või:

```dns
vasemba.oige.local.    IN    A    10.0.x.40
```

Kontrolli tsooni:

```bash
sudo named-checkzone oige.local /etc/bind/db.oige.local
```

Taaskäivita DNS:

```bash
sudo systemctl restart bind9
```

Kontroll kliendist:

```bash
nslookup vasemba.oige.local 10.0.x.5
```

---

## 14. SSL sertifikaadi loomine

DebianPilet1 serveris:

```bash
sudo mkdir -p /etc/ssl/vasemba
```

Genereeri sertifikaat:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/vasemba/vasemba.key \
-out /etc/ssl/vasemba/vasemba.crt \
-subj "/CN=vasemba.oige.local"
```

---

## 15. Apache HTTPS seadistus

Luba moodulid:

```bash
sudo a2enmod ssl
sudo a2enmod rewrite
```

Loo VirtualHost fail:

```bash
sudo nano /etc/apache2/sites-available/vasemba.conf
```

Sisu:

```apache
<VirtualHost *:80>
    ServerName vasemba.oige.local
    Redirect permanent / https://vasemba.oige.local/
</VirtualHost>

<VirtualHost *:443>
    ServerName vasemba.oige.local
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/ssl/vasemba/vasemba.crt
    SSLCertificateKeyFile /etc/ssl/vasemba/vasemba.key

    <Directory /var/www/html>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Luba sait:

```bash
sudo a2ensite vasemba.conf
```

Keela default sait, kui vaja:

```bash
sudo a2dissite 000-default.conf
```

Kontrolli konfiguratsiooni:

```bash
sudo apache2ctl configtest
```

Kui tulemus on `Syntax OK`, tee restart:

```bash
sudo systemctl restart apache2
```

Kontroll:

```bash
curl -I http://vasemba.oige.local
curl -k -I https://vasemba.oige.local
```

---

## 16. WordPressi URL muutmine HTTPS peale

```bash
sudo mysql
```

```sql
USE wordpress;
UPDATE wp_options SET option_value='https://vasemba.oige.local' WHERE option_name='siteurl';
UPDATE wp_options SET option_value='https://vasemba.oige.local' WHERE option_name='home';
EXIT;
```

---

## 17. Tulemüüri seadistamine DebianPilet1 serveris

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Luba vajalikud pordid:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Aktiveeri tulemüür:

```bash
sudo ufw enable
```

Kontroll:

```bash
sudo ufw status verbose
```

---

## 18. Crontab WordPressi andmebaasi backupiks

Loo backup kaust:

```bash
sudo mkdir -p /var/backups/wordpress
```

Ava root crontab:

```bash
sudo crontab -e
```

Lisa rida:

```cron
0 2 * * * mysqldump wordpress > /var/backups/wordpress/wordpress-db-$(date +\%F).sql
```

Kontroll:

```bash
sudo crontab -l
```

---

## 19. Paroolihalduri paigaldamine UbuntuServerisse

Soovituslik valik: Vaultwarden.

Põhjendus:

```text
Vaultwarden on isehostitav veebipõhine paroolihaldur, mida saab lihtsalt Dockeriga paigaldada. See sobib testkeskkonda ning andmeid saab säilitada hosti kaustas või Docker volume'is.
```

Paigalda Docker:

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin -y
```

Käivita Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Lisa kasutaja Docker gruppi:

```bash
sudo usermod -aG docker kasutaja
```

Logi välja ja sisse tagasi.

Loo Vaultwardeni kaust:

```bash
sudo mkdir -p /opt/vaultwarden
sudo chown -R kasutaja:kasutaja /opt/vaultwarden
cd /opt/vaultwarden
```

Loo compose fail:

```bash
nano docker-compose.yml
```

Sisu:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./vw-data:/data
```

Käivita:

```bash
docker compose up -d
```

Kontroll:

```bash
docker ps
```

Ava brauseris:

```text
http://10.0.x.10:8080
```

---

## 20. DNS kirje `paroolihaldur.oige.local`

DNS serveris lisa:

```dns
paroolihaldur    IN    A    10.0.x.10
```

Kontroll:

```bash
nslookup paroolihaldur.oige.local 10.0.x.5
```

Ava brauseris:

```text
http://paroolihaldur.oige.local:8080
```

---

## 21. Pilet 1 kontrollnimekiri

### Ansible

```bash
ansible all -i inventory.ini -m ping
```

### AlmaServer veebiserver

```bash
curl http://10.0.x.20
```

### DebianServer andmebaas

```bash
systemctl status mariadb
```

### DebianPilet1 hostname

```bash
hostname -f
```

Oodatav:

```text
vasemba.oige.local
```

### DNS

```bash
nslookup vasemba.oige.local 10.0.x.5
```

### HTTPS

```bash
curl -k -I https://vasemba.oige.local
```

### HTTP redirect

```bash
curl -I http://vasemba.oige.local
```

### Debian versioon

```bash
cat /etc/debian_version
```

### Tulemüür

```bash
sudo ufw status verbose
```

### Vaultwarden

```bash
docker ps
```

---

## Pilet 1 tüüpilised probleemid ja lahendused

### Ansible annab `UNREACHABLE`

Kontrolli:

```bash
ping 10.0.x.20
ssh kasutaja@10.0.x.20
```

Võimalikud põhjused:

- vale IP
- SSH ei tööta
- SSH võti pole kopeeritud
- tulemüür blokib porti 22
- vale kasutajanimi

---

### SSH ei lase enam sisse pärast `PasswordAuthentication no`

Mine Proxmoxi konsooli ja luba ajutiselt:

```conf
PasswordAuthentication yes
```

Kopeeri SSH võti uuesti ning testi võtmega login.  
Alles siis keela parooliga login uuesti.

---

### WordPress näitab valget lehte

Vaata Apache logi:

```bash
sudo tail -f /var/log/apache2/error.log
```

Keela pluginad ajutiselt:

```bash
cd /var/www/html/wp-content
sudo mv plugins plugins.disabled
```

---

### HTTPS annab sertifikaadi hoiatuse

Self-signed sertifikaadiga on see sisevõrgus normaalne.

---

### DNS ei lahendu

Kontrolli:

```bash
nslookup vasemba.oige.local 10.0.x.5
```

Kontrolli ka:

- kas DNS tsoonifailis on kirje
- kas serial number sai suurendatud
- kas bind9 restart tehti
- kas klient kasutab õiget DNS serverit

---

# Linux. Pilet 2

## Märksõnad

- Logiserveri paigaldus
- Logimise seadistamine
- GRUB alglaadur
- BASH skriptimine
- Crontab
- Varundamine
- Lisaketta vormindamine
- DNS
- Tulemüür

---

## Ülesande lühikokkuvõte

AS Oige vajab keskset logiserverit Linuxi serverite logide kogumiseks. Logiserver peab olema isehostitav ja tulevikus peab olema võimalik koguda ka Windowsi serverite logisid.

Lisaks on UbuntuPilet2 masinal alglaadur katki. Masinat ei tohi üle kirjutada, sest seal on oluline memo. Samuti tuleb vormindada lisatud kõvaketas, ühendada see püsivalt süsteemi ning teha ülemuse kodukaustast sinna varukoopia.

Lahenduse eesmärk:

1. Paigaldada UbuntuServerisse Ansible.
2. Seadistada Ansible ligipääs AlmaServerisse ja DebianServerisse.
3. Paigaldada Ansible abil:
   - AlmaServerisse veebiserver
   - DebianServerisse andmebaasiserver
4. Paigaldada UbuntuServerisse keskne logiserver.
5. Muuta UbuntuServeri hostname `logger.oige.local`.
6. Lisada DNS kirje `logger.oige.local`.
7. Seadistada tulemüür.
8. Seadistada teised Linuxi masinad logisid saatma.
9. Kontrollida, et logid jõuavad kohale.
10. Parandada UbuntuPilet2 GRUB.
11. Taastada memo.
12. Vormindada UbuntuPilet2 lisaketas.
13. Ühendada ketas püsivalt süsteemi.
14. Teha ülemuse kodukaustast backup.
15. Luua backupi jaoks Bash skript.
16. Lisada backup crontabi.
17. Dokumenteerida kogu protsess.

---

## Logiserveri valik

Soovituslik valik: Wazuh.

Põhjendus:

```text
Wazuh on isehostitav logide ja turvasündmuste kogumise platvorm. See sobib Linuxi logide keskseks kogumiseks ning toetab tulevikus ka Windowsi logide kogumist Wazuh agendi abil.
```

---

## Näidis IP-plaan

`x` tuleb asendada enda Proxmoxi vmbr numbriga.

| Masin | Roll | IP |
|---|---|---|
| UbuntuServer | Ansible + Wazuh logiserver | `10.0.x.10` |
| AlmaServer | Webservers grupp / logi klient | `10.0.x.20` |
| DebianServer | Dbservers grupp / logi klient | `10.0.x.30` |
| UbuntuPilet2 | Parandatav masin / logi klient | `10.0.x.40` |
| DNS server | AS Oige nimeserver | `10.0.x.5` |
| Klientmasin | Haldusmasin | `10.0.x.100` |

---

## Soovituslik tööjärjekord

1. Koosta IP-plaan.
2. Seadista UbuntuServeri võrk.
3. Muuda UbuntuServeri hostname `logger.oige.local`.
4. Lisa DNS kirje `logger.oige.local`.
5. Paigalda UbuntuServerisse Ansible.
6. Loo SSH võtmed.
7. Seadista SSH ligipääs AlmaServerisse ja DebianServerisse.
8. Loo Ansible inventory.
9. Loo Ansible playbook.
10. Paigalda Ansible abil AlmaServerisse Apache ja DebianServerisse MariaDB.
11. Paigalda UbuntuServerisse Docker.
12. Paigalda UbuntuServerisse Wazuh.
13. Seadista logiserveri tulemüür.
14. Seadista Linuxi serverid logisid saatma.
15. Kontrolli logide kohalejõudmist.
16. Paranda UbuntuPilet2 GRUB.
17. Leia ja päästa memo.
18. Vorminda lisatud ketas.
19. Ühenda ketas püsivalt `/backup` alla.
20. Tee ülemuse kodukaustast backup.
21. Loo backupi Bash skript.
22. Lisa skript crontabi.
23. Dokumenteeri kogu protsess.

---

## 1. UbuntuServeri hostname muutmine

```bash
sudo hostnamectl set-hostname logger.oige.local
```

Ava hosts fail:

```bash
sudo nano /etc/hosts
```

Lisa:

```text
10.0.x.10 logger.oige.local logger
```

Kontroll:

```bash
hostnamectl
hostname -f
```

Oodatav:

```text
logger.oige.local
```

---

## 2. DNS kirje `logger.oige.local`

DNS serveris lisa tsoonifaili:

```dns
logger    IN    A    10.0.x.10
```

või:

```dns
logger.oige.local.    IN    A    10.0.x.10
```

Kontrolli tsooni:

```bash
sudo named-checkzone oige.local /etc/bind/db.oige.local
```

Taaskäivita DNS:

```bash
sudo systemctl restart bind9
```

Kontroll kliendist:

```bash
nslookup logger.oige.local 10.0.x.5
```

---

## 3. Ansible paigaldamine UbuntuServerisse

```bash
sudo apt update
sudo apt install ansible openssh-client -y
```

Kontroll:

```bash
ansible --version
```

---

## 4. SSH võtmete loomine

UbuntuServeris:

```bash
ssh-keygen
```

Kopeeri võti AlmaServerisse:

```bash
ssh-copy-id kasutaja@10.0.x.20
```

Kopeeri võti DebianServerisse:

```bash
ssh-copy-id kasutaja@10.0.x.30
```

Kontroll:

```bash
ssh kasutaja@10.0.x.20
exit
```

```bash
ssh kasutaja@10.0.x.30
exit
```

---

## 5. SSH turvaseadistus

Kõigis Linuxi serverites:

```bash
sudo nano /etc/ssh/sshd_config
```

Kontrolli või lisa:

```conf
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Taaskäivita SSH.

Ubuntu/Debian:

```bash
sudo systemctl restart ssh
```

AlmaLinux:

```bash
sudo systemctl restart sshd
```

---

## 6. Ansible inventory

UbuntuServeris:

```bash
mkdir ~/ansible
cd ~/ansible
nano inventory.ini
```

Sisu:

```ini
[webservers]
alma ansible_host=10.0.x.20 ansible_user=kasutaja

[dbservers]
debian ansible_host=10.0.x.30 ansible_user=kasutaja

[linux_clients]
alma ansible_host=10.0.x.20 ansible_user=kasutaja
debian ansible_host=10.0.x.30 ansible_user=kasutaja

[all:vars]
ansible_become=yes
ansible_become_method=sudo
```

Kontroll:

```bash
ansible all -i inventory.ini -m ping
```

---

## 7. Ansible playbook

Loo fail:

```bash
nano install_services.yml
```

Sisu:

```yaml
---
- name: Paigalda veebiserver AlmaServerisse
  hosts: webservers
  become: yes
  tasks:
    - name: Paigalda Apache AlmaLinuxis
      ansible.builtin.dnf:
        name: httpd
        state: present

    - name: Käivita ja luba Apache
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: yes

- name: Paigalda andmebaasiserver DebianServerisse
  hosts: dbservers
  become: yes
  tasks:
    - name: Uuenda apt cache
      ansible.builtin.apt:
        update_cache: yes

    - name: Paigalda MariaDB
      ansible.builtin.apt:
        name: mariadb-server
        state: present

    - name: Käivita ja luba MariaDB
      ansible.builtin.service:
        name: mariadb
        state: started
        enabled: yes
```

Käivita:

```bash
ansible-playbook -i inventory.ini install_services.yml
```

Kontroll:

```bash
ansible webservers -i inventory.ini -m shell -a "systemctl status httpd --no-pager"
ansible dbservers -i inventory.ini -m shell -a "systemctl status mariadb --no-pager"
```

---

## 8. Dockeri paigaldamine UbuntuServerisse

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin git -y
```

Käivita Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Lisa kasutaja Docker gruppi:

```bash
sudo usermod -aG docker kasutaja
```

Logi välja ja sisse tagasi.

Kontroll:

```bash
docker --version
docker compose version
```

---

## 9. Wazuh logiserveri paigaldamine Dockeriga

Liigu `/opt` kausta:

```bash
cd /opt
```

Laadi Wazuh Docker projekt:

```bash
sudo git clone https://github.com/wazuh/wazuh-docker.git
```

Anna kasutajale õigused:

```bash
sudo chown -R kasutaja:kasutaja /opt/wazuh-docker
```

Mine single-node kausta:

```bash
cd /opt/wazuh-docker/single-node
```

Kui olemas on sertifikaatide genereerimise compose fail, käivita:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

Käivita Wazuh:

```bash
docker compose up -d
```

Kontroll:

```bash
docker ps
```

Brauseris ava:

```text
https://logger.oige.local
```

või:

```text
https://10.0.x.10
```

Self-signed sertifikaadi hoiatus on testkeskkonnas normaalne.

---

## 10. Logiserveri tulemüür

UbuntuServeris:

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Luba SSH:

```bash
sudo ufw allow 22/tcp
```

Luba Wazuh dashboard:

```bash
sudo ufw allow 443/tcp
```

Luba Wazuh agentide ühendused:

```bash
sudo ufw allow 1514/tcp
```

Luba agentide registreerimine:

```bash
sudo ufw allow 1515/tcp
```

Luba syslog:

```bash
sudo ufw allow 514/tcp
sudo ufw allow 514/udp
```

Aktiveeri tulemüür:

```bash
sudo ufw enable
```

Kontroll:

```bash
sudo ufw status verbose
```

---

## 11. Wazuh syslog vastuvõtu seadistamine

Mine Wazuh manager konteinerisse:

```bash
docker exec -it single-node-wazuh.manager bash
```

Ava konfiguratsioon:

```bash
vi /var/ossec/etc/ossec.conf
```

Lisa `<ossec_config>` ploki sisse:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>tcp</protocol>
  <allowed-ips>10.0.x.0/24</allowed-ips>
  <local_ip>10.0.x.10</local_ip>
</remote>
```

Välju konteinerist:

```bash
exit
```

Taaskäivita Wazuh manager:

```bash
docker restart single-node-wazuh.manager
```

---

## 12. AlmaServeri logide saatmine logiserverisse

AlmaServeris:

```bash
sudo dnf install rsyslog -y
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

Ava rsyslog config:

```bash
sudo nano /etc/rsyslog.conf
```

Lisa faili lõppu:

```text
*.info@@10.0.x.10:514
```

Taaskäivita:

```bash
sudo systemctl restart rsyslog
```

Test:

```bash
logger "TEST AlmaServer saadab logi Wazuh logiserverisse"
```

---

## 13. DebianServeri logide saatmine logiserverisse

DebianServeris:

```bash
sudo apt update
sudo apt install rsyslog -y
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

Ava config:

```bash
sudo nano /etc/rsyslog.conf
```

Lisa faili lõppu:

```text
*.info@@10.0.x.10:514
```

Taaskäivita:

```bash
sudo systemctl restart rsyslog
```

Test:

```bash
logger "TEST DebianServer saadab logi Wazuh logiserverisse"
```

---

## 14. UbuntuPilet2 logide saatmine

Kui UbuntuPilet2 on käima saadud:

```bash
sudo apt update
sudo apt install rsyslog -y
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

Ava config:

```bash
sudo nano /etc/rsyslog.conf
```

Lisa:

```text
*.info@@10.0.x.10:514
```

Taaskäivita:

```bash
sudo systemctl restart rsyslog
```

Test:

```bash
logger "TEST UbuntuPilet2 saadab logi Wazuh logiserverisse"
```

---

## 15. Logide kohalejõudmise kontroll

Wazuh manager konteineris:

```bash
docker exec -it single-node-wazuh.manager bash
```

Otsi testlogisid:

```bash
grep -R "TEST AlmaServer" /var/ossec/logs/
grep -R "TEST DebianServer" /var/ossec/logs/
grep -R "TEST UbuntuPilet2" /var/ossec/logs/
```

Välju:

```bash
exit
```

---

## 16. UbuntuPilet2 GRUB alglaaduri parandamine

Käivita UbuntuPilet2 Live ISO pealt.

Vali:

```text
Try Ubuntu
```

Ava terminal.

Leia kettad:

```bash
lsblk
```

Kontrolli failisüsteeme:

```bash
sudo blkid
```

Leia Ubuntu root partitsioon. Näiteks `/dev/sda2`.

Mounti root partitsioon:

```bash
sudo mount /dev/sda2 /mnt
```

Kui olemas on EFI partitsioon, näiteks `/dev/sda1`, mounti see:

```bash
sudo mount /dev/sda1 /mnt/boot/efi
```

Seo süsteemikaustad:

```bash
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo mount --bind /run /mnt/run
```

Sisene parandatavasse süsteemi:

```bash
sudo chroot /mnt
```

Kui süsteem on BIOS/Legacy:

```bash
grub-install /dev/sda
update-grub
```

Kui süsteem on UEFI:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
update-grub
```

Välju:

```bash
exit
```

Unmount:

```bash
sudo umount -R /mnt
```

Taaskäivita:

```bash
sudo reboot
```

Eemalda Live ISO bootist.

---

## 17. Ülemuse memo leidmine

Kui UbuntuPilet2 käivitub, vaata kasutajaid:

```bash
ls /home
```

Otsi memo faile:

```bash
sudo find /home -iname "*memo*"
```

Otsi võimalikke tekstidokumente:

```bash
sudo find /home -type f \( -iname "*.txt" -o -iname "*.odt" -o -iname "*.docx" \)
```

Kui memo on näiteks:

```text
/home/ylemus/memo.txt
```

Kontroll:

```bash
cat /home/ylemus/memo.txt
```

---

## 18. Lisatud kõvaketta leidmine

UbuntuPilet2 masinas:

```bash
lsblk
```

Näide:

```text
sda    süsteemiketas
sdb    uus lisatud ketas
```

Kontrolli veel:

```bash
sudo fdisk -l
```

Ole väga ettevaatlik, et ei vormindaks valet ketast.

---

## 19. Lisaketta partitsioneerimine

Kui uus ketas on `/dev/sdb`:

```bash
sudo fdisk /dev/sdb
```

Vali järjest:

```text
g    loob GPT partitsioonitabeli
n    loob uue partitsiooni
Enter
Enter
Enter
w    salvestab muudatused
```

Kontroll:

```bash
lsblk
```

Peaks tekkima:

```text
/dev/sdb1
```

---

## 20. Lisaketta vormindamine

```bash
sudo mkfs.ext4 /dev/sdb1
```

Soovi korral anna kettale label:

```bash
sudo e2label /dev/sdb1 BACKUP
```

---

## 21. Lisaketta püsiv ühendamine

Loo mount kaust:

```bash
sudo mkdir -p /backup
```

Leia UUID:

```bash
sudo blkid /dev/sdb1
```

Ava fstab:

```bash
sudo nano /etc/fstab
```

Lisa lõppu:

```fstab
UUID=SIIN-SINU-UUID /backup ext4 defaults,nofail 0 2
```

Näide:

```fstab
UUID=abcd-1234-5678 /backup ext4 defaults,nofail 0 2
```

Testi:

```bash
sudo mount -a
```

Kontroll:

```bash
df -h
```

Peab näitama `/backup`.

---

## 22. Ülemuse kodukausta backup

Kontrolli kasutajanime:

```bash
ls /home
```

Näiteks kasutaja on `ylemus`.

Loo backup kaust:

```bash
sudo mkdir -p /backup/ylemus
```

Tee backup:

```bash
sudo rsync -avh /home/ylemus/ /backup/ylemus/
```

Kontroll:

```bash
ls -la /backup/ylemus
```

---

## 23. Bash skript backupi jaoks

Loo skript:

```bash
sudo nano /usr/local/sbin/backup_ylemus.sh
```

Sisu:

```bash
#!/bin/bash

SOURCE="/home/ylemus/"
DEST="/backup/ylemus/"
LOG="/var/log/backup_ylemus.log"

echo "Backup algas: $(date)" >> "$LOG"

if mountpoint -q /backup; then
    rsync -avh --delete "$SOURCE" "$DEST" >> "$LOG" 2>&1
    echo "Backup lõppes edukalt: $(date)" >> "$LOG"
else
    echo "VIGA: /backup ei ole ühendatud: $(date)" >> "$LOG"
    exit 1
fi
```

Tee käivitatavaks:

```bash
sudo chmod +x /usr/local/sbin/backup_ylemus.sh
```

Testi:

```bash
sudo /usr/local/sbin/backup_ylemus.sh
```

Kontrolli logi:

```bash
sudo tail -n 20 /var/log/backup_ylemus.log
```

---

## 24. Crontab automaatseks backupiks

Ava root crontab:

```bash
sudo crontab -e
```

Lisa:

```cron
0 2 * * * /usr/local/sbin/backup_ylemus.sh
```

Kontroll:

```bash
sudo crontab -l
```

---

## 25. UbuntuPilet2 tulemüür

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

---

## Pilet 2 kontrollnimekiri

### DNS

```bash
nslookup logger.oige.local 10.0.x.5
```

### Ansible

```bash
ansible all -i inventory.ini -m ping
```

### AlmaServer veebiserver

```bash
curl http://10.0.x.20
```

### DebianServer MariaDB

```bash
systemctl status mariadb
```

### Wazuh konteinerid

```bash
docker ps
```

### Wazuh dashboard

```text
https://logger.oige.local
```

### Logide testimine

AlmaServer:

```bash
logger "TEST AlmaServer"
```

DebianServer:

```bash
logger "TEST DebianServer"
```

UbuntuPilet2:

```bash
logger "TEST UbuntuPilet2"
```

Logiserveris:

```bash
grep -R "TEST AlmaServer" /var/ossec/logs/
grep -R "TEST DebianServer" /var/ossec/logs/
grep -R "TEST UbuntuPilet2" /var/ossec/logs/
```

### GRUB

UbuntuPilet2 peab käivituma ilma Live ISO-ta.

### Lisaketas

```bash
df -h
```

Peab näitama:

```text
/backup
```

### fstab

```bash
sudo mount -a
```

Ei tohi anda viga.

### Backup

```bash
sudo /usr/local/sbin/backup_ylemus.sh
ls -la /backup/ylemus
sudo tail -n 20 /var/log/backup_ylemus.log
```

### Crontab

```bash
sudo crontab -l
```

Peab sisaldama:

```cron
0 2 * * * /usr/local/sbin/backup_ylemus.sh
```

---

## Pilet 2 tüüpilised probleemid ja lahendused

### Wazuh dashboard ei avane

Kontrolli konteinerid:

```bash
docker ps
```

Kontrolli porte:

```bash
sudo ss -tulpn
```

Kontrolli tulemüüri:

```bash
sudo ufw status verbose
```

Kui port 443 pole lubatud:

```bash
sudo ufw allow 443/tcp
```

---

### DNS nimi `logger.oige.local` ei tööta

Kontrolli:

```bash
nslookup logger.oige.local 10.0.x.5
```

Võimalikud põhjused:

- DNS tsoonifailis pole kirjet
- serial number jäi suurendamata
- bind9 restart jäi tegemata
- klient ei kasuta õiget DNS serverit

---

### Logid ei jõua logiserverisse

Kontrolli kliendis rsyslog:

```bash
systemctl status rsyslog
```

Kontrolli rsyslog configi:

```bash
cat /etc/rsyslog.conf
```

Seal peab olema:

```text
*.info@@10.0.x.10:514
```

Kontrolli logiserveri tulemüüri:

```bash
sudo ufw status verbose
```

Kontrolli Wazuh configis:

```xml
<allowed-ips>10.0.x.0/24</allowed-ips>
```

---

### GRUB parandamine ei tööta

Kontrolli:

```bash
lsblk
sudo blkid
```

Kui UEFI süsteem, peab EFI partitsioon olema mountitud:

```bash
sudo mount /dev/sda1 /mnt/boot/efi
```

BIOS puhul:

```bash
grub-install /dev/sda
update-grub
```

UEFI puhul:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
update-grub
```

---

### `mount -a` annab vea

Kontrolli UUID:

```bash
sudo blkid /dev/sdb1
```

Kontrolli `/etc/fstab` faili:

```bash
sudo nano /etc/fstab
```

Kasuta `nofail`, et katkine lisaketas ei takistaks bootimist:

```fstab
UUID=SIIN-SINU-UUID /backup ext4 defaults,nofail 0 2
```

---

### Backup skript ütleb, et `/backup` pole ühendatud

Kontrolli:

```bash
df -h
mount | grep backup
```

Proovi:

```bash
sudo mount -a
```

---

### `rsync` puudub

Paigalda:

```bash
sudo apt install rsync -y
```

---

# Lühikokkuvõte

## Pilet 1

Pilet 1 keskendub WordPressi, Debian serveri, HTTPS-i, DNS-i, tulemüüri ja paroolihalduri seadistamisele.

Kõige olulisemad kontrollid:

```bash
ansible all -i inventory.ini -m ping
curl -k -I https://vasemba.oige.local
nslookup vasemba.oige.local 10.0.x.5
sudo ufw status verbose
docker ps
```

---

## Pilet 2

Pilet 2 keskendub kesksele logiserverile, logide saatmisele, GRUB parandamisele ja varundamisele.

Kõige olulisemad kontrollid:

```bash
ansible all -i inventory.ini -m ping
nslookup logger.oige.local 10.0.x.5
docker ps
grep -R "TEST" /var/ossec/logs/
df -h
sudo crontab -l
```
