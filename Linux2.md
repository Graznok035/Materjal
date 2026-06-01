# Linux pilet 2 ülesannete lahenduskäik

See juhend keskendub ainult **Linux pilet 2 eriosa ülesannetele**.

Siin ei ole lahti kirjutatud üldist Linuxi baasosa:

- Ansible paigaldus UbuntuServerisse
- Ansible inventory loomine
- `hkhk` kasutaja loomine
- sudo gruppi lisamine
- SSH võtmega ligipääs
- SSH parooliga sisselogimise keelamine

---

## Pilet 2 põhiteemad

| Teema | Mida tuleb teha |
|---|---|
| Logiserver | Paigaldada UbuntuServerisse logiserveri tarkvara |
| Tulemüür | Lubada ainult vajalikud pordid |
| FQDN | Luua logiserverile nimi `logi.sinuNimi.local` |
| Logide saatmine | AlmaServer ja DebianServer saadavad logid logiserverisse |
| GRUB / alglaadur | Teha UbuntuPilet2 masina alglaadur korda |
| Ketas | Vormindada ühendatud kõvaketas |
| Püsiv ühendamine | Lisada ketas `/etc/fstab` faili |
| Varukoopiad | Teha `/home` kaustast varukoopia ühendatud kettale |
| Dokumentatsioon | Dokumenteerida kogu protsess |

---

# 1. Algkontroll UbuntuServeris

## Eesmärk

Kõigepealt tuleb kontrollida, mis masinas oled, mis IP tal on ja kas võrk/DNS töötab.

---

## 1.1 Kontrolli hostname’i

```bash
hostname
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse serveri nimi, näiteks `ubuntuserver` | Kui nimi on vale, dokumenteeri praegune nimi ja muuda hiljem käsuga `sudo hostnamectl set-hostname logiserver` |

Kontrolli täielikku nime:

```bash
hostname -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse FQDN või vähemalt hostname | Kui tuleb error, siis DNS või `/etc/hosts` pole FQDN jaoks korras |

---

## 1.2 Kontrolli IP-aadressi

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
| Näed `default via ...` rida | Kui default route puudub, ei pruugi masin saada võrku ega internetti |

---

## 1.3 Kontrolli DNS-i ja internetti

```bash
ping -c 4 8.8.8.8
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ping töötab | Kui ei tööta, on võrgutee või gateway probleem |

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

# 2. Muuda logiserveri hostname

## Eesmärk

Logiserveril peab olema loogiline nimi ja FQDN.

Näide:

```text
logi.sinuNimi.local
```

---

## 2.1 Muuda hostname

```bash
sudo hostnamectl set-hostname logi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk lõpeb veata | Kui tuleb permission denied, kasuta `sudo`. Kui käsk puudub, kontrolli süsteemi |

Kontrolli:

```bash
hostname
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse `logi` | Kui vana nimi jäi alles, logi välja/sisse või tee restart |

---

## 2.2 Muuda `/etc/hosts` faili

```bash
sudo nano /etc/hosts
```

Lisa või paranda rida:

```text
127.0.1.1 logi.sinuNimi.local logi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei saa salvestada, kontrolli, et kasutasid `sudo nano` |

Kontrolli FQDN-i:

```bash
hostname -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse `logi.sinuNimi.local` | Kui ei kuvata, kontrolli `/etc/hosts` kirjavigu |

---

# 3. Lisa DNS kirje logiserverile

## Eesmärk

Kõik masinad peaksid leidma logiserveri nimega:

```text
logi.sinuNimi.local
```

Kui DNS on Windows Serveris, lisa DNS Manageris A-kirje:

| Nimi | IP |
|---|---|
| `logi` | UbuntuServeri IP |

Näide:

```text
logi.sinuNimi.local -> UbuntuServeri IP
```

---

## 3.1 Kontrolli nime lahendumist Linuxis

```bash
getent hosts logi.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse logiserveri IP | Kui väljund puudub, DNS kirje puudub või klient kasutab valet DNS serverit |

