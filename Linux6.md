# Linux pilet 6 lahenduskäik

See juhend kirjeldab ainult **Linux pilet 6 eriosa**.

Siin ei korrata Linuxi üldosa:

- Ansible paigaldus UbuntuServerisse
- Ansible inventory loomine
- `hkhk` kasutaja loomine
- sudo/wheel gruppi lisamine
- SSH võtmega ligipääs AlmaServerisse ja DebianServerisse
- SSH parooliga sisselogimise keelamine

---

## 1. Linux pilet 6 eesmärk

Linux pilet 6 põhiteemad on:

| Teema | Mida tuleb teha | Milleks seda vaja on |
|---|---|---|
| NFS teenus | Paigaldada AlmaServerisse NFS server | Linuxi masinatele jagatud võrgukausta pakkumiseks |
| iSCSI teenus | Paigaldada AlmaServerisse iSCSI target teenus | Plokiseadme jagamiseks serveritele üle võrgu |
| Kettad | Vormindada kaks lisatud kõvaketast | Üks NFS jaoks ja teine iSCSI jaoks |
| NFS jagamine | Seadistada NFS share kõigile vajalikele serveritele | Et serverid saaksid võrgukausta kasutada |
| iSCSI targetid | Seadistada iSCSI targetid DC1, DebianServer ja UbuntuServer jaoks | Et serverid saaksid iSCSI ketta külge ühendada |
| Püsiv ühendamine | Ühendada NFS ja iSCSI püsivalt klientmasinates | Et ühendused jääksid tööle ka pärast restarti |
| APT taastamine | Leida UbuntuPilet6 masinas puuduvad paketid ja need taastada | Et paketihaldus ja vajalikud tööriistad töötaksid |
| AD domeen | Liita UbuntuPilet6 Windows AD domeeni | Et Ubuntu masin oleks domeenis hallatav |
| Dokumentatsioon | Kirjeldada seadistus ja kontrollid | Et tõendada lahenduse toimimist |

---

# 2. Soovitatav rollijaotus

| Masin | Roll |
|---|---|
| UbuntuServer | Ansible juhtmasin, iSCSI klient |
| AlmaServer | NFS server ja iSCSI target server |
| DebianServer | NFS/iSCSI klient |
| DC1 | Windows Server, iSCSI klient |
| UbuntuPilet6 | APT parandamine ja AD domeeniga liitmine |

---

# 3. Oluline märkus iSCSI kohta

iSCSI jagab **plokiseadet**, mitte tavalist failijagamist.

See tähendab:

```text
Sama iSCSI ketast ei tohiks mitmes masinas korraga tavalise ext4/NTFS failisüsteemiga read-write kasutada.
```

Miks?

```text
Kui mitu serverit kirjutavad samale iSCSI LUN-ile ilma klasterfailisüsteemita,
võib failisüsteem katki minna.
```

Turvalisem eksamil:

```text
Loo igale kliendile eraldi iSCSI LUN:
- DC1 jaoks eraldi LUN
- DebianServer jaoks eraldi LUN
- UbuntuServer jaoks eraldi LUN
```

Kui ülesandes nõutakse ühte ühist iSCSI sihtmärki, dokumenteeri:

```text
iSCSI target on loodud ja ühendatav nõutud masinatest,
kuid sama LUN-i korraga mitmes masinas write-režiimis kasutamine ei ole tavapärase failisüsteemiga ohutu.
```

---

# 4. Vajalikud teenused ja paketid

## 4.1 AlmaServer

AlmaServerisse on vaja:

| Pakett / teenus | Milleks |
|---|---|
| `nfs-utils` | NFS serveri jaoks |
| `targetcli` | iSCSI targeti seadistamiseks |
| `firewalld` | Tulemüüri haldamiseks |
| `parted` või `fdisk` | Ketaste partitsioneerimiseks |
| `xfsprogs` või `e2fsprogs` | Failisüsteemi loomiseks |

Paigaldus AlmaServeris:

```bash
sudo dnf install nfs-utils targetcli firewalld parted xfsprogs e2fsprogs -y
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab NFS, iSCSI ja ketaste halduse tööriistad | Paigaldus lõpeb veata |

---

## 4.2 DebianServer ja UbuntuServer

Linux klientidesse on vaja:

| Pakett | Milleks |
|---|---|
| `nfs-common` | NFS share ühendamiseks |
| `open-iscsi` | iSCSI targeti ühendamiseks |

Debian/Ubuntu:

```bash
sudo apt update
sudo apt install nfs-common open-iscsi -y
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab NFS ja iSCSI klienditööriistad | Kliendid saavad NFS/iSCSI teenuseid kasutada |

---

## 4.3 UbuntuPilet6 AD domeeniga liitumiseks

UbuntuPilet6 masinasse on vaja:

| Pakett | Milleks |
|---|---|
| `realmd` | Domeeni avastamiseks ja liitumiseks |
| `sssd` | AD kasutajate autentimiseks |
| `sssd-tools` | SSSD haldamiseks |
| `adcli` | AD domeeniga liitumiseks |
| `krb5-user` | Kerberose autentimiseks |
| `samba-common-bin` | AD/Samba tööriistad |
| `packagekit` | realm tööks vajalik |
| `oddjob` / `oddjob-mkhomedir` | Kodukaustade automaatseks loomiseks |

Ubuntu/Debian:

```bash
sudo apt update
sudo apt install realmd sssd sssd-tools adcli krb5-user samba-common-bin packagekit oddjob oddjob-mkhomedir -y
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab AD domeeniga liitumiseks vajalikud paketid | `realm` käsk töötab |

---

# 5. AlmaServeri algkontroll

## 5.1 Kontrolli hostname’i

```bash
hostname
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse `AlmaServer` või muu loogiline nimi | Kui nimi on vale, dokumenteeri või muuda käsuga `sudo hostnamectl set-hostname AlmaServer` |

---

## 5.2 Kontrolli IP-aadressi

```bash
ip a
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Näitab võrguliideseid ja IP-aadresse | AlmaServeril on IP-aadress |

Kontrolli gateway’d:

```bash
ip route
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed `default via ...` rida | Kui puudub, kontrolli võrgu seadistust |

---

## 5.3 Kontrolli DNS-i

```bash
resolvectl status
```

või AlmaLinuxis:

```bash
cat /etc/resolv.conf
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| DNS serverid on õiged | Kui masin peab domeeniga suhtlema, peab DNS olema DC1/DC2, mitte 1.1.1.1 või 8.8.8.8 |

---

# 6. Kontrolli lisatud kõvakettaid AlmaServeris

## 6.1 Vaata kettaid

```bash
lsblk
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed süsteemiketast ja kahte lisaketast, näiteks `/dev/sdb` ja `/dev/sdc` | Kui lisakettaid ei näe, kontrolli Proxmoxis/VM-is, kas kettad on masinale lisatud |

Kontrolli failisüsteeme:

```bash
lsblk -f
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Näitab, kas ketastel on failisüsteemid | Lisaketastel pole veel mountpointi või failisüsteemi |

Näidis:

```text
/dev/sda  süsteemiketas
/dev/sdb  NFS ketas
/dev/sdc  iSCSI ketas
```

---

# 7. Vorminda NFS ketas

Näites kasutatakse:

```text
/dev/sdb = NFS ketas
```

Kontrolli enne vormindamist väga hoolikalt:

```bash
lsblk
```

Ära vorminda süsteemiketast.

---

## 7.1 Loo partitsioon NFS kettale

```bash
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart primary xfs 0% 100%
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob `/dev/sdb` kettale GPT partitsioonitabeli ja ühe partitsiooni | Tekib `/dev/sdb1` |

Kontroll:

```bash
lsblk
```

Oodatav:

```text
/dev/sdb1 on olemas
```

---

## 7.2 Vorminda NFS partitsioon

```bash
sudo mkfs.xfs -f /dev/sdb1
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob XFS failisüsteemi NFS kettale | Vormindamine lõpeb veata |

Alternatiiv ext4-ga:

```bash
sudo mkfs.ext4 -F /dev/sdb1
```

---

## 7.3 Loo mountpoint ja ühenda ketas

```bash
sudo mkdir -p /srv/nfs/yhine
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob NFS jagamise kausta | Kaust on olemas |

Leia UUID:

```bash
sudo blkid /dev/sdb1
```

Näide:

```text
UUID="abcd-1234" TYPE="xfs"
```

Ava fstab:

```bash
sudo nano /etc/fstab
```

Lisa rida:

```text
UUID=SIIN-SINU-UUID /srv/nfs/yhine xfs defaults 0 0
```

Kui kasutasid ext4:

```text
UUID=SIIN-SINU-UUID /srv/nfs/yhine ext4 defaults 0 2
```

Rakenda:

```bash
sudo mount -a
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna viga | Kui tuleb error, kontrolli UUID-d, failisüsteemi tüüpi ja mountpointi |

Kontroll:

```bash
df -h
```

Oodatav:

```text
/dev/sdb1 on mountitud /srv/nfs/yhine alla
```

---

# 8. Seadista NFS server AlmaServeris

## 8.1 Paigalda ja käivita NFS

```bash
sudo dnf install nfs-utils -y
```

```bash
sudo systemctl enable --now nfs-server
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab ja käivitab NFS serveri | Teenus töötab |

Kontroll:

```bash
systemctl status nfs-server
```

Oodatav:

```text
active (running)
```

---

## 8.2 Seadista NFS kausta õigused

Lihtne eksamilahendus:

```bash
sudo chown -R nobody:nobody /srv/nfs/yhine
sudo chmod -R 0777 /srv/nfs/yhine
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Annab NFS testimiseks laiad õigused | Kliendid saavad kausta kirjutada |

Turvalisem variant oleks kasutada kindlat gruppi ja õiguseid, kuid eksamil on oluline, et jagamine töötaks ja oleks dokumenteeritud.

---

## 8.3 Lisa NFS export

Ava exports fail:

```bash
sudo nano /etc/exports
```

Lisa näiteks oma sisevõrgule:

```text
/srv/nfs/yhine 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```

Asenda `10.0.0.0/24` oma tegeliku võrguga.

Kui sinu võrk on näiteks `10.31.0.0/24`, siis:

```text
/srv/nfs/yhine 10.31.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```

| Valik | Tähendus |
|---|---|
| `rw` | lubab lugeda ja kirjutada |
| `sync` | kirjutab andmed turvalisemalt kettale |
| `no_subtree_check` | vähendab jagatud alamkausta kontrolliprobleeme |
| `no_root_squash` | lubab root õiguseid kliendist; mugav labis, aga päriselus ettevaatlikult |

Rakenda exportid:

```bash
sudo exportfs -ra
```

Kontrolli:

```bash
sudo exportfs -v
```

Oodatav:

```text
/srv/nfs/yhine on välja jagatud sinu võrgule
```

---

## 8.4 Ava tulemüüris NFS teenused

AlmaServeris:

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --add-service=nfs --permanent
sudo firewall-cmd --add-service=mountd --permanent
sudo firewall-cmd --add-service=rpc-bind --permanent
sudo firewall-cmd --reload
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lubab NFS jaoks vajalikud teenused tulemüüris | Kliendid saavad NFS share’i mountida |

Kontroll:

```bash
sudo firewall-cmd --list-services
```

Oodatav:

```text
nfs mountd rpc-bind
```

---

# 9. Ühenda NFS DebianServeris ja UbuntuServeris

Tee see samm Linux klientides, näiteks DebianServeris ja UbuntuServeris.

---

## 9.1 Paigalda NFS klient

```bash
sudo apt update
sudo apt install nfs-common -y
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab NFS kliendi | `mount -t nfs` töötab |

---

## 9.2 Kontrolli NFS share’i nähtavust

```bash
showmount -e ALMASERVER_IP
```

Näide:

```bash
showmount -e 10.0.0.20
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse `/srv/nfs/yhine` | Kui ei kuvata, kontrolli AlmaServeri tulemüüri, NFS teenust ja `/etc/exports` faili |

---

## 9.3 Mounti NFS käsitsi testiks

```bash
sudo mkdir -p /mnt/nfs-yhine
sudo mount -t nfs ALMASERVER_IP:/srv/nfs/yhine /mnt/nfs-yhine
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Ühendab NFS share’i kliendi kausta alla | Kaust avaneb kliendis |

Kontroll:

```bash
df -h
```

Oodatav:

```text
ALMASERVER_IP:/srv/nfs/yhine on mountitud /mnt/nfs-yhine alla
```

Testi kirjutamist:

```bash
echo "NFS test $(hostname)" | sudo tee /mnt/nfs-yhine/test-$(hostname).txt
```

Oodatav:

```text
Fail tekib NFS jagatud kausta
```

---

## 9.4 Tee NFS ühendus püsivaks

Ava fstab:

```bash
sudo nano /etc/fstab
```

Lisa:

```text
ALMASERVER_IP:/srv/nfs/yhine /mnt/nfs-yhine nfs defaults,_netdev 0 0
```

Testi:

```bash
sudo mount -a
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Käsk ei anna viga | Kui tuleb timeout või permission denied, kontrolli NFS serverit ja võrku |

Kontroll pärast restarti:

```bash
sudo reboot
```

Pärast restarti:

```bash
df -h | grep nfs
```

Oodatav:

```text
NFS share on automaatselt ühendatud
```

---

# 10. Seadista iSCSI ketas AlmaServeris

Näites kasutatakse:

```text
/dev/sdc = iSCSI ketas
```

iSCSI puhul on parem mitte vormindada kogu ketast NFS moodi, vaid kasutada seda kas:

```text
- terve plokiseadmena
- partitsioonina
- LVM logical volume’idena
- failipõhiste backstore’idena
```

Lihtne ja turvaline eksamilahendus:

```text
Kasuta iSCSI jaoks LVM-i ja tee igale kliendile eraldi logical volume.
```

---

## 10.1 Paigalda LVM ja targetcli

```bash
sudo dnf install lvm2 targetcli -y
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab LVM ja iSCSI targeti tööriistad | `targetcli` töötab |

---

## 10.2 Loo LVM iSCSI kettale

Kontrolli enne:

```bash
lsblk
```

Loo physical volume:

```bash
sudo pvcreate /dev/sdc
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| PV luuakse `/dev/sdc` kettale | Kui ketas on vale või kasutuses, kontrolli `lsblk` |

Loo volume group:

```bash
sudo vgcreate vg_iscsi /dev/sdc
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob iSCSI jaoks LVM volume group’i | `vg_iscsi` on olemas |

Loo kolm LUN-i:

```bash
sudo lvcreate -L 5G -n lv_dc1 vg_iscsi
sudo lvcreate -L 5G -n lv_debian vg_iscsi
sudo lvcreate -L 5G -n lv_ubuntu vg_iscsi
```

Kui ketas on väiksem, kasuta väiksemaid mahtusid, näiteks `1G`.

Kontroll:

```bash
sudo lvs
```

Oodatav:

```text
lv_dc1, lv_debian ja lv_ubuntu on olemas
```

---

# 11. Loo iSCSI targetid AlmaServeris

## 11.1 Ava targetcli

```bash
sudo targetcli
```

Oodatav:

```text
Avaneb targetcli prompt
```

---

## 11.2 Loo backstore’id

targetcli sees:

```text
/backstores/block create name=dc1_lun dev=/dev/vg_iscsi/lv_dc1
/backstores/block create name=debian_lun dev=/dev/vg_iscsi/lv_debian
/backstores/block create name=ubuntu_lun dev=/dev/vg_iscsi/lv_ubuntu
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Seob LVM logical volume’id iSCSI backstore’ideks | Backstore’id tekivad targetcli alla |

Kontroll targetcli sees:

```text
/backstores/block ls
```

---

## 11.3 Loo iSCSI target

IQN näide:

```text
iqn.2026-06.local.sinuNimi:almaserver.storage
```

targetcli sees:

```text
/iscsi create iqn.2026-06.local.sinuNimi:almaserver.storage
```

Asenda `sinuNimi` enda nime/domeeniga.

Kontroll:

```text
/iscsi ls
```

---

## 11.4 Loo LUN-id targeti alla

Mine targeti TPG alla:

```text
cd /iscsi/iqn.2026-06.local.sinuNimi:almaserver.storage/tpg1
```

Lisa LUN-id:

```text
/luns create /backstores/block/dc1_lun
/luns create /backstores/block/debian_lun
/luns create /backstores/block/ubuntu_lun
```

Kontroll:

```text
ls
```

Oodatav:

```text
LUN 0, LUN 1 ja LUN 2 on olemas
```

---

## 11.5 Luba demo mode ehk lihtne ligipääs

Eksamil lihtsuse mõttes võib lubada discovery/auth ilma ACL-ita:

```text
set attribute authentication=0
set attribute generate_node_acls=1
set attribute demo_mode_write_protect=0
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lubab klientidel targetit avastada ja kasutada lihtsama seadistusega | iSCSI ühendamine on lihtsam |

Turvalisem variant oleks luua iga initiatori IQN järgi ACL, aga eksamil on lihtsam demo mode.

---

## 11.6 Salvesta konfiguratsioon

targetcli sees:

```text
saveconfig
exit
```

Kontroll:

```bash
sudo targetcli ls
```

Oodatav:

```text
Target, backstore’id ja LUN-id on alles
```

---

## 11.7 Käivita ja luba target teenus

```bash
sudo systemctl enable --now target
```

Kontroll:

```bash
systemctl status target
```

Oodatav:

```text
active (running)
```

---

## 11.8 Ava tulemüüris iSCSI port

iSCSI port:

```text
3260/tcp
```

AlmaServeris:

```bash
sudo firewall-cmd --add-port=3260/tcp --permanent
sudo firewall-cmd --reload
```

Kontroll:

```bash
sudo firewall-cmd --list-ports
```

Oodatav:

```text
3260/tcp
```

Kontrolli, kas teenus kuulab:

```bash
sudo ss -tulpen | grep 3260
```

Oodatav:

```text
target teenus kuulab pordil 3260
```

---

# 12. Ühenda iSCSI DebianServeris

## 12.1 Paigalda iSCSI klient

DebianServeris:

```bash
sudo apt update
sudo apt install open-iscsi -y
```

Käivita teenus:

```bash
sudo systemctl enable --now iscsid open-iscsi
```

Kontroll:

```bash
systemctl status iscsid
```

Oodatav:

```text
active (running)
```

---

## 12.2 Avasta iSCSI target

```bash
sudo iscsiadm -m discovery -t sendtargets -p ALMASERVER_IP
```

Näide:

```bash
sudo iscsiadm -m discovery -t sendtargets -p 10.0.0.20
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse AlmaServeri IQN | Kui ei kuvata, kontrolli porti 3260, tulemüüri ja target teenust |

---

## 12.3 Logi targetisse

```bash
sudo iscsiadm -m node --login
```

Oodatav:

```text
Login to [iface: default, target: iqn..., portal: ...] successful
```

Kontrolli ketast:

```bash
lsblk
```

Oodatav:

```text
Ilmub uus ketas, näiteks /dev/sdb
```

---

## 12.4 Vorminda ja mounti Debiani iSCSI LUN

Kui Debianile mõeldud iSCSI LUN ilmus näiteks `/dev/sdb`:

```bash
sudo mkfs.ext4 /dev/sdb
```

Loo mountpoint:

```bash
sudo mkdir -p /mnt/iscsi
```

Leia UUID:

```bash
sudo blkid /dev/sdb
```

Ava fstab:

```bash
sudo nano /etc/fstab
```

Lisa:

```text
UUID=SIIN-SINU-UUID /mnt/iscsi ext4 _netdev,nofail 0 2
```

Testi:

```bash
sudo mount -a
df -h
```

Oodatav:

```text
iSCSI ketas on mountitud /mnt/iscsi alla
```

---

## 12.5 Tee iSCSI login püsivaks

```bash
sudo iscsiadm -m node -o update -n node.startup -v automatic
```

Kontroll:

```bash
sudo iscsiadm -m node
```

Restarti test:

```bash
sudo reboot
```

Pärast restarti:

```bash
lsblk
df -h | grep iscsi
```

Oodatav:

```text
iSCSI ketas tuleb automaatselt külge
```

---

# 13. Ühenda iSCSI UbuntuServeris

UbuntuServeris tee samad sammud:

```bash
sudo apt update
sudo apt install open-iscsi -y
sudo systemctl enable --now iscsid open-iscsi
sudo iscsiadm -m discovery -t sendtargets -p ALMASERVER_IP
sudo iscsiadm -m node --login
lsblk
```

Seejärel vorminda UbuntuServerile mõeldud LUN.

Näide, kui uus ketas on `/dev/sdb`:

```bash
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /mnt/iscsi
sudo blkid /dev/sdb
sudo nano /etc/fstab
```

Lisa:

```text
UUID=SIIN-SINU-UUID /mnt/iscsi ext4 _netdev,nofail 0 2
```

Püsiv login:

```bash
sudo iscsiadm -m node -o update -n node.startup -v automatic
```

Test:

```bash
sudo mount -a
df -h
```

---

# 14. Ühenda iSCSI DC1 Windows Serveris

## 14.1 Ava iSCSI Initiator

DC1-s:

```text
Start
→ iSCSI Initiator
```

Kui küsib teenuse käivitamist, vali:

```text
Yes
```

---

## 14.2 Lisa target portal

iSCSI Initiatoris:

```text
Discovery
→ Discover Portal
```

Sisesta AlmaServeri IP:

```text
ALMASERVER_IP
```

Port:

```text
3260
```

---

## 14.3 Ühenda target

Ava:

```text
Targets
```

Vali target:

```text
iqn.2026-06.local.sinuNimi:almaserver.storage
```

Vajuta:

```text
Connect
```

Märgi:

```text
Add this connection to the list of Favorite Targets
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Ühendab iSCSI targeti Windows Serverisse | Target on Connected |

---

## 14.4 Ava Disk Management

```text
Server Manager
→ Tools
→ Computer Management
→ Disk Management
```

või:

```text
diskmgmt.msc
```

Uus ketas peaks ilmuma.

Tee:

```text
Online
Initialize Disk
GPT
New Simple Volume
NTFS
Drive letter näiteks S:
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Valmistab iSCSI ketta Windowsis kasutamiseks | DC1 näeb uut ketast |

Kontroll PowerShellis:

```powershell
Get-Disk
Get-Volume
```

---

# 15. UbuntuPilet6 APT pakettide parandamine

## 15.1 Kontrolli, kas apt töötab

UbuntuPilet6 masinas:

```bash
sudo apt update
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Paketiloend uuendatakse | Kui tuleb DNS/repo error, kontrolli võrku, DNS-i ja `/etc/apt/sources.list` faili |

Kontrolli katkiseid pakette:

```bash
sudo apt --fix-broken install
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Parandab katkised sõltuvused | Käsk lõpeb veata |

Kontrolli poolelijäänud dpkg seadistusi:

```bash
sudo dpkg --configure -a
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Seadistab pooleli jäänud paketid | Käsk lõpeb veata |

---

## 15.2 Kontrolli APT allikaid

```bash
cat /etc/apt/sources.list
```

või uuemates Ubuntu versioonides:

```bash
ls /etc/apt/sources.list.d/
```

Kui allikad on katki, ava:

```bash
sudo nano /etc/apt/sources.list
```

Tüüpiline Ubuntu 22.04 näidis:

```text
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-security main restricted universe multiverse
```

Pärast parandust:

```bash
sudo apt update
```

---

## 15.3 Leia puuduvad vajalikud paketid

Kontrolli domeeniga liitumiseks vajalikke käske:

```bash
which realm
which sssd
which adcli
which kinit
```

| Käsk | Kui puudub |
|---|---|
| `realm` | paigalda `realmd` |
| `sssd` | paigalda `sssd` |
| `adcli` | paigalda `adcli` |
| `kinit` | paigalda `krb5-user` |

Paigalda vajalikud paketid:

```bash
sudo apt install realmd sssd sssd-tools adcli krb5-user samba-common-bin packagekit oddjob oddjob-mkhomedir -y
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab domeeniga liitumiseks vajalikud paketid | Kõik vajalikud käsud on olemas |

Kontroll:

```bash
realm --version
sssd --version
adcli --version
```

---

# 16. Valmista UbuntuPilet6 AD domeeniga liitumiseks

## 16.1 DNS peab olema DC1/DC2

UbuntuPilet6 masinas kontrolli DNS-i:

```bash
resolvectl status
```

või:

```bash
cat /etc/resolv.conf
```

Oodatav:

```text
DNS serverid on DC1 ja DC2 IP-aadressid
```

Oluline:

```text
AD domeeniga liitumisel ei tohi DNS olla 1.1.1.1 ega 8.8.8.8.
UbuntuPilet6 peab kasutama domeeni DNS servereid ehk DC1/DC2.
```

Kui kasutad netplani:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Näide:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses:
        - 10.x.x.50/24
      routes:
        - to: default
          via: 10.x.x.1
      nameservers:
        addresses:
          - DC1_IP
          - DC2_IP
        search:
          - sinuNimi.local
```

Rakenda:

```bash
sudo netplan apply
```

Kontrolli nimelahendust:

```bash
nslookup sinuNimi.local
nslookup DC1.sinuNimi.local
```

Oodatav:

```text
Domeen ja DC1 lahenduvad õigeks IP-ks
```

---

## 16.2 Kontrolli aega

AD/Kerberos vajab õiget kellaaega.

```bash
timedatectl
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kellaaeg on DC-dega sarnane | Kui aeg on vale, seadista NTP |

Luba NTP:

```bash
sudo timedatectl set-ntp true
```

---

## 16.3 Avasta domeen

```bash
realm discover sinuNimi.local
```

Näide:

```bash
realm discover ekristal.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse domeeni info ja vajalikud paketid | Kui ei leia domeeni, kontrolli DNS-i ja ühendust DC1/DC2-ga |

---

# 17. Liida UbuntuPilet6 AD domeeni

## 17.1 Liitu domeeniga

```bash
sudo realm join sinuNimi.local -U Haldur
```

või:

```bash
sudo realm join sinuNimi.local -U Administrator
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Liidab UbuntuPilet6 masina AD domeeni | Käsk lõpeb veata ja küsib domeeni kasutaja parooli |

Kontroll:

```bash
realm list
```

Oodatav:

```text
sinuNimi.local domeen kuvatakse
configured: kerberos-member
```

---

## 17.2 Kontrolli AD kasutajat

```bash
id 'kasutaja@sinuNimi.local'
```

Näide:

```bash
id 'testkasutaja@ekristal.local'
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse AD kasutaja UID/GID info | Kui kasutajat ei leita, kontrolli SSSD teenust ja domeeni liitumist |

Kontrolli SSSD-d:

```bash
systemctl status sssd
```

Oodatav:

```text
active (running)
```

---

## 17.3 Luba kodukaustade automaatne loomine

Ubuntu puhul:

```bash
sudo pam-auth-update
```

Vali:

```text
Create home directory on login
```

Kui `pam-auth-update` ei ole mugav, lisa:

```bash
sudo pam-auth-update --enable mkhomedir
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob AD kasutajale esimesel sisselogimisel kodukausta | Domeeni kasutaja saab sisse logides `/home/...` kausta |

---

## 17.4 Luba domeeni adminidele sudo õigused

Ava sudoers drop-in:

```bash
sudo visudo -f /etc/sudoers.d/domain-admins
```

Lisa:

```text
%domain\ admins@sinuNimi.local ALL=(ALL) ALL
```

Kui grupinimi ei tööta tühiku tõttu, kontrolli grupi täpset nime:

```bash
getent group | grep -i "domain"
```

Alternatiiv kasutada konkreetset AD gruppi, näiteks:

```text
%linux-admins@sinuNimi.local ALL=(ALL) ALL
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Annab AD administraatorigrupile sudo õiguse UbuntuPilet6 masinas | Domeeni admin saab sudo käske kasutada |

---

# 18. Kontrolli domeeni sisselogimist UbuntuPilet6 masinas

## 18.1 SSH või lokaalne login

Proovi:

```bash
su - 'kasutaja@sinuNimi.local'
```

või SSH-ga:

```bash
ssh 'kasutaja@sinuNimi.local'@UBUNTUPILET6_IP
```

Oodatav:

```text
Domeeni kasutajaga saab sisse logida
```

Kontroll:

```bash
whoami
id
pwd
```

Oodatav:

```text
Kasutaja on AD kasutaja ja kodukaust on loodud
```

---

# 19. Lõppkontroll

