# Linuxi üldosa lahenduskäik

See juhend kirjeldab Linuxi piletite **üldosa**, mis tuleb teha enne konkreetse Linuxi pileti eriosa lahendamist.

Linuxi üldosa eesmärk on valmistada ette Linuxi serverite keskne haldus Ansible abil ning seadistada turvaline SSH ligipääs.

---

## 1. Üldosa eesmärk

Kõik Linuxi piletid sisaldavad esimese ülesandena Ansible kasutamist UbuntuServeris.

Üldosa lõpuks peab olema tehtud:

| Ülesande osa | Mida tuleb teha | Milleks seda vaja on |
|---|---|---|
| Ansible juhtmasin | Paigalda UbuntuServerisse Ansible | Et hallata Linuxi masinaid ühest kohast |
| Inventory | Lisa inventory faili AlmaServer ja DebianServer | Et Ansible teaks, milliseid masinaid hallata |
| Playbook | Loo Ansible playbook | Et automatiseerida kasutaja, sudo ja SSH seadistamine |
| Kasutaja `hkhk` | Loo AlmaServerisse ja DebianServerisse kasutaja `hkhk` | Ühine halduskasutaja |
| Sudo õigused | Lisa `hkhk` sudo õigustega gruppi | Et kasutaja saaks teha administraatori tegevusi |
| SSH võti | Kopeeri SSH avalik võti sihtserveritesse | Et sisselogimine toimuks võtmega |
| SSH turvamine | Keela parooliga SSH sisselogimine | Et serveritesse pääseks ainult SSH võtmega |
| Root SSH keeld | Keela root kasutajana SSH sisselogimine | Turvalisuse suurendamiseks |
| Tulemüür | Luba ainult vajalikud pordid | Et serveritel ei oleks liigseid avatud teenuseid |
| Kontroll | Testi Ansible ühendust | Et veenduda, et baasosa töötab |

---

# 2. Vajalikud paketid ja teenused

## 2.1 UbuntuServer ehk Ansible juhtmasin

UbuntuServerisse on vaja paigaldada:

| Pakett | Milleks vajalik |
|---|---|
| `ansible` | Linuxi masinate keskseks haldamiseks |
| `openssh-client` | SSH ühenduste loomiseks |
| `sshpass` | Vajadusel ajutiseks parooliga Ansible testimiseks |
| `python3` | Ansible tööks vajalik |
| `ufw` | Tulemüüri seadistamiseks |

Paigalda vajalikud paketid:

```bash
sudo apt update
sudo apt install ansible openssh-client sshpass python3 ufw -y
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Uuendab paketiloendi ja paigaldab Ansible ning vajalikud tööriistad | Paigaldus lõpeb veata |

Kontrolli Ansible paigaldust:

```bash
ansible --version
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse Ansible versioon | Kui tuleb `command not found`, ei ole Ansible paigaldatud |

---

## 2.2 AlmaServer ja DebianServer ehk hallatavad masinad

Hallatavates masinates peab olema:

| Pakett / teenus | Milleks vajalik |
|---|---|
| `openssh-server` | Et serverisse saaks SSH-ga sisse |
| `python3` | Et Ansible moodulid töötaksid |
| `sudo` | Et `hkhk` kasutaja saaks administraatori õiguseid kasutada |
| `ufw` või `firewalld` | Tulemüüri jaoks |

Debian/Ubuntu põhises masinas:

```bash
sudo apt update
sudo apt install openssh-server python3 sudo ufw -y
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab SSH serveri, Python3, sudo ja UFW | SSH teenus töötab ja masinat saab hallata |

AlmaLinuxis:

```bash
sudo dnf install openssh-server python3 sudo firewalld -y
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab SSH serveri, Python3, sudo ja tulemüüri | SSH teenus töötab ja masinat saab hallata |

Kontrolli SSH teenust Debian/Ubuntu masinas:

```bash
systemctl status ssh
```

Kontrolli SSH teenust AlmaLinuxis:

```bash
systemctl status sshd
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Teenus on `active (running)` | Kui teenus ei tööta, käivita `sudo systemctl enable --now ssh` või `sudo systemctl enable --now sshd` |

---

# 3. Serverite algkontroll

## 3.1 Kontrolli kasutajat

```bash
whoami
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Näitab, millise kasutajaga oled sisse logitud | Kuvatakse praeguse kasutaja nimi |

Kui oled vale kasutajaga sees, logi õige kasutajaga uuesti sisse.

---

## 3.2 Kontrolli hostname’i

