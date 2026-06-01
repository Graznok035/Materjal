# Linux pilet 4 ülesannete lahenduskäik

See juhend keskendub ainult **Linux pilet 4 eriosa ülesannetele**.

Siin ei ole pikalt lahti kirjutatud üldist Linuxi baasosa:

- Ansible paigaldus UbuntuServerisse
- Ansible inventory loomine
- `hkhk` kasutaja loomine
- sudo gruppi lisamine
- SSH võtmega ligipääs
- SSH parooliga sisselogimise keelamine

Pilet 4 puhul on baasosa siiski oluline, sest osa tarkvarast tuleb ülesande järgi paigaldada **Ansible abil**.

---

## Pilet 4 põhiteemad

| Teema | Mida tuleb teha |
|---|---|
| Veebiserver | Paigaldada ja seadistada PHP toega veebiserver |
| Ansible | Veebiserver ja andmebaasiserver tuleks paigaldada Ansible abil |
| SSL | Veebiserverile tuleb seadistada SSL sertifikaat |
| FQDN | Veebirakendus peab töötama domeeninimega |
| MariaDB | Paigaldada ja seadistada andmebaasiserver |
| phpMyAdmin | Paigaldada andmebaasi haldamiseks phpMyAdmin |
| Andmebaas | Luua andmebaas kasutajatoe rakenduse jaoks |
| CRUD | Veebirakendus peab lubama andmeid lisada, vaadata, muuta ja kustutada |
| Adminiliides | Admin osa peab olema parooliga kaitstud |
| Crontab | Luua automaatne varundusskript |
| CSS / Bootstrap | Leht peab olema kujundatud ja valideeruv |
| Dokumentatsioon | Kõik tegevused ja kontrollid tuleb dokumenteerida |

---

# 1. Soovitatav masinate rollijaotus

Piletis on mõistlik teha nii:

| Masin | Roll |
|---|---|
| UbuntuServer | Ansible juhtmasin |
| AlmaServer või DebianServer | Veebiserver + PHP + rakendus |
| AlmaServer või DebianServer | MariaDB + phpMyAdmin |

Lihtsuse mõttes võib eksamil panna **veebiserveri ja andmebaasi samasse masinasse**, kui ülesanne ei nõua eraldi servereid.

Näidis selles juhendis:

| Masin | Roll |
|---|---|
| UbuntuServer | Ansible |
| DebianServer | Apache + PHP + MariaDB + phpMyAdmin + kasutajatoe rakendus |

Näidis FQDN:

```text
kasutajatugi.sinuNimi.local
```

Asenda `sinuNimi.local` enda domeeniga.

---

# 2. Algkontroll sihtserveris

## Eesmärk

Enne paigaldust kontrollin, kas sihtserveril on:

- IP-aadress
- DNS töötab
- internet töötab
- piisavalt kettaruumi
- õige hostname

---

## 2.1 Kontrolli hostname’i

```bash
hostname
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse serveri nimi, näiteks `debianserver` | Kui nimi on vale, dokumenteeri see või muuda käsuga `sudo hostnamectl set-hostname veebiserver` |

Kontrolli FQDN-i:

```bash
hostname -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse täielik nimi või vähemalt hostname | Kui tuleb error, kontrolli `/etc/hosts` faili ja DNS kirjeid |

---

## 2.2 Kontrolli IP-aadressi

```bash
ip a
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Võrguliidesel on IP-aadress | Kui IP puudub, kontrolli VM võrku, DHCP-d või staatilist IP seadistust |

Kontrolli gateway’d:

```bash
ip route
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `default via ...` rida | Kui default route puudub, ei saa server internetti ega teistesse võrkudesse |

---

## 2.3 Kontrolli DNS-i ja internetti

```bash
ping -c 4 8.8.8.8
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ping töötab | Kui ei tööta, on probleem gateway või võrguühendusega |

```bash
ping -c 4 google.com
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Nimi lahendub ja ping töötab | Kui IP ping töötab, aga domeeninimi mitte, on DNS probleem |

Kontrolli DNS-i:

```bash
resolvectl status
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed DNS serveri aadressi | Kui DNS puudub või on vale, paranda võrgu DNS seadistus |

---

# 3. Lisa DNS kirje veebirakendusele

## Eesmärk

Veebirakendus peab avanema FQDN nimega.

Näide:

```text
kasutajatugi.sinuNimi.local
```

Kui DNS on Windows Serveris, lisa DNS Manageris A-kirje:

| Nimi | IP |
|---|---|
| `kasutajatugi` | veebiserveri IP |

Näide:

```text
kasutajatugi.sinuNimi.local -> DebianServeri IP
```

---

## 3.1 Kontroll Linuxis

```bash
getent hosts kasutajatugi.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse veebiserveri IP | Kui vastust pole, DNS kirje puudub või klient kasutab valet DNS serverit |

```bash
ping -c 4 kasutajatugi.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Nimi lahendub õigeks IP-ks | Kui nimi ei lahendu, kontrolli DNS kirjet või lisa ajutiselt `/etc/hosts` kirje |

---

## 3.2 Ajutine `/etc/hosts` lahendus

Kui DNS pole kohe valmis, lisa ajutiselt:

```bash
sudo nano /etc/hosts
```

Lisa:

```text
SERVERI_IP kasutajatugi.sinuNimi.local kasutajatugi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Nimi hakkab selles masinas lahenduma | Kui ei lahendu, kontrolli IP ja kirjavigu |

---

# 4. Paigalda veebiserver PHP toega Ansible abil

## Eesmärk

Ülesanne ütleb, et veebiserveri tarkvara tuleb paigaldada **Ansible abil**.  
Kui Ansible lahendus ei õnnestu, võib teha käsitsi, aga see tuleb dokumentatsioonis eraldi välja tuua.

---

## 4.1 Kontrolli Ansible ühendust UbuntuServerist

UbuntuServeris:

```bash
ansible all -m ping
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Sihtmasinad vastavad `pong` | Kui tuleb SSH error, kontrolli SSH võtit, kasutajat, inventory faili ja võrku |

Kui kasutad kindlat inventory faili:

```bash
ansible all -i inventory.ini -m ping
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kõik inventory masinad vastavad `pong` | Kui mõni ei vasta, kontrolli selle masina IP-d ja SSH ligipääsu |

---

## 4.2 Näidis inventory

```bash
nano inventory.ini
```

Sisu näiteks:

```ini
[web]
debianserver ansible_host=10.0.0.20 ansible_user=hkhk

[db]
debianserver ansible_host=10.0.0.20 ansible_user=hkhk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Inventory fail sisaldab õiget IP-d ja kasutajat | Kui Ansible ei saa ühendust, on IP, kasutaja või SSH võti vale |

---

## 4.3 Loo Ansible playbook veebiserveri jaoks

```bash
nano install_web.yml
```

Lisa:

```yaml
---
- name: Paigalda Apache ja PHP
  hosts: web
  become: yes

  tasks:
    - name: Uuenda paketiloendit
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Paigalda Apache ja PHP paketid
      apt:
        name:
          - apache2
          - php
          - php-mysql
          - php-cli
          - php-curl
          - php-gd
          - php-mbstring
          - php-xml
          - php-zip
          - libapache2-mod-php
        state: present
      when: ansible_os_family == "Debian"

    - name: Luba Apache käivitumisel
      service:
        name: apache2
        enabled: yes
        state: started
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei salvestu, kontrolli, et oled õiges kaustas |

---

## 4.4 Käivita playbook

```bash
ansible-playbook -i inventory.ini install_web.yml
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Playbook lõpeb `failed=0` | Kui on `failed`, loe errorit: tavaliselt on probleem sudo õigustes, apt repos või SSH ühenduses |

Kontrolli sihtserveris Apache staatust:

```bash
systemctl status apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui `inactive`, käivita `sudo systemctl start apache2`. Kui `failed`, kontrolli `sudo apache2ctl configtest` |

Kontrolli PHP-d:

```bash
php -v
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse PHP versioon | Kui `command not found`, PHP ei paigaldunud |

Kontrolli Apache vastust:

```bash
curl -I http://localhost
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP vastus `200 OK`, `301` või `302` | Kui `connection refused`, Apache ei tööta või port 80 ei kuula |

---

# 5. Paigalda MariaDB ja phpMyAdmin Ansible abil

## Eesmärk

Andmebaasiserver ja phpMyAdmin tuleb võimalusel paigaldada Ansible abil.

---

## 5.1 Loo Ansible playbook andmebaasi jaoks

UbuntuServeris:

```bash
nano install_db.yml
```

Lisa:

```yaml
---
- name: Paigalda MariaDB ja phpMyAdmin
  hosts: db
  become: yes

  tasks:
    - name: Uuenda paketiloendit
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Paigalda MariaDB ja phpMyAdmin
      apt:
        name:
          - mariadb-server
          - mariadb-client
          - phpmyadmin
          - php-mbstring
          - php-zip
          - php-gd
          - php-json
          - php-curl
        state: present
      when: ansible_os_family == "Debian"

    - name: Luba MariaDB käivitumisel
      service:
        name: mariadb
        enabled: yes
        state: started
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Playbook fail salvestub | Kui fail ei salvestu, kontrolli õiguseid ja asukohta |

---

## 5.2 Käivita playbook

```bash
ansible-playbook -i inventory.ini install_db.yml
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Playbook lõpeb `failed=0` | Kui phpMyAdmin küsib interaktiivseid valikuid ja Ansible jääb kinni, paigalda phpMyAdmin käsitsi ning dokumenteeri see |

Kontrolli MariaDB staatust sihtserveris:

```bash
systemctl status mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui `failed`, vaata `sudo journalctl -xeu mariadb` |

Kontrolli MariaDB versiooni:

```bash
mariadb --version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse MariaDB versioon | Kui käsk puudub, MariaDB ei paigaldunud |

---

# 6. Kui Ansible ei õnnestu: käsitsi paigaldus

Kui Ansible osa ei tööta ja aeg hakkab kaduma, tee käsitsi ning kirjuta dokumentatsiooni:

```text
Veebiserveri ja/või andmebaasi paigaldus tehti käsitsi, sest Ansible playbook ebaõnnestus.
Viga: ...
Kontroll: teenused töötavad.
```

Käsitsi paigaldus Debian/Ubuntu serveris:

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui tuleb DNS/repo error, kontrolli internetti ja DNS-i |

```bash
sudo apt install apache2 php php-mysql php-cli php-curl php-gd php-mbstring php-xml php-zip libapache2-mod-php -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache ja PHP paigaldatakse | Kui paketti ei leita, kontrolli apt repository seadistust |

```bash
sudo apt install mariadb-server mariadb-client phpmyadmin -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| MariaDB ja phpMyAdmin paigaldatakse | Kui phpMyAdmin küsib valikuid, vali Apache ja seadista dbconfig-common vajadusel |

---

# 7. Seadista MariaDB root parool ja turvaseaded

## 7.1 Käivita turvaseadistus

```bash
sudo mysql_secure_installation
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Saad määrata root parooli ja eemaldada ebaturvalised vaikeseaded | Kui käsk puudub, kontrolli MariaDB paigaldust |

Soovitatavad vastused:

| Küsimus | Vastus |
|---|---|
| Switch to unix_socket authentication? | N või Y, sõltub süsteemist |
| Change root password? | Y |
| Remove anonymous users? | Y |
| Disallow root login remotely? | Y |
| Remove test database? | Y |
| Reload privilege tables? | Y |

---

## 7.2 Testi MariaDB ligipääsu

```bash
sudo mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Avaneb MariaDB käsurida | Kui access denied, kasuta `sudo mysql` või kontrolli root autentimist |

MariaDB sees:

```sql
SHOW DATABASES;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse andmebaaside nimekiri | Kui tuleb error, pole õiguseid või MariaDB ei tööta |

Välju:

```sql
EXIT;
```

---

# 8. Loo andmebaas kasutajatoe rakenduse jaoks

## Eesmärk

Luua andmebaas, tabelid ja kasutaja, mida PHP rakendus kasutab.

Vajalikud tabelid:

| Tabel | Milleks |
|---|---|
| `tickets` | Kasutajate IT-probleemide salvestamiseks |
| `admins` | Admin kasutaja parooli salvestamiseks |

---

## 8.1 Logi MariaDB-sse

```bash
sudo mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Avaneb MariaDB prompt | Kui ei avane, kontrolli `systemctl status mariadb` |

---

## 8.2 Loo andmebaas ja kasutaja

MariaDB sees:

```sql
CREATE DATABASE kasutajatugi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Query OK` | Kui andmebaas on juba olemas, jätka või kustuta ainult juhul, kui oled kindel |

Loo andmebaasi kasutaja:

```sql
CREATE USER 'kasutajatugi_user'@'localhost' IDENTIFIED BY 'TugevParool123!';
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Query OK` | Kui kasutaja on juba olemas, kasuta `ALTER USER` |

Kui kasutaja on olemas:

```sql
ALTER USER 'kasutajatugi_user'@'localhost' IDENTIFIED BY 'TugevParool123!';
```

Anna õigused:

```sql
GRANT ALL PRIVILEGES ON kasutajatugi.* TO 'kasutajatugi_user'@'localhost';
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Query OK` | Kui tuleb õiguste viga, veendu, et oled MariaDB-s root õigustes |

Rakenda õigused:

```sql
FLUSH PRIVILEGES;
```

Vali andmebaas:

```sql
USE kasutajatugi;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Database changed` | Kui andmebaasi pole, kontrolli `SHOW DATABASES;` |

---

## 8.3 Loo `tickets` tabel

```sql
CREATE TABLE tickets (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nimi VARCHAR(100) NOT NULL,
    osakond VARCHAR(100) NOT NULL,
    kontakt VARCHAR(150) NOT NULL,
    kirjeldus TEXT NOT NULL,
    staatus ENUM('uus','toös','lahendatud') DEFAULT 'uus',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Query OK` | Kui tabel on juba olemas, kontrolli `SHOW TABLES;` |

---

## 8.4 Loo `admins` tabel

```sql
CREATE TABLE admins (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL
);
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Query OK` | Kui tabel on juba olemas, jätka admin kasutaja loomisega |

---

## 8.5 Loo admin kasutaja

PHP password_hash genereerimiseks kasuta Linuxi käsureal:

```bash
php -r "echo password_hash('AdminParool123!', PASSWORD_DEFAULT), PHP_EOL;"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse pikk hash, mis algab tihti `$2y$` või `$argon2` | Kui `php` puudub, paigalda PHP või kasuta ajutiselt mõnda muud hash’i loomise meetodit |

Kopeeri saadud hash.

Mine tagasi MariaDB-sse:

```bash
sudo mariadb
```

Vali andmebaas:

```sql
USE kasutajatugi;
```

Lisa admin:

```sql
INSERT INTO admins (username, password_hash)
VALUES ('admin', 'SIIA_KOPEERI_HASH');
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Query OK` | Kui username on juba olemas, tee `UPDATE admins SET password_hash='HASH' WHERE username='admin';` |

Kontrolli tabeleid:

```sql
SHOW TABLES;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `admins` ja `tickets` | Kui tabeleid pole, SQL käsud ei jooksnud õiges andmebaasis |

Kontrolli admini:

```sql
SELECT id, username FROM admins;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed kasutajat `admin` | Kui ei näe, lisa admin uuesti |

Välju:

```sql
EXIT;
```

---

# 9. Loo veebirakenduse kaust

## Eesmärk

Luua rakenduse failid Apache veebikausta.

Näidis asukoht:

```text
/var/www/kasutajatugi
```

---

## 9.1 Loo kaust

```bash
sudo mkdir -p /var/www/kasutajatugi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse | Kui permission denied, kasuta `sudo` |

Mine kausta:

```bash
cd /var/www/kasutajatugi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Oled rakenduse kaustas | Kui kausta pole, loo see eelmise käsuga |

---

## 9.2 Loo andmebaasi ühenduse fail

```bash
sudo nano db.php
```

Lisa:

```php
<?php
$host = 'localhost';
$db   = 'kasutajatugi';
$user = 'kasutajatugi_user';
$pass = 'TugevParool123!';
$charset = 'utf8mb4';

$dsn = "mysql:host=$host;dbname=$db;charset=$charset";

$options = [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
];

try {
    $pdo = new PDO($dsn, $user, $pass, $options);
} catch (PDOException $e) {
    die("Andmebaasi ühendus ebaõnnestus.");
}
?>
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei saa salvestada, kontrolli `sudo nano` kasutamist |

---

# 10. Loo avalik kasutajatoe leht

## 10.1 Loo `index.php`

```bash
sudo nano index.php
```

Lisa:

```php
<?php
require 'db.php';

$message = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $nimi = trim($_POST['nimi'] ?? '');
    $osakond = trim($_POST['osakond'] ?? '');
    $kontakt = trim($_POST['kontakt'] ?? '');
    $kirjeldus = trim($_POST['kirjeldus'] ?? '');

    if ($nimi && $osakond && $kontakt && $kirjeldus) {
        $stmt = $pdo->prepare("INSERT INTO tickets (nimi, osakond, kontakt, kirjeldus) VALUES (?, ?, ?, ?)");
        $stmt->execute([$nimi, $osakond, $kontakt, $kirjeldus]);
        $message = "Pöördumine on edukalt salvestatud.";
    } else {
        $message = "Palun täida kõik väljad.";
    }
}
?>
<!doctype html>
<html lang="et">
<head>
    <meta charset="utf-8">
    <title>Kasutajatugi</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link href="style.css" rel="stylesheet">
</head>
<body>
<header class="banner">
    <h1>Kasutajatugi</h1>
    <p>IT-probleemide registreerimise keskkond</p>
</header>

<nav class="menu">
    <a href="index.php">Avaleht</a>
    <a href="kkk.php">KKK</a>
    <a href="kontakt.php">Kontakt</a>
    <a href="admin.php">Admin</a>
</nav>

<main class="container">
    <h2>Esita IT-probleem</h2>

    <?php if ($message): ?>
        <div class="notice"><?= htmlspecialchars($message) ?></div>
    <?php endif; ?>

    <form method="post">
        <label>Nimi</label>
        <input type="text" name="nimi" required>

        <label>Osakond</label>
        <input type="text" name="osakond" required>

        <label>Kontakt e-post või telefon</label>
        <input type="text" name="kontakt" required>

        <label>Probleemi kirjeldus</label>
        <textarea name="kirjeldus" rows="6" required></textarea>

        <button type="submit">Saada pöördumine</button>
    </form>
</main>
</body>
</html>
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui veebis kuvatakse PHP kood tekstina, PHP moodul ei tööta Apachega |

---

## 10.2 Loo KKK leht

```bash
sudo nano kkk.php
```

Lisa:

```php
<!doctype html>
<html lang="et">
<head>
    <meta charset="utf-8">
    <title>KKK - Kasutajatugi</title>
    <link href="style.css" rel="stylesheet">
</head>
<body>
<header class="banner">
    <h1>Kasutajatugi</h1>
    <p>Korduma kippuvad küsimused</p>
</header>

<nav class="menu">
    <a href="index.php">Avaleht</a>
    <a href="kkk.php">KKK</a>
    <a href="kontakt.php">Kontakt</a>
    <a href="admin.php">Admin</a>
</nav>