```bash
ping -c 4 logi.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ping jõuab logiserverini | Kui `Name or service not known`, on DNS probleem. Kui nimi lahendub, aga ping ei tööta, kontrolli võrku/tulemüüri |

---

# 4. Paigalda logiserver UbuntuServerisse

## Eesmärk

Lihtne ja sobiv lahendus on kasutada `rsyslog` teenust.

Ubuntu süsteemis on `rsyslog` sageli juba olemas, aga vajadusel paigaldame selle.

---

## 4.1 Paigalda rsyslog

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui tuleb DNS/repo error, kontrolli võrku ja DNS-i |

```bash
sudo apt install rsyslog -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `rsyslog` paigaldatakse või on juba olemas | Kui paketti ei leita, kontrolli apt allikaid |

Kontrolli teenust:

```bash
systemctl status rsyslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui `inactive`, käivita `sudo systemctl start rsyslog`. Kui `failed`, vaata `sudo journalctl -xeu rsyslog` |

Luba teenus käivitumisel:

```bash
sudo systemctl enable rsyslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Teenus lubatakse käivitumisel | Kui teenust ei leita, kontrolli rsyslog paigaldust |

---

## 4.2 Luba rsyslogil võrgu kaudu logisid vastu võtta

Ava konfiguratsioon:

```bash
sudo nano /etc/rsyslog.conf
```

Otsi ja eemalda kommentaarimärgid järgmistelt ridadelt või lisa need:

```text
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei saa salvestada, kontrolli `sudo` kasutamist |

Kontrolli konfiguratsiooni süntaksit:

```bash
sudo rsyslogd -N1
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Vigu ei kuvata või lõpus on `End of config validation run` | Kui näitab errorit, paranda viidatud fail ja rida |

Taaskäivita rsyslog:

```bash
sudo systemctl restart rsyslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Teenus käivitub uuesti | Kui `failed`, kontrolli `sudo journalctl -xeu rsyslog` |

Kontrolli, kas port 514 kuulab:

```bash
sudo ss -tulpen | grep :514
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `rsyslogd` kuulamas pordil 514 TCP ja/või UDP | Kui väljund puudub, ei ole võrgu vastuvõtt rsyslogis lubatud või teenus ei käivitunud |

---

# 5. Seadista logiserveri tulemüür

## Eesmärk

Logiserveris peavad olema avatud ainult vajalikud pordid.

Tavaliselt vajalik:

| Port | Teenus |
|---|---|
| 22/tcp | SSH |
| 514/tcp | Syslog TCP |
| 514/udp | Syslog UDP |

Kui kasutad ainult TCP-d või ainult UDP-d, dokumenteeri see.

---

## 5.1 Paigalda UFW

```bash
sudo apt install ufw -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW paigaldatakse või on juba olemas | Kui repo error, kontrolli `apt update` ja võrku |

---

## 5.2 Määra vaikereeglid

```bash
sudo ufw default deny incoming
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Vaikimisi keelatakse sissetulev liiklus | Kui `ufw` käsku pole, paigalda UFW |

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
| SSH lubatakse | Kui kasutad teist SSH porti, luba enne UFW sisselülitamist õige port |

```bash
sudo ufw allow 514/tcp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Syslog TCP lubatakse | Kui kasutad ainult UDP-d, võib TCP puududa, aga dokumenteeri otsus |

```bash
sudo ufw allow 514/udp
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Syslog UDP lubatakse | Kui kasutad ainult TCP-d, võib UDP puududa, aga dokumenteeri otsus |

---

## 5.4 Lülita UFW sisse

```bash
sudo ufw enable
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UFW aktiveerub | Kui oled SSH-ga sees ja 22/tcp pole lubatud, võid ühenduse kaotada |

Kontrolli reegleid:

```bash
sudo ufw status numbered
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed lubatud porte 22/tcp, 514/tcp ja 514/udp | Kui mõni vajalik port puudub, lisa see `sudo ufw allow PORT/protocol` käsuga |

---

# 6. Seadista AlmaServer ja DebianServer logisid saatma

## Eesmärk

AlmaServer ja DebianServer peavad saatma oma logid logiserverisse.

Näide logiserveri aadressist:

```text
logi.sinuNimi.local
```

Soovitatav on kasutada TCP-d:

```text
*.* @@logi.sinuNimi.local:514
```

Üks `@` tähendab UDP-d.  
Kaks `@@` tähendab TCP-d.

---

## 6.1 Kontrolli kliendimasinas rsyslog teenust

Tee seda nii AlmaServeris kui DebianServeris.