```bash
hostname
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Näitab serveri nime | Kuvatakse masina nimi, näiteks `UbuntuServer`, `AlmaServer` või `DebianServer` |

Vajadusel muuda hostname:

```bash
sudo hostnamectl set-hostname UUS-NIMI
```

Näide:

```bash
sudo hostnamectl set-hostname AlmaServer
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Määrab serverile uue nime | Pärast kontrolli kuvab `hostname` uue nime |

Kontroll:

```bash
hostname
```

---

## 3.3 Kontrolli IP-aadressi

```bash
ip a
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Näitab võrguliideseid ja IP-aadresse | Serveril on korrektne IP-aadress |

Kontrolli gateway’d:

```bash
ip route
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Näitab marsruute | Olemas on `default via ...` rida |

Kui `default via` puudub, ei pruugi server saada teistesse võrkudesse ega internetti.

---

# 4. SSH võtme loomine UbuntuServeris

## 4.1 Loo SSH võti

UbuntuServeris:

```bash
ssh-keygen -t ed25519
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob SSH võtmepaari | Tekivad failid `~/.ssh/id_ed25519` ja `~/.ssh/id_ed25519.pub` |

Kui küsib asukohta, vajuta Enter.

Kui küsib parooli, võib eksami lihtsustamiseks vajutada Enter ehk jätta võtme paroolita.

Kontroll:

```bash
ls -lah ~/.ssh/
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `id_ed25519` ja `id_ed25519.pub` faile | Kui faile pole, loo võti uuesti |

---

# 5. Ansible inventory loomine

## 5.1 Loo inventory fail

UbuntuServeris:

```bash
nano inventory.ini
```

Lisa:

```ini
[linux]
almaserver ansible_host=ALMASERVER_IP ansible_user=hkhk
debianserver ansible_host=DEBIANSERVER_IP ansible_user=hkhk

[alma]
almaserver ansible_host=ALMASERVER_IP ansible_user=hkhk

[debian]
debianserver ansible_host=DEBIANSERVER_IP ansible_user=hkhk
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob Ansible inventory faili | Failis on AlmaServer ja DebianServer õigete IP-aadressidega |

Asenda:

```text
ALMASERVER_IP
DEBIANSERVER_IP
```

oma tegelike IP-aadressidega.

Kontroll:

```bash
cat inventory.ini
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed AlmaServeri ja DebianServeri kirjeid | Kui IP või kasutaja on vale, paranda fail |

---

# 6. Ajutine ühenduse test parooliga

Seda sammu võib kasutada enne SSH võtme kopeerimist.

```bash
ansible all -i inventory.ini -m ping --ask-pass
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Testib, kas Ansible saab serveritesse parooliga SSH kaudu sisse | Masinad vastavad `pong` |

Kui tuleb SSH error:

| Probleem | Lahendus |
|---|---|
| Vale IP | Kontrolli sihtserveris `ip a` |
| Vale kasutaja | Kontrolli, kas kasutaja on olemas |
| SSH ei tööta | Kontrolli `systemctl status ssh` või `sshd` |
| Server ei vasta | Kontrolli võrku ja pingimist |
| Python puudub | Paigalda sihtserverisse `python3` |

---

# 7. Ansible playbook üldosa tegemiseks

## 7.1 Loo playbook

UbuntuServeris:

```bash
nano linux_uldosa.yml
```

Lisa:

```yaml
---
- name: Linuxi üldosa seadistamine
  hosts: linux
  become: yes

  vars:
    admin_user: hkhk
    ssh_public_key: "{{ lookup('file', lookup('env','HOME') + '/.ssh/id_ed25519.pub') }}"

  tasks:
    - name: Veendu, et vajalikud paketid on Debian/Ubuntu masinas olemas
      apt:
        name:
          - sudo
          - python3
          - openssh-server
          - ufw
        state: present
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Veendu, et vajalikud paketid on Alma/RHEL masinas olemas
      dnf:
        name:
          - sudo
          - python3
          - openssh-server
          - firewalld
        state: present
      when: ansible_os_family == "RedHat"

    - name: Loo kasutaja hkhk
      user:
        name: "{{ admin_user }}"
        shell: /bin/bash
        create_home: yes
        state: present

    - name: Lisa hkhk Debian/Ubuntu sudo gruppi
      user:
        name: "{{ admin_user }}"
        groups: sudo
        append: yes
      when: ansible_os_family == "Debian"

    - name: Lisa hkhk Alma/RHEL wheel gruppi
      user:
        name: "{{ admin_user }}"
        groups: wheel
        append: yes
      when: ansible_os_family == "RedHat"

    - name: Lisa SSH avalik võti kasutajale hkhk
      authorized_key:
        user: "{{ admin_user }}"
        key: "{{ ssh_public_key }}"
        state: present

    - name: Keela SSH parooliga sisselogimine
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
        backup: yes

    - name: Luba SSH võtmega sisselogimine
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PubkeyAuthentication'
        line: 'PubkeyAuthentication yes'
        backup: yes

    - name: Keela root kasutaja SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
        backup: yes

    - name: Kontrolli SSH konfiguratsiooni
      command: sshd -t
      changed_when: false

    - name: Taaskäivita SSH teenus Debian/Ubuntu masinas
      service:
        name: ssh
        state: restarted
        enabled: yes
      when: ansible_os_family == "Debian"

    - name: Taaskäivita SSH teenus Alma/RHEL masinas
      service:
        name: sshd
        state: restarted
        enabled: yes
      when: ansible_os_family == "RedHat"
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob playbooki, mis teeb üldosa automaatselt | Fail salvestub ja on valmis käivitamiseks |