<main class="container">
    <h2>KKK</h2>
    <h3>Mida teha, kui arvuti ei käivitu?</h3>
    <p>Kontrolli toitekaablit ja ekraani ühendust. Vajadusel esita pöördumine.</p>

    <h3>Mida teha, kui internet ei tööta?</h3>
    <p>Kontrolli võrguühendust ja proovi arvuti taaskäivitada.</p>
</main>
</body>
</html>
```

---

## 10.3 Loo kontakt leht

```bash
sudo nano kontakt.php
```

Lisa:

```php
<!doctype html>
<html lang="et">
<head>
    <meta charset="utf-8">
    <title>Kontakt - Kasutajatugi</title>
    <link href="style.css" rel="stylesheet">
</head>
<body>
<header class="banner">
    <h1>Kasutajatugi</h1>
    <p>Kontaktinfo</p>
</header>

<nav class="menu">
    <a href="index.php">Avaleht</a>
    <a href="kkk.php">KKK</a>
    <a href="kontakt.php">Kontakt</a>
    <a href="admin.php">Admin</a>
</nav>

<main class="container">
    <h2>Kontakt</h2>
    <p>IT kasutajatugi</p>
    <p>E-post: itabi@example.local</p>
    <p>Telefon: +372 5555 5555</p>
</main>
</body>
</html>
```

---

# 11. Loo adminiliides

## Eesmärk

Adminiliides peab võimaldama:

- adminina sisse logida
- näha kõiki pöördumisi
- lisada uus pöördumine
- muuta pöördumise staatust
- kustutada pöördumine jäädavalt

---

## 11.1 Loo `admin.php`

```bash
sudo nano admin.php
```

Lisa:

```php
<?php
session_start();
require 'db.php';

$error = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $username = $_POST['username'] ?? '';
    $password = $_POST['password'] ?? '';

    $stmt = $pdo->prepare("SELECT * FROM admins WHERE username = ?");
    $stmt->execute([$username]);
    $admin = $stmt->fetch();

    if ($admin && password_verify($password, $admin['password_hash'])) {
        $_SESSION['admin'] = $admin['username'];
        header("Location: admin_panel.php");
        exit;
    } else {
        $error = "Vale kasutajanimi või parool.";
    }
}
?>
<!doctype html>
<html lang="et">
<head>
    <meta charset="utf-8">
    <title>Admin login</title>
    <link href="style.css" rel="stylesheet">
</head>
<body>
<header class="banner">
    <h1>Kasutajatugi</h1>
    <p>Administreerimisliides</p>
</header>

<main class="container">
    <h2>Admin login</h2>

    <?php if ($error): ?>
        <div class="notice error"><?= htmlspecialchars($error) ?></div>
    <?php endif; ?>

    <form method="post">
        <label>Kasutajanimi</label>
        <input type="text" name="username" required>

        <label>Parool</label>
        <input type="password" name="password" required>

        <button type="submit">Logi sisse</button>
    </form>
</main>
</body>
</html>
```

---

## 11.2 Loo `admin_panel.php`

```bash
sudo nano admin_panel.php
```

Lisa:

```php
<?php
session_start();
require 'db.php';

if (!isset($_SESSION['admin'])) {
    header("Location: admin.php");
    exit;
}

if (isset($_GET['delete'])) {
    $id = (int)$_GET['delete'];
    $stmt = $pdo->prepare("DELETE FROM tickets WHERE id = ?");
    $stmt->execute([$id]);
    header("Location: admin_panel.php");
    exit;
}

if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['update_status'])) {
    $id = (int)$_POST['id'];
    $staatus = $_POST['staatus'];

    $stmt = $pdo->prepare("UPDATE tickets SET staatus = ? WHERE id = ?");
    $stmt->execute([$staatus, $id]);
}

if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['add_ticket'])) {
    $stmt = $pdo->prepare("INSERT INTO tickets (nimi, osakond, kontakt, kirjeldus, staatus) VALUES (?, ?, ?, ?, ?)");
    $stmt->execute([
        $_POST['nimi'],
        $_POST['osakond'],
        $_POST['kontakt'],
        $_POST['kirjeldus'],
        $_POST['staatus']
    ]);
}

$tickets = $pdo->query("SELECT * FROM tickets ORDER BY created_at DESC")->fetchAll();
?>
<!doctype html>
<html lang="et">
<head>
    <meta charset="utf-8">
    <title>Admin paneel</title>
    <link href="style.css" rel="stylesheet">
</head>
<body>
<header class="banner">
    <h1>Kasutajatugi</h1>
    <p>Admin paneel</p>
</header>

<nav class="menu">
    <a href="index.php">Avaleht</a>
    <a href="admin_panel.php">Admin paneel</a>
    <a href="logout.php">Logi välja</a>
</nav>

<main class="container">
    <h2>Lisa uus pöördumine</h2>

    <form method="post">
        <input type="hidden" name="add_ticket" value="1">

        <label>Nimi</label>
        <input type="text" name="nimi" required>

        <label>Osakond</label>
        <input type="text" name="osakond" required>

        <label>Kontakt</label>
        <input type="text" name="kontakt" required>

        <label>Kirjeldus</label>
        <textarea name="kirjeldus" required></textarea>

        <label>Staatus</label>
        <select name="staatus">
            <option value="uus">uus</option>
            <option value="toös">töös</option>
            <option value="lahendatud">lahendatud</option>
        </select>

        <button type="submit">Lisa</button>
    </form>

    <h2>Kõik pöördumised</h2>

    <table>
        <tr>
            <th>ID</th>
            <th>Nimi</th>
            <th>Osakond</th>
            <th>Kontakt</th>
            <th>Kirjeldus</th>
            <th>Staatus</th>
            <th>Aeg</th>
            <th>Tegevus</th>
        </tr>

        <?php foreach ($tickets as $ticket): ?>
        <tr>
            <td><?= htmlspecialchars($ticket['id']) ?></td>
            <td><?= htmlspecialchars($ticket['nimi']) ?></td>
            <td><?= htmlspecialchars($ticket['osakond']) ?></td>
            <td><?= htmlspecialchars($ticket['kontakt']) ?></td>
            <td><?= htmlspecialchars($ticket['kirjeldus']) ?></td>
            <td>
                <form method="post">
                    <input type="hidden" name="update_status" value="1">
                    <input type="hidden" name="id" value="<?= htmlspecialchars($ticket['id']) ?>">
                    <select name="staatus">
                        <option value="uus" <?= $ticket['staatus'] === 'uus' ? 'selected' : '' ?>>uus</option>
                        <option value="toös" <?= $ticket['staatus'] === 'toös' ? 'selected' : '' ?>>töös</option>
                        <option value="lahendatud" <?= $ticket['staatus'] === 'lahendatud' ? 'selected' : '' ?>>lahendatud</option>
                    </select>
                    <button type="submit">Muuda</button>
                </form>
            </td>
            <td><?= htmlspecialchars($ticket['created_at']) ?></td>
            <td>
                <a href="admin_panel.php?delete=<?= htmlspecialchars($ticket['id']) ?>" onclick="return confirm('Kas kustutan jäädavalt?')">Kustuta</a>
            </td>
        </tr>
        <?php endforeach; ?>
    </table>
