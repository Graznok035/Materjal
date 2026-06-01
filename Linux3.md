# Linux pilet 3 ülesannete lahenduskäik

See juhend keskendub ainult **Linux pilet 3 eriosa ülesannetele**.

Siin ei ole lahti kirjutatud üldist Linuxi baasosa:

- Ansible paigaldus UbuntuServerisse
- Ansible inventory loomine
- `hkhk` kasutaja loomine
- sudo gruppi lisamine
- SSH võtmega ligipääs
- SSH parooliga sisselogimise keelamine

Need kuuluvad Linuxi üldise baasosa alla.

---

## Pilet 3 põhiteemad

| Teema | Mida tuleb teha |
|---|---|
| Monitooringulahendus | Valida ja paigaldada monitooring UbuntuServerisse |
| DNS kirjed | Luua monitooringulahendusele DNS kirjed |
| Hostide lisamine | Lisada monitooringusse nõutud Linuxi, Windowsi ja vajadusel võrguseadmed |
| Monitooringu protokollid | Kasutada agenti, SNMP-d või muud sobivat seireviisi |
| Tulemüür | Lubada ainult halduseks ja monitooringuks vajalikud pordid |
| DebianPilet3 failserver | Uurida, mis ei tööta, ja failserver korda teha |
| Partitsiooni suurendamine | Suurendada DebianPilet3 kasutatava kõvaketta partitsioon maksimaalsele mahule |
| Dokumentatsioon | Dokumenteerida kogu protsess |

---

# 1. Soovitatav lahendus: Zabbix

Selle juhendi näites kasutan monitooringuks **Zabbixit**, sest see sobib hästi Linuxi, Windowsi ja võrguseadmete jälgimiseks.

Zabbixiga saab jälgida:

| Seadme tüüp | Kuidas jälgida |
|---|---|
| Linux serverid | Zabbix agent |
| Windows serverid | Zabbix agent |
| Võrguseadmed | SNMP |
| Teenused | TCP portide kontroll, HTTP kontroll |
| Ressursid | CPU, RAM, ketas, võrk |

---

# 2. Algkontroll UbuntuServeris

## Eesmärk

UbuntuServer on selles piletis monitooringuserver.

Enne paigaldust kontrollin:

- hostname
- IP-aadress
- DNS
- internetiühendus
- vaba kettaruum
- kas vajalikud pordid on vabad

---

## 2.1 Kontrolli hostname’i

```bash
hostname
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse UbuntuServeri nimi | Kui nimi on vale või segane, dokumenteeri see ja muuda hiljem käsuga `sudo hostnamectl set-hostname monitooring` |

Kontrolli FQDN-i:

```bash
hostname -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse täielik nimi või vähemalt hostname | Kui tuleb error, siis `/etc/hosts` või DNS pole korrektselt seadistatud |

---

## 2.2 Soovi korral muuda monitooringuserveri hostname

```bash
sudo hostnamectl set-hostname monitooring
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk lõpeb veata | Kui tuleb `permission denied`, kasuta `sudo`. Kui hostname ei muutu kohe, logi välja/sisse või tee restart |

Kontroll:

```bash
hostname
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse `monitooring` | Kui vana nimi jäi alles, tee `sudo reboot` |

---

## 2.3 Kontrolli IP-aadressi

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
| Näed `default via ...` rida | Kui default route puudub, ei saa server tõenäoliselt internetti ega teistesse võrkudesse |

---

## 2.4 Kontrolli internetti ja DNS-i

```bash
ping -c 4 8.8.8.8
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ping töötab | Kui ei tööta, on probleem gateway, võrgu või tulemüüriga |

```bash
ping -c 4 google.com
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Nimi lahendub ja ping töötab | Kui IP ping töötab, aga nimi mitte, on DNS probleem |

Kontrolli DNS servereid:

```bash
resolvectl status
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed DNS serveri aadressi | Kui DNS puudub või on vale, paranda võrgu/DNS seadistus |

---

## 2.5 Kontrolli kettaruumi

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Root partitsioonil on piisavalt vaba ruumi | Kui ruum on otsas, puhasta pakette käsuga `sudo apt autoremove --purge -y` või suurenda ketast |

---

# 3. Lisa DNS kirje monitooringule

## Eesmärk

Monitooring peab olema kättesaadav FQDN nimega, näiteks:

```text
monitooring.sinuNimi.local
```

Kui DNS on Windows Serveris, lisa DNS Manageris A-kirje:

| Nimi | IP |
|---|---|
| `monitooring` | UbuntuServeri IP |

Näide:

```text
monitooring.sinuNimi.local -> UbuntuServeri IP
```

---

## 3.1 Kontroll Linuxis

```bash
getent hosts monitooring.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse UbuntuServeri IP | Kui vastust pole, puudub DNS kirje või klient kasutab valet DNS serverit |

```bash
ping -c 4 monitooring.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ping jõuab monitooringuserverini | Kui nimi ei lahendu, kontrolli DNS-i. Kui nimi lahendub, aga ping ei tööta, kontrolli võrku/tulemüüri |

---

## 3.2 Ajutine `/etc/hosts` lahendus

Kui DNS serverit pole kohe käepärast, saab testimiseks lisada kirje käsitsi.

```bash
sudo nano /etc/hosts
```

Lisa:

```text
UBUNTU_IP monitooring.sinuNimi.local monitooring
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Nimi hakkab selles masinas lahenduma | Kui ei lahendu, kontrolli kirjavigu ja IP-aadressi |

Kontroll:

```bash
getent hosts monitooring.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse UbuntuServeri IP | Kui väljund puudub, on `/etc/hosts` kirje vale |

---

# 4. Paigalda Zabbix UbuntuServerisse Dockeriga

## Miks Docker?

Dockeriga on Zabbixi paigaldus eksami jaoks lihtsam, sest ei pea eraldi käsitsi seadistama andmebaasi, PHP-d ja Apache/Nginx-i.

Kasutame kolme konteinerit:

| Konteiner | Roll |
|---|---|
| PostgreSQL | Zabbixi andmebaas |
| Zabbix server | Monitooringu põhisüsteem |
| Zabbix web | Veebiliides |

---

## 4.1 Paigalda Docker

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui tuleb DNS/repo error, kontrolli internetti ja DNS-i |

```bash
sudo apt install docker.io docker-compose-plugin -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Docker ja Docker Compose plugin paigaldatakse | Kui pakette ei leita, kontrolli Ubuntu repo seadistust |

Luba Docker käivitumisel:

```bash
sudo systemctl enable docker
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Docker lubatakse käivitumisel | Kui teenust ei leita, ei õnnestunud Dockeri paigaldus |

Käivita Docker:

```bash
sudo systemctl start docker
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Docker käivitub | Kui `failed`, vaata `sudo journalctl -xeu docker` |

Kontrolli Dockerit:

```bash
docker --version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse Dockeri versioon | Kui `command not found`, Docker ei paigaldunud |

```bash
sudo docker ps
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse konteinerite tabel, isegi kui see on tühi | Kui tuleb daemon error, Docker ei tööta |

---

## 4.2 Loo Zabbixi kaust

```bash
sudo mkdir -p /opt/zabbix
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse | Kui tuleb `permission denied`, kasuta `sudo` |

```bash
cd /opt/zabbix
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Oled kaustas `/opt/zabbix` | Kui kaust puudub, loo see eelmise käsuga |

---

## 4.3 Loo Docker Compose fail

```bash
sudo nano docker-compose.yml
```

Lisa sisu:

```yaml
services:
  postgres:
    image: postgres:15
    container_name: zabbix-postgres
    restart: always
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: ZabbixDBpass123!
      POSTGRES_DB: zabbix
    volumes:
      - ./postgres-data:/var/lib/postgresql/data

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:alpine-latest
    container_name: zabbix-server
    restart: always
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: ZabbixDBpass123!
      POSTGRES_DB: zabbix
    depends_on:
      - postgres
    ports:
      - "10051:10051"

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:alpine-latest
    container_name: zabbix-web
    restart: always
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: ZabbixDBpass123!
      POSTGRES_DB: zabbix
      ZBX_SERVER_HOST: zabbix-server
      PHP_TZ: Europe/Tallinn
    depends_on:
      - postgres
      - zabbix-server
    ports:
      - "8080:8080"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei saa salvestada, kontrolli, et oled `/opt/zabbix` kaustas ja kasutasid `sudo nano` |

---

## 4.4 Käivita Zabbix

```bash
sudo docker compose up -d
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Docker tõmbab image’id ja käivitab konteinerid | Kui image download ei õnnestu, kontrolli internetti ja DNS-i |

Kontrolli konteinereid:

```bash
sudo docker ps
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `zabbix-postgres`, `zabbix-server`, `zabbix-web` konteinerid staatusega `Up` | Kui mõni konteiner on `Exited`, vaata selle logi käsuga `sudo docker logs KONTEINERI_NIMI` |

Kontrolli kõiki konteinereid:

```bash
sudo docker ps -a
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kõik Zabbixi konteinerid on olemas | Kui mõni puudub, kontrolli `docker-compose.yml` süntaksit |

---

## 4.5 Kontrolli Zabbixi veebiliidest

```bash
curl -I http://localhost:8080
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| HTTP vastus, näiteks `200 OK`, `301` või `302` | Kui `connection refused`, Zabbix web ei tööta või port 8080 pole avatud |

Kontroll väljastpoolt:

```text
http://monitooring.sinuNimi.local:8080
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Brauseris avaneb Zabbixi sisselogimisleht | Kui nimi ei lahendu, kontrolli DNS-i. Kui port ei avane, kontrolli UFW-d |

Vaikimisi sisselogimine:

```text
Username: Admin
Password: zabbix
```

Pärast sisselogimist muuda parool, kui aega on.

---

# 5. Seadista monitooringuserveri tulemüür

## Eesmärk

Lubatud peavad olema ainult halduseks ja monitooringuks vajalikud pordid.

Zabbixi Docker lahenduse puhul:

| Port | Teenus | Milleks |
|---|---|---|
| 22/tcp | SSH | Serveri haldus |
| 8080/tcp | Zabbix web | Veebiliides |
| 10051/tcp | Zabbix server | Agentid saadavad andmeid serverisse |

Kui kasutad SNMP-d võrguseadmete jälgimiseks, tuleb sihtseadmetes lubada SNMP port 161/udp, kuid Zabbix serveris ei pea tavaliselt 161/udp inbound olema avatud, sest Zabbix teeb päringu välja.

---

## 5.1 Paigalda UFW

```bash
sudo apt install ufw -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW paigaldatakse või on juba olemas | Kui repo error, kontrolli `sudo apt update` ja võrku |

---

## 5.2 Määra vaikereeglid

```bash
sudo ufw default deny incoming
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Sissetulev liiklus keelatakse vaikimisi | Kui `ufw` käsku pole, paigalda UFW |

```bash
sudo ufw default allow outgoing
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Väljaminev liiklus lubatakse | Kui tuleb error, kontrolli UFW paigaldust |

---

## 5.3 Luba vajalikud pordid

```bash
sudo ufw allow 22/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SSH lubatakse | Kui kasutad teist SSH porti, luba õige port enne UFW sisselülitamist |

```bash
sudo ufw allow 8080/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Zabbixi veebiliides lubatakse | Kui veeb töötab ainult lokaalselt, kontrolli Docker port mappingut |

```bash
sudo ufw allow 10051/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Zabbix serveri port lubatakse agentide jaoks | Kui agentid ei saa ühendust, kontrolli seda porti |

