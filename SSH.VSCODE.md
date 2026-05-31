# VS Code Remote-SSH seadistuse kokkuvõte

## Eesmärk

Eesmärk oli seadistada Windows UI masinast VS Code kaudu SSH ühendus kolme Linuxi masinasse:

| Masin | IP-aadress | Hosti nimi VS Code'is |
|---|---:|---|
| Debian | 10.0.24.10 | debian |
| Ubuntu | 10.0.24.11 | ubuntu |
| AlmaLinux | 10.0.24.12 | alma |

Oluline lõpptulemus:

- Windows PowerShellist SSH ühendus töötab kõigisse kolme masinasse.
- VS Code terminalist saab SSH-ga masinatesse sisse.
- Ubuntu Remote-SSH ühendus näitas olekut `connected`.
- AlmaLinuxis tekkis VS Code Serveri paigaldamisel viga `UnpackFailed`.

---

## 1. SSH config fail Windowsis

Faili asukoht:

```text
C:\Users\Kasutaja\.ssh\config
```

Sisu:

```sshconfig
Host debian
    HostName 10.0.24.10
    User kasutaja
    Port 22
    PreferredAuthentications password
    PubkeyAuthentication no

Host ubuntu
    HostName 10.0.24.11
    User kasutaja
    Port 22
    PreferredAuthentications password
    PubkeyAuthentication no

Host alma
    HostName 10.0.24.12
    User kasutaja
    Port 22
    PreferredAuthentications password
    PubkeyAuthentication no
```

### Milleks seda tegime?

Selle failiga määrasime VS Code'ile ja Windowsi SSH kliendile lühinimed:

```powershell
ssh debian
ssh ubuntu
ssh alma
```

Ilma selle failita peaks iga kord kirjutama:

```powershell
ssh kasutaja@10.0.24.10
ssh kasutaja@10.0.24.11
ssh kasutaja@10.0.24.12
```

`HostName` on päris IP-aadress.  
`Host` on lihtsalt lühinimi, mida kasutad VS Code'is või PowerShellis.

---

## 2. VS Code settings.json fail

Faili asukoht:

```text
C:\Users\Kasutaja\AppData\Roaming\Code\User\settings.json
```

Sisu:

```json
{
    "remote.SSH.remotePlatform": {
        "debian": "linux",
        "ubuntu": "linux",
        "alma": "linux"
    },
    "remote.SSH.path": "C:\\Windows\\System32\\OpenSSH\\ssh.exe",
    "remote.SSH.useLocalServer": false,
    "remote.SSH.showLoginTerminal": true
}
```

### Milleks seda tegime?

| Rida | Selgitus |
|---|---|
| `remote.SSH.remotePlatform` | Ütleb VS Code'ile, et `debian`, `ubuntu` ja `alma` on Linuxi masinad. |
| `remote.SSH.path` | Määrab täpse Windowsi SSH programmi asukoha. |
| `remote.SSH.useLocalServer` | Lülitasime välja, sest Remote-SSH jäi ühendamisel kinni. |
| `remote.SSH.showLoginTerminal` | Näitab parooli küsimist ja ühenduse logi terminalis. |

---

## 3. Kontrollisime SSH olemasolu Windowsis

PowerShellis kontrollisime:

```powershell
where.exe ssh
```

Tulemus oli:

```text
C:\Windows\System32\OpenSSH\ssh.exe
```

See tähendab, et Windowsi OpenSSH klient on olemas.

---

## 4. Kontrollisime SSH ühendust PowerShellist

PowerShellis töötasid ühendused IP kaudu:

```powershell
ssh kasutaja@10.0.24.10
ssh kasutaja@10.0.24.11
ssh kasutaja@10.0.24.12
```

See tähendab:

- Linuxi masinad on võrgus kättesaadavad.
- SSH server töötab.
- Kasutaja/parool toimivad.
- Probleem ei ole Linuxi SSH seadistuses.

---

## 5. Kontrollisime VS Code terminalist SSH-d

VS Code terminalist sai samuti SSH-ga sisse logida.

See tähendab, et ka VS Code sees töötab tavaline SSH käsurealt.

Näide:

```powershell
ssh kasutaja@10.0.24.11
```

---

## 6. Ubuntu seis

VS Code Remote Explorer näitas:

```text
ubuntu connected
```

See tähendab, et Ubuntu Remote-SSH ühendus töötas.

Kui VS Code terminal avaneb ja käsuga:

```bash
hostname
```

tuleb Ubuntu masina nimi, siis oled Ubuntu masinas sees.

Kontrollkäsud Ubuntu sees:

```bash
hostname
ip a
pwd
whoami
```

---

## 7. Debianiga tehtud kontroll

Debiani puhul proovis VS Code Remote-SSH serverit käivitada.

Logis oli näha midagi sellist:

```text
Found existing installation at /home/kasutaja/.vscode-server
Starting VS Code CLI...
listeningOn==127.0.0.1:42711==
osReleaseId==debian==
arch==x86_64==
platform==linux==
end
```

See näitab, et VS Code proovis Debianis serveripoolset osa käivitada.

Kui Debian jääb ühendamisel kinni, siis saab Debianis puhastada vana VS Code serveri:

```bash
rm -rf ~/.vscode-server
rm -rf ~/.vscode-remote
```

Seejärel VS Code'is:

```text
Remote-SSH: Connect to Host → debian
```

Soovitus: kasutada `Connect in New Window`.

---

## 8. AlmaLinuxi probleem

AlmaLinuxi puhul tuli viga:

```text
UnpackFailed
Failed to install the VS Code Server
```

See tähendab:

- SSH ühendus ise töötab.
- VS Code saab Almasse sisse.
- Probleem tekib VS Code Serveri lahtipakkimisel või paigaldamisel AlmaLinuxis.

---

## 9. AlmaLinuxi paranduskäsud

Logi PowerShellist Almasse:

```powershell
ssh kasutaja@10.0.24.12
```

Seejärel AlmaLinuxis:

```bash
rm -rf ~/.vscode-server
rm -rf ~/.vscode-remote
```

Kontrolli kettaruumi:

```bash
df -h
```

Kui ruumi on, paigalda vajalikud paketid:

```bash
sudo dnf install tar gzip glibc libstdc++ -y
```

Loo VS Code serveri kaust uuesti:

```bash
mkdir -p ~/.vscode-server
chmod 700 ~/.vscode-server
```

Seejärel proovi VS Code'is uuesti:

```text
Remote-SSH: Connect to Host → alma
```

või Remote Explorerist:

```text
alma → Connect in New Window
```

---

## 10. Kui Remote-SSH ei tööta, aga SSH töötab

Kui ülesandes ei ole otseselt kirjas, et peab kasutama VS Code Remote-SSH failipuud, siis võib kasutada VS Code terminalist tavalist SSH-d.

Näited:

```powershell
ssh kasutaja@10.0.24.10
ssh kasutaja@10.0.24.11
ssh kasutaja@10.0.24.12
```

või kui `.ssh/config` on olemas:

```powershell
ssh debian
ssh ubuntu
ssh alma
```

See on piisav, et:

- käske käivitada;
- teenuseid seadistada;
- võrku kontrollida;
- faile muuta terminalipõhiselt;
- Git käske kasutada.

---

## 11. Remote-SSH eelis

Remote-SSH on mugavam, sest:

- VS Code failipuu näitab Linuxi masina faile;
- faile saab avada ja salvestada graafiliselt;
- saab kasutada otsingut;
- saab korraga mitut faili avada;
- Git ja terminal on mugavamalt samas kohas.

Aga kui Remote-SSH jonnib ja ülesandes seda otseselt ei nõuta, siis ei tasu sellele liiga palju aega kulutada. SSH PowerShellist või VS Code terminalist on täiesti toimiv varuplaan.

---

## 12. Lühike lõppseis

| Kontroll | Tulemus |
|---|---|
| Windows `ssh.exe` olemas | Jah |
| PowerShellist SSH Debianisse | Töötab |
| PowerShellist SSH Ubuntusse | Töötab |
| PowerShellist SSH Almasse | Töötab |
| VS Code terminalist SSH | Töötab |
| VS Code Remote-SSH Ubuntu | Ühendatud |
| VS Code Remote-SSH Debian | Vajab vajadusel puhastust/uus ühendus |
| VS Code Remote-SSH Alma | Viga `UnpackFailed`, vaja paigaldada/puhastada paketid |

---

## 13. Kõige olulisem meeles pidada

Kui PowerShellist töötab:

```powershell
ssh kasutaja@IP
```

siis Linuxi SSH pool on korras.

Kui VS Code Remote-SSH ei tööta, on probleem VS Code Remote-SSH serveri paigalduses või laienduse seadistuses, mitte põhivõrgus ega SSH teenuses.