</main>
</body>
</html>
```

---

## 11.3 Loo logout fail

```bash
sudo nano logout.php
```

Lisa:

```php
<?php
session_start();
session_destroy();
header("Location: admin.php");
exit;
```

---

# 12. Loo CSS fail

## Eesmärk

Leht peab olema kujundatud. Ülesandes mainitakse Bootstrap raamistikku ja CSS-i valideerimist.  
Lihtsuse mõttes kasutan siin oma CSS-i. Kui tahad Bootstrapit, saab selle juurde lisada CDN-iga.

```bash
sudo nano style.css
```

Lisa:

```css
body {
    margin: 0;
    font-family: Arial, sans-serif;
    background: #f4f6f8;
    color: #222;
}

.banner {
    background: #1f2937;
    color: white;
    padding: 30px;
    text-align: center;
}

.banner h1 {
    margin: 0;
    font-size: 36px;
}

.menu {
    background: #374151;
    padding: 12px;
    text-align: center;
}

.menu a {
    color: white;
    text-decoration: none;
    margin: 0 15px;
    font-weight: bold;
}

.menu a:hover {
    text-decoration: underline;
}

.container {
    max-width: 1000px;
    margin: 30px auto;
    background: white;
    padding: 25px;
    border-radius: 8px;
}

form {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

input,
textarea,
select {
    padding: 10px;
    border: 1px solid #bbb;
    border-radius: 4px;
}

button {
    background: #2563eb;
    color: white;
    border: 0;
    padding: 10px;
    border-radius: 4px;
    cursor: pointer;
}

button:hover {
    background: #1d4ed8;
}

.notice {
    background: #d1fae5;
    padding: 12px;
    border: 1px solid #10b981;
    margin-bottom: 15px;
}

.notice.error {
    background: #fee2e2;
    border-color: #ef4444;
}

table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 20px;
}

td,
th {
    border: 1px solid #ddd;
    padding: 8px;
    vertical-align: top;
}

th {
    background: #f3f4f6;
}
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub ja leht saab kujunduse | Kui kujundus ei rakendu, kontrolli, kas `style.css` on samas kaustas ja HTML viitab õigesti |

---

# 13. Seadista Apache VirtualHost

## Eesmärk

Rakendus peab avanema FQDN kaudu:

```text
https://kasutajatugi.sinuNimi.local
```

---

## 13.1 Loo Apache konfiguratsioon

```bash
sudo nano /etc/apache2/sites-available/kasutajatugi.conf
```

Lisa HTTP osa:

```apache
<VirtualHost *:80>
    ServerName kasutajatugi.sinuNimi.local
    DocumentRoot /var/www/kasutajatugi

    <Directory /var/www/kasutajatugi>
        AllowOverride All
        Require all granted
        Options -Indexes
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/kasutajatugi_error.log
    CustomLog ${APACHE_LOG_DIR}/kasutajatugi_access.log combined
</VirtualHost>
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| VirtualHost fail salvestub | Kui ei saa salvestada, kontrolli `sudo nano` kasutamist |

Oluline rida:

```apache
Options -Indexes
```

See keelab kataloogide sirvimise.

---

## 13.2 Luba sait ja moodulid

```bash
sudo a2ensite kasutajatugi.conf
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Sait lubatakse | Kui failinimi vale, kontrolli `ls /etc/apache2/sites-available/` |

```bash
sudo a2dissite 000-default.conf
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Vaikimisi sait keelatakse | Kui juba keelatud, on korras |

```bash
sudo a2enmod rewrite
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Rewrite moodul lubatakse või oli juba lubatud | Kui käsk puudub, Apache pole korralikult paigaldatud |

Kontrolli Apache konfiguratsiooni:

```bash
sudo apache2ctl configtest
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Syntax OK` | Kui näitab veaga faili ja rida, paranda see fail |

Laadi Apache uuesti:

```bash
sudo systemctl reload apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache laeb seadistuse | Kui error, kontrolli `systemctl status apache2` |

---

# 14. Seadista õigused

```bash
sudo chown -R www-data:www-data /var/www/kasutajatugi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Rakenduse failide omanikuks saab `www-data` | Kui kaust puudub, kontrolli asukohta |

```bash
sudo find /var/www/kasutajatugi -type d -exec chmod 755 {} \;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaustade õigused seatakse | Kui tuleb error, kontrolli kausta olemasolu |

```bash
sudo find /var/www/kasutajatugi -type f -exec chmod 644 {} \;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failide õigused seatakse | Kui tuleb error, kontrolli failiõiguseid |

---

# 15. Testi veebirakendust HTTP kaudu

```bash
curl -I http://kasutajatugi.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP vastus `200 OK` või `302` | Kui nimi ei lahendu, kontrolli DNS. Kui connection refused, kontrolli Apache staatust |

Testi lokaalselt:

```bash
curl -I http://localhost
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache vastab | Kui ei vasta, Apache ei tööta või VirtualHost on vigane |

Brauseris ava:

```text
http://kasutajatugi.sinuNimi.local
```

Kontrolli:

| Kontroll | Oodatud tulemus |
|---|---|
| Avaleht avaneb | Näed vormi |
| Vorm salvestab | Pöördumine tekib andmebaasi |
| KKK avaneb | KKK leht töötab |
| Kontakt avaneb | Kontakt leht töötab |
| Admin avaneb | Login vorm kuvatakse |

---

# 16. Kontrolli, kas vorm salvestab andmebaasi

Täida veebivorm ja seejärel kontrolli MariaDB-s:

```bash
sudo mariadb
```

```sql
USE kasutajatugi;
SELECT id, nimi, osakond, kontakt, staatus, created_at FROM tickets;
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed veebivormist sisestatud pöördumist | Kui tabel on tühi, kontrolli `db.php`, PHP error logi ja andmebaasi õiguseid |

Välju:

```sql
EXIT;
```

---

# 17. Seadista SSL sertifikaat

## Eesmärk

Veebileht peab avanema HTTPS kaudu.

Kui ametlikku sertifikaati ei saa kasutada, sobib eksamil self-signed sertifikaat.

---

## 17.1 Luba SSL moodul

```bash
sudo a2enmod ssl
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SSL moodul lubatakse või oli juba lubatud | Kui käsk puudub, Apache pole paigaldatud |