---

## 7.2 Käivita playbook

Kui sihtserverites on veel parooliga SSH lubatud:

```bash
ansible-playbook -i inventory.ini linux_uldosa.yml --ask-pass --ask-become-pass
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Käivitab üldosa playbooki ja küsib SSH/sudo parooli | Playbook lõpeb veata, `failed=0` |

Kui SSH võtmega ligipääs juba töötab:

```bash
ansible-playbook -i inventory.ini linux_uldosa.yml --ask-become-pass
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Käivitab playbooki SSH võtmega | Playbook lõpeb veata |

Kui tulemus on `failed`, loe errorit. Tavaliselt on põhjus üks neist:

| Viga | Lahendus |
|---|---|
| SSH ühendus ei tööta | Kontrolli IP-d, kasutajat ja SSH teenust |
| sudo parool vale | Sisesta õige parool või kontrolli sudo õiguseid |
| Python puudub | Paigalda sihtserverisse `python3` |
| SSH konfiguratsiooni viga | Ava sihtserveris `/etc/ssh/sshd_config` ja paranda rida |

---

# 8. SSH võtmega ligipääsu kontroll

UbuntuServerist testi AlmaServerit:

```bash
ssh hkhk@ALMASERVER_IP
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Ühendub AlmaServerisse kasutajana `hkhk` | Sisse saab SSH võtmega |

Välju:

```bash
exit
```

Testi DebianServerit:

```bash
ssh hkhk@DEBIANSERVER_IP
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Ühendub DebianServerisse kasutajana `hkhk` | Sisse saab SSH võtmega |

Välju:

```bash
exit
```

---

# 9. Parooliga SSH sisselogimise kontroll

Testi, et parooliga SSH enam ei töötaks:

```bash
ssh -o PubkeyAuthentication=no hkhk@SERVER_IP
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Proovib sisse logida ilma SSH võtmeta | Sisselogimine peab ebaõnnestuma |

Kui parooliga saab endiselt sisse, kontrolli sihtserveris:

```bash
sudo grep -i "PasswordAuthentication" /etc/ssh/sshd_config
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `PasswordAuthentication no` | Kui on `yes`, muuda `no` peale ja taaskäivita SSH |

---

# 10. Root SSH sisselogimise kontroll

Kontrolli sihtserveris:

```bash
sudo grep -i "PermitRootLogin" /etc/ssh/sshd_config
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| `PermitRootLogin no` | Kui on `yes`, muuda `no` peale ja taaskäivita SSH |

---

# 11. Ansible ühenduse lõppkontroll

UbuntuServeris:

```bash
ansible all -i inventory.ini -m ping
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Kontrollib Ansible ühendust SSH võtmega | Mõlemad masinad vastavad `pong` |

Näide:

```text
almaserver | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

debianserver | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Kontrolli hostname’e:

```bash
ansible all -i inventory.ini -m command -a "hostname"
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Käivitab sihtserverites `hostname` käsu | Kuvatakse AlmaServeri ja DebianServeri nimed |

Kontrolli sudo õiguseid:

```bash
ansible all -i inventory.ini -m command -a "whoami" -b --ask-become-pass
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Käivitab käsu sihtserverites administraatori õigustes | Väljund on `root` |

Kui väljund ei ole `root`, siis `hkhk` kasutajal pole sudo/wheel õiguseid.

---

# 12. Tulemüüri üldpõhimõte

Linuxi üldnõue ütleb, et serverites tohib olla avatud ainult vajalikud pordid.

Üldosas peab vähemalt SSH olema lubatud.

## Debian/Ubuntu UFW näide

```bash
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status numbered
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lubab SSH pordi ja aktiveerib tulemüüri | `22/tcp` on lubatud |