---

## 5.4 Lülita UFW sisse

```bash
sudo ufw enable
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW aktiveerub | Kui oled SSH-ga sees ja 22/tcp pole lubatud, võid ühenduse kaotada |

Kontrolli tulemüüri:

```bash
sudo ufw status numbered
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed lubatud porte 22/tcp, 8080/tcp ja 10051/tcp | Kui mõni puudub, lisa see `sudo ufw allow PORT/tcp` käsuga |

---

# 6. Paigalda Zabbix agent Linuxi masinatesse

## Eesmärk

AlmaServer ja DebianServer tuleb monitooringusse lisada.  
Selleks paigaldame neisse Zabbix agenti.

Tee allolevad sammud nii AlmaServeris kui DebianServeris.

---

## 6.1 Kontrolli kliendi hostname’i

```bash
hostname
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse masina nimi, näiteks `almaserver` või `debianserver` | Kui nimi on vale, dokumenteeri praegune nimi või muuda `hostnamectl` abil |

Kontrolli IP-d:

```bash
ip a
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Masinal on IP-aadress | Kui IP puudub, kontrolli võrku või DHCP-d |

Kontrolli, kas monitooringuserveri nimi lahendub:

```bash
getent hosts monitooring.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse Zabbix serveri IP | Kui ei lahendu, lisa DNS kirje või `/etc/hosts` kirje |

---

## 6.2 Paigalda agent Debian/Ubuntu põhises serveris

DebianServeris:

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui DNS/repo error, kontrolli võrku ja DNS-i |

```bash
sudo apt install zabbix-agent -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Zabbix agent paigaldatakse | Kui paketti ei leita, võib vaja olla Zabbixi repo lisamist või kasutada `zabbix-agent2`, kui see on saadaval |

Kui `zabbix-agent` ei leidu, proovi:

```bash
sudo apt search zabbix-agent
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed saadaolevaid Zabbixi agenti pakette | Kui tulemusi pole, repo ei sisalda Zabbixi pakette |

---

## 6.3 Paigalda agent AlmaServeris

AlmaServeris:

```bash
sudo dnf install zabbix-agent -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Zabbix agent paigaldatakse | Kui paketti ei leita, kontrolli repo seadistust või kasuta EPEL/Zabbixi repo lahendust |

Kui paketti ei leita:

```bash
sudo dnf search zabbix
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed saadaolevaid Zabbixi pakette | Kui tulemusi pole, repo ei sisalda Zabbixit |

---

## 6.4 Seadista Zabbix agent

Ava agenti konfiguratsioon.

Debian/Ubuntu:

```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

AlmaServeris on asukoht tavaliselt sama:

```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

Muuda või kontrolli read:

```text
Server=MONITOORINGU_SERVERI_IP
ServerActive=MONITOORINGU_SERVERI_IP
Hostname=KLIENDI_HOSTNAME
```

Näide AlmaServeri puhul:

```text
Server=10.0.0.10
ServerActive=10.0.0.10
Hostname=almaserver
```

Näide DebianServeri puhul:

```text
Server=10.0.0.10
ServerActive=10.0.0.10
Hostname=debianserver
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub ja sisaldab õiget Zabbix serveri IP-d ning hostname’i | Kui hostname Zabbixis ei klapi agendi hostname’iga, host võib jääda Zabbixis kättesaamatuks |

---

## 6.5 Käivita agent

```bash
sudo systemctl enable zabbix-agent
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Agent lubatakse käivitumisel | Kui teenust ei leita, agent ei paigaldunud |

```bash
sudo systemctl restart zabbix-agent
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Agent käivitub | Kui `failed`, vaata `sudo journalctl -xeu zabbix-agent` |

Kontroll:

```bash
systemctl status zabbix-agent
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui `inactive` või `failed`, kontrolli konfiguratsiooni ja logisid |

---

## 6.6 Luba agenti port kliendi tulemüüris

Zabbix agent kuulab tavaliselt pordil:

```text
10050/tcp
```

Debian/Ubuntu UFW korral:

```bash
sudo ufw allow 10050/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Zabbix agenti port lubatakse | Kui UFW pole kasutusel, kontrolli muud tulemüüri |

Kontroll:

```bash
sudo ufw status numbered
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed 10050/tcp lubatud | Kui ei näe, lisa reegel uuesti |

AlmaServeris firewalld korral:

```bash
sudo firewall-cmd --add-port=10050/tcp --permanent
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse `success` | Kui `firewall-cmd` puudub, firewalld ei tööta või pole paigaldatud |

```bash
sudo firewall-cmd --reload
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse `success` | Kui reload ebaõnnestub, kontrolli firewalld staatust |

Kontroll:

```bash
sudo firewall-cmd --list-ports
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `10050/tcp` | Kui puudub, lisa port uuesti |

---

## 6.7 Testi, kas agent kuulab

Kliendimasinas:

```bash
sudo ss -tulpen | grep :10050
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `zabbix_agentd` kuulamas pordil 10050 | Kui väljund puudub, agent ei tööta või ei kuula porti |

Monitooringuserverist:

```bash
nc -zv KLIENDI_IP 10050
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ühendus õnnestub | Kui `nc` puudub, paigalda `sudo apt install netcat-openbsd -y`. Kui ühendus ei õnnestu, kontrolli kliendi tulemüüri ja agenti |

Kui `nc` puudub:

```bash
sudo apt install netcat-openbsd -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Netcat paigaldatakse | Kui paketti ei leita, kontrolli apt repo |

---

# 7. Lisa Linux hostid Zabbixi veebiliideses

## Eesmärk

AlmaServer ja DebianServer tuleb lisada Zabbixi monitooringusse.

---

## Sammud veebiliideses

Ava brauseris:

```text
http://monitooring.sinuNimi.local:8080
```

Logi sisse:

```text
Admin / zabbix
```

Mine:

```text
Data collection → Hosts → Create host
```

Lisa host:

| Väli | Näide |
|---|---|
| Host name | `almaserver` |
| Visible name | `AlmaServer` |
| Groups | `Linux servers` |
| Interfaces | Agent |
| IP address | AlmaServeri IP |
| Port | `10050` |

Lisa template:

```text
Linux by Zabbix agent
```

Sama tee DebianServerile.

---

## Kontroll Zabbixis

| Kontroll | Oodatav tulemus |
|---|---|
| Availability | ZBX muutub roheliseks |
| Latest data | Hostilt tulevad CPU, RAM, ketta ja võrgu andmed |
| Problems | Kui vigu pole, host on korras |

Kui ZBX jääb halliks või punaseks:

| Probleem | Lahendus |
|---|---|
| Agent ei tööta | Kontrolli `systemctl status zabbix-agent` |
| Tulemüür blokeerib | Kontrolli 10050/tcp kliendis |
| Vale IP | Paranda hosti interface Zabbixis |
| Vale hostname | Kontrolli `Hostname=` agenti konfiguratsioonis |
| DNS ei tööta | Kasuta IP-aadressi või paranda DNS |

---

# 8. Windows serverite lisamine monitooringusse

## Eesmärk

Kui ülesandes on Windowsi serverid, tuleb ka need monitooringusse lisada.

Lihtsaim variant on paigaldada Windowsi Zabbix Agent.

---

## Windowsis tehtavad tegevused

1. Laadi alla Zabbix Agent Windowsile.
2. Paigalda agent.
3. Seadista Zabbix serveri IP.
4. Kontrolli, et Windows Firewall lubab Zabbix agenti.
5. Lisa Windows host Zabbixi veebiliideses.

---

## Windowsi agenti olulisemad seaded

| Seade | Väärtus |
|---|---|
| Server | Zabbix serveri IP |
| ServerActive | Zabbix serveri IP |
| Hostname | Windows serveri nimi |
| Port | 10050/tcp |

Zabbixis vali template:

```text
Windows by Zabbix agent
```

---

## Kontroll Windowsi poolel

PowerShellis:

```powershell
Get-Service *zabbix*
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Zabbix Agent teenus on Running | Kui teenust pole, agent pole paigaldatud. Kui Stopped, käivita teenus |

Kontrolli porti:

```powershell
netstat -ano | findstr 10050
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Windows kuulab pordil 10050 | Kui ei kuula, kontrolli agenti teenust ja konfiguratsiooni |

---

# 9. Võrguseadmete lisamine monitooringusse SNMP kaudu

## Eesmärk

Kui piletis on nõutud võrguseadmete jälgimine, sobib selleks SNMP.

SNMP port:

```text
161/udp
```

---

## Cisco seadme näidiskonfiguratsioon

Cisco switchis või ruuteris:

```cisco
enable
configure terminal
snmp-server community public RO
end
write memory
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SNMP community luuakse ja konfiguratsioon salvestub | Kui käsk ei tööta, kontrolli seadme IOS versiooni või õiguseid |

Turvalisem variant lubada ainult monitooringuserveri IP-lt:

```cisco
enable
configure terminal
access-list 10 permit MONITOORINGU_SERVERI_IP
snmp-server community public RO 10
end
write memory
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SNMP päringud lubatakse ainult monitooringuserverilt | Kui ACL vale, Zabbix ei saa seadet lugeda |

---

## Testi SNMP-d UbuntuServerist

Paigalda SNMP tööriistad:

```bash
sudo apt install snmp -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SNMP tööriistad paigaldatakse | Kui paketti ei leita, kontrolli apt repo |

Testi SNMP päringut:

```bash
snmpwalk -v2c -c public VORGUSEADME_IP sysName
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse seadme nimi | Kui timeout, kontrolli SNMP seadistust, community stringi, ACL-i ja võrguteed |

---

## Lisa võrguseade Zabbixis

Zabbixi veebiliideses:

```text
Data collection → Hosts → Create host
```

| Väli | Näide |
|---|---|
| Host name | `SW1` |
| Groups | `Network devices` |
| Interface | SNMP |
| IP address | switchi või ruuteri IP |
| Port | `161` |
| SNMP version | v2c |
| Community | `public` |

Lisa template:

```text
Cisco IOS by SNMP
```

või üldisem:

```text
Generic SNMP
```

---

# 10. DebianPilet3 failserveri algkontroll

## Eesmärk

Ülesandes on kirjas, et DebianPilet3 failserver, mis on mõeldud serverite varukoopiate tegemiseks, ei tööta enam.  
Kõigepealt tuleb aru saada, mis täpselt ei tööta.

Kontrollin:

- kas serveril on IP
- kas kettad on olemas
- kas backup kaust on mountitud
- kas Samba või NFS töötab
- kas jagatud kaustad on olemas
- kas õigused on korras
- kas tulemüür lubab ligipääsu

---

## 10.1 Kontrolli serverit ja võrku

```bash
hostname
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse `DebianPilet3` või muu loogiline nimi | Kui nimi on vale, dokumenteeri ja vajadusel muuda |

```bash
ip a
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Serveril on IP-aadress | Kui IP puudub, kontrolli võrguadapterit või DHCP/staatilist seadistust |

```bash
ip route
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed default route’i | Kui puudub, võib server olla teistest võrkudest kättesaamatu |

Testi ühendust:

```bash
ping -c 4 MONITOORINGU_SERVERI_IP
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ping töötab | Kui ei tööta, kontrolli võrguseadeid ja tulemüüri |

---

## 10.2 Kontrolli kettaid ja mountpoint’e

```bash
lsblk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed süsteemiketast ja backup/failserveri ketast | Kui lisaketast ei näe, kontrolli VM-is ketta ühendust |

