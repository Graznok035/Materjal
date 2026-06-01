# Windowsi üldosa lahenduskäik

See juhend kirjeldab Windowsi piletite **üldosa**, mis tuleb teha enne konkreetse Windowsi pileti eriosa lahendamist.

Arvestus: põhiline Windows Server on **graafilise kasutajaliidesega server** ehk GUI server.  
Server Core masinat kasutatakse teise domeenikontrollerina ja DHCP failover partnerina.

---

## 1. Windowsi üldosa eesmärk

Kõik Windowsi piletid sisaldavad sama baasosa.

Üldosa lõpuks peab olema loodud AS Õige Windowsi terviklahendus.

| Ülesande osa | Mida tuleb teha | Milleks seda vaja on |
|---|---|---|
| DC1 | Windows Server GUI masin seadistada esimeseks domeenikontrolleriks | Domeeni põhiserver |
| DC2 | Windows Server Core masin seadistada teiseks domeenikontrolleriks | Varudomeenikontroller |
| Domeen | Luua domeen `sinuNimi.local` | Kasutajate, arvutite ja poliitikate keskseks haldamiseks |
| DNS | Paigaldada ja seadistada DNS roll | Domeeni nimelahenduseks |
| DHCP | Paigaldada ja seadistada DHCP roll | Klientidele IP-aadresside jagamiseks |
| DHCP failover | Seadistada DHCP failover DC1 ja DC2 vahel | DHCP töökindluse tagamiseks |
| DHCP reservation’id | Luua staatilised DHCP rendid klientidele | Et kliendid saaksid alati sama IP |
| OU-d | Luua OU-d `Kasutajad` ja `Arvutid` | Kasutajate ja arvutite korrastamiseks |
| Haldur | Luua kasutaja `Haldur` ja lisada `Domain Admins` gruppi | Administraatori õigustega halduskasutaja |
| Windows 11 klient | Lisada Windows 11 klient domeeni | Domeeni kasutajate ja GPO-de testimiseks |
| CSV import | Importida kasutajad failist `kasutajad.csv` | Kasutajate automaatseks loomiseks |
| GPO-d | Luua parooli, kontolukustuse ja USB piirangu poliitikad | Turvanõuete täitmiseks |

---

# 2. Vajalikud rollid ja tööriistad

## 2.1 DC1 ehk graafiline Windows Server

DC1-s kasutatakse Server Managerit:

```text
Server Manager → Manage → Add Roles and Features
```

Vajalikud rollid ja tööriistad:

| Roll / tööriist | Vajalik millal | Milleks kasutatakse |
|---|---|---|
| Active Directory Domain Services | Kõik Windowsi piletid | Domeeni ja domeenikontrolleri loomiseks |
| DNS Server | Kõik Windowsi piletid | Domeeni nimelahenduseks |
| DHCP Server | Kõik Windowsi piletid | IP-aadresside jagamiseks klientidele |
| Group Policy Management | Kõik Windowsi piletid | GPO-de loomiseks ja haldamiseks |
| Active Directory Users and Computers | Kõik Windowsi piletid | Kasutajate, gruppide ja OU-de haldamiseks |
| DNS Manager | Kõik Windowsi piletid | DNS kirjete haldamiseks |
| DHCP Manager | Kõik Windowsi piletid | DHCP scope’i, reservation’ite ja failoveri haldamiseks |

---

## 2.2 DC2 ehk Server Core

Server Core masinast saab DC2.

Vajalikud rollid:

| Roll | Milleks |
|---|---|
| AD DS | Teiseks domeenikontrolleriks |
| DNS | Teiseks DNS serveriks |
| DHCP Server | DHCP failover partneriks |

DC2 saab seadistada PowerShelliga või hallata hiljem DC1 Server Managerist.

---

# 3. Serverite algkontroll

## 3.1 Kontrolli DC1 võrku

GUI kaudu:

```text
Control Panel
→ Network and Internet
→ Network Connections
→ paremklõps adapteril
→ Properties
→ Internet Protocol Version 4 (TCP/IPv4)
```

PowerShellis:

```powershell
ipconfig /all
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Näitab IP-aadresse, DNS servereid ja võrgukaardi infot | Serveril on õige IP, subnet mask, gateway ja DNS |

---

## 3.2 Määra DC1-le staatiline IP

Näidis:

| Väli | Näide |
|---|---|
| IP address | `10.x.x.10` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `10.x.x.1` |
| Preferred DNS server | `10.x.x.10` |
| Alternate DNS server | hiljem `10.x.x.11` ehk DC2 |

Kontroll:

```powershell
ipconfig /all
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| DC1 kasutab staatilist IP-d ja DNS on DC1 enda IP | Kui DNS on `1.1.1.1` või `8.8.8.8`, muuda see DC1/DC2 IP-ks |

Oluline:

```text
Domeenimasinate DNS ei tohi olla 1.1.1.1 ega 8.8.8.8.
Klientide ja serverite DNS peab olema DC1 ja DC2.
Avalikke DNS-e võib kasutada ainult DNS forwarder’itena DNS serveri sees.
```

---

## 3.3 Muuda DC1 serveri nimi

GUI kaudu:

```text
Server Manager → Local Server → Computer name
```

Nimeks pane:

```text
DC1
```

PowerShelli alternatiiv:

```powershell
Rename-Computer -NewName "DC1" -Restart
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Muudab serveri nime ja teeb restardi | Pärast restarti on serveri nimi `DC1` |

Kontroll:

```powershell
hostname
```

Oodatav:

```text
DC1
```

---

# 4. AD DS ja DNS paigaldamine DC1-s

## 4.1 Paigalda AD DS ja DNS rollid

GUI kaudu:

```text
Server Manager
→ Manage
→ Add Roles and Features
→ Role-based or feature-based installation
→ Select server DC1
→ Active Directory Domain Services
→ Add Features
→ DNS Server
→ Add Features
→ Next
→ Install
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab AD DS ja DNS rollid | Server Manageris tekib teade, et server tuleb promoda domeenikontrolleriks |

PowerShelli alternatiiv:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

---

## 4.2 Promote this server to a domain controller

Pärast rollide paigaldust vali Server Manageris:

```text
Promote this server to a domain controller
```

Vali:

```text
Add a new forest
```

Domeeni nimi:

```text
sinuNimi.local
```

Näide:

```text
ekristal.local
```

Määra DSRM parool ja lõpeta wizard.

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob uue AD foresti ja domeeni | Server teeb restardi ja DC1 on domeenikontroller |

---

## 4.3 Kontrolli domeeni olemasolu

Ava:

```text
Server Manager → Tools → Active Directory Users and Computers
```

Oodatav:

```text
Näed domeeni sinuNimi.local
```

PowerShelli kontroll:

```powershell
Get-ADDomain
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse domeeni info | Kui käsk ei tööta, AD DS ei ole õigesti paigaldatud või PowerShell moodul puudub |

Kontroll DNS-is:

```text
Server Manager → Tools → DNS
```

Oodatav:

```text
Forward Lookup Zones all on sinuNimi.local tsoon
```

---

# 5. DC2 ehk Server Core lisamine domeeni

## 5.1 Seadista DC2 võrk

Server Core masinas ava:

```text
sconfig
```

Vali:

```text
8) Network Settings
```

Määra:

| Väli | Väärtus |
|---|---|
| IP | DC2 staatiline IP |
| Subnet mask | sama võrk |
| Gateway | võrgu gateway |
| Preferred DNS | DC1 IP |
| Alternate DNS | esialgu tühi või hiljem DC2 IP |

Kontroll:

```powershell
ipconfig /all
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| DC2 DNS on DC1 IP | Kui DNS on avalik DNS, domeeniga liitumine ei tööta korrektselt |