```bash
systemctl status rsyslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui teenust pole, paigalda see. Debian/Ubuntu: `sudo apt install rsyslog -y`. Alma: `sudo dnf install rsyslog -y` |

Debianis vajadusel:

```bash
sudo apt install rsyslog -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| rsyslog paigaldatakse | Kui repo/DNS error, kontrolli võrku |

Almas vajadusel:

```bash
sudo dnf install rsyslog -y
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| rsyslog paigaldatakse | Kui repo/DNS error, kontrolli võrku ja nimeserverit |

Luba käivitumisel:

```bash
sudo systemctl enable rsyslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| rsyslog lubatakse käivitumisel | Kui teenust ei leita, paigaldus ei õnnestunud |

Käivita teenus:

```bash
sudo systemctl start rsyslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Teenus käivitub | Kui failed, vaata `sudo journalctl -xeu rsyslog` |

---

## 6.2 Kontrolli, kas logiserveri nimi lahendub

```bash
getent hosts logi.sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse logiserveri IP | Kui vastust pole, lisa DNS kirje või ajutiselt `/etc/hosts` kirje |

Ajutine hosts lahendus:

```bash
sudo nano /etc/hosts
```

Lisa:

```text
LOGISERVERI_IP logi.sinuNimi.local logi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub ja nimi lahendub | Kui ikka ei lahendu, kontrolli kirjavigu |

---

## 6.3 Lisa logide edastamise konfiguratsioon

Loo eraldi rsyslog konfiguratsioonifail:

```bash
sudo nano /etc/rsyslog.d/60-logiserver.conf
```

Lisa TCP jaoks:

```text
*.* @@logi.sinuNimi.local:514
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail salvestub | Kui ei saa salvestada, kasuta `sudo` |

Kui tahad kasutada UDP-d, lisa:

```text
*.* @logi.sinuNimi.local:514
```

Kontrolli rsyslog konfiguratsiooni:

```bash
sudo rsyslogd -N1
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Konfiguratsioonis pole vigu | Kui näitab errorit, paranda faili `/etc/rsyslog.d/60-logiserver.conf` sisu |

Taaskäivita rsyslog:

```bash
sudo systemctl restart rsyslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| rsyslog käivitub veata | Kui failed, vaata `sudo journalctl -xeu rsyslog` |

---

## 6.4 Testi logi saatmist

Kliendimasinas:

```bash
logger "TEST: logi saatmine serverisse masinast $(hostname)"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna viga | Kui `logger` puudub, paigalda util-linux pakett või kontrolli süsteemi |

Logiserveris kontrolli logisid:

```bash
sudo grep "TEST: logi saatmine" /var/log/syslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed kliendimasinast tulnud testlogi | Kui ei näe, kontrolli porti 514, UFW-d, rsyslog konfiguratsiooni ja DNS-i |

Kui Ubuntu logiserveris pole `/var/log/syslog` faili, proovi:

```bash
sudo journalctl | grep "TEST: logi saatmine"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed testlogi | Kui ei näe, logi ei jõudnud serverisse või rsyslog ei kirjuta seda sinna |

---

# 7. UbuntuPilet2 alglaaduri ehk GRUB-i parandamine

## Eesmärk

Kui UbuntuPilet2 ei käivitu, tuleb alglaadur taastada.

Seda tehakse tavaliselt Live ISO või rescue keskkonna abil.

---

## 7.1 Kontrolli kettaid rescue/live keskkonnas

```bash
lsblk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed süsteemiketast ja partitsioone, näiteks `/dev/sda1`, `/dev/sda2` | Kui ketast ei näe, kontrolli VM-is, kas ketas on ühendatud |

Kontrolli failisüsteeme:

```bash
sudo blkid
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed partitsioonide UUID-sid ja failisüsteeme | Kui infot pole, võib ketas või partitsioonitabel katki olla |

---

## 7.2 Mounti Ubuntu süsteem

Näide, kui Ubuntu juurpartitsioon on `/dev/sda2`.

```bash
sudo mount /dev/sda2 /mnt
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Partitsioon ühendatakse `/mnt` alla | Kui tuleb `wrong fs type`, valisid vale partitsiooni. Kontrolli `lsblk -f` |

Kui eraldi EFI partitsioon on näiteks `/dev/sda1`, mounti see:

```bash
sudo mount /dev/sda1 /mnt/boot/efi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| EFI partitsioon ühendatakse | Kui `/mnt/boot/efi` puudub, loo `sudo mkdir -p /mnt/boot/efi` |

Kui kaust puudub:

```bash
sudo mkdir -p /mnt/boot/efi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse | Kui permission denied, kasuta `sudo` |

---

## 7.3 Seo süsteemikataloogid

```bash
sudo mount --bind /dev /mnt/dev
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/dev` seotakse chroot keskkonda | Kui error, kontrolli, kas `/mnt` on õigesti mountitud |

```bash
sudo mount --bind /proc /mnt/proc
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/proc` seotakse | Kui error, kontrolli mounti |

```bash
sudo mount --bind /sys /mnt/sys
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/sys` seotakse | Kui error, kontrolli mounti |

```bash
sudo mount --bind /run /mnt/run
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/run` seotakse | Kui error, enamasti pole kriitiline, aga proovi uuesti |

---

## 7.4 Mine chroot keskkonda

```bash
sudo chroot /mnt
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Prompt muutub ja oled Ubuntu süsteemi sees root õigustes | Kui tuleb error, pole süsteemipartitsioon õigesti mountitud |

---

## 7.5 Paigalda GRUB uuesti

BIOS/Legacy süsteemi puhul:

```bash
grub-install /dev/sda
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `Installation finished. No error reported.` | Kui error, kontrolli, et valisid ketta `/dev/sda`, mitte partitsiooni `/dev/sda1` |

UEFI süsteemi puhul:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| GRUB paigaldatakse EFI partitsioonile | Kui error, kontrolli, kas EFI partitsioon on mountitud `/boot/efi` alla |

Uuenda GRUB konfiguratsioon:

```bash
update-grub
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Leitakse Linux kernelid ja luuakse grub.cfg | Kui error, kontrolli `/boot` olemasolu ja mountimist |

Välju chrootist:

```bash
exit
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Jõuad tagasi live/rescue keskkonda | Kui ei välju, kasuta `Ctrl+D` |

---

## 7.6 Unmount ja restart

```bash
sudo umount /mnt/run
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust unmountitakse | Kui ütleb busy, jätka teistega või kasuta hiljem `sudo umount -l` |

```bash
sudo umount /mnt/sys
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust unmountitakse | Kui busy, kasuta `sudo umount -l /mnt/sys` |

```bash
sudo umount /mnt/proc
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust unmountitakse | Kui busy, kasuta lazy unmounti |

```bash
sudo umount /mnt/dev
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust unmountitakse | Kui busy, kasuta `sudo umount -l /mnt/dev` |

Kui EFI oli mountitud:

```bash
sudo umount /mnt/boot/efi
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| EFI partitsioon unmountitakse | Kui seda ei olnud mountitud, võib tulla veateade, see pole probleem |

Unmounti juurpartitsioon:

```bash
sudo umount /mnt
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/mnt` unmountitakse | Kui busy, kontrolli, et terminal pole `/mnt` sees |

Restart:

```bash
sudo reboot
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| UbuntuPilet2 käivitub kettalt | Kui ei käivitu, kontrolli boot orderit ja korda GRUB taastamist õige kettaga |

---

# 8. Vorminda UbuntuPilet2 külge ühendatud kõvaketas

## Eesmärk

Ülesandes on vaja vormindada UbuntuPilet2 masinasse ühendatud kõvaketas ja ühendada see püsivalt.

Oluline: ära vorminda süsteemiketast.

---

## 8.1 Kontrolli kettaid

```bash
lsblk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed lisaketast, näiteks `/dev/sdb` | Kui lisaketast ei näe, kontrolli Proxmox/VM seadistust |

Kontrolli failisüsteeme:

```bash
lsblk -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed, millisel kettal pole failisüsteemi või mountpointi | Kui ei tea, kumb on lisaketas, ära vorminda enne, kui oled kindel |

---

## 8.2 Loo partitsioon

Näide lisakettaga `/dev/sdb`.

```bash
sudo fdisk /dev/sdb
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Avaneb fdisk käsurežiim | Kui ketast pole, kontrolli `lsblk` väljundit |

Fdiskis sisesta järjest:

```text
n
p
1
ENTER
ENTER
w
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Luuakse uus partitsioon `/dev/sdb1` | Kui saad veateate, kontrolli, kas ketas on kasutuses või valisid vale seadme |

