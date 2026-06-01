# DebianPilet5 restartimise põhjuse uurimine

## Eesmärk

DebianPilet5 masin taaskäivitub iseenesest. Vihje järgi on põhjus **tarkvaraline või mingi tarkvaraga seotud**, seega kontrollin eelkõige:

- cron töid
- systemd teenuseid ja taimereid
- automaatseid uuendusi
- skripte, mis käivitavad `reboot`, `shutdown` või `systemctl reboot`
- watchdog teenust
- logisid
- kasutajate käsuajalugu
- scheduled reboot seadistusi

---

## 1. Kontrolli, millal masin restartis

```bash
last reboot
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed varasemaid taaskäivitusi ja nende kellaaegu | Kui restartid toimuvad kindla intervalliga, viitab see cronile, systemd timerile või skriptile |

Kontrolli süsteemi käivitumise aega:

```bash
uptime
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed, kui kaua süsteem on üleval olnud | Kui uptime on väga lühike, on masin hiljuti restartinud |

---

## 2. Vaata eelmise boot’i logisid

```bash
journalctl -b -1 -e
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed eelmise käivituse lõpu logisid | Kui logis on `reboot`, `shutdown`, `systemd-shutdown` või mõni skript, siis on põhjus tõenäoliselt tarkvaraline |

Otsi restartiga seotud ridu:

```bash
journalctl | grep -i "reboot\|shutdown\|poweroff\|restart"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Võib näidata, mis protsess restarti algatas | Kui midagi ei leita, kontrolli cron ja systemd timerid eraldi |

Vaata eelmise boot’i restartimisega seotud ridu:

```bash
journalctl -b -1 | grep -i "reboot\|shutdown\|poweroff\|restart"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed eelmise sessiooni lõpu põhjuseid | Kui rida näitab näiteks `CRON`, `systemd`, `unattended-upgrades` või mõnda skripti, on suund käes |

---

## 3. Kontrolli cron töid

Tarkvaraline automaatne restart on väga tihti cron tööga tehtud.

Kontrolli root kasutaja crontabi:

```bash
sudo crontab -l
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed root cron töid või teadet, et crontabi pole | Kui seal on `reboot`, `shutdown`, `systemctl reboot` või skript, mis seda teeb, on põhjus tõenäoliselt leitud |

Kontrolli süsteemi crontab faili:

```bash
cat /etc/crontab
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed süsteemseid cron ridu | Kui seal on restarti käsk, kommenteeri see välja või eemalda |

Kontrolli cron katalooge:

```bash
ls -lah /etc/cron.d/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed cron lisafaile | Kui mõni fail tundub kahtlane, ava see ja kontrolli sisu |

```bash
ls -lah /etc/cron.hourly/
ls -lah /etc/cron.daily/
ls -lah /etc/cron.weekly/
ls -lah /etc/cron.monthly/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed perioodilisi skripte | Kui mõni skript sisaldab restarti käsku, on põhjus leitud |

Otsi cronidest restarti käske:

```bash
sudo grep -R "reboot\|shutdown\|poweroff\|systemctl reboot" /etc/cron* 2>/dev/null
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ideaalis ei leia midagi | Kui leiab faili ja rea, ava see fail ning eemalda või kommenteeri restarti käsk |

Näide parandusest:

```bash
sudo nano /etc/cron.d/kahtlane-fail
```

Pane restarti rea ette `#`:

```text
# * * * * * root reboot
```

---

## 4. Kontrolli systemd taimereid

Systemd timer võib samuti regulaarselt skripte käivitada.

```bash
systemctl list-timers --all
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed aktiivseid ja mitteaktiivseid timereid | Kui mõni timer käivitub täpselt restartimise ajal, kontrolli selle teenust |

Otsi kahtlaseid teenuseid:

```bash
systemctl list-units --type=service --all | grep -i "reboot\|restart\|shutdown\|watchdog"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Tavaliselt ei tohiks olla kahtlast restart teenust | Kui leiad kahtlase teenuse, kontrolli seda `systemctl cat TEENUS` käsuga |