---

## 5.2 Muuda DC2 nimi

Server Core masinas:

```text
sconfig
```

Vali:

```text
2) Computer Name
```

Pane nimeks:

```text
DC2
```

Tee restart.

Kontroll:

```powershell
hostname
```

Oodatav:

```text
DC2
```

---

## 5.3 Lisa DC2 domeeni

Server Core masinas:

```text
sconfig
```

Vali:

```text
1) Domain/Workgroup
```

Sisesta domeen:

```text
sinuNimi.local
```

Kasuta domeeni admin kontot:

```text
sinuNimi\Administrator
```

või:

```text
sinuNimi\Haldur
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lisab DC2 domeeni liikmeks | Pärast restarti on DC2 domeenis |

Kontroll DC1-s:

```text
Active Directory Users and Computers → Computers
```

Oodatav:

```text
DC2 arvutiobjekt on olemas
```

---

# 6. Tee DC2 teiseks domeenikontrolleriks

## 6.1 Paigalda AD DS ja DNS DC2-s

Server Core PowerShellis:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab DC2 masinasse AD DS ja DNS rolli | Rollid paigaldatakse edukalt |

---

## 6.2 Promote DC2 olemasoleva domeeni domeenikontrolleriks

Server Core PowerShellis:

```powershell
Install-ADDSDomainController -DomainName "sinuNimi.local" -InstallDns
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Teeb DC2 masinast olemasoleva domeeni teise domeenikontrolleri | Masin teeb restardi ja muutub DC-ks |

Kui küsib domeeni administraatori andmeid, sisesta domeeni admin konto.

Pärast restarti kontrolli DC1-s:

```text
Active Directory Users and Computers
→ Domain Controllers
```

Oodatav:

```text
DC1 ja DC2 on mõlemad Domain Controllers OU all
```

---

## 6.3 Kontrolli DC-de olemasolu

DC1 PowerShellis:

```powershell
Get-ADDomainController -Filter *
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse DC1 ja DC2 | Kui DC2 puudub, ei ole DC2 õigesti domeenikontrolleriks promootud |

Soovi korral kontrolli replikatsiooni:

```powershell
repadmin /replsummary
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Replikatsiooni vead puuduvad või neid on 0 | Kui on vead, kontrolli DNS-i ja DC-de ühendust |

---

# 7. DHCP rolli paigaldamine

## 7.1 Paigalda DHCP roll DC1-s

GUI kaudu:

```text
Server Manager
→ Manage
→ Add Roles and Features
→ DHCP Server
→ Add Features
→ Install
```

Pärast paigaldust vali:

```text
Complete DHCP configuration
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab DHCP serveri ja autoriseerib selle AD domeenis | DHCP Manageris on DC1 nähtav ja autoriseeritud |

PowerShelli alternatiiv:

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

---

## 7.2 Paigalda DHCP roll DC2-s

DC2 Server Core PowerShellis:

```powershell
Install-WindowsFeature DHCP
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab DHCP rolli DC2 masinasse | DC2 saab olla DHCP failover partner |

DC2 DHCP server tuleb samuti AD-s autoriseerida. Seda saab teha DC1 DHCP Managerist või PowerShelliga.

---

# 8. DHCP scope’i loomine

## 8.1 Ava DHCP Manager

DC1 GUI serveris:

```text
Server Manager → Tools → DHCP
```

Liigu:

```text
DC1 → IPv4
```

Paremklõps:

```text
New Scope
```

---

## 8.2 Loo IPv4 scope

Näidisväljad:

| Väli | Näide |
|---|---|
| Scope name | `LAN-Scope` |
| Start IP | `10.x.x.100` |
| End IP | `10.x.x.200` |
| Subnet mask | `255.255.255.0` |
| Exclusions | serverite IP-d, kui jäävad vahemikku |
| Lease duration | `4 hours` |
| Router / Gateway | `10.x.x.1` |
| DNS servers | `DC1 IP` ja `DC2 IP` |
| DNS domain name | `sinuNimi.local` |

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob DHCP aadressivahemiku | Kliendid saavad IP-aadressi sellest vahemikust |

