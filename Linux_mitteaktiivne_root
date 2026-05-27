# Linuxi root kasutaja mitteaktiivseks jätmine

Linuxi virtuaalmasinates peab `root` kasutaja olema mitteaktiivne, välja arvatud juhul, kui ülesanne või pilet ütleb eraldi, et `root` kasutaja peab olema aktiivne.

Tavapärane töö käib tavalise kasutajaga, kellel on vajadusel `sudo` õigused.

---

## 1. Mida tähendab, et root kasutaja on mitteaktiivne?

Linuxis on `root` kõige kõrgemate õigustega kasutaja.

`root` kasutaja saab muuta kogu süsteemi, näiteks:

```bash
apt install
systemctl restart ssh
nano /etc/ssh/sshd_config
rm -rf /kaust
```

Kui `root` kasutaja on mitteaktiivne, tähendab see tavaliselt seda, et `root` kontoga ei saa otse sisse logida.

Näiteks ei peaks saama sisse logida nii:

```bash
su root
```

või SSH kaudu:

```bash
ssh root@192.168.1.50
```

Selle asemel kasutatakse tavalist kasutajat ja vajadusel `sudo` käsku.

Näide:

```bash
sudo apt update
sudo systemctl restart ssh
```

---

## 2. Miks root kasutaja mitteaktiivseks jäetakse?

`root` kasutaja jäetakse mitteaktiivseks turvalisuse pärast.

Kui `root` kontoga saab otse sisse logida, on ründajal lihtsam proovida süsteemi sisse murda, sest kasutajanimi `root` on alati teada.

Turvalisem lahendus on:

```text
tavaline kasutaja + sudo õigused
```

Näiteks:

```bash
kasutaja
```

on tavaline kasutaja, aga ta kuulub `sudo` gruppi.

Siis saab ta administraatori käske teha nii:

```bash
sudo käsk
```

Näiteks:

```bash
sudo apt update
```

---

## 3. Kontrolli, kas oled tavalise kasutajaga sisse logitud

Kontrolli praegust kasutajat:

```bash
whoami
```

Kui väljund on näiteks:

```text
kasutaja
```

siis oled tavalise kasutajana sees.

Kui väljund on:

```text
root
```

siis oled root kasutajana sees.

Üldjuhul ei peaks igapäevane töö käima `root` kasutajana.

---

## 4. Kontrolli, kas kasutajal on sudo õigused

Kontrolli kasutaja gruppe:

```bash
groups
```

Näide korrektsest väljundist:

```text
kasutaja sudo
```

Kui väljundis on `sudo`, saab kasutaja kasutada administraatori õigusi käsuga `sudo`.

Testi:

```bash
sudo whoami
```

Kui väljund on:

```text
root
```

siis sudo õigused töötavad.

---

## 5. Lisa kasutaja sudo gruppi

Kui kasutaja ei ole veel `sudo` grupis, lisa ta sinna.

Näide, kui kasutajanimi on `kasutaja`:

```bash
sudo usermod -aG sudo kasutaja
```

Kui oled käsu teinud, logi välja ja sisse tagasi:

```bash
exit
```

Seejärel kontrolli uuesti:

```bash
groups
```

ja testi:

```bash
sudo whoami
```

Kui väljund on:

```text
root
```

siis on kõik korras.

---

## 6. Kontrolli root kasutaja olekut

Root kasutaja parooli olekut saab kontrollida käsuga:

```bash
sudo passwd -S root
```

Näide väljundist:

```text
root L ...
```

Tähtis osa on täht pärast kasutajanime.

| Tähis | Tähendus |
|---|---|
| `L` | konto parool on lukus ehk root parooliga sisselogimine on keelatud |
| `P` | parool on määratud ehk root kasutaja võib olla aktiivne |
| `NP` | parooli pole määratud |

Kui näed `L`, siis on root parool lukus.

---

## 7. Muuda root kasutaja mitteaktiivseks

Kui root kasutaja on aktiivne ja ülesanne ei nõua selle kasutamist, saab root parooli lukustada käsuga:

```bash
sudo passwd -l root
```

See lukustab root kasutaja parooli.

Pärast seda kontrolli:

```bash
sudo passwd -S root
```

Kui väljundis on `L`, siis on root parool lukus.

---

## 8. Keela root kasutaja SSH kaudu sisselogimine

Lisaks root parooli lukustamisele on oluline kontrollida, et root ei saaks SSH kaudu sisse logida.

Ava SSH seadistusfail:

```bash
sudo nano /etc/ssh/sshd_config
```

Otsi rida:

```text
PermitRootLogin
```

Kui seda ei ole, lisa failis sobivasse kohta rida:

```text
PermitRootLogin no
```

Kui seal on näiteks:

```text
PermitRootLogin yes
```

muuda see selliseks:

```text
PermitRootLogin no
```

Salvesta fail:

```text
Ctrl + O
Enter
Ctrl + X
```

Taaskäivita SSH teenus:

```bash
sudo systemctl restart ssh
```

---

## 9. Kontrolli, et root SSH login ei tööta

Teisest masinast proovi:

```bash
ssh root@192.168.1.50
```

Kui root login on keelatud, siis ei tohiks root kasutajaga sisse saada.

Võib tulla näiteks:

```text
Permission denied
```

See on õige tulemus, kui root peab olema mitteaktiivne.

---

## 10. Õige tööviis administraatori käskude jaoks

Igapäevaselt ei logita sisse root kasutajana.

Õige tööviis on:

```bash
ssh kasutaja@192.168.1.50
```

Seejärel kasutatakse vajadusel `sudo` käsku:

```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl restart ssh
sudo nano /etc/ssh/sshd_config
```

See tähendab, et kasutaja töötab tavakasutajana, aga saab vajadusel teha administraatori tegevusi.

---

## 11. Kui ülesanne nõuab root kasutajat

Kui pilet või ülesanne ütleb eraldi, et root kasutaja peab olema aktiivne, siis võib root kasutaja aktiveerida.

Root parooli määramiseks:

```bash
sudo passwd root
```

Seejärel sisesta uus root parool.

Kontrollimiseks:

```bash
sudo passwd -S root
```

Kui väljundis on `P`, siis root kasutajal on parool määratud.

Oluline: kui ülesanne ei nõua root kasutajat, siis jäta root mitteaktiivseks.

---

## 12. Root kasutaja uuesti lukustamine

Kui root kasutajat oli ajutiselt vaja, saab selle pärast uuesti lukustada:

```bash
sudo passwd -l root
```

Kontroll:

```bash
sudo passwd -S root
```

Kui väljundis on `L`, siis root on jälle lukus.

---

## 13. Tüüpilised vead ja lahendused

### Viga: kasutaja ei saa sudo käsku kasutada

Kui tuleb teade:

```text
kasutaja is not in the sudoers file
```

siis kasutaja ei kuulu `sudo` gruppi.

Lahendus:

```bash
sudo usermod -aG sudo kasutaja
```

Seejärel logi välja ja sisse tagasi.

---

### Viga: root kasutajaga saab SSH kaudu sisse

Kui root kasutajaga saab SSH kaudu sisse, kontrolli SSH seadistust:

```bash
sudo nano /etc/ssh/sshd_config
```

Seal peab olema:

```text
PermitRootLogin no
```

Seejärel taaskäivita SSH:

```bash
sudo systemctl restart ssh
```

---

### Viga: sudo passwd -S root näitab P

Kui käsk:

```bash
sudo passwd -S root
```

näitab:

```text
root P ...
```

siis root kasutajal on parool määratud.

Kui root ei pea olema aktiivne, lukusta see:

```bash
sudo passwd -l root
```

Kontrolli uuesti:

```bash
sudo passwd -S root
```

---

## 14. Näidis algusest lõpuni

Näide: Linuxi kasutaja nimi on `kasutaja`.

Kontrolli, kes on sisse logitud:

```bash
whoami
```

Kontrolli, kas kasutajal on sudo õigused:

```bash
groups
sudo whoami
```

Kontrolli root olekut:

```bash
sudo passwd -S root
```

Lukusta root kasutaja:

```bash
sudo passwd -l root
```

Keela root SSH login:

```bash
sudo nano /etc/ssh/sshd_config
```

Lisa või muuda rida:

```text
PermitRootLogin no
```

Taaskäivita SSH:

```bash
sudo systemctl restart ssh
```

Kontrolli root olekut uuesti:

```bash
sudo passwd -S root
```

---

## 15. Kõige lühem spikker

```bash
whoami
groups
sudo whoami
sudo passwd -S root
sudo passwd -l root
sudo nano /etc/ssh/sshd_config
sudo systemctl restart ssh
```

SSH seadistusfailis peab olema:

```text
PermitRootLogin no
```

Root kasutaja peab olema mitteaktiivne, välja arvatud juhul, kui pilet nõuab teisiti.