```bash
sudo systemctl restart apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache käivitub uuesti | Kui failed, kontrolli `sudo apache2ctl configtest` |

---

## 17.2 Loo sertifikaadi kaust

```bash
sudo mkdir -p /etc/ssl/localcerts
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse või oli olemas | Kui permission denied, kasuta sudo |

---

## 17.3 Loo self-signed sertifikaat

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/localcerts/kasutajatugi.key \
-out /etc/ssl/localcerts/kasutajatugi.crt
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Luuakse `.key` ja `.crt` fail | Kui `openssl` puudub, paigalda `sudo apt install openssl -y` |

Common Name küsimuse juures sisesta:

```text
kasutajatugi.sinuNimi.local
```

Kontrolli faile:

```bash
ls -lah /etc/ssl/localcerts/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `kasutajatugi.key` ja `kasutajatugi.crt` | Kui faile pole, korda openssl käsku |

---

## 17.4 Lisa HTTPS VirtualHost

Ava konfiguratsioon:

```bash
sudo nano /etc/apache2/sites-available/kasutajatugi-ssl.conf
```

Lisa:

```apache
<VirtualHost *:443>
    ServerName kasutajatugi.sinuNimi.local
    DocumentRoot /var/www/kasutajatugi

    SSLEngine on
    SSLCertificateFile /etc/ssl/localcerts/kasutajatugi.crt
    SSLCertificateKeyFile /etc/ssl/localcerts/kasutajatugi.key

    <Directory /var/www/kasutajatugi>
        AllowOverride All
        Require all granted
        Options -Indexes
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/kasutajatugi_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/kasutajatugi_ssl_access.log combined
</VirtualHost>
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTPS VirtualHost salvestub | Kui ei salvestu, kontrolli `sudo nano` |

Luba sait:

```bash
sudo a2ensite kasutajatugi-ssl.conf
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SSL sait lubatakse | Kui failinimi vale, kontrolli `sites-available` kausta |

Kontrolli Apache konfiguratsiooni:

```bash
sudo apache2ctl configtest
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Syntax OK` | Kui näitab errorit, paranda viidatud fail |

Taaskäivita Apache:

```bash
sudo systemctl restart apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache käivitub | Kui failed, vaata `journalctl -xeu apache2` |

---

## 17.5 Testi HTTPS

```bash
curl -k -I https://kasutajatugi.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP vastus `200 OK`, `301` või `302` | Kui DNS error, kontrolli DNS kirjet. Kui connection refused, kontrolli porti 443 ja Apache staatust |

Kontrolli, kas port 443 kuulab:

```bash
sudo ss -tulpen | grep :443
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed Apache protsessi pordil 443 | Kui ei näe, SSL VirtualHost ei tööta või Apache ei käivitunud |

---

# 18. Paigalda ja kontrolli phpMyAdmin

## 18.1 Kontrolli phpMyAdmini olemasolu

```bash
ls -lah /usr/share/phpmyadmin
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust on olemas | Kui puudub, phpMyAdmin ei paigaldunud |

Kui phpMyAdmin ei avane Apache all, lisa alias.

```bash
sudo nano /etc/apache2/conf-available/phpmyadmin.conf
```

Lisa:

```apache
Alias /phpmyadmin /usr/share/phpmyadmin

<Directory /usr/share/phpmyadmin>
    Options SymLinksIfOwnerMatch
    DirectoryIndex index.php
    Require all granted
</Directory>
```

Luba konfiguratsioon:

```bash
sudo a2enconf phpmyadmin.conf
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| phpMyAdmin conf lubatakse | Kui failinimi vale, kontrolli `conf-available` kausta |

Laadi Apache uuesti:

```bash
sudo systemctl reload apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Apache laeb seadistuse | Kui error, kontrolli Apache configtesti |

Testi:

```bash
curl -k -I https://kasutajatugi.sinuNimi.local/phpmyadmin
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP vastus phpMyAdminilt | Kui 404, alias puudub või Apache conf pole lubatud |

---

# 19. Seadista UFW tulemüür

## Eesmärk

Lubatud peavad olema ainult vajalikud pordid.

Veebiserveri puhul:

| Port | Teenus |
|---|---|
| 22/tcp | SSH |
| 80/tcp | HTTP |
| 443/tcp | HTTPS |

MariaDB porti 3306 ei pea avama, kui andmebaas on samas masinas ja PHP ühendub `localhost` kaudu.

---

## 19.1 Paigalda UFW

```bash
sudo apt install ufw -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW paigaldatakse | Kui paketti ei leita, kontrolli apt repo |

---

## 19.2 Määra vaikereeglid

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

## 19.3 Luba vajalikud pordid

```bash
sudo ufw allow 22/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SSH lubatakse | Kui kasutad muud SSH porti, luba see enne UFW aktiveerimist |

```bash
sudo ufw allow 80/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP lubatakse | Kui HTTP pole vajalik, võib hiljem eemaldada, aga testimiseks on kasulik |

```bash
sudo ufw allow 443/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTPS lubatakse | Kui HTTPS ei avane, kontrolli ka Apache porti 443 |

---

## 19.4 Lülita UFW sisse

```bash
sudo ufw enable
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW aktiveerub | Kui oled SSH-ga sees ja 22/tcp pole lubatud, võid ühenduse kaotada |

Kontroll:

```bash
sudo ufw status numbered
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed 22/tcp, 80/tcp ja 443/tcp | Kui mõni puudub, lisa see `sudo ufw allow PORT/tcp` käsuga |

---

# 20. Loo varundusskript crontab jaoks

## Eesmärk

Ülesandes on vaja luua skript, mis teeb varukoopiaid.

Varukoopia peab:

- sisaldama veebirakenduse faile
- sisaldama andmebaasi dumpi
- olema kokku pakitud
- sisaldama failinimes kuupäeva ja kellaaega
- kustutama vanemad kui 7 päeva varukoopiad
- olema piisavalt kommenteeritud
- käivituma crontabiga

---

## 20.1 Loo backup kaust

```bash
sudo mkdir -p /var/backups/kasutajatugi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse | Kui permission denied, kasuta sudo |

---

## 20.2 Loo skript

```bash
sudo nano /usr/local/sbin/backup-kasutajatugi.sh
```

Lisa:

```bash
#!/bin/bash