## AlmaLinux firewalld näide

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --add-service=ssh --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Käivitab firewalld ja lubab SSH teenuse | SSH on lubatud teenuste nimekirjas |

Piletite eriosades tuleb juurde lubada ainult vastava teenuse pordid, näiteks:

| Teenus | Port |
|---|---|
| HTTP | 80/tcp |
| HTTPS | 443/tcp |
| Syslog | 514/tcp või 514/udp |
| Zabbix agent | 10050/tcp |
| Zabbix server | 10051/tcp |
| DHCP | 67/udp |
| NFS | 2049/tcp |
| SMB | 445/tcp |

---

# 13. Üldosa lõpptulemus

Linuxi üldosa lõpuks peab olema selline seis:

| Kontrollitav asi | Lõpptulemus |
|---|---|
| UbuntuServer | Ansible on paigaldatud |
| Inventory | AlmaServer ja DebianServer on inventory failis |
| Playbook | Üldosa playbook on loodud ja käivitatud |
| Kasutaja `hkhk` | Olemas AlmaServeris ja DebianServeris |
| Sudo õigused | `hkhk` kuulub Debianis `sudo` gruppi ja Almas `wheel` gruppi |
| SSH võti | UbuntuServerist saab SSH võtmega sisse AlmaServerisse ja DebianServerisse |
| Parooliga SSH | Parooliga SSH sisselogimine on keelatud |
| Root SSH | Root kasutajana SSH sisselogimine on keelatud |
| Ansible test | `ansible all -m ping` annab `pong` |
| Ansible sudo test | Ansible saab käske käivitada root õigustes |
| Tulemüür | Lubatud on ainult vajalikud pordid |

---

# 14. Väike lõppkontroll

## UbuntuServeris

```bash
ansible --version
```

Oodatav tulemus:

```text
Ansible versioon kuvatakse
```

```bash
cat inventory.ini
```

Oodatav tulemus:

```text
AlmaServer ja DebianServer on kirjas õigete IP-aadressidega
```

```bash
ansible all -i inventory.ini -m ping
```

Oodatav tulemus:

```text
Kõik masinad vastavad pong
```

```bash
ansible all -i inventory.ini -m command -a "hostname"
```

Oodatav tulemus:

```text
Kuvatakse hallatavate masinate nimed
```

```bash
ansible all -i inventory.ini -m command -a "whoami" -b --ask-become-pass
```

Oodatav tulemus:

```text
Väljund on root
```

---

## AlmaServeris ja DebianServeris

```bash
id hkhk
```

Oodatav tulemus:

```text
Kasutaja hkhk on olemas
```

```bash
groups hkhk
```

Oodatav Debian/Ubuntu puhul:

```text
hkhk kuulub sudo gruppi
```

Oodatav AlmaLinuxi puhul:

```text
hkhk kuulub wheel gruppi
```

```bash
sudo sshd -t
```

Oodatav tulemus:

```text
Väljund puudub ehk SSH konfiguratsioon on korras
```

Debian/Ubuntu:

```bash
systemctl status ssh
```

AlmaLinux:

```bash
systemctl status sshd
```

Oodatav tulemus:

```text
SSH teenus on active (running)
```

Kontrolli SSH turvaseadeid:

```bash
sudo grep -Ei "PasswordAuthentication|PubkeyAuthentication|PermitRootLogin" /etc/ssh/sshd_config
```

Oodatav tulemus:

```text
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

---

# 15. Kokkuvõte

Linuxi üldosa eesmärk on teha valmis turvaline ja toimiv halduskeskkond.

Lühidalt peab üldosa lõpuks olema:

```text
UbuntuServer = Ansible juhtmasin
AlmaServer + DebianServer = hallatavad masinad
hkhk kasutaja = olemas ja sudo/wheel õigustega
SSH võtmed = töötavad
Parooliga SSH = keelatud
Root SSH = keelatud
Ansible ping = pong
Ansible sudo test = root
Tulemüür = lubatud ainult vajalikud pordid
```

Kui see osa töötab, saab edasi liikuda konkreetse Linuxi pileti eriosa juurde:

```text
Pilet 1 = WordPress, Debian upgrade, Vaultwarden
Pilet 2 = logiserver, GRUB, ketas ja backup
Pilet 3 = monitooring ja failserver
Pilet 4 = PHP veebirakendus ja MariaDB
Pilet 5 = DHCP server ja DebianPilet5 parandamine
Pilet 6 = NFS, iSCSI ja APT paketihaldur
```