```bash
lsblk -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed failisüsteeme ja mountpoint’e | Kui backup ketas pole mountitud, tuleb see ühendada |

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed failserveri kasutatavat mountpointi, näiteks `/srv/backup` või `/backup` | Kui mountpoint puudub, kontrolli `/etc/fstab` faili |

Kontrolli fstab faili:

```bash
cat /etc/fstab
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failis on backup ketta mountimise rida | Kui rida puudub või UUID vale, tuleb see parandada |

Testi fstab-i:

```bash
sudo mount -a
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna viga | Kui tuleb UUID või mount error, paranda `/etc/fstab` rida |

---

# 11. Uuri, kas failserver kasutab Sambat või NFS-i

## Eesmärk

Failserver võib töötada kas:

| Teenus | Milleks |
|---|---|
| Samba | Windowsi ja Linuxi klientidele SMB jagatud kaustad |
| NFS | Linuxi serveritele jagatud kaustad |

---

## 11.1 Kontrolli Samba teenuseid

```bash
systemctl status smbd
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kui Samba on kasutusel, peaks olema `active (running)` | Kui teenust pole, võib failserver kasutada NFS-i või Samba pole paigaldatud |

```bash
systemctl status nmbd
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Samba NetBIOS teenus võib olla aktiivne | Kui ei tööta, ei pruugi see olla kriitiline, kui kasutatakse ainult IP/FQDN ligipääsu |

Kontrolli Samba konfiguratsiooni:

```bash
testparm
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näitab Samba konfiguratsiooni ja lõpus pole kriitilisi vigu | Kui näitab errorit, paranda `/etc/samba/smb.conf` |

---

## 11.2 Kontrolli NFS teenust

```bash
systemctl status nfs-server
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kui NFS on kasutusel, peaks olema `active (running)` | Kui teenust pole, võib failserver kasutada Sambat |

Kontrolli exporte:

```bash
sudo exportfs -v
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed jagatud kaustu ja lubatud võrke | Kui midagi ei kuvata, pole NFS jagamised seadistatud |

Kontrolli faili:

```bash
cat /etc/exports
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed NFS jagamise ridu | Kui fail on tühi, NFS jagamist pole seadistatud |

---

# 12. Paranda Samba failserver

Kasuta seda osa, kui failserver peab töötama SMB/Samba kaudu.

---

## 12.1 Paigalda Samba

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui repo/DNS error, kontrolli võrku |

```bash
sudo apt install samba -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Samba paigaldatakse või on juba olemas | Kui paketti ei leita, kontrolli apt allikaid |

---

## 12.2 Loo backup kaust

Näide:

```bash
sudo mkdir -p /srv/backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse | Kui permission denied, kasuta `sudo` |

Anna õigused:

```bash
sudo chown -R nobody:nogroup /srv/backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kausta omanik muudetakse | Kui kasutajat/gruppi pole, kasuta eraldi backup kasutajat |

```bash
sudo chmod -R 0775 /srv/backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust saab lugemis/kirjutamisõigused omanikule ja grupile | Kui õigused ei muutu, kontrolli, kas failisüsteem on read-only |

---

## 12.3 Lisa Samba share

Tee smb.conf varukoopia:

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Varukoopia luuakse | Kui fail puudub, Samba pole paigaldatud |

Ava konfiguratsioon:

```bash
sudo nano /etc/samba/smb.conf
```

Lisa faili lõppu:

```ini
[backup]
   path = /srv/backup
   browseable = yes
   writable = yes
   guest ok = yes
   read only = no
   force user = nobody
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei saa salvestada, kontrolli `sudo` kasutamist |

Kontrolli Samba konfiguratsiooni:

```bash
testparm
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Konfiguratsioon on OK ja näed `[backup]` share’i | Kui on süntaksiviga, paranda viidatud rida `smb.conf` failis |

Taaskäivita Samba:

```bash
sudo systemctl restart smbd
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Samba käivitub | Kui failed, vaata `sudo journalctl -xeu smbd` |

Luba käivitumisel:

```bash
sudo systemctl enable smbd
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Samba lubatakse käivitumisel | Kui teenust pole, kontrolli Samba paigaldust |

---

## 12.4 Testi Samba share’i DebianPilet3 serveris

```bash
smbclient -L localhost -N
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `backup` share’i | Kui `smbclient` puudub, paigalda `sudo apt install smbclient -y` |

Kui `smbclient` puudub:

```bash
sudo apt install smbclient -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| smbclient paigaldatakse | Kui paketti ei leita, kontrolli apt repo |

Testi kirjutamist:

```bash
smbclient //localhost/backup -N -c "put /etc/hostname test-hostname.txt"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail kopeeritakse share’i | Kui access denied, kontrolli kausta õiguseid ja Samba konfiguratsiooni |

Kontrolli faili:

```bash
ls -lah /srv/backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `test-hostname.txt` faili | Kui faili pole, kirjutamine ei õnnestunud |

---

# 13. Paranda NFS failserver

Kasuta seda osa, kui failserver peab töötama NFS kaudu.

---

## 13.1 Paigalda NFS server

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui repo/DNS error, kontrolli võrku |

```bash
sudo apt install nfs-kernel-server -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| NFS server paigaldatakse või on juba olemas | Kui paketti ei leita, kontrolli apt allikaid |

---

## 13.2 Loo NFS jagatav kaust

```bash
sudo mkdir -p /srv/backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse | Kui permission denied, kasuta sudo |

```bash
sudo chown -R nobody:nogroup /srv/backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Omanik määratakse | Kui kasutajat pole, kontrolli süsteemi kasutajaid |