Näide:

```bash
systemctl cat kahtlane.service
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed, mida teenus käivitab | Kui `ExecStart` sisaldab `reboot` või skripti, mis teeb restarti, on põhjus leitud |

Keela kahtlane teenus:

```bash
sudo systemctl disable --now kahtlane.service
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Teenus peatatakse ja ei käivitu enam automaatselt | Kui teenust ei leita, kontrolli täpset nime |

Keela kahtlane timer:

```bash
sudo systemctl disable --now kahtlane.timer
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Timer peatatakse | Kui timer puudub, kontrolli nime `systemctl list-timers --all` väljundist |

---

## 5. Otsi kogu süsteemist restarti käske

See on väga kasulik, kui keegi on pannud restarti käsu skripti sisse.

```bash
sudo grep -R "reboot" /etc /usr/local /opt 2>/dev/null
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ideaalis ei leia kahtlaseid skripte | Kui leiab skripti, ava see ja kontrolli, kas see põhjustab restarti |

Otsi ka `shutdown` käske:

```bash
sudo grep -R "shutdown" /etc /usr/local /opt 2>/dev/null
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ei tohiks olla automaatset shutdown käsku | Kui leiab, kontrolli faili sisu |

Otsi `systemctl reboot` käske:

```bash
sudo grep -R "systemctl reboot" /etc /usr/local /opt 2>/dev/null
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ei tohiks midagi leida | Kui leiab, on väga tõenäoline, et see on restartimise põhjus |

Näiteks kui leitakse:

```text
/etc/cron.d/restart-server:* * * * * root /sbin/reboot
```

siis põhjus on leitud: cron käivitab iga minut restarti.

---

## 6. Kontrolli kasutajate käsuajalugu

Kui restart on lisatud käsitsi või skripti kaudu, võib jälg olla käsuajaloos.

Kontrolli root ajalugu:

```bash
sudo cat /root/.bash_history | grep -i "reboot\|shutdown\|systemctl"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Võib näidata, kas keegi on restarti käske käivitanud | Kui näed kahtlase skripti loomist või cron muutmist, kontrolli vastavat faili |

Kontrolli kasutajate ajalugu:

```bash
for user in /home/*; do echo "=== $user ==="; sudo cat "$user/.bash_history" 2>/dev/null | grep -i "reboot\|shutdown\|crontab\|systemctl"; done
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed kasutajate võimalikke käske | Kui keegi on lisanud crontabi või teenuse, kontrolli seda kohta |

---

## 7. Kontrolli automaatseid uuendusi

Debianis võib automaatne uuendussüsteem mõnikord restarti vajada või restarti käivitada, kui see on nii seadistatud.

Kontrolli unattended-upgrades seadistust:

```bash
dpkg -l | grep unattended-upgrades
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed, kas `unattended-upgrades` on paigaldatud | Kui pole paigaldatud, siis see pole põhjus |

Kontrolli seadistusfaile:

```bash
grep -R "Automatic-Reboot" /etc/apt/apt.conf.d/ 2>/dev/null
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kui `Automatic-Reboot "true";`, siis automaatne restart on lubatud | Kui restart pole soovitud, muuda väärtuseks `false` |

Ava seadistus:

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Muuda vajadusel:

```text
Unattended-Upgrade::Automatic-Reboot "false";
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Automaatne restart keelatakse | Kui faili pole, otsi seadistust muudest apt config failidest |

Kontrolli apt periodic seadistust:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed, kas automaatsed uuendused on lubatud | Kui fail puudub, automaatuuendused ei pruugi olla seadistatud |

---

## 8. Kontrolli watchdog teenust

Watchdog võib süsteemi restartida, kui arvab, et süsteem ei vasta.

```bash
systemctl status watchdog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Tavaliselt võiks olla inactive või puududa | Kui `active (running)`, kontrolli seadistust |

Kontrolli watchdog seadistust:

```bash
cat /etc/watchdog.conf
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed watchdog reegleid | Kui seadistus on vale, võib watchdog restarti põhjustada |

Keela watchdog, kui see põhjustab restarte:

```bash
sudo systemctl disable --now watchdog
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Watchdog peatatakse ja keelatakse | Kui teenust pole, ei ole see põhjus |

---

## 9. Kontrolli rc.local ja käivitusskripte

Vanemates süsteemides võib restart olla lisatud käivitusskripti.

```bash
ls -lah /etc/rc.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Fail võib puududa | Kui fail on olemas, kontrolli selle sisu |

```bash
cat /etc/rc.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Failis ei tohiks olla `reboot` või `shutdown` käsku | Kui on, eemalda või kommenteeri see välja |

Otsi init.d skriptidest:

```bash
sudo grep -R "reboot\|shutdown\|poweroff" /etc/init.d/ 2>/dev/null
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ei tohiks leida automaatset restarti | Kui leiab, kontrolli vastavat skripti |

---

## 10. Kontrolli, kas restarti põhjustab mõni teenus pärast crashi

Mõnel teenusel võib olla systemd seadistus, mis teeb vea korral restarti, aga see restartib tavaliselt ainult teenust, mitte kogu masinat. Siiski tasub kontrollida kahtlaseid custom teenuseid.

```bash
systemctl list-unit-files | grep enabled
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed automaatselt käivituvaid teenuseid | Kui mõni nimi tundub kahtlane, kontrolli seda |

Kontrolli konkreetset teenust:

```bash
systemctl cat TEENUSE_NIMI
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed teenuse käske | Kui `ExecStart` viitab skriptile, kontrolli skripti sisu |

---

## 11. Kontrolli ajastatud shutdown käske

Linuxis saab restarti/shutdowni ajastada ka käsuga `shutdown -r +10`.

Kontrolli, kas on aktiivne ajastatud shutdown:

```bash
shutdown -c
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kui ajastatud shutdown oli olemas, see tühistatakse | Kui ütleb, et pole ajastatud shutdowni, on okei |

Märkus: see käsk sobib ka kiireks paranduseks, kui restart on ajastatud `shutdown` käsuga.

---

# 12. Kõige tõenäolisemad tarkvaralised põhjused

| Põhjus | Kuidas tuvastada | Kuidas parandada |
|---|---|---|
| Cron käivitab rebooti | `grep -R "reboot" /etc/cron*` | Eemalda või kommenteeri cron rida |
| Systemd timer käivitab reboot skripti | `systemctl list-timers --all` | `systemctl disable --now kahtlane.timer` |
| Systemd service teeb rebooti | `systemctl cat teenus.service` | Keela teenus |
| unattended-upgrades teeb automaatrebooti | `grep -R "Automatic-Reboot" /etc/apt/apt.conf.d/` | Pane `Automatic-Reboot "false";` |
| watchdog restartib süsteemi | `systemctl status watchdog` | Keela watchdog või paranda seadistus |
| rc.local sisaldab rebooti | `cat /etc/rc.local` | Eemalda reboot käsk |
| kasutaja lisas restart skripti | `.bash_history` ja `/usr/local/bin` kontroll | Eemalda skript või automaatkäivitus |
| pahatahtlik/katsetuseks tehtud skript | `grep -R "shutdown\|reboot" /etc /opt /usr/local` | Eemalda käivitusmehhanism |

---

# 13. Näidis: kui leidsid cronist reboot käsu

Kui käsk:

```bash
sudo grep -R "reboot\|shutdown\|systemctl reboot" /etc/cron* 2>/dev/null
```

annab näiteks:

```text
/etc/cron.d/restart:*/5 * * * * root /sbin/reboot
```

siis ava fail:

```bash
sudo nano /etc/cron.d/restart
```

Kommenteeri rida välja:

```text
# */5 * * * * root /sbin/reboot
```

Või kustuta fail:

```bash
sudo rm /etc/cron.d/restart
```

Seejärel taaskäivita cron:

```bash
sudo systemctl restart cron
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Cron käivitub uuesti ja restart enam ei kordu | Kui masin ikka restartib, kontrolli systemd timereid ja muid skripte |