| Kontroll | Käsk / koht | Oodatav tulemus |
|---|---|---|
| Alma NFS teenus | `systemctl status nfs-server` | active |
| Alma NFS export | `exportfs -v` | `/srv/nfs/yhine` jagatud |
| Alma NFS tulemüür | `firewall-cmd --list-services` | nfs, mountd, rpc-bind |
| NFS klient | `showmount -e ALMASERVER_IP` | NFS share kuvatakse |
| NFS mount | `df -h` | NFS ühendatud |
| iSCSI target | `systemctl status target` | active |
| iSCSI port | `ss -tulpen \| grep 3260` | port 3260 kuulab |
| targetcli | `targetcli ls` | target ja LUN-id olemas |
| Linux iSCSI klient | `iscsiadm -m node --login` | login successful |
| Linux iSCSI ketas | `lsblk` | uus ketas olemas |
| Windows iSCSI | iSCSI Initiator | target connected |
| UbuntuPilet6 apt | `sudo apt update` | töötab |
| UbuntuPilet6 paketid | `which realm adcli kinit` | käsud olemas |
| Domeeni avastamine | `realm discover sinuNimi.local` | domeen leitakse |
| Domeeniga liitumine | `realm list` | domeen kuvatakse |
| AD kasutaja | `id kasutaja@sinuNimi.local` | kasutaja info kuvatakse |

---

# 20. Dokumentatsiooni näidis

```markdown
## Linux pilet 6 dokumentatsioon

### Eesmärk

Eesmärk oli seadistada AlmaServerisse NFS ja iSCSI teenused, vormindada kaks eraldi kõvaketast NFS ja iSCSI jaoks, teha NFS jagamine serveritele kättesaadavaks, seadistada iSCSI targetid DC1, DebianServer ja UbuntuServer jaoks ning ühendada need püsivalt. Lisaks tuli UbuntuPilet6 masinas taastada puuduvad APT paketid ja liita UbuntuPilet6 Windows AD domeeni.

### Kasutatud masinad

| Masin | Roll |
|---|---|
| AlmaServer | NFS server ja iSCSI target server |
| DebianServer | NFS ja iSCSI klient |
| UbuntuServer | NFS ja iSCSI klient |
| DC1 | Windows iSCSI klient ja AD domeenikontroller |
| UbuntuPilet6 | APT parandamine ja AD domeeniga liitmine |

### Tehtud seadistused

- Kontrollisin AlmaServeri võrguühendust ja lisatud kettaid.
- Vormindasin ühe lisaketta NFS teenuse jaoks.
- Ühendasin NFS ketta püsivalt `/srv/nfs/yhine` alla.
- Paigaldasin AlmaServerisse `nfs-utils`.
- Seadistasin NFS exporti `/etc/exports` failis.
- Avasin tulemüüris NFS teenused.
- Testisin NFS share’i DebianServerist ja UbuntuServerist.
- Vormindasin teise lisaketta iSCSI jaoks LVM-ina.
- Lõin iSCSI jaoks logical volume’id.
- Paigaldasin ja seadistasin `targetcli`.
- Lõin iSCSI targeti ja LUN-id.
- Avasin tulemüüris pordi 3260/tcp.
- Ühendasin iSCSI targeti DebianServeris ja UbuntuServeris.
- Ühendasin iSCSI targeti DC1 Windows Serveris iSCSI Initiatoriga.
- Seadistasin iSCSI ühendused püsivaks.
- Kontrollisin UbuntuPilet6 APT paketihaldust.
- Taastasin puuduvad paketid domeeniga liitumiseks.
- Seadistasin UbuntuPilet6 DNS-i kasutama DC1/DC2 servereid.
- Liitsin UbuntuPilet6 masina AD domeeni.
- Testisin domeeni kasutajaga sisselogimist.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `systemctl status nfs-server` | NFS server töötab |
| `exportfs -v` | NFS share on välja jagatud |
| `showmount -e ALMASERVER_IP` | Kliendid näevad NFS share’i |
| `df -h` | NFS on kliendis mountitud |
| `systemctl status target` | iSCSI target töötab |
| `targetcli ls` | iSCSI target ja LUN-id olemas |
| `ss -tulpen \| grep 3260` | iSCSI port kuulab |
| `iscsiadm -m discovery` | Linux klient leiab targeti |
| `lsblk` | iSCSI ketas ilmub kliendis |
| DC1 iSCSI Initiator | Target on Connected |
| `sudo apt update` | UbuntuPilet6 APT töötab |
| `realm discover` | AD domeen leitakse |
| `realm list` | UbuntuPilet6 on domeenis |
| `id kasutaja@sinuNimi.local` | AD kasutaja info kuvatakse |

### Kokkuvõte

Linux pilet 6 tulemusena töötab AlmaServer NFS ja iSCSI serverina. Üks lisaketas on kasutusel NFS jagatud kaustana ning teine ketas on jagatud iSCSI LUN-idena. DebianServer, UbuntuServer ja DC1 saavad iSCSI targetiga ühenduda. UbuntuPilet6 masinas on APT paketihaldus taastatud, vajalikud paketid paigaldatud ning masin on liidetud Windows AD domeeniga.
```

---

# 21. Troubleshooting

## NFS share ei ole kliendis nähtav

Kontrolli AlmaServeris:

```bash
sudo exportfs -v
systemctl status nfs-server
sudo firewall-cmd --list-services
```

Kliendis:

```bash
showmount -e ALMASERVER_IP
```

Kui ei näe:

| Põhjus | Lahendus |
|---|---|
| NFS teenus ei tööta | `sudo systemctl enable --now nfs-server` |
| `/etc/exports` viga | Paranda võrk ja õigused |
| exportfs rakendamata | `sudo exportfs -ra` |
| tulemüür blokeerib | luba `nfs`, `mountd`, `rpc-bind` |

---

## NFS mount annab permission denied

Kontrolli `/etc/exports`:

```bash
cat /etc/exports
```

Kontrolli õiguseid:

```bash
ls -ld /srv/nfs/yhine
```

Lihtne labiparandus:

```bash
sudo chmod -R 0777 /srv/nfs/yhine
sudo exportfs -ra
```

---

## iSCSI discovery ei leia targetit

Kontrolli AlmaServeris:

```bash
systemctl status target
sudo targetcli ls
sudo ss -tulpen | grep 3260
sudo firewall-cmd --list-ports
```

Kui port puudub:

```bash
sudo firewall-cmd --add-port=3260/tcp --permanent
sudo firewall-cmd --reload
```

Kliendis:

```bash
ping ALMASERVER_IP
sudo iscsiadm -m discovery -t sendtargets -p ALMASERVER_IP
```

---

## iSCSI ketas ei ilmu pärast login’it

Kliendis:

```bash
sudo iscsiadm -m node --login
lsblk
dmesg | tail
```

Kui ikka ei ilmu:

| Põhjus | Lahendus |
|---|---|
| LUN puudub targetis | Kontrolli `targetcli ls` |
| ACL/auth probleem | Luba demo mode või loo õige ACL |
| tulemüür blokeerib | Ava 3260/tcp |
| vale target | tee discovery uuesti |

---

## iSCSI ei tule pärast restarti tagasi

Linux kliendis:

```bash
sudo iscsiadm -m node -o update -n node.startup -v automatic
sudo systemctl enable --now iscsid open-iscsi
```

fstab reas peab olema:

```text
_netdev,nofail
```

Näide:

```text
UUID=... /mnt/iscsi ext4 _netdev,nofail 0 2
```

---

## UbuntuPilet6 apt update ei tööta

Kontrolli võrku:

```bash
ping -c 4 8.8.8.8
ping -c 4 archive.ubuntu.com
```

Kui IP ping töötab, aga nimi mitte:

```text
DNS probleem.
```

Kontrolli:

```bash
cat /etc/resolv.conf
resolvectl status
```

Kui repo allikad on katki:

```bash
sudo nano /etc/apt/sources.list
sudo apt update
```

Paranda katkised paketid:

```bash
sudo dpkg --configure -a
sudo apt --fix-broken install
```

---

## UbuntuPilet6 ei leia AD domeeni

Kontrolli DNS-i:

```bash
nslookup sinuNimi.local
nslookup DC1.sinuNimi.local
```

Kui ei lahendu:

```text
DNS peab olema DC1/DC2 IP.
```

Kontrolli aega:

```bash
timedatectl
```

Kui kell erineb DC-st liiga palju:

```bash
sudo timedatectl set-ntp true
```

Kontrolli domeeni avastamist:

```bash
realm discover sinuNimi.local
```

---

## realm join ebaõnnestub

Kontrolli paketid:

```bash
which realm
which adcli
which kinit
```

Kontrolli Kerberost:

```bash
kinit Haldur@sinuNimi.local
```

Mõnikord peab realm olema suurtähtedega:

```bash
kinit Haldur@SINUNIMI.LOCAL
```

Kui Kerberos töötab, proovi uuesti:

```bash
sudo realm join sinuNimi.local -U Haldur
```

---

# 22. Kõige lühem spikker

```text
1. Kontrolli AlmaServeri lisakettaid: lsblk
2. Vali üks ketas NFS jaoks ja teine iSCSI jaoks
3. Vorminda NFS ketas
4. Mounti NFS ketas /srv/nfs/yhine alla
5. Paigalda nfs-utils
6. Seadista /etc/exports
7. Käivita nfs-server
8. Ava tulemüüris nfs, mountd, rpc-bind
9. Testi NFS klientidest showmount ja mount
10. Tee NFS mount püsivaks /etc/fstab failis
11. Paigalda targetcli ja lvm2
12. Loo iSCSI kettale LVM PV, VG ja LV-d
13. Loo targetcli backstore’id
14. Loo iSCSI target ja LUN-id
15. Ava tulemüüris 3260/tcp
16. DebianServeris ja UbuntuServeris paigalda open-iscsi
17. Tee iscsiadm discovery ja login
18. Vorminda Linux klientides iSCSI ketas ja lisa fstab
19. DC1-s ava iSCSI Initiator ja ühenda target
20. UbuntuPilet6 masinas paranda apt
21. Paigalda realmd, sssd, adcli, krb5-user jne
22. Seadista DNS DC1/DC2 peale
23. realm discover sinuNimi.local
24. realm join sinuNimi.local -U Haldur
25. Kontrolli realm list ja id kasutaja@sinuNimi.local
26. Dokumenteeri
```