Oluline:

```text
DHCP peab klientidele DNS serveritena jagama DC1 ja DC2 IP-aadresse.
Mitte 1.1.1.1 ega 8.8.8.8.
```

---

## 8.3 Kontrolli DHCP scope’i

DHCP Manageris:

```text
IPv4 → Scope
```

Kontrolli:

| Kontroll | Oodatav tulemus |
|---|---|
| Scope on Active | DHCP jagab aadresse |
| Address Pool olemas | Vahemik on õige |
| Scope Options olemas | Router, DNS ja Domain Name on õiged |

PowerShelli kontroll:

```powershell
Get-DhcpServerv4Scope
```

Oodatav:

```text
Scope on olemas ja aktiivne
```

---

# 9. DHCP failover DC1 ja DC2 vahel

## 9.1 Seadista failover

DHCP Manageris:

```text
DHCP → DC1 → IPv4 → Scope peal paremklõps → Configure Failover
```

Vali partneriks:

```text
DC2
```

Vali režiim:

```text
Load balance
```

või:

```text
Hot standby
```

Eksami jaoks sobib enamasti vaikimisi või õpetaja nõutud valik.

| Mida see teeb | Oodatav tulemus |
|---|---|
| Jagab DHCP scope’i DC1 ja DC2 vahel | Kui üks DHCP server ei tööta, saab teine aadresse jagada |

Kontroll:

```powershell
Get-DhcpServerv4Failover
```

Oodatav:

```text
Failover relationship on olemas
```

Oluline pärast muudatusi:

```text
Kui muudad failoveriga scope’i seadeid, tee DHCP Manageris:
IPv4 → paremklõps → Replicate Failover Scopes
```

---

# 10. DHCP staatilised rendid ehk reservation’id

## 10.1 Leia kliendi MAC-aadress

Windows kliendis:

```cmd
ipconfig /all
```

Otsi:

```text
Physical Address
```

Linux kliendis:

```bash
ip link
```

või:

```bash
ip a
```

---

## 10.2 Lisa reservation DHCP Manageris

DHCP Manageris:

```text
Scope → Reservations → New Reservation
```

Täida:

| Väli | Näide |
|---|---|
| Reservation name | `WIN11` |
| IP address | `10.x.x.101` |
| MAC address | kliendi MAC |
| Supported types | DHCP |

| Mida see teeb | Oodatav tulemus |
|---|---|
| Seob kliendi MAC-aadressi kindla IP-ga | Klient saab alati sama IP-aadressi |

Kontrolli kliendis:

```cmd
ipconfig /release
ipconfig /renew
ipconfig /all
```

Oodatav:

```text
Klient saab reservation’is määratud IP
```

---

# 11. OU-de loomine

## 11.1 Ava Active Directory Users and Computers

```text
Server Manager → Tools → Active Directory Users and Computers
```

Loo domeeni alla OU-d:

```text
Kasutajad
Arvutid
```

Soovi korral loo kasutajate jaoks alam-OU-d vastavalt CSV failile.

Näide:

```text
Kasutajad
├── Personal
├── Juhtkond
├── IT
└── Tootmine

Arvutid
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob AD struktuuri kasutajate ja arvutite jaoks | Kasutajad ja arvutid saab õigesse OU-sse tõsta |

---

# 12. Haldur kasutaja loomine

## 12.1 Loo kasutaja Haldur

ADUC-is:

```text
OU Kasutajad → New → User
```

Kasutajanimi:

```text
Haldur
```

Määra parool.

Lisa kasutaja gruppi:

```text
Domain Admins
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob administraatori õigustega domeenikasutaja | Haldur saab domeeni hallata |