```bash
sudo chmod -R 0775 /srv/backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Õigused muudetakse | Kui failisüsteem on read-only, kontrolli mounti |

---

## 13.3 Lisa NFS export

```bash
sudo nano /etc/exports
```

Lisa näiteks kogu sisevõrgule:

```text
/srv/backup 10.0.0.0/24(rw,sync,no_subtree_check)
```

Asenda `10.0.0.0/24` oma tegeliku võrguga.

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Export rida salvestub | Kui ei saa salvestada, kasuta `sudo nano` |

Rakenda exportid:

```bash
sudo exportfs -ra
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna viga | Kui tuleb süntaksiviga, paranda `/etc/exports` |

Kontrolli:

```bash
sudo exportfs -v
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `/srv/backup` jagamist | Kui ei näe, export rida pole korrektne |

Käivita NFS:

```bash
sudo systemctl restart nfs-server
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| NFS käivitub | Kui failed, vaata `sudo journalctl -xeu nfs-server` |

Luba käivitumisel:

```bash
sudo systemctl enable nfs-server
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| NFS lubatakse käivitumisel | Kui teenust ei leita, paigaldus ei õnnestunud |

---

## 13.4 Testi NFS-i kliendist

Kliendimasinas paigalda NFS klient.

Debian/Ubuntu:

```bash
sudo apt install nfs-common -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| NFS klient paigaldatakse | Kui paketti ei leita, kontrolli apt repo |

AlmaServer:

```bash
sudo dnf install nfs-utils -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| NFS tööriistad paigaldatakse | Kui paketti ei leita, kontrolli repo seadistust |

Loo mountpoint:

```bash
sudo mkdir -p /mnt/backup-test
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse | Kui permission denied, kasuta sudo |

Mounti NFS share:

```bash
sudo mount DEBIANPILET3_IP:/srv/backup /mnt/backup-test
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Share mountitakse | Kui timeout või permission denied, kontrolli NFS serverit, tulemüüri ja `/etc/exports` võrku |

Testi kirjutamist:

```bash
sudo touch /mnt/backup-test/testfail.txt
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail luuakse | Kui permission denied, kontrolli õiguseid `/srv/backup` kaustal |

Kontrolli:

```bash
ls -lah /mnt/backup-test
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `testfail.txt` faili | Kui ei näe, kirjutamine ei õnnestunud |

---

# 14. DebianPilet3 tulemüür failserveri jaoks

## Eesmärk

Lubatud peaksid olema ainult vajalikud pordid.

Kui kasutad Sambat:

| Port | Teenus |
|---|---|
| 22/tcp | SSH |
| 445/tcp | SMB |
| 139/tcp | NetBIOS SMB, vajadusel |
| 137/udp | NetBIOS, vajadusel |
| 138/udp | NetBIOS, vajadusel |

Kui kasutad NFS-i:

| Port | Teenus |
|---|---|
| 22/tcp | SSH |
| 2049/tcp | NFS |
| 2049/udp | NFS |

---

## 14.1 UFW Samba puhul

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
| Sissetulev liiklus keelatakse vaikimisi | Kui UFW puudub, paigalda UFW |

```bash
sudo ufw default allow outgoing
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Väljaminev liiklus lubatakse | Kui error, kontrolli UFW-d |

```bash
sudo ufw allow 22/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SSH lubatakse | Kui SSH port on teine, luba õige port |

```bash
sudo ufw allow 445/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SMB põhiport lubatakse | Kui Windows ei saa ligi, kontrolli ka 139/tcp ja NetBIOS porte |

```bash
sudo ufw enable
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW aktiveerub | Kui oled SSH-ga sees, veendu enne, et 22/tcp on lubatud |

Kontroll:

```bash
sudo ufw status numbered
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed 22/tcp ja 445/tcp | Kui port puudub, lisa see uuesti |

---

## 14.2 UFW NFS puhul

```bash
sudo ufw allow 22/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| SSH lubatakse | Kui SSH port on teine, luba õige port |

```bash
sudo ufw allow 2049/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| NFS TCP lubatakse | Kui NFS ei tööta, kontrolli ka UDP-d |

```bash
sudo ufw allow 2049/udp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| NFS UDP lubatakse | Kui kasutad ainult TCP-d, ei pruugi UDP vajalik olla |

```bash
sudo ufw enable
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW aktiveerub | Kui kaotad SSH ühenduse, kasuta Proxmoxi konsooli ja paranda reeglid |

Kontroll:

```bash
sudo ufw status numbered
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed 22/tcp ja 2049/tcp/udp | Kui mõni puudub, lisa vastav reegel |

---

# 15. Suurenda DebianPilet3 failserveri partitsioon maksimaalsele mahule

## Eesmärk

Failserveri kasutatav kõvaketas tuleb suurendada maksimaalsele võimalikule mahule.

Oluline: enne muutmist tee kindlaks, milline ketas ja partitsioon on failserveri andmete jaoks.

---

## 15.1 Kontrolli kettaid

```bash
lsblk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed ketast ja partitsioone, näiteks `/dev/sdb` ja `/dev/sdb1` | Kui lisaketast ei näe, kontrolli Proxmoxis, kas ketas on VM küljes |

```bash
lsblk -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed failisüsteemi tüüpi ja mountpointi | Kui mountpoint puudub, kontrolli `/etc/fstab` ja `df -h` |

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed failserveri kausta mountpointi, näiteks `/srv/backup` | Kui mountpointi pole, ketas pole ühendatud |

Näide:

```text
/dev/sdb1  /srv/backup
```

Sellisel juhul on suurendatav partitsioon:

```text
/dev/sdb1
```

ja ketas:

```text
/dev/sdb
```

---

## 15.2 Paigalda growpart tööriist

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui DNS/repo error, kontrolli võrku |

```bash
sudo apt install cloud-guest-utils -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paigaldatakse `growpart` tööriist | Kui paketti ei leita, saab kasutada `parted` või `fdisk`, aga `growpart` on lihtsam |

