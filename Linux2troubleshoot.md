# Troubleshoot ja kontrollnimekiri

## 1. Ansible probleemid

### Probleem: ansible all -m ping annab UNREACHABLE

Kontrolli SSH ühendust:

```bash
ssh kasutaja@almaserver
```

või

```bash
ssh kasutaja@debianserver
```

Kui SSH ei tööta, ei tööta ka Ansible.

---

### Probleem: Permission denied (publickey)

Kontrolli, kas SSH võti on kopeeritud:

```bash
ssh-copy-id kasutaja@almaserver
ssh-copy-id kasutaja@debianserver
```

Kontroll:

```bash
ls ~/.ssh
```

Peab sisaldama:

```text
id_rsa
id_rsa.pub
```

---

### Probleem: sudo käsud ei tööta Ansible kaudu

Kontrolli:

```yaml
become: yes
```

Playbookis peab see olemas olema.

---

### Probleem: hkhk kasutajat ei loodud

Kontroll Alma või Debiani peal:

```bash
id hkhk
```

Kui kasutajat ei eksisteeri:

```bash
ansible-playbook create_hkhk.yml
```

käivita uuesti.

---

## 2. Logiserveri probleemid

### Probleem: rsyslog ei käivitu

Kontroll:

```bash
sudo systemctl status rsyslog
```

Veateated:

```bash
sudo journalctl -xeu rsyslog
```

---

### Probleem: port 514 ei kuula

Kontroll:

```bash
sudo ss -tulpn | grep 514
```

Peab näitama:

```text
514/tcp
514/udp
```

Kui ei näita:

Kontrolli faili:

```bash
sudo nano /etc/rsyslog.conf
```

Kas read on kommenteerimata:

```text
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```

---

### Probleem: logid ei jõua serverisse

Klientmasinas:

```bash
logger TESTLOG
```

Serveris:

```bash
grep TESTLOG /var/log/remote/*/*
```

Kui ei leia:

Kontrolli:

```bash
ping logger.sinunimi.local
```

---

### Probleem: DNS nimi logger.sinunimi.local ei tööta

Kontroll:

```bash
nslookup logger.sinunimi.local
```

või

```bash
host logger.sinunimi.local
```

Kui vastust ei tule:

- DNS kirje puudub
- vale IP
- klient kasutab valet DNS serverit

---

### Probleem: tulemüür blokeerib logid

Kontroll:

```bash
sudo ufw status
```

Peavad olema lubatud:

```text
22/tcp
514/tcp
514/udp
```

Lisa vajadusel:

```bash
sudo ufw allow 514/tcp
sudo ufw allow 514/udp
```

---

## 3. Hostname ja DNS

### Probleem: hostname ei muutunud

Kontroll:

```bash
hostnamectl
```

Muuda:

```bash
sudo hostnamectl set-hostname logger
```

---

### Probleem: vana hostname kuvatakse endiselt

Kontroll:

```bash
cat /etc/hosts
```

Vajadusel muuda:

```bash
sudo nano /etc/hosts
```

---

## 4. GRUB taastamine

### Probleem: ainult must ekraan

Proovi GRUB menüüd:

BIOS:

```text
Shift
```

UEFI:

```text
Esc
```

---

### Probleem: Recovery Mode puudub

Käivita Ubuntu ISO pealt:

```text
Try Ubuntu
```

Ava terminal.

Leia partitsioon:

```bash
sudo fdisk -l
```

Mount:

```bash
sudo mount /dev/sda2 /mnt
```

Seejärel chroot.

---

### Probleem: grub-install annab vea

Kontrolli ketast:

```bash
lsblk
```

Näiteks:

```text
sda
sda1
sda2
```

Paigalda kettale:

```bash
sudo grub-install /dev/sda
```

MITTE:

```bash
sudo grub-install /dev/sda1
```

---

### Probleem: süsteem käivitub GRUB-i, aga mitte Ubuntu-sse

Uuenda menüü:

```bash
sudo update-grub
```

---

## 5. Kõvaketta vormindamine

### Probleem: ei tea milline ketas on uus

Kontroll:

```bash
lsblk
```

Enne ketta lisamist tee pilt.

Pärast ketta lisamist:

```bash
lsblk
```

Uus seade on tavaliselt:

```text
sdb
```

või

```text
sdc
```

---

### Probleem: mkfs.ext4 ütleb "device busy"

Kontroll:

```bash
mount
```

Kui on mountitud:

```bash
sudo umount /dev/sdb1
```

---

### Probleem: mount kaob pärast restarti

Kontroll:

```bash
cat /etc/fstab
```

Test:

```bash
sudo mount -a
```

Kui veateadet ei tule, on korras.

---

## 6. Varukoopiad

### Probleem: rsync ei kopeeri midagi

Kontroll:

```bash
ls /home/ulemus
```

Kas kaust on olemas.

---

### Probleem: rsync ütleb Permission denied

Kasuta:

```bash
sudo rsync -avh /home/ulemus/ /backup/ulemus_backup/
```

---

### Probleem: backup kaust tühi

Kontroll:

```bash
ls -la /backup/ulemus_backup
```

---

## 7. Crontab

### Probleem: cron ei käivitu

Kontroll:

```bash
systemctl status cron
```

Kui ei tööta:

```bash
sudo systemctl enable --now cron
```

---

### Probleem: cron ei käivita skripti

Kas skript on käivitatav?

```bash
ls -l backup.sh
```

Peab sisaldama:

```text
x
```

Kui ei:

```bash
chmod +x backup.sh
```

---

### Probleem: cron töötab käsitsi, aga mitte automaatselt

Kasuta absoluutseid teid.

Vale:

```bash
rsync -avh home/ulemus backup/
```

Õige:

```bash
rsync -avh /home/ulemus/ /backup/
```

---

## 8. Eksami lõppkontroll

### Ansible

```bash
ansible all -m ping
```

Kõik peavad vastama:

```text
SUCCESS
```

---

### hkhk kasutaja

```bash
id hkhk
```

Peab sisaldama:

```text
sudo
```

---

### Logiserver

```bash
sudo ss -tulpn | grep 514
```

Peab näitama:

```text
514/tcp
514/udp
```

---

### DNS

```bash
nslookup logger.sinunimi.local
```

Peab lahenduma õigeks IP-ks.

---

### Logid

Klient:

```bash
logger TEST
```

Server:

```bash
grep TEST /var/log/remote/*/*
```

---

### Ketas

```bash
df -h
```

Peab näitama:

```text
/backup
```

---

### Varukoopia

```bash
ls /backup
```

Peab sisaldama kasutaja andmeid.

---

### Crontab

```bash
crontab -l
```

Peab näitama varundamise kirjet.

---

### GRUB

Masin peab pärast restarti käivituma ilma veateadeteta.

```bash
sudo reboot
```

Kui süsteem käivitub normaalselt, on ülesanne edukalt lahendatud.