Kontrolli:

```bash
lsblk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `/dev/sdb1` | Kui partitsiooni ei tekkinud, korda fdisk sammu või kontrolli ketast |

---

## 8.3 Vorminda partitsioon ext4 failisüsteemiks

```bash
sudo mkfs.ext4 /dev/sdb1
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failisüsteem luuakse edukalt | Kui küsib kinnitust, kontrolli veel kord, et see pole süsteemipartitsioon |

Kontrolli:

```bash
lsblk -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/dev/sdb1` real on `ext4` | Kui failisüsteemi pole, vormindamine ei õnnestunud |

---

# 9. Ühenda ketas püsivalt süsteemi

## Eesmärk

Ketas peab pärast restarti automaatselt ühenduma.

Näidis mountpoint:

```text
/backup
```

---

## 9.1 Loo mountpoint

```bash
sudo mkdir -p /backup
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust `/backup` luuakse | Kui permission denied, kasuta `sudo` |

---

## 9.2 Leia partitsiooni UUID

```bash
sudo blkid /dev/sdb1
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse UUID | Kui UUID puudub, kontrolli, kas partitsioon on vormindatud |

Näide:

```text
/dev/sdb1: UUID="abcd-1234" TYPE="ext4"
```

---

## 9.3 Lisa ketas `/etc/fstab` faili

```bash
sudo nano /etc/fstab
```

Lisa faili lõppu rida:

```text
UUID=SIIN-SINU-UUID /backup ext4 defaults 0 2
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Rida salvestub faili | Kui teed UUID-s vea, võib mount ebaõnnestuda. Kontrolli hoolikalt |

Testi mountimist:

```bash
sudo mount -a
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna viga | Kui tuleb fstab error, ava `/etc/fstab` ja paranda UUID või mountpoint |

Kontrolli:

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `/backup` mountpointi | Kui ei näe, kontrolli `mount -a` veateadet |

Kontrolli ka:

```bash
lsblk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/dev/sdb1` on ühendatud `/backup` alla | Kui mountpoint puudub, kontrolli fstab rida |

---

# 10. Tee `/home` kaustast varukoopia

## Eesmärk

UbuntuPilet2 masina `/home` kaust tuleb varundada ühendatud kettale.

---

## 10.1 Loo varukoopiate kaust

```bash
sudo mkdir -p /backup/home-backups
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kaust luuakse | Kui `/backup` pole mountitud, kontrolli `df -h` |

---

## 10.2 Tee käsitsi esimene varukoopia rsynciga

```bash
sudo rsync -aAXv /home/ /backup/home-backups/home/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/home` sisu kopeeritakse `/backup/home-backups/home/` alla | Kui tuleb permission denied, kasuta `sudo`. Kui ruum saab täis, kontrolli `df -h` |

Kontrolli varukoopiat:

```bash
ls -lah /backup/home-backups/home/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed kasutajate kodukaustu | Kui kaust on tühi, rsync ei töötanud või `/home` oli tühi |

---

## 10.3 Loo varundusskript

```bash
sudo nano /usr/local/sbin/backup-home.sh
```

Lisa:

```bash
#!/bin/bash

BACKUP_DIR="/backup/home-backups"
DATE=$(date +%F_%H-%M-%S)

mkdir -p "$BACKUP_DIR"

rsync -aAX --delete /home/ "$BACKUP_DIR/home-current/"

tar -czf "$BACKUP_DIR/home-$DATE.tar.gz" /home

find "$BACKUP_DIR" -name "home-*.tar.gz" -type f -mtime +7 -delete
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Skript salvestub | Kui ei saa salvestada, kontrolli `sudo` kasutamist |

Anna käivitusõigus:

```bash
sudo chmod +x /usr/local/sbin/backup-home.sh
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Skript saab käivitusõiguse | Kui fail puudub, kontrolli faili asukohta |

Testi skripti:

```bash
sudo /usr/local/sbin/backup-home.sh
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Varukoopia luuakse `/backup/home-backups` alla | Kui tuleb error, kontrolli skripti ridu ja `/backup` mounti |

Kontrolli tulemust:

```bash
ls -lah /backup/home-backups/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `home-current` kausta ja `.tar.gz` varukoopiat | Kui faile pole, skript ei käivitunud edukalt |

---

## 10.4 Lisa cron töö automaatseks varundamiseks