Kontrolli cron staatust:

```bash
systemctl status cron
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `active (running)` | Kui cron ei tööta, vaata `journalctl -xeu cron` |

---

# 14. Näidis: kui leidsid automaatse rebooti unattended-upgrades seadistusest

Kontroll:

```bash
grep -R "Automatic-Reboot" /etc/apt/apt.conf.d/
```

Kui näed:

```text
Unattended-Upgrade::Automatic-Reboot "true";
```

siis ava fail:

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Muuda:

```text
Unattended-Upgrade::Automatic-Reboot "false";
```

Salvesta ja kontrolli uuesti:

```bash
grep -R "Automatic-Reboot" /etc/apt/apt.conf.d/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `Automatic-Reboot "false"` | Kui ikka `true`, muutsid valet faili või sama seadistus on teises failis üle kirjutatud |

---

# 15. Näidis: kui leidsid systemd timeri

Kontroll:

```bash
systemctl list-timers --all
```

Kui leiad näiteks:

```text
reboot.timer
```

kontrolli:

```bash
systemctl cat reboot.timer
```

ja seotud teenust:

```bash
systemctl cat reboot.service
```

Kui teenus teeb restarti, keela see:

```bash
sudo systemctl disable --now reboot.timer
sudo systemctl disable --now reboot.service
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Timer ja teenus peatatakse | Kui nime ei leita, kontrolli täpset nime `systemctl list-timers --all` väljundist |

---

# 16. Lõputest pärast parandust

Pärast kahtlase cron töö, timeri, teenuse või seadistuse eemaldamist kontrolli:

```bash
last reboot
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Uusi restarte ei lisandu | Kui lisandub, põhjus pole veel kõrvaldatud |

Kontrolli uptime’i:

```bash
uptime
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Uptime suureneb ega lähe iga paari minuti järel nulli | Kui uptime läheb jälle nulli, jätka cron/systemd/watchdog otsinguga |

Vaata jooksvaid logisid:

```bash
journalctl -f
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed jooksvaid logisid ja restarti algatajat võib enne restarti märgata | Kui enne restarti ilmub kindel teenus või skript, uuri seda |

---

# 17. Dokumentatsiooni näidis

```markdown
## DebianPilet5 restartimise põhjuse uurimine

DebianPilet5 masin taaskäivitus iseenesest. Kuna vihje järgi oli põhjus tarkvaraline, kontrollisin cron töid, systemd timereid, automaatseid uuendusi, watchdog teenust ja süsteemiloge.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `last reboot` | Näitas korduvaid restarte |
| `journalctl -b -1 -e` | Kontrollisin eelmise boot’i lõppu |
| `sudo crontab -l` | Kontrollisin root kasutaja cron töid |
| `grep -R "reboot" /etc/cron*` | Otsisin automaatset reboot käsku |
| `systemctl list-timers --all` | Kontrollisin systemd timereid |
| `grep -R "Automatic-Reboot" /etc/apt/apt.conf.d/` | Kontrollisin automaatuuenduste restarti |
| `systemctl status watchdog` | Kontrollisin watchdog teenust |

### Leitud põhjus

Leidsin, et restarti põhjustas:

```text
KIRJUTA SIIA LEITUD PÕHJUS
```

Näiteks:

```text
/etc/cron.d/restart failis oli rida, mis käivitas regulaarselt /sbin/reboot käsu.
```

### Parandus

Parandasin probleemi järgmiselt:

```text
KIRJUTA SIIA, MIDA TEGID
```

Näiteks:

```text
Kommenteerisin cron failis restarti käsu välja ja taaskäivitasin cron teenuse.
```

### Tulemus

Pärast parandust kontrollisin:

```bash
last reboot
uptime
journalctl -f
```

Masin enam automaatselt ei restartinud.
```

