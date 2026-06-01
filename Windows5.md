# Windows pilet 5 lahenduskäik

See juhend kirjeldab ainult **Windows pilet 5 eriosa**.

Siin ei korrata Windowsi üldosa:

- DC1 ja DC2
- AD domeen `sinuNimi.local`
- DNS
- DHCP ja DHCP failover
- OU-d `Kasutajad` ja `Arvutid`
- kasutaja `Haldur`
- Windows 11 klient domeenis
- CSV kasutajate import
- baas-GPO-d

Windows pilet 5 põhiülesanne on **WDS ehk Windows Deployment Services** seadistamine ning Windows 11 paigaldamine üle võrgu PXE boot’i kaudu. Ülesandes on kirjas, et tuleb paigaldada WDS roll, laadida WDS-i Windows 10 Enterprise ja Windows 11 paigaldusmeedia failid, teha DHCP serveris WDS-i jaoks vajalikud seadistused ning paigaldada testmasinasse üle võrgu Windows 11. :contentReference[oaicite:0]{index=0}

---

## 1. Windows pilet 5 eesmärk

| Teema | Mida tuleb teha | Milleks seda vaja on |
|---|---|---|
| WDS roll | Paigaldada Windows Deployment Services | Windowsi paigaldamiseks üle võrgu |
| WDS algseadistus | Luua WDS serveri kaust ja seadistada PXE vastamine | Et kliendid saaksid võrgust bootida |
| Boot image | Lisada WDS-i `boot.wim` | Windows PE käivitamiseks PXE kaudu |
| Install image | Lisada WDS-i `install.wim` või `install.esd` | Windowsi päris paigaldamiseks |
| Windows 10 Enterprise image | Lisada WDS-i Windows 10 Enterprise paigaldusfailid | Ülesande nõue |
| Windows 11 image | Lisada WDS-i Windows 11 paigaldusfailid | Testmasina paigaldamiseks |
| DHCP seadistus | Kontrollida DHCP ja PXE toimimist | Et klient leiaks WDS serveri |
| Testmasin | Seadistada võrgust alglaadimine | Et Windows 11 paigalduks WDS-i kaudu |
| Testimine | Kontrollida PXE boot’i ja Windows 11 paigaldust | Et tõendada lahenduse toimimist |

---

# 2. Vajalikud rollid ja failid

## 2.1 Serveri rollid

WDS serveris on vaja:

| Roll / teenus | Milleks |
|---|---|
| Windows Deployment Services | PXE boot ja Windowsi paigaldus üle võrgu |
| Deployment Server | WDS põhiteenus |
| Transport Server | Võrgupaigalduse andmeedastus |
| DHCP Server | IP-aadresside jagamine PXE klientidele |
| DNS Server | Domeeni nimelahendus |
| AD DS | Kui WDS töötab domeeniga integreeritult |

Soovitus eksamil:

```text
Kui eraldi serverit pole nõutud, võib WDS rolli paigaldada DC1 peale.
Kui eraldi Windows Server on olemas, on parem kasutada eraldi WDS serverit.
```

---

## 2.2 Vajalikud ISO failid

Sul peab olema kättesaadav:

| ISO / paigaldusmeedia | Milleks |
|---|---|
| Windows 10 Enterprise ISO | Ülesande järgi tuleb lisada WDS-i |
| Windows 11 ISO | Testmasina paigaldamiseks WDS-i kaudu |

ISO sees on olulised failid:

```text
\sources\boot.wim
\sources\install.wim
```

või mõnikord:

```text
\sources\install.esd
```

Oluline:

```text
boot.wim = käivitab Windows Setup / Windows PE keskkonna
install.wim või install.esd = sisaldab Windowsi paigaldatavaid versioone
```

---

# 3. Kontroll enne WDS seadistamist

## 3.1 Kontrolli serveri IP-d ja DNS-i

WDS serveris PowerShellis:

```powershell
ipconfig /all
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Näitab serveri IP, DNS ja võrguinfot | Serveril on staatiline IP ja DNS serverid on DC1/DC2 |

Oluline:

```text
WDS server peab olema samas võrgus või ruutimise kaudu PXE klientidele kättesaadav.
DNS ei tohi domeenimasinatel olla 1.1.1.1 ega 8.8.8.8.
```

---

## 3.2 Kontrolli DHCP scope’i

DC1 PowerShellis:

```powershell
Get-DhcpServerv4Scope
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| DHCP scope on olemas ja aktiivne | Kui scope puudub, tee esmalt Windowsi üldosa DHCP seadistus korda |

Kontrolli DHCP lease’e:

```powershell
Get-DhcpServerv4Lease -ScopeId 10.x.x.0
```

Asenda `10.x.x.0` oma scope ID-ga.

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Klientidele jagatakse IP-aadresse | Kui lease’e pole, kontrolli DHCP teenust ja kliendi võrku |

---

## 3.3 Kontrolli, kas WDS server on domeenis

```powershell
whoami
```

Oodatav domeenikontoga:

```text
sinuNimi\Haldur
```

Kontrolli domeeni:

```powershell
Get-ADDomain
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Domeeni info kuvatakse | Kui käsk ei tööta, pole AD moodul või domeeniühendus korras |

---

# 4. Paigalda WDS roll

## 4.1 Paigaldus Server Manageriga

GUI serveris:

```text
Server Manager
→ Manage
→ Add Roles and Features
→ Role-based or feature-based installation
→ Select server
→ Windows Deployment Services
→ Add Features
```

Vali rolliteenused:

```text
Deployment Server
Transport Server
```

Seejärel:

```text
Install
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab WDS rolli | Server Manageris on WDS roll olemas |

---

## 4.2 PowerShelli alternatiiv

```powershell
Install-WindowsFeature WDS -IncludeManagementTools
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab Windows Deployment Services rolli ja haldustööriistad | Käsk lõpeb edukalt |

Kontroll:

```powershell
Get-WindowsFeature WDS
```

Oodatav:

```text
Install State = Installed
```

---

# 5. WDS algseadistus

## 5.1 Ava WDS konsool

```text
Server Manager
→ Tools
→ Windows Deployment Services
```

Ava:

```text
Servers
→ serveri nimi
```

Kui server pole veel seadistatud, näed hoiatust.

Paremklõps serveril:

```text
Configure Server
```

---

## 5.2 WDS seadistamise valikud

Wizardis vali:

| Samm | Valik |
|---|---|
| Install Options | Integrated with Active Directory |
| Remote Installation Folder | `C:\RemoteInstall` või parem eraldi ketas `D:\RemoteInstall` |
| PXE Server Initial Settings | Respond to all client computers |
| Require administrator approval | Eksamil võib jätta välja, kui tahad lihtsamat testimist |

Soovitus:

```text
Kui D: ketas on olemas, kasuta D:\RemoteInstall.
Kui eraldi ketast pole, kasuta C:\RemoteInstall.
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Seadistab WDS serveri, PXE vastamise ja RemoteInstall kausta | WDS server muutub konsoolis roheliseks / aktiivseks |

---

## 5.3 Kontrolli WDS teenuseid

PowerShellis:

```powershell
Get-Service WDSServer
```

Oodatav:

```text
Running
```

Kui teenus ei tööta:

```powershell
Start-Service WDSServer
```

Kontrolli WDS kausta:

```powershell
Test-Path "C:\RemoteInstall"
```

või:

```powershell
Test-Path "D:\RemoteInstall"
```

Oodatav:

```text
True
```

---

# 6. DHCP seadistus WDS jaoks

## 6.1 Oluline loogika

WDS ja DHCP käitumine sõltub sellest, kus teenused asuvad.

| Olukord | Mida teha |
|---|---|
| WDS ja DHCP on samas serveris | WDS peab olema seadistatud mitte kuulama DHCP porti 67 ja DHCP option 60 peab viitama PXEClientile |
| WDS ja DHCP on eri serverites samas võrgus | Tavaliselt pole DHCP option 66/67 vaja, klient leiab WDS-i broadcastiga |
| WDS ja klient on eri subnetis/VLAN-is | Eelistatud lahendus on IP Helper / DHCP relay, mis suunab päringud DHCP ja WDS serverisse |
| Kui IP Helper pole võimalik | Võib kasutada DHCP option 66 ja 67, aga UEFI/Legacy erinevused võivad probleeme teha |