Ava root crontab:

```bash
sudo crontab -e
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Avaneb crontab redaktor | Kui küsib redaktorit, vali näiteks `nano` |

Lisa rida igapäevaseks varukoopiaks kell 02:00:

```text
0 2 * * * /usr/local/sbin/backup-home.sh >> /var/log/home-backup.log 2>&1
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
| Näed lisatud backup rida | Kui rida puudub, lisa uuesti `sudo crontab -e` kaudu |

---

# 11. Kontrolli pärast restarti

## Eesmärk

Veendu, et ketas ühendub automaatselt ja teenused töötavad.

Restart:

```bash
sudo reboot
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Masin taaskäivitub edukalt | Kui masin ei boodi, kontrolli GRUB-i ja `/etc/fstab` vigu rescue režiimis |

Pärast restarti kontrolli ketast:

```bash
df -h
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `/backup` on mountitud | Kui pole, kontrolli `sudo mount -a` ja `/etc/fstab` |

Kontrolli backup faile:

```bash
ls -lah /backup/home-backups/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Varukoopiad on alles | Kui kaust on tühi, kontrolli, kas õige ketas on mountitud |

Kontrolli rsyslogit:

```bash
systemctl status rsyslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui failed, kontrolli rsyslog konfiguratsiooni |

---

# 12. Lõputestid logiserveris

## 12.1 Kontrolli rsyslog porti

```bash
sudo ss -tulpen | grep :514
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Port 514 kuulab TCP ja/või UDP peal | Kui ei kuula, kontrolli `/etc/rsyslog.conf` ja taaskäivita rsyslog |

---

## 12.2 Kontrolli tulemüüri

```bash
sudo ufw status verbose
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Lubatud ainult vajalikud pordid: 22/tcp, 514/tcp, 514/udp | Kui üleliigseid porte on avatud, eemalda need käsuga `sudo ufw delete allow PORT/protocol` |

---

## 12.3 Saada testlogi kliendist

AlmaServeris või DebianServeris:

```bash
logger "TESTLOGI $(hostname) -> logiserver"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk lõpeb veata | Kui `logger` puudub, kontrolli util-linux paigaldust |

Logiserveris:

```bash
sudo grep "TESTLOGI" /var/log/syslog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed kliendi saadetud testlogi | Kui ei näe, kontrolli DNS-i, UFW-d, rsyslog forwardingut ja porti 514 |

Alternatiiv:

```bash
sudo journalctl | grep "TESTLOGI"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed testlogi journalis | Kui ei näe, logi ei jõua serverisse |

---

# 13. Dokumentatsiooni näidis

```markdown
## Linux pilet 2 dokumentatsioon

### Eesmärk

Eesmärk oli seadistada UbuntuServer masin logiserveriks, seadistada AlmaServer ja DebianServer logisid logiserverisse saatma, piirata tulemüüriga avatud pordid, luua logiserverile FQDN kirje, taastada UbuntuPilet2 alglaadur, vormindada lisaketas, ühendada see püsivalt süsteemi ning seadistada `/home` kausta varukoopiad.

### Kasutatud masinad

| Masin | Roll |
|---|---|
| UbuntuServer | Logiserver |
| AlmaServer | Logide saatja |
| DebianServer | Logide saatja |
| UbuntuPilet2 | GRUB taastamine, ketas ja varukoopiad |
| DNS server | FQDN kirje `logi.sinuNimi.local` |

### Tehtud seadistused

- Muutsin logiserveri hostname’i.
- Lisasin DNS kirje `logi.sinuNimi.local`.
- Paigaldasin ja seadistasin `rsyslog` teenuse.
- Lubasin rsyslogil võtta vastu logisid pordil 514.
- Seadistasin UFW tulemüüri.
- Seadistasin AlmaServeri ja DebianServeri logisid logiserverisse saatma.
- Testisin logide saatmist käsuga `logger`.
- Taastasin UbuntuPilet2 alglaaduri GRUB-i.
- Vormindasin lisaketta ext4 failisüsteemiks.
- Ühendasin ketta püsivalt `/backup` alla.
- Lõin `/home` kausta varundusskripti.
- Lisasin varunduse cron tööna.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `systemctl status rsyslog` | rsyslog töötab |
| `ss -tulpen | grep :514` | logiserver kuulab pordil 514 |
| `ufw status verbose` | lubatud ainult vajalikud pordid |
| `getent hosts logi.sinuNimi.local` | FQDN lahendub õigeks IP-ks |
| `logger "TESTLOGI"` | testlogi jõuab logiserverisse |
| `df -h` | `/backup` on mountitud |
| `ls -lah /backup/home-backups/` | varukoopiad on olemas |
| `crontab -l` | automaatne backup töö on lisatud |