Kontrolli:

```bash
which growpart
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse näiteks `/usr/bin/growpart` | Kui väljund puudub, pakett ei paigaldunud |

---

## 15.3 Suurenda partitsioon

Näide, kui ketas on `/dev/sdb` ja partitsioon on `/dev/sdb1`.

```bash
sudo growpart /dev/sdb 1
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Partitsioon suurendatakse ketta lõpuni | Kui ütleb `NOCHANGE`, on partitsioon juba maksimaalne või ketas pole Proxmoxis suurendatud |

Kontrolli:

```bash
lsblk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/dev/sdb1` suurus on suurenenud | Kui ei muutunud, kontrolli, kas valisid õige ketta ja partitsiooni |

---

## 15.4 Suurenda failisüsteem ext4 puhul

Kontrolli failisüsteemi:

```bash
lsblk -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed, kas failisüsteem on `ext4` või `xfs` | Kui failisüsteemi ei näe, kontrolli mounti |

Kui failisüsteem on ext4:

```bash
sudo resize2fs /dev/sdb1
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failisüsteem suurendatakse partitsiooni suuruseni | Kui tuleb error, kontrolli, et partitsioon on õige ja failisüsteem on ext4 |

Kontrolli:

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Mountpointi suurus on suurenenud | Kui suurus ei muutunud, kontrolli, kas kasutasid õiget partitsiooni |

---

## 15.5 Suurenda failisüsteem XFS puhul

Kui failisüsteem on XFS, kasuta mountpointi, mitte partitsiooni.

Näide:

```bash
sudo xfs_growfs /srv/backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| XFS failisüsteem suurendatakse | Kui käsk puudub, paigalda `sudo apt install xfsprogs -y` |

Kui xfs tööriistu pole:

```bash
sudo apt install xfsprogs -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| XFS tööriistad paigaldatakse | Kui paketti ei leita, kontrolli apt repo |

Kontroll:

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Mountpointi suurus on suurenenud | Kui ei suurenenud, kontrolli partitsiooni suurendamist |

---

# 16. Kontrolli failserver pärast suurendamist

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failserveri mountpoint kasutab uut maksimaalset mahtu | Kui maht pole muutunud, jäi kas partitsioon või failisüsteem suurendamata |

```bash
lsblk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Partitsioon kasutab kogu ketta ruumi | Kui partitsioon on väiksem kui ketas, korda `growpart` sammu |

Testi kirjutamist:

```bash
sudo touch /srv/backup/test-write.txt
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail luuakse | Kui permission denied, kontrolli õiguseid |

Kontroll:

```bash
ls -lah /srv/backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `test-write.txt` faili | Kui ei näe, fail ei tekkinud või oled vales kaustas |

---

# 17. Lõputestid

## 17.1 Monitooringuserver

```bash
sudo docker ps
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Zabbixi konteinerid on `Up` | Kui mõni on `Exited`, kontrolli `sudo docker logs KONTEINERI_NIMI` |

```bash
curl -I http://localhost:8080
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Zabbix web vastab | Kui connection refused, kontrolli `zabbix-web` konteinerit |

```bash
sudo ufw status verbose
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Lubatud ainult 22/tcp, 8080/tcp ja 10051/tcp | Kui üleliigseid porte on avatud, eemalda need |

---

## 17.2 Linux agentid

AlmaServeris ja DebianServeris:

```bash
systemctl status zabbix-agent
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Agent on `active (running)` | Kui failed, kontrolli agendi konfiguratsiooni ja logisid |

```bash
sudo ss -tulpen | grep :10050
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Agent kuulab pordil 10050 | Kui ei kuula, agent ei tööta või konfiguratsioon on vale |

---

## 17.3 DebianPilet3 failserver

```bash
systemctl status smbd
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Samba puhul `active (running)` | Kui kasutad NFS-i, kontrolli hoopis `systemctl status nfs-server` |

```bash
systemctl status nfs-server
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| NFS puhul `active (running)` | Kui kasutad Sambat, pole NFS vajalik |

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Backup/failserveri partitsioon on suurendatud | Kui mitte, kontrolli growpart ja resize sammu |

---

# 18. Dokumentatsiooni näidis

```markdown
## Linux pilet 3 dokumentatsioon

### Eesmärk

Eesmärk oli seadistada UbuntuServerisse monitooringulahendus, lisada sinna nõutud virtuaalmasinad ja vajadusel võrguseadmed, seadistada vajalikud DNS kirjed ja tulemüürireeglid ning taastada DebianPilet3 failserveri töö. Lisaks tuli suurendada failserveri kasutatava kõvaketta partitsioon maksimaalsele võimalikule mahule.

### Kasutatud masinad

| Masin | Roll |
|---|---|
| UbuntuServer | Zabbix monitooringuserver |
| AlmaServer | Monitooritav Linux server |
| DebianServer | Monitooritav Linux server |
| Windows Server | Monitooritav Windows server |
| DebianPilet3 | Failserver varukoopiate jaoks |
| Võrguseadmed | SNMP kaudu jälgitavad seadmed |

### Valitud monitooringulahendus

Valisin monitooringulahenduseks Zabbixi, sest see võimaldab jälgida Linuxi ja Windowsi servereid agentide kaudu ning võrguseadmeid SNMP kaudu.

### Tehtud seadistused

- Kontrollisin UbuntuServeri võrguühendust, DNS-i ja kettaruumi.
- Lisasin DNS kirje `monitooring.sinuNimi.local`.
- Paigaldasin Dockeriga Zabbixi monitooringulahenduse.
- Seadistasin UFW tulemüüri, lubades ainult vajalikud pordid.
- Paigaldasin AlmaServerisse ja DebianServerisse Zabbix agendi.
- Lisasin Linuxi hostid Zabbixi veebiliidesesse.
- Lisasin Windows serveri Zabbixi monitooringusse.
- Seadistasin vajadusel võrguseadmetel SNMP.
- Uurisin DebianPilet3 failserveri probleemi.
- Parandasin Samba/NFS jagamise.
- Kontrollisin ja parandasin failserveri õigused.
- Suurendasin failserveri kasutatava partitsiooni maksimaalsele mahule.
- Testisin failserverisse kirjutamist.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `docker ps` | Zabbixi konteinerid töötavad |
| `curl -I http://localhost:8080` | Zabbixi veebiliides vastab |
| `ufw status verbose` | Lubatud ainult vajalikud pordid |
| `systemctl status zabbix-agent` | Linux agentid töötavad |
| Zabbix Latest data | Hostid saadavad andmeid |
| `testparm` | Samba konfiguratsioon korras |
| `exportfs -v` | NFS export olemas, kui kasutati NFS-i |
| `df -h` | Failserveri partitsioon suurendatud |
| `touch /srv/backup/test-write.txt` | Failserverisse saab kirjutada |