---

# Kõige lühem spikker

```text
1. last reboot
2. uptime
3. journalctl -b -1 -e
4. journalctl | grep -i "reboot\|shutdown\|restart"
5. sudo crontab -l
6. cat /etc/crontab
7. sudo grep -R "reboot\|shutdown\|systemctl reboot" /etc/cron* 2>/dev/null
8. systemctl list-timers --all
9. systemctl list-units --type=service --all | grep -i "reboot\|shutdown\|watchdog"


# DebianPilet5 restartimise põhjuse lahendamise järjekord

Kui masin restardib ise, siis enne DHCP või muu põhiseadistuse tegemist on mõistlik **kõigepealt lahendada restartimise põhjus**.

Muidu võib juhtuda nii:

```text
seadistad DHCP → masin teeb restarti → töö katkeb / teenus ei käivitu / failid jäävad pooleli
```

Õige järjekord Linux pilet 5 puhul:

```text
1. Mine DebianPilet5 masinasse sisse
2. Taasta ligipääs / root või peakasutaja õigused
3. Uuri, miks masin restartib
4. Peata või keela restarti põhjustav tarkvaraline asi
5. Kontrolli, et masin püsib üleval
6. Alles siis tee DHCP serveri seadistus
7. Testi DHCP klientidega
8. Dokumenteeri
```

Ehk **DebianPilet5 puhul esimene suur eesmärk ei ole kohe DHCP**, vaid:

```text
masin stabiilseks → õigused korda → siis teenused
```

---

## Kiired käsud restartimise põhjuse otsimiseks

```bash
last reboot
uptime
journalctl -b -1 -e
sudo crontab -l
cat /etc/crontab
sudo grep -R "reboot\|shutdown\|systemctl reboot" /etc/cron* 2>/dev/null
systemctl list-timers --all
grep -R "Automatic-Reboot" /etc/apt/apt.conf.d/ 2>/dev/null
systemctl status watchdog
```

---

## Mida need käsud kontrollivad?

| Käsk | Mida kontrollib | Mida otsida |
|---|---|---|
| `last reboot` | Näitab varasemaid restartimisi | Kas restartid korduvad kindla intervalliga |
| `uptime` | Näitab, kaua masin on üleval olnud | Kui aeg on väga lühike, restartis masin hiljuti |
| `journalctl -b -1 -e` | Näitab eelmise käivituse lõpu logisid | Mis toimus vahetult enne restarti |
| `sudo crontab -l` | Näitab root kasutaja cron töid | Kas seal on `reboot`, `shutdown` või skript |
| `cat /etc/crontab` | Näitab süsteemset crontabi | Kas süsteem käivitab restarti käsu |
| `grep -R ... /etc/cron*` | Otsib cron failidest restarti käske | Leia fail, mis restarti käivitab |
| `systemctl list-timers --all` | Näitab systemd timereid | Kas mõni timer käivitab restarti |
| `grep -R "Automatic-Reboot"` | Kontrollib automaatuuenduste restarti | Kas unattended-upgrades teeb restarti |
| `systemctl status watchdog` | Kontrollib watchdog teenust | Kas watchdog võib süsteemi restartida |

---

## Näide: kui leiad cronist restarti käsu

Kui leiad näiteks sellise rea:

```text
*/5 * * * * root /sbin/reboot
```

siis see tähendab, et masin teeb restarti iga 5 minuti tagant.

Ava vastav cron fail:

```bash
sudo nano /etc/cron.d/failinimi
```

Pane restarti rea ette `#`:

```text
# */5 * * * * root /sbin/reboot
```

Seejärel taaskäivita cron:

```bash
sudo systemctl restart cron
```

Kontrolli, et cron töötab:

```bash
systemctl status cron
```

Oodatav tulemus:

```text
active (running)
```

Kontrolli, et masin püsib üleval:

```bash
uptime
last reboot
```