### Kokkuvõte

Linux pilet 2 lahenduse tulemusena töötab UbuntuServer logiserverina, AlmaServer ja DebianServer saadavad sinna logisid, tulemüüris on lubatud ainult vajalikud pordid, UbuntuPilet2 alglaadur on taastatud, lisaketas on ühendatud `/backup` alla ning `/home` kaustast tehakse varukoopiaid.
```

---

# 14. Troubleshooting

## Logid ei jõua logiserverisse

Kontrolli logiserveris:

```bash
sudo ss -tulpen | grep :514
```

Kui port ei kuula, kontrolli `/etc/rsyslog.conf`.

Kontrolli tulemüüri:

```bash
sudo ufw status verbose
```

Kui 514 puudub, lisa:

```bash
sudo ufw allow 514/tcp
sudo ufw allow 514/udp
```

Kontrolli kliendis:

```bash
cat /etc/rsyslog.d/60-logiserver.conf
```

Kui seal pole logiserveri rida, lisa:

```text
*.* @@logi.sinuNimi.local:514
```

---

## DNS nimi `logi.sinuNimi.local` ei lahendu

Kontrolli:

```bash
getent hosts logi.sinuNimi.local
```

Kui vastust pole:

- lisa DNS serverisse A-kirje
- kontrolli, et klient kasutab õiget DNS serverit
- ajutiselt lisa `/etc/hosts` kirje

---

## rsyslog ei käivitu

Kontrolli konfiguratsiooni:

```bash
sudo rsyslogd -N1
```

Kui näitab errorit, paranda viidatud fail.

Vaata logi:

```bash
sudo journalctl -xeu rsyslog
```

---

## UFW blokeerib logid

Kontrolli:

```bash
sudo ufw status numbered
```

Kui 514 puudub:

```bash
sudo ufw allow 514/tcp
sudo ufw allow 514/udp
```

---

## GRUB ei taastu

Kontrolli, kas mountisid õige partitsiooni:

```bash
lsblk -f
```

BIOS puhul kasuta:

```bash
grub-install /dev/sda
```

UEFI puhul kontrolli, et EFI partitsioon on mountitud:

```bash
mount | grep efi
```

Kui pole:

```bash
sudo mount /dev/sda1 /mnt/boot/efi
```

---

## `/backup` ei mounti pärast restarti

Kontrolli fstab faili:

```bash
sudo nano /etc/fstab
```

Testi:

```bash
sudo mount -a
```

Kui tuleb UUID error, kontrolli:

```bash
sudo blkid /dev/sdb1
```

Paranda UUID `/etc/fstab` failis.

---

## Backup ei teki

Käivita skript käsitsi:

```bash
sudo /usr/local/sbin/backup-home.sh
```

Kui tuleb error, kontrolli:

```bash
df -h
ls -lah /backup
ls -lah /usr/local/sbin/backup-home.sh
```

Kontrolli cron tööd:

```bash
sudo crontab -l
```

Kontrolli logi:

```bash
sudo tail -n 50 /var/log/home-backup.log
```

---

# Kõige lühem spikker

```text
1. Kontrolli UbuntuServeri IP, hostname ja DNS
2. Muuda hostname logi
3. Lisa FQDN logi.sinuNimi.local
4. Paigalda rsyslog
5. Luba rsyslogil kuulata porti 514 TCP/UDP
6. Ava UFW-s ainult 22/tcp, 514/tcp, 514/udp
7. Seadista AlmaServer ja DebianServer logisid saatma
8. Testi logger käsuga
9. Paranda UbuntuPilet2 GRUB
10. Leia lisaketas lsblk abil
11. Loo partitsioon ja vorminda ext4
12. Mounti ketas /backup alla
13. Lisa ketas /etc/fstab faili
14. Tee /home varukoopia rsynciga
15. Loo backup skript
16. Lisa cron töö
17. Testi pärast restarti
18. Dokumenteeri
```