# Kasutajatoe veebirakenduse ja andmebaasi varundusskript
# Skript loob andmebaasi dumpi, pakib veebifailid ja andmebaasi kokku
# ning kustutab vanemad kui 7 päeva varukoopiad.

BACKUP_DIR="/var/backups/kasutajatugi"
WEB_DIR="/var/www/kasutajatugi"
DATE=$(date +"%Y-%m-%d_%H-%M-%S")

DB_NAME="kasutajatugi"
DB_USER="kasutajatugi_user"
DB_PASS="TugevParool123!"

TMP_DIR="/tmp/kasutajatugi_backup_$DATE"
ARCHIVE_NAME="kasutajatugi_backup_$DATE.tar.gz"

mkdir -p "$BACKUP_DIR"
mkdir -p "$TMP_DIR"

# Andmebaasi varukoopia
mysqldump -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" > "$TMP_DIR/database.sql"

# Veebifailide kopeerimine
cp -a "$WEB_DIR" "$TMP_DIR/web"

# Varukoopia pakkimine
tar -czf "$BACKUP_DIR/$ARCHIVE_NAME" -C "$TMP_DIR" .

# Ajutise kausta eemaldamine
rm -rf "$TMP_DIR"

# Vanemate kui 7 päeva varukoopiate kustutamine
find "$BACKUP_DIR" -name "kasutajatugi_backup_*.tar.gz" -type f -mtime +7 -delete
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Skript salvestub | Kui ei saa salvestada, kasuta `sudo nano` |

---

## 20.3 Anna skriptile käivitusõigus

```bash
sudo chmod +x /usr/local/sbin/backup-kasutajatugi.sh
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Skript saab käivitusõiguse | Kui fail puudub, kontrolli asukohta |

Kontrolli:

```bash
ls -lah /usr/local/sbin/backup-kasutajatugi.sh
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failil on `x` käivitusõigus | Kui `x` puudub, korda `chmod +x` |

---

## 20.4 Testi skripti käsitsi

```bash
sudo /usr/local/sbin/backup-kasutajatugi.sh
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk lõpeb veata ja varukoopia tekib | Kui tuleb MySQL access denied, kontrolli DB kasutajat ja parooli skriptis |

Kontrolli varukoopiaid:

```bash
ls -lah /var/backups/kasutajatugi/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `.tar.gz` faili kuupäeva ja kellaajaga | Kui faili pole, skript ebaõnnestus |

Kontrolli arhiivi sisu:

```bash
tar -tzf /var/backups/kasutajatugi/kasutajatugi_backup_*.tar.gz | head
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `database.sql` ja `web` faile/kaustu | Kui arhiiv vigane, kontrolli `tar` ja skripti käske |

---

## 20.5 Lisa cron töö

Ava root crontab:

```bash
sudo crontab -e
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Avaneb crontab redaktor | Kui küsib redaktorit, vali `nano` |

Kui ülesanne nõuab iga minuti tagant varundamist, lisa:

```text
* * * * * /usr/local/sbin/backup-kasutajatugi.sh >> /var/log/kasutajatugi-backup.log 2>&1
```

Kui soovid mõistlikumat varianti, näiteks kord ööpäevas kell 02:00:

```text
0 2 * * * /usr/local/sbin/backup-kasutajatugi.sh >> /var/log/kasutajatugi-backup.log 2>&1
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Cron töö salvestub | Kui crontab annab errori, kontrolli süntaksit |

Kontrolli crontabi:

```bash
sudo crontab -l
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed lisatud backup rida | Kui rida puudub, lisa see uuesti |

Kontrolli backup logi pärast cron käivitumist:

```bash
sudo tail -n 50 /var/log/kasutajatugi-backup.log
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed varunduse väljundit või logi on tühi, kui vigu polnud | Kui näed errorit, paranda skripti vastavalt |

---

# 21. Lõputestid

## 21.1 Apache ja PHP

```bash
systemctl status apache2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui failed, kontrolli `apache2ctl configtest` ja logisid |

```bash
php -v
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse PHP versioon | Kui puudub, paigalda PHP |

```bash
curl -I http://kasutajatugi.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP vastus tuleb | Kui DNS error, kontrolli DNS kirjet |

```bash
curl -k -I https://kasutajatugi.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTPS vastus tuleb | Kui ei tule, kontrolli SSL VirtualHosti ja UFW-d |

---

## 21.2 MariaDB ja andmebaas

```bash
systemctl status mariadb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui failed, kontrolli `journalctl -xeu mariadb` |

```bash
sudo mariadb -e "SHOW DATABASES;"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `kasutajatugi` andmebaasi | Kui ei näe, andmebaas jäi loomata |

```bash
sudo mariadb -e "USE kasutajatugi; SHOW TABLES;"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `tickets` ja `admins` tabeleid | Kui ei näe, tabelid jäid loomata |

---

## 21.3 Veebirakendus

Brauseris kontrolli:

| Kontroll | Oodatav tulemus |
|---|---|
| `https://kasutajatugi.sinuNimi.local` | Avaleht avaneb |
| Vorm täidetakse | Andmed salvestuvad |
| `KKK` menüü | KKK leht avaneb |
| `Kontakt` menüü | Kontakt leht avaneb |
| `Admin` | Login avaneb |
| Admin login | Sisse saab admin kasutajaga |
| Admin paneel | Näeb pöördumisi |
| Staatuse muutmine | Staatus muutub |
| Kustutamine | Pöördumine kustub |

---

## 21.4 Varukoopiad

```bash
sudo /usr/local/sbin/backup-kasutajatugi.sh
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Varukoopia luuakse | Kui viga, kontrolli MySQL parooli ja kaustade õiguseid |

```bash
ls -lah /var/backups/kasutajatugi/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed varukoopia `.tar.gz` faili | Kui ei näe, skript ei töötanud |

```bash
sudo crontab -l
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed cron rida | Kui rida puudub, lisa uuesti |

---

# 22. Dokumentatsiooni näidis

```markdown
## Linux pilet 4 dokumentatsioon

### Eesmärk

Eesmärk oli luua ettevõtte sisene kasutajatoe veebirakendus, mille kaudu töötajad saavad sisestada IT-probleeme ning administraator saab neid hallata. Lisaks tuli seadistada PHP toega veebiserver, SSL, MariaDB andmebaasiserver, phpMyAdmin, CRUD funktsionaalsus ja automaatsed varukoopiad cron abil.

### Kasutatud masinad