Kui masin enam iga paari minuti järel ei restarti, saad minna edasi DHCP osa juurde.

---

## Näide: kui põhjus on automaatuuenduste restart

Kontrolli:

```bash
grep -R "Automatic-Reboot" /etc/apt/apt.conf.d/ 2>/dev/null
```

Kui näed:

```text
Unattended-Upgrade::Automatic-Reboot "true";
```

siis automaatuuendused võivad restarti teha.

Ava seadistusfail:

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Muuda väärtus:

```text
Unattended-Upgrade::Automatic-Reboot "false";
```

Kontrolli uuesti:

```bash
grep -R "Automatic-Reboot" /etc/apt/apt.conf.d/ 2>/dev/null
```

Oodatav tulemus:

```text
Unattended-Upgrade::Automatic-Reboot "false";
```

---

## Näide: kui põhjus on systemd timer

Kontrolli timereid:

```bash
systemctl list-timers --all
```

Kui leiad kahtlase timeri, näiteks:

```text
reboot.timer
```

kontrolli seda:

```bash
systemctl cat reboot.timer
```

Kontrolli seotud teenust:

```bash
systemctl cat reboot.service
```

Kui teenus käivitab restarti, keela see:

```bash
sudo systemctl disable --now reboot.timer
sudo systemctl disable --now reboot.service
```

Kontrolli uuesti:

```bash
systemctl list-timers --all
```

---

## Näide: kui põhjus on watchdog

Kontrolli watchdog teenust:

```bash
systemctl status watchdog
```

Kui watchdog töötab ja põhjustab restarti, keela see:

```bash
sudo systemctl disable --now watchdog
```

Kontrolli uuesti:

```bash
systemctl status watchdog
```

Oodatav tulemus:

```text
inactive
```

või:

```text
disabled
```

---

## Lõputest pärast restartimise põhjuse parandamist

Pärast kahtlase cron töö, timeri, teenuse või seadistuse eemaldamist kontrolli:

```bash
last reboot
```

Oodatav tulemus:

```text
uusi restarte ei lisandu
```

Kontrolli uptime’i:

```bash
uptime
```

Oodatav tulemus:

```text
uptime suureneb ega lähe iga paari minuti järel nulli
```

Vaata jooksvaid logisid:

```bash
journalctl -f
```

Oodatav tulemus:

```text
restarti põhjustavat teenust või skripti enam ei ilmu
```

---

## Millal minna DHCP seadistuse juurde?

DHCP seadistuse juurde mine alles siis, kui:

| Kontroll | Oodatav tulemus |
|---|---|
| `uptime` | masin on püsinud üleval |
| `last reboot` | uusi restarte ei teki |
| cron kontroll | restarti käsku pole |
| systemd timer kontroll | kahtlast timerit pole |
| watchdog kontroll | watchdog ei põhjusta restarti |
| logid | restarti algatajat enam ei ilmu |

Kui masin on stabiilne, siis jätka DHCP osaga:

```text
1. Paigalda isc-dhcp-server
2. Määra DHCP serveri võrguliides
3. Seadista DHCP scope
4. Lisa staatilised lease’id
5. Käivita DHCP teenus
6. Testi klientidega
```

---

## Kokkuvõte

Linux pilet 5 puhul on mõistlik järjekord:

```text
1. DebianPilet5 ligipääs korda
2. Restartimise põhjus üles leida
3. Restartimist põhjustav tarkvaraline asi keelata
4. Kontrollida, et masin püsib üleval
5. Alles siis seadistada DHCP
```

Lühidalt:

```text
masin stabiilseks → õigused korda → DHCP teenus tööle → testimine → dokumentatsioon
```
10. grep -R "Automatic-Reboot" /etc/apt/apt.conf.d/
11. systemctl status watchdog
12. sudo grep -R "reboot\|shutdown" /etc /usr/local /opt 2>/dev/null
13. leitud põhjus välja kommenteerida või teenus/timer keelata
14. uptime ja last reboot abil kontrollida, et restart enam ei kordu
```
