# Kasutaja lisamine sudo gruppi Linuxis

See juhend näitab, kuidas lisada Linuxi kasutaja `sudo` gruppi, et kasutaja saaks kasutada administraatori õigustega käske.

Näiteks saab kasutaja pärast seda kasutada käske kujul:

```bash
sudo apt update
sudo systemctl restart ssh
sudo nano /etc/hosts
```

---

## 1. Mis on sudo grupp?

Linuxis ei anta tavakasutajale vaikimisi täielikke administraatori õigusi.

Kui kasutaja kuulub `sudo` gruppi, saab ta vajadusel käivitada käske administraatori õigustes, kasutades käsu ees `sudo`.

Näide:

```bash
sudo apt update
```

Kui kasutaja ei kuulu `sudo` gruppi, võib tulla veateade:

```text
kasutaja is not in the sudoers file
```

või:

```text
Permission denied
```

---

## 2. Kontrolli praegust kasutajat

Linuxi masinas saab kontrollida, mis kasutajaga oled sisse logitud:

```bash
whoami
```

Näide:

```text
kasutaja
```

---

## 3. Kontrolli, millistes gruppides kasutaja on

Kasuta käsku:

```bash
groups kasutaja
```

Näide:

```bash
groups kasutaja
```

Kui kasutaja on juba `sudo` grupis, näed midagi sellist:

```text
kasutaja : kasutaja sudo
```

Kui `sudo` puudub, siis kasutajal ei ole veel sudo õigusi.

---

## 4. Lisa kasutaja sudo gruppi

Kasutaja lisamiseks `sudo` gruppi kasuta käsku:

```bash
sudo usermod -aG sudo kasutaja
```

Näide:

```bash
sudo usermod -aG sudo kasutaja
```

Käsu selgitus:

| Osa | Tähendus |
|---|---|
| `sudo` | käivitab käsu administraatori õigustes |
| `usermod` | muudab kasutaja seadeid |
| `-aG` | lisab kasutaja gruppi ilma olemasolevaid gruppe eemaldamata |
| `sudo` | grupp, kuhu kasutaja lisatakse |
| `kasutaja` | kasutajanimi, kellele õigused antakse |

---

## 5. Oluline märkus käsu kohta

Kasuta kindlasti parameetrit:

```bash
-aG
```

Mitte ainult:

```bash
-G
```

Õige käsk:

```bash
sudo usermod -aG sudo kasutaja
```

Vale või ohtlik variant:

```bash
sudo usermod -G sudo kasutaja
```

Ilma `-a` parameetrita võib kasutaja teistest gruppidest eemalduda.

---

## 6. Logi välja ja sisse tagasi

Pärast kasutaja lisamist `sudo` gruppi tuleb kasutaja seanss uuendada.

Kõige lihtsam viis:

```bash
exit
```

Seejärel logi uuesti sisse.

Kui kasutad VM-i, võid teha ka lihtsalt kasutaja välja logimise ja uuesti sisse logimise.

---

## 7. Kontrolli, kas kasutaja lisati sudo gruppi

Pärast uuesti sisse logimist kontrolli:

```bash
groups
```

või:

```bash
groups kasutaja
```

Kui kõik on korras, peaks väljundis olema `sudo`.

Näide:

```text
kasutaja : kasutaja sudo
```

---

## 8. Testi sudo õigusi

Testimiseks kasuta lihtsat käsku:

```bash
sudo whoami
```

Kui kõik töötab, küsitakse parooli ja väljund peaks olema:

```text
root
```

See tähendab, et kasutaja saab käske käivitada administraatori õigustes.

---

## 9. Kui sudo käsk ei tööta

### Viga: user is not in the sudoers file

See tähendab, et kasutaja ei ole `sudo` grupis.

Lahendus: logi sisse kasutajaga, kellel on juba sudo õigused, ja käivita:

```bash
sudo usermod -aG sudo kasutaja
```

---

### Viga: Permission denied

See tähendab, et käsu käivitamiseks pole piisavalt õigusi.

Lahendus:

```bash
sudo käsk
```

Näide:

```bash
sudo nano /etc/hosts
```

---

### Viga: sudo: command not found

See tähendab, et `sudo` ei ole süsteemi paigaldatud.

Debian/Ubuntu põhises süsteemis saab selle paigaldada root kasutajana:

```bash
su -
apt update
apt install sudo -y
```

Seejärel lisa kasutaja sudo gruppi:

```bash
usermod -aG sudo kasutaja
```

---

## 10. Näidis algusest lõpuni

Näide: Linuxi kasutaja nimi on `kasutaja`.

Kontrolli kasutajat:

```bash
whoami
```

Kontrolli gruppe:

```bash
groups kasutaja
```

Lisa kasutaja sudo gruppi:

```bash
sudo usermod -aG sudo kasutaja
```

Logi välja:

```bash
exit
```

Logi uuesti sisse ja kontrolli:

```bash
groups
```

Testi sudo õigusi:

```bash
sudo whoami
```

Kui väljund on:

```text
root
```

siis kasutaja on edukalt sudo gruppi lisatud.

---

## 11. Kõige lühem spikker

```bash
groups kasutaja
sudo usermod -aG sudo kasutaja
exit
groups kasutaja
sudo whoami
```

Kui `sudo whoami` vastab:

```text
root
```

siis sudo õigused töötavad.