| Masin | Roll |
|---|---|
| UbuntuServer | Ansible juhtmasin |
| DebianServer | Apache, PHP, MariaDB, phpMyAdmin ja kasutajatoe rakendus |
| DNS server | FQDN kirje `kasutajatugi.sinuNimi.local` |

### Tehtud seadistused

- Kontrollisin sihtserveri võrguühendust, DNS-i ja hostname’i.
- Lisasin DNS kirje `kasutajatugi.sinuNimi.local`.
- Paigaldasin Apache ja PHP Ansible abil.
- Paigaldasin MariaDB ja phpMyAdmin Ansible abil.
- Lõin MariaDB andmebaasi `kasutajatugi`.
- Lõin tabelid `tickets` ja `admins`.
- Lõin admin kasutaja parooliräsi abil.
- Lõin PHP veebirakenduse IT-probleemide sisestamiseks.
- Lõin adminiliidese pöördumiste haldamiseks.
- Seadistasin Apache VirtualHosti.
- Keelasin kataloogide sirvimise reaga `Options -Indexes`.
- Seadistasin self-signed SSL sertifikaadi.
- Seadistasin UFW tulemüüri.
- Lõin varundusskripti ja lisasin selle crontabi.
- Testisin vormi, andmebaasi, adminiliidest, HTTPS-i ja varukoopiaid.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `systemctl status apache2` | Apache töötab |
| `php -v` | PHP on paigaldatud |
| `systemctl status mariadb` | MariaDB töötab |
| `SHOW DATABASES;` | Andmebaas `kasutajatugi` olemas |
| `SHOW TABLES;` | Tabelid `tickets` ja `admins` olemas |
| `curl -k -I https://kasutajatugi.sinuNimi.local` | HTTPS vastab |
| Veebivorm | Pöördumine salvestub andmebaasi |
| Adminiliides | Admin näeb ja haldab pöördumisi |
| `ufw status numbered` | Lubatud ainult vajalikud pordid |
| `backup-kasutajatugi.sh` | Varukoopia tekib |
| `crontab -l` | Automaatne backup töö olemas |

### Kokkuvõte

Linux pilet 4 tulemusena valmis PHP ja MariaDB põhine kasutajatoe veebirakendus. Töötajad saavad sisestada IT-probleeme ning administraator saab pöördumisi vaadata, lisada, muuta ja kustutada. Veebileht töötab FQDN nimega HTTPS kaudu, kataloogide sirvimine on keelatud ning andmetest tehakse cron abil automaatseid varukoopiaid.
```

---

# 23. Troubleshooting

## Veebileht ei avane

Kontrolli Apache staatust:

```bash
systemctl status apache2
```

Kontrolli konfiguratsiooni:

```bash
sudo apache2ctl configtest
```

Kontrolli porti:

```bash
sudo ss -tulpen | grep :80
sudo ss -tulpen | grep :443
```

Kui port ei kuula, Apache ei tööta või VirtualHost pole lubatud.

---

## PHP kood kuvatakse brauseris tekstina

Kontrolli PHP paigaldust:

```bash
php -v
```

Paigalda Apache PHP moodul:

```bash
sudo apt install libapache2-mod-php php -y
```

Taaskäivita Apache:

```bash
sudo systemctl restart apache2
```

---

## Vorm ei salvesta andmebaasi

Kontrolli `db.php` faili:

```bash
sudo nano /var/www/kasutajatugi/db.php
```

Kontrolli andmebaasi kasutajat:

```bash
mysql -u kasutajatugi_user -p kasutajatugi
```

Kui tuleb `Access denied`, muuda MariaDB-s kasutaja parool ja kontrolli õiguseid:

```sql
GRANT ALL PRIVILEGES ON kasutajatugi.* TO 'kasutajatugi_user'@'localhost';
FLUSH PRIVILEGES;
```

---

## Admin login ei tööta

Kontrolli, kas admin on olemas:

```bash
sudo mariadb -e "USE kasutajatugi; SELECT id, username FROM admins;"
```

Kui admin puudub, loo uus hash:

```bash
php -r "echo password_hash('AdminParool123!', PASSWORD_DEFAULT), PHP_EOL;"
```

Lisa või uuenda admin:

```sql
UPDATE admins SET password_hash='HASH' WHERE username='admin';
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

Kontrolli sertifikaadifaile:

```bash
ls -lah /etc/ssl/localcerts/
```

Kontrolli Apache konfiguratsiooni:

```bash
sudo apache2ctl configtest
```

---

## phpMyAdmin annab 404

Kontrolli alias konfiguratsiooni:

```bash
ls /etc/apache2/conf-enabled/ | grep phpmyadmin
```

Kui puudub:

```bash
sudo a2enconf phpmyadmin.conf
sudo systemctl reload apache2
```

---

## Backup skript ei tööta

Käivita käsitsi:

```bash
sudo /usr/local/sbin/backup-kasutajatugi.sh
```

Kontrolli logi:

```bash
sudo tail -n 50 /var/log/kasutajatugi-backup.log
```

Kontrolli MySQL dumpi:

```bash
mysqldump -u kasutajatugi_user -p kasutajatugi > /tmp/test.sql
```

Kui see ei tööta, on probleem andmebaasi kasutaja või parooliga.

---

## Cron ei käivita backupit

Kontrolli crontabi:

```bash
sudo crontab -l
```

Kontrolli cron teenust:

```bash
systemctl status cron
```

Kui cron ei tööta:

```bash
sudo systemctl enable cron
sudo systemctl restart cron
```

---

# 24. Kõige lühem spikker

```text
1. Kontrolli sihtserveri IP, DNS ja hostname
2. Lisa DNS kirje kasutajatugi.sinuNimi.local
3. Kontrolli Ansible ühendust: ansible all -m ping
4. Paigalda Apache + PHP Ansible playbookiga
5. Paigalda MariaDB + phpMyAdmin Ansible playbookiga
6. Kui Ansible ei õnnestu, paigalda käsitsi ja dokumenteeri
7. Loo MariaDB andmebaas kasutajatugi
8. Loo tickets ja admins tabelid
9. Loo admin parooli hash
10. Loo PHP failid: db.php, index.php, kkk.php, kontakt.php
11. Loo admin.php, admin_panel.php ja logout.php
12. Loo style.css
13. Seadista Apache VirtualHost
14. Keela kataloogide sirvimine Options -Indexes
15. Testi HTTP
16. Loo SSL sertifikaat
17. Seadista HTTPS VirtualHost
18. Ava UFW-s 22, 80, 443
19. Testi vormi ja adminiliidest
20. Loo backup skript
21. Lisa crontab
22. Kontrolli varukoopiaid
23. Dokumenteeri
```