Kontroll:

```powershell
Get-ADUser Haldur -Properties MemberOf
```

Oodatav:

```text
Kasutaja on Domain Admins grupis
```

---

# 13. Kasutajate import CSV failist

## 13.1 Kontrolli CSV faili

Fail:

```text
kasutajad.csv
```

Kontroll PowerShellis:

```powershell
Import-Csv .\kasutajad.csv
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse CSV sisu | Kui tulevad valed veerud, ava CSV ja kontrolli eraldajat ning päiseid |

---

## 13.2 Näidis PowerShell kasutajate loomiseks

Näidis, mida tuleb kohandada CSV veergude järgi:

```powershell
$users = Import-Csv .\kasutajad.csv

foreach ($u in $users) {
    $ouPath = "OU=$($u.OU),OU=Kasutajad,DC=sinuNimi,DC=local"

    New-ADUser `
        -Name "$($u.Eesnimi) $($u.Perenimi)" `
        -GivenName $u.Eesnimi `
        -Surname $u.Perenimi `
        -SamAccountName $u.Kasutajanimi `
        -UserPrincipalName "$($u.Kasutajanimi)@sinuNimi.local" `
        -Path $ouPath `
        -AccountPassword (ConvertTo-SecureString "Parool123!" -AsPlainText -Force) `
        -Enabled $true `
        -ChangePasswordAtLogon $true
}
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob CSV põhjal AD kasutajad õigesse OU-sse | Kasutajad tekivad ADUC-is |

Kontroll:

```powershell
Get-ADUser -Filter *
```

või ADUC-is:

```text
Kasutajad OU
```

---

# 14. Windows 11 kliendi lisamine domeeni

## 14.1 Kontrolli kliendi DNS-i

Windows 11 kliendis:

```cmd
ipconfig /all
```

Oodatav:

```text
DNS Servers = DC1 IP ja DC2 IP
```

Kui DNS on `1.1.1.1` või `8.8.8.8`, siis domeeniga liitumine võib ebaõnnestuda.

---

## 14.2 Lisa klient domeeni

Windows 11 GUI kaudu:

```text
Settings
→ System
→ About
→ Domain or workgroup
→ Change
→ Domain
```

Sisesta:

```text
sinuNimi.local
```

Kui küsib kasutajat, sisesta:

```text
sinuNimi\Haldur
```

või domeeni Administrator.

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lisab Windows 11 kliendi domeeni | Tuleb teade `Welcome to the sinuNimi.local domain` |

Pärast seda tee restart.

---

## 14.3 Tõsta klient OU-sse Arvutid

ADUC-is:

```text
Computers → leia Windows 11 arvuti → Move → OU Arvutid
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paneb kliendi õigesse OU-sse | Arvutile rakenduvad Arvutid OU külge lingitud GPO-d |

---

# 15. GPO-de loomine

## 15.1 Ava Group Policy Management

```text
Server Manager → Tools → Group Policy Management
```

---

## 15.2 Paroolipoliitika

Oluline:

```text
Domeeni kasutajate paroolipoliitika tuleb rakendada domeeni tasemel.
Ära lingi seda ainult OU-le, sest tavaline domeeni Account Policy kehtib domeeni kaudu.
```

Variant A: muuda olemasolevat `Default Domain Policy`.

Variant B: loo uus GPO ja lingi see domeeni juurele.

GPO nimi:

```text
ParooliKehtivus
```

Seadistus:

```text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Account Policies
→ Password Policy
```

Määra:

| Seade | Väärtus |
|---|---|
| Maximum password age | 30 days |

| Mida see teeb | Oodatav tulemus |
|---|---|
| Määrab parooli maksimaalse kehtivusaja | Parool kehtib kuni 30 päeva |

---

## 15.3 Kontolukustuse poliitika

Ka see tuleb rakendada domeeni tasemel.

GPO nimi:

```text
KontodeLukustamine
```

Seadistus:

```text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Account Policies
→ Account Lockout Policy
```

Määra:

| Seade | Väärtus |
|---|---|
| Account lockout threshold | 5 invalid logon attempts |
| Account lockout duration | 15 minutes |
| Reset account lockout counter after | 15 minutes |

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lukustab konto pärast 5 valet parooli | Konto avaneb 15 minuti pärast või admin avab selle |

---

## 15.4 USB piirangu GPO

See GPO sobib linkida OU-le:

```text
Arvutid
```

GPO nimi:

```text
KeelaUSB
```

Seadistus:

```text
Computer Configuration
→ Policies
→ Administrative Templates
→ System
→ Removable Storage Access
```

Luba:

```text
All Removable Storage classes: Deny all access
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Keelab eemaldatavate andmekandjate kasutamise | Windows 11 kliendis USB lugemine/kirjutamine on keelatud |

---

# 16. GPO testimine kliendis

Windows 11 kliendis ava CMD või PowerShell administraatorina.

Uuenda poliitikaid:

```cmd
gpupdate /force
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Rakendab GPO-d kohe | Tulemus ütleb, et Computer/User Policy update completed successfully |

Kontrolli rakendunud poliitikaid:

```cmd
gpresult /r
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Näed rakendunud GPO-sid | Kui GPO puudub, kontrolli OU-d ja GPO linki |

Täpsem HTML raport:

```cmd
gpresult /h C:\gpresult.html
```

Ava fail:

```text
C:\gpresult.html
```

---

# 17. DNS kontroll

## 17.1 Kontrolli domeeni nimelahendust

DC1 või kliendi PowerShellis:

```powershell
nslookup sinuNimi.local
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Domeen lahendub DC IP-ks | Kui ei lahendu, klient kasutab valet DNS serverit või DNS tsoon on vigane |

Kontrolli DC1 nime:

```powershell
nslookup DC1.sinuNimi.local
```

Kontrolli DC2 nime:

```powershell
nslookup DC2.sinuNimi.local
```

Oodatav:

```text
Nimed lahenduvad õigeks IP-ks
```

---

# 18. DHCP kontroll kliendis

Windows 11 kliendis:

```cmd
ipconfig /all
```

Kontrolli:

| Väli | Oodatav |
|---|---|
| DHCP Enabled | Yes |
| IPv4 Address | DHCP scope’ist või reservation’ist |
| Default Gateway | õige gateway |
| DNS Servers | DC1 IP ja DC2 IP |
| Connection-specific DNS Suffix | `sinuNimi.local` |

IP uuendamine:

```cmd
ipconfig /release
ipconfig /renew
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Küsib DHCP serverilt uue aadressi | Klient saab õige IP ja DNS seaded |

---

# 19. Üldosa lõpptulemus

Windowsi üldosa lõpuks peab olema selline seis:

| Kontrollitav asi | Lõpptulemus |
|---|---|
| DC1 | GUI Windows Server, esimene domeenikontroller |
| DC2 | Server Core, teine domeenikontroller |
| Domeen | `sinuNimi.local` töötab |
| DNS | DC1 ja DC2 lahendavad domeeni nimesid |
| DHCP | DHCP scope jagab klientidele IP-aadresse |
| DHCP failover | DC1 ja DC2 vahel töötab failover |
| Reservation’id | Klientidele on määratud staatilised DHCP rendid |
| OU-d | Olemas `Kasutajad` ja `Arvutid` |
| Haldur | Kasutaja olemas ja `Domain Admins` grupis |
| CSV kasutajad | Kasutajad imporditud AD-sse |
| Windows 11 klient | Domeeniga liidetud |
| Windows 11 OU | Klient asub OU-s `Arvutid` |
| Paroolipoliitika | Parooli max eluiga 30 päeva, domeeni tasemel |
| Kontolukustus | Konto lukustub pärast 5 vale parooli, domeeni tasemel |
| USB piirang | USB andmekandjad keelatud, OU `Arvutid` tasemel |
| DNS kliendis | DNS serverid on DC1 ja DC2, mitte avalikud DNS-id |

---

# 20. Väike lõppkontroll

## DC1 PowerShellis

```powershell
Get-ADDomain
```

Oodatav:

```text
Domeen kuvatakse
```

```powershell
Get-ADDomainController -Filter *
```

Oodatav:

```text
DC1 ja DC2 kuvatakse
```

```powershell
Get-DhcpServerv4Scope
```

Oodatav:

```text
DHCP scope on olemas
```

```powershell
Get-DhcpServerv4Failover
```

Oodatav:

```text
DHCP failover relationship on olemas
```

```powershell
Get-ADUser Haldur -Properties MemberOf
```

Oodatav:

```text
Haldur on Domain Admins grupis
```

---

## Windows 11 kliendis

```cmd
ipconfig /all
```

Oodatav:

```text
IP tuleb DHCP-st
DNS serverid on DC1 ja DC2
DNS suffix on sinuNimi.local
```

```cmd
whoami
```

Oodatav domeeni kasutajaga:

```text
sinuNimi\kasutaja
```

```cmd
gpupdate /force
```

Oodatav:

```text
Policy update completed successfully
```

```cmd
gpresult /r
```

Oodatav:

```text
Näha on rakendunud GPO-d
```

---

# 21. Ametlikud juhendid piltidega

| Teema | Ametlik juhend |
|---|---|
| AD DS paigaldamine ja domeenikontrolleriks promote | [Install Active Directory Domain Services on Windows Server](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services--level-100-) |
| AD DS wizardi lehtede selgitused | [AD DS Configuration Wizard Page Descriptions](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/deploy/ad-ds-installation-and-removal-wizard-page-descriptions) |
| DHCP failover ülevaade | [DHCP failover overview](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-failover) |
| DHCP failover haldus | [Manage DHCP failover relationships in Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/manage-dhcp-failover-relationships) |
| DHCP failover replikatsioon | [Replicate DHCP failover in Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/replicate-dhcp-failover) |
| DNS kirjete haldus | [Managing DNS resource records](https://learn.microsoft.com/en-us/windows-server/networking/dns/manage-resource-records) |
| DNS A-kirje lisamise näide | [Add a Host A Resource Record](https://learn.microsoft.com/en-us/windows-server/identity/ad-fs/deployment/add-a-host--a--resource-record-to-corporate-dns-for-a-federation-server) |
| Group Policy Management Console | [Group Policy Management Console in Windows](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-management-console) |

---

# 22. Kokkuvõte

Windowsi üldosa eesmärk on teha valmis toimiv domeenikeskkond.

Lühidalt peab üldosa lõpuks olema:

```text
DC1 = GUI Windows Server, AD DS + DNS + DHCP
DC2 = Server Core, teine DC + DNS + DHCP failover partner
Domeen = sinuNimi.local
DNS = DC1 ja DC2
DHCP = scope + reservation’id + failover
OU-d = Kasutajad ja Arvutid
Haldur = Domain Admins grupis
Windows 11 = domeenis ja OU-s Arvutid
GPO-d = parool, kontolukustus, USB piirang
Klientide DNS = DC1 ja DC2, mitte 1.1.1.1 ega 8.8.8.8
```

Kui see osa töötab, saab edasi liikuda konkreetse Windowsi pileti eriosa juurde:

```text
Pilet 1 = IIS + AD autentimine + AD CS + HTTPS
Pilet 2 = DFS + DFS Replication + FSRM
Pilet 3 = PowerShell skriptid AD ja DHCP jaoks
Pilet 4 = GPO tarkvarapaigaldus ja kasutajapiirangud
Pilet 5 = WDS ja Windowsi paigaldus üle võrgu
```