---

## 6.2 Kui DHCP ja WDS on samas serveris

WDS konsoolis:

```text
Windows Deployment Services
→ Servers
→ serveri nimi
→ Properties
→ DHCP
```

Märgi:

```text
Do not listen on DHCP ports
Configure DHCP options to indicate that this is also a PXE server
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Väldib konflikti DHCP ja WDS vahel samas serveris | PXE klient saab IP ja WDS vastuse |

PowerShell / käsurea kontroll:

```powershell
wdsutil /get-server /show:config
```

Oodatav:

```text
DHCP ja PXE seadistused on nähtavad
```

---

## 6.3 Kui DHCP ja WDS on eri serverites samas võrgus

Sellisel juhul üldiselt:

```text
DHCP option 66 ja 67 ei ole samas subnetis tavaliselt vajalikud.
```

Kontrolli ainult, et:

| Kontroll | Oodatav |
|---|---|
| DHCP annab kliendile IP | Jah |
| WDS teenus töötab | Jah |
| WDS PXE vastab klientidele | Jah |
| Klient ja WDS on samas võrgus | Jah |

Kui PXE klient ei leia WDS serverit, kontrolli tulemüüri ja PXE vastamise seadeid.

---

## 6.4 Kui klient ja WDS on eri subnetis

Õige lahendus on võrguseadmes IP Helper / DHCP Relay.

IP helper peaks suunama PXE kliendi päringud:

```text
DHCP serveri IP
WDS serveri IP
```

Kui seda Packet Traceris või Proxmox labis teha ei saa, võib kasutada DHCP option 66/67.

---

## 6.5 DHCP option 66 ja 67 varuvariant

DHCP Manageris:

```text
IPv4
→ Scope Options
→ Configure Options
```

Lisa vajadusel:

| Option | Väärtus |
|---|---|
| 066 Boot Server Host Name | WDS serveri IP või FQDN |
| 067 Bootfile Name UEFI x64 jaoks | `boot\x64\wdsmgfw.efi` |
| 067 Bootfile Name Legacy BIOS jaoks | `boot\x64\wdsnbp.com` |

Oluline:

```text
UEFI ja Legacy BIOS kasutavad erinevat boot faili.
Windows 11 jaoks kasuta pigem UEFI boot’i.
```

Soovitus eksamil:

```text
Kui testmasin on UEFI, kasuta boot\x64\wdsmgfw.efi.
Kui testmasin on Legacy BIOS, kasuta boot\x64\wdsnbp.com.
```

Pärast DHCP muudatusi, kui kasutusel on DHCP failover:

```text
DHCP Manager
→ IPv4
→ paremklõps
→ Replicate Failover Scopes
```

---

# 7. Tulemüüri kontroll

WDS vajab mitut võrguteenust.

Kontrolli Windows Defender Firewallis, et WDS reeglid oleksid lubatud.

PowerShellis:

```powershell
Get-NetFirewallRule | Where-Object DisplayName -like "*Windows Deployment*"
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| WDS reeglid on olemas ja Enabled | Kui reeglid on keelatud, luba need |

Luba WDS reeglid:

```powershell
Get-NetFirewallRule | Where-Object DisplayName -like "*Windows Deployment*" | Enable-NetFirewallRule
```

DHCP serveris peab DHCP liiklus töötama:

| Port | Teenus |
|---|---|
| UDP 67 | DHCP server |
| UDP 68 | DHCP klient |
| UDP 69 | TFTP |
| UDP 4011 | PXE/WDS |
| TCP/UDP dünaamilised pordid | WDS / RPC haldus |

---

# 8. Lisa Windows 11 boot image

## 8.1 Mounti Windows 11 ISO

GUI kaudu:

```text
Paremklõps Windows 11 ISO failil
→ Mount
```

Näiteks tekib draiv:

```text
E:
```

Kontrolli:

```powershell
Test-Path "E:\sources\boot.wim"
```

Oodatav:

```text
True
```

---

## 8.2 Lisa boot image WDS-i

WDS konsoolis:

```text
Boot Images
→ Add Boot Image
```

Vali:

```text
E:\sources\boot.wim
```

Image name näiteks:

```text
Windows 11 Boot Image
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lisab PXE käivitamiseks Windows PE boot image’i | Boot Images all on Windows 11 Boot Image |

Kontroll PowerShellis:

```powershell
Get-WdsBootImage
```

Oodatav:

```text
Windows 11 Boot Image on nimekirjas
```

---

# 9. Lisa Windows 11 install image

## 9.1 Kontrolli install faili

Windows 11 ISO sees kontrolli:

```powershell
Test-Path "E:\sources\install.wim"
Test-Path "E:\sources\install.esd"
```

Üks neist peaks olema:

```text
True
```

Kui olemas on `install.wim`, saab selle otse lisada.

Kui olemas on `install.esd`, võib WDS uuemates versioonides selle vahel lisada, aga kindlam on teisendada see WIM-iks.

---

## 9.2 Kui olemas on install.wim

WDS konsoolis:

```text
Install Images
→ Add Image Group
```

Image group name:

```text
Windows 11
```

Seejärel:

```text
Windows 11
→ Add Install Image
```

Vali:

```text
E:\sources\install.wim
```

Vali sobiv edition, näiteks:

```text
Windows 11 Pro
Windows 11 Enterprise
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lisab Windows 11 paigaldatava image’i WDS-i | Install Images all on Windows 11 image |

---

## 9.3 Kui olemas on install.esd

Kontrolli image indeksid:

```powershell
dism /Get-WimInfo /WimFile:E:\sources\install.esd
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Kuvab ISO sees olevad Windowsi edition’id ja indeksid | Näed, milline indeks vastab soovitud Windows 11 versioonile |

Näide teisendamiseks:

```powershell
New-Item -ItemType Directory -Path "C:\WDS-Images" -Force

dism /Export-Image `
 /SourceImageFile:E:\sources\install.esd `
 /SourceIndex:1 `
 /DestinationImageFile:C:\WDS-Images\install-windows11.wim `
 /Compress:max `
 /CheckIntegrity
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Teisendab valitud Windows 11 image’i ESD-st WIM-iks | Tekib `C:\WDS-Images\install-windows11.wim` |

Seejärel lisa WDS-is:

```text
Install Images
→ Windows 11
→ Add Install Image
→ C:\WDS-Images\install-windows11.wim
```

---

# 10. Lisa Windows 10 Enterprise image

## 10.1 Mounti Windows 10 Enterprise ISO

Paremklõps ISO-l:

```text
Mount
```

Näiteks draiv:

```text
F:
```

Kontrolli:

```powershell
Test-Path "F:\sources\boot.wim"
Test-Path "F:\sources\install.wim"
Test-Path "F:\sources\install.esd"
```

---

## 10.2 Lisa Windows 10 boot image

Kui tahad eraldi Windows 10 boot image’i lisada:

```text
Boot Images
→ Add Boot Image
→ F:\sources\boot.wim
```

Image name:

```text
Windows 10 Boot Image
```

Märkus:

```text
Praktikas võib Windows 11 boot.wim paigaldada ka Windows 10 image’i,
aga ülesande täitmiseks on mõistlik lisada mõlema meedia failid.
```

---

## 10.3 Lisa Windows 10 Enterprise install image

WDS konsoolis:

```text
Install Images
→ Add Image Group
```

Image group name:

```text
Windows 10 Enterprise
```

Lisa install image:

```text
F:\sources\install.wim
```

Kui on `install.esd`, teisenda WIM-iks samamoodi nagu Windows 11 puhul:

```powershell
dism /Get-WimInfo /WimFile:F:\sources\install.esd
```

Seejärel:

```powershell
dism /Export-Image `
 /SourceImageFile:F:\sources\install.esd `
 /SourceIndex:1 `
 /DestinationImageFile:C:\WDS-Images\install-windows10-enterprise.wim `
 /Compress:max `
 /CheckIntegrity
```

Lisa WDS-i:

```text
C:\WDS-Images\install-windows10-enterprise.wim
```

---

# 11. Kontrolli WDS image’e

PowerShellis:

```powershell
Get-WdsBootImage
```

Oodatav:

```text
Windows 10 Boot Image
Windows 11 Boot Image
```

Kontrolli install image’e:

```powershell
Get-WdsInstallImage
```

Oodatav:

```text
Windows 10 Enterprise image
Windows 11 image
```

Kui käsud ei tööta, ava WDS konsool ja kontrolli graafiliselt:

```text
Boot Images
Install Images
```

---

# 12. Seadista WDS PXE vastamine

WDS konsoolis:

```text
Servers
→ serveri nimi
→ Properties
→ PXE Response
```

Soovitatav eksamil:

```text
Respond to all client computers
```

Kui tahad turvalisemat varianti:

```text
Respond only to known client computers
```

aga siis pead arvuti eelnevalt AD-s prestage’ima.

Lihtsam testimiseks:

```text
Respond to all client computers
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| WDS vastab PXE boot päringutele | Testmasin saab PXE menüü kätte |

---

# 13. Testmasina ettevalmistus

## 13.1 Loo uus VM või kasuta olemasolevat testmasinat

Testmasin peab olema samas võrgus, kus DHCP ja WDS töötavad.

Soovitus Proxmoxis:

| Seade | Väärtus |
|---|---|
| Boot mode | UEFI, eriti Windows 11 jaoks |
| Network | sama bridge/võrk, kus DHCP/WDS |
| Disk | vähemalt 64 GB |
| RAM | vähemalt 4 GB |
| CPU | vähemalt 2 tuuma |
| TPM | Windows 11 jaoks võib vaja minna |
| Secure Boot | Windows 11 puhul võib vaja minna, sõltub ISO ja labist |

Kui Windows 11 paigaldus ütleb, et nõuded pole täidetud, on probleem tavaliselt:

```text
TPM puudub
Secure Boot puudub
RAM või ketas liiga väike
```

---

## 13.2 Seadista boot order

VM-is pane esimeseks:

```text
Network boot / PXE
```

või käivita boot menüü ja vali:

```text
PXE Network Boot
```

Oodatav:

```text
Klient küsib DHCP-st IP ja leiab WDS serveri.
```

---

# 14. Windows 11 paigaldus WDS kaudu

## 14.1 Käivita PXE boot

Testmasinas peaks ilmuma midagi sarnast:

```text
Press F12 for network service boot
```

Vajuta:

```text
F12
```

Seejärel laaditakse:

```text
Windows Deployment Services boot image
```

---

## 14.2 Vali boot image

Kui WDS küsib boot image’i, vali:

```text
Windows 11 Boot Image
```

Oodatav:

```text
Windows Setup käivitub
```

---

## 14.3 Autendi domeeni kasutajaga

Windows Setup võib küsida credentials.

Sisesta domeeni admin või WDS õigustega kasutaja:

```text
sinuNimi\Haldur
```

või:

```text
sinuNimi\Administrator
```

---

## 14.4 Vali install image

Vali:

```text
Windows 11
```

või konkreetne edition:

```text
Windows 11 Pro / Enterprise
```

Seejärel vali ketas ja alusta paigaldust.

Oodatav:

```text
Windows 11 failid kopeeritakse üle võrgu ja paigaldus algab.
```

---

# 15. Paigalduse järel kontroll

Kui Windows 11 on paigaldatud:

```cmd
winver
```

Oodatav:

```text
Windows 11 versioon kuvatakse
```

Kontrolli võrku:

```cmd
ipconfig /all
```

Oodatav:

```text
Klient saab DHCP-st IP-aadressi
DNS serverid on DC1 ja DC2
```

Kui vaja, lisa masin domeeni:

```text
Settings
→ System
→ About
→ Domain or workgroup
→ Change
→ Domain
→ sinuNimi.local
```

Kui ülesande järgi piisab Windows 11 paigaldusest, domeeni liitmine võib olla juba üldosas teise kliendi peal tehtud.

---

# 16. Lõppkontroll

| Kontroll | Käsk / koht | Oodatav tulemus |
|---|---|---|
| WDS roll | `Get-WindowsFeature WDS` | Installed |
| WDS teenus | `Get-Service WDSServer` | Running |
| RemoteInstall | `Test-Path C:\RemoteInstall` | True |
| DHCP scope | `Get-DhcpServerv4Scope` | Scope aktiivne |
| Boot images | `Get-WdsBootImage` | Windows 10 ja Windows 11 boot image’id |
| Install images | `Get-WdsInstallImage` | Windows 10 Enterprise ja Windows 11 image’id |
| PXE response | WDS Properties | Respond to clients enabled |
| Tulemüür | WDS firewall rules | Enabled |
| Testmasin | PXE boot | Saab WDS boot image’i kätte |
| Windows 11 install | Testmasin | Windows 11 paigaldub üle võrgu |
| Võrk pärast installi | `ipconfig /all` | DHCP IP ja DC1/DC2 DNS |

---

# 17. Dokumentatsiooni näidis

```markdown
## Windows pilet 5 dokumentatsioon