### Kokkuvõte

Linux pilet 3 tulemusena töötab UbuntuServeris Zabbix monitooringulahendus. Linuxi ja Windowsi serverid on lisatud monitooringusse ning võrguseadmeid saab vajadusel jälgida SNMP kaudu. Tulemüüris on lubatud ainult halduseks ja monitooringuks vajalikud pordid. DebianPilet3 failserveri töö on taastatud ning failserveri kasutatav partitsioon on suurendatud maksimaalsele võimalikule mahule.
```

---

# 19. Troubleshooting

## Zabbix veeb ei avane

Kontrolli:

```bash
sudo docker ps
```

Kui `zabbix-web` ei tööta:

```bash
sudo docker logs zabbix-web
```

Kui port ei vasta:

```bash
sudo ss -tulpen | grep :8080
```

Kui 8080 ei kuula, kontrolli `docker-compose.yml` port mappingut.

---

## Zabbix agent ei ilmu roheliseks

Kliendis:

```bash
systemctl status zabbix-agent
```

Kui agent ei tööta:

```bash
sudo journalctl -xeu zabbix-agent
```

Kontrolli konfiguratsiooni:

```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

Peavad olema õiged:

```text
Server=ZABBIX_SERVER_IP
ServerActive=ZABBIX_SERVER_IP
Hostname=SAMA_NIMI_MIS_ZABBIXIS
```

---

## Tulemüür blokeerib monitooringu

Monitooringuserveris:

```bash
sudo ufw status numbered
```

Peavad olema lubatud:

```text
22/tcp
8080/tcp
10051/tcp
```

Kliendis:

```bash
sudo ufw status numbered
```

või AlmaServeris:

```bash
sudo firewall-cmd --list-ports
```

Kliendi poolel peab olema lubatud:

```text
10050/tcp
```

---

## DNS nimi ei lahendu

Kontroll:

```bash
getent hosts monitooring.sinuNimi.local
```

Kui vastust pole:

- lisa DNS serverisse A-kirje
- kontrolli, et klient kasutab õiget DNS serverit
- lisa ajutiselt `/etc/hosts` kirje

---

## DebianPilet3 failserver ei tööta

Kontrolli esmalt, kas kasutatakse Sambat või NFS-i.

Samba:

```bash
systemctl status smbd
testparm
smbclient -L localhost -N
```

NFS:

```bash
systemctl status nfs-server
sudo exportfs -v
cat /etc/exports
```

Kui teenus ei tööta, vaata logisid:

```bash
sudo journalctl -xeu smbd
```

või:

```bash
sudo journalctl -xeu nfs-server
```

---

## Failserverisse ei saa kirjutada

Kontrolli mountpointi:

```bash
df -h
```

Kontrolli õiguseid:

```bash
ls -ld /srv/backup
```

Paranda vajadusel:

```bash
sudo chown -R nobody:nogroup /srv/backup
sudo chmod -R 0775 /srv/backup
```

Testi:

```bash
sudo touch /srv/backup/test-write.txt
```

---

## Partitsioon ei suurene

Kontrolli:

```bash
lsblk
```

Kui ketas on suurem kui partitsioon, aga partitsioon ei kasva:

```bash
sudo growpart /dev/sdb 1
```

Kui failisüsteem on ext4:

```bash
sudo resize2fs /dev/sdb1
```

Kui failisüsteem on XFS:

```bash
sudo xfs_growfs /srv/backup
```

Kontroll:

```bash
df -h
```

---

# 20. Kõige lühem spikker

```text
1. Kontrolli UbuntuServeri IP, hostname, DNS ja internet
2. Lisa DNS kirje monitooring.sinuNimi.local
3. Paigalda Docker
4. Loo /opt/zabbix/docker-compose.yml
5. Käivita Zabbix docker compose up -d
6. Ava Zabbix http://monitooring.sinuNimi.local:8080
7. Luba UFW-s 22/tcp, 8080/tcp, 10051/tcp
8. Paigalda AlmaServerisse ja DebianServerisse zabbix-agent
9. Seadista agentides Server, ServerActive ja Hostname
10. Luba kliendi tulemüüris 10050/tcp
11. Lisa hostid Zabbixi veebiliideses
12. Vajadusel seadista võrguseadmetel SNMP
13. Kontrolli DebianPilet3 võrku, kettaid ja mountpointi
14. Uuri, kas failserver kasutab Sambat või NFS-i
15. Paranda Samba või NFS jagamine
16. Testi failserverisse kirjutamist
17. Suurenda failserveri partitsioon growpartiga
18. Suurenda failisüsteem resize2fs või xfs_growfs käsuga
19. Kontrolli df -h ja Zabbix Latest data
20. Dokumenteeri
```