### Eesmärk

Eesmärk oli seadistada Windows Deployment Services ehk WDS lahendus, mille abil saab Windowsi klientarvuteid paigaldada üle võrgu PXE boot’i kaudu. Lahendusse tuli lisada Windows 10 Enterprise ja Windows 11 paigaldusmeedia failid, teha DHCP serveris WDS-i toimimiseks vajalikud seadistused ning paigaldada testmasinasse üle võrgu Windows 11.

### Kasutatud serverid

| Server | Roll |
|---|---|
| DC1 | AD DS, DNS, DHCP, WDS |
| DC2 | Teine domeenikontroller ja DHCP failover partner |
| Test VM | PXE boot ja Windows 11 paigaldus |

### Tehtud seadistused

- Kontrollisin DHCP scope’i ja DNS seadeid.
- Paigaldasin Windows Deployment Services rolli.
- Valisin WDS rolliteenused Deployment Server ja Transport Server.
- Seadistasin WDS serveri Active Directoryga integreeritult.
- Lõin RemoteInstall kausta.
- Seadistasin WDS-i vastama PXE klientidele.
- Kontrollisin DHCP ja WDS koos toimimist.
- Vajadusel seadistasin DHCP option 60 või 66/67 vastavalt WDS ja DHCP paiknemisele.
- Mountisin Windows 11 ISO faili.
- Lisasin WDS-i Windows 11 boot image’i.
- Lisasin WDS-i Windows 11 install image’i.
- Mountisin Windows 10 Enterprise ISO faili.
- Lisasin WDS-i Windows 10 Enterprise paigaldusfailid.
- Kontrollisin WDS teenuse ja image’ite olemasolu.
- Seadistasin testmasina võrgust alglaadima.
- Käivitasin PXE boot’i ja valisin Windows 11 paigalduse.
- Paigaldasin testmasinasse Windows 11 üle võrgu.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `Get-WindowsFeature WDS` | WDS roll on paigaldatud |
| `Get-Service WDSServer` | WDS teenus töötab |
| `Get-WdsBootImage` | Boot image’id on lisatud |
| `Get-WdsInstallImage` | Install image’id on lisatud |
| DHCP scope | Klient saab IP-aadressi |
| PXE boot | Testmasin saab WDS boot menüü |
| Windows 11 install | Windows 11 paigaldub üle võrgu |
| `ipconfig /all` | Paigaldatud klient saab DHCP IP ja DC DNS seaded |

### Kokkuvõte

Windows pilet 5 tulemusena töötab WDS-põhine Windowsi võrgupaigaldus. WDS serveris on olemas Windows 10 Enterprise ja Windows 11 boot/install image’id. DHCP ja PXE seadistus võimaldab testmasinal võrgust alglaadida ning Windows 11 paigaldada üle võrgu.
```

---

# 18. Troubleshooting

## PXE klient ei saa IP-aadressi

Kontrolli DHCP serveris:

```powershell
Get-DhcpServerv4Scope
Get-DhcpServerv4Lease -ScopeId 10.x.x.0
```

Kui klient IP-d ei saa:

| Põhjus | Lahendus |
|---|---|
| DHCP scope pole aktiivne | Aktiveeri scope |
| Klient on vales võrgus | Kontrolli VM network/bridge |
| DHCP failover ei replitseerunud | Tee Replicate Failover Scopes |
| DHCP teenus ei tööta | Kontrolli `Get-Service DHCPServer` |

---

## PXE klient saab IP, aga ei leia WDS serverit

Kontrolli WDS teenust:

```powershell
Get-Service WDSServer
```

Kontrolli PXE vastamist:

```text
WDS → Server Properties → PXE Response
```

Kui WDS ja DHCP on samas serveris:

```text
WDS Properties → DHCP:
Do not listen on DHCP ports
Configure DHCP option 60
```

Kui klient on teises subnetis:

```text
Seadista IP Helper DHCP ja WDS serveri IP peale.
```

---

## TFTP timeout või boot file error

Võimalikud põhjused:

| Põhjus | Lahendus |
|---|---|
| Vale DHCP option 67 | UEFI jaoks kasuta `boot\x64\wdsmgfw.efi` |
| Legacy/UEFI mismatch | Kontrolli VM boot mode’i |
| Tulemüür blokeerib | Luba WDS firewall rules |
| Boot image puudub | Lisa `boot.wim` uuesti |

---

## Windows 11 install ei käivitu

Kontrolli:

| Kontroll | Lahendus |
|---|---|
| Testmasin on UEFI? | Windows 11 jaoks kasuta UEFI |
| TPM olemas? | Lisa VM-ile TPM, kui installer nõuab |
| RAM piisav? | Anna vähemalt 4 GB |
| Ketas piisav? | Anna vähemalt 64 GB |
| Install image olemas? | Kontrolli `Get-WdsInstallImage` |

---

## WDS-is ei saa install.esd faili lisada

Teisenda ESD WIM-iks:

```powershell
dism /Get-WimInfo /WimFile:E:\sources\install.esd
```

Seejärel:

```powershell
dism /Export-Image `
 /SourceImageFile:E:\sources\install.esd `
 /SourceIndex:1 `
 /DestinationImageFile:C:\WDS-Images\install.wim `
 /Compress:max `
 /CheckIntegrity
```

Lisa WDS-i:

```text
Install Images → Add Install Image → C:\WDS-Images\install.wim
```

---

## Windows Setup küsib kasutajanime ja parooli

Kasuta domeeni admin kontot:

```text
sinuNimi\Haldur
```

või:

```text
sinuNimi\Administrator
```

Kui ei tööta:

| Põhjus | Lahendus |
|---|---|
| Vale parool | Kontrolli kasutajat |
| WDS pole AD-ga integreeritud | Kontrolli WDS seadistust |
| DNS ei tööta | Kontrolli DC1/DC2 DNS-i |
| Testmasin ei saa domeeniga ühendust | Kontrolli võrku |

---

# 19. Kõige lühem spikker

```text
1. Kontrolli DHCP scope’i ja DNS-i
2. Paigalda WDS roll
3. Vali Deployment Server ja Transport Server
4. Ava WDS konsool
5. Configure Server
6. Vali Integrated with Active Directory
7. Loo C:\RemoteInstall või D:\RemoteInstall
8. Seadista PXE Response: Respond to all client computers
9. Kui WDS ja DHCP on samas serveris, märgi WDS DHCP seaded õigeks
10. Mounti Windows 11 ISO
11. Lisa Windows 11 boot.wim Boot Images alla
12. Lisa Windows 11 install.wim või teisendatud WIM Install Images alla
13. Mounti Windows 10 Enterprise ISO
14. Lisa Windows 10 Enterprise boot/install image’id
15. Kontrolli Get-WdsBootImage ja Get-WdsInstallImage
16. Luba WDS firewall rules
17. Loo test VM
18. Sea test VM võrgust bootima
19. Vajuta PXE boot’is F12
20. Vali Windows 11 boot image
21. Vali Windows 11 install image
22. Paigalda Windows 11 üle võrgu
23. Kontrolli ipconfig /all
24. Dokumenteeri
```
