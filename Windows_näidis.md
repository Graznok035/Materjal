# Windows. Pilet 1 — Active Directory, DHCP, DNS, IIS ja siseportaal

## Märksõnad

- Active Directory kataloogiteenused
- Domeenikontrollerid
- Windows Server Core
- DNS serveri haldus
- DHCP serveri haldus
- DHCP failover
- OU struktuur
- AD kasutajate import CSV failist
- Group Policy Object ehk GPO
- IIS veebiserver
- Veebirakenduse / siseportaali paigaldus
- AD autentimine veebilehele
- SSL sertifikaat
- DNS CNAME kirjed

---

# 1. Ülesande lühikokkuvõte

AS Oige vajab Windowsi-põhist terviklahendust.

Tuleb luua Active Directory domeen:

```text
oige.local
```

Domeenikontrollerid:

```text
DC1 = Windows Server 2025 Desktop Experience ehk graafilise liidesega server
DC2 = Windows Server Core 2025 ehk käsureapõhine server
```

NB! **DC2 Server Core’il ei ole tavalist graafilist kasutajaliidest ega Server Manageri akent.**  
DC2 seadistamine tehakse:

```text
sconfig
PowerShell
kaugelt DC1 haldustööriistade kaudu
```

Lisaks tuleb seadistada:

- DNS roll mõlemale domeenikontrollerile
- DHCP roll DC1 ja DC2 serverile
- DHCP failover DC1 ja DC2 vahel
- staatilised DHCP rendid klientarvutitele
- DHCP rendi kestus 4 tundi
- DHCP jagab DNS serveritena välja DC1 ja DC2
- OU-d `Kasutajad` ja `Arvutid`
- kasutaja `Haldur`, kes kuulub gruppi `Domain Admins`
- Windows 11 klient domeeni ja OU-sse `Arvutid`
- DNS A-kirjed Linuxi serveritele
- CSV fail `kasutajad.csv`
- PowerShelli skript kasutajate importimiseks
- GPO `Teade`, mis kuvab OU `Personal` kasutajatele sisselogimisel teate
- IIS veebiserver
- siseportaali veebileht / lihtne sisuhalduslahendus
- AD autentimine toimetajatele
- SSL sertifikaat
- DNS CNAME kirjed:
  - `siseportaal.oige.local`
  - `siseveeb.oige.local`

---

# 2. Olulised mõisted

## Active Directory ehk AD

Active Directory on Windowsi keskne kataloogiteenus.  
Selle kaudu hallatakse domeeni kasutajaid, arvuteid, gruppe, õiguseid ja poliitikaid.

AD abil saab hallata näiteks:

```text
kasutajaid
arvuteid
gruppe
OU-sid
GPO-sid
domeeni sisselogimist
```

---

## Domeen

Domeen on keskne Windowsi võrk, kus arvutid ja kasutajad alluvad ühisele haldusele.

Selles piletis on domeen:

```text
oige.local
```

---

## Domeenikontroller ehk DC

Domeenikontroller on server, kus töötab Active Directory Domain Services.

Selles piletis:

```text
DC1 = esimene domeenikontroller
DC2 = teine domeenikontroller
```

Kui üks domeenikontroller ei tööta, saab teine domeeniteenuseid edasi pakkuda.

---

## Windows Server Core

Windows Server Core on Windows Serveri käsureapõhine paigaldus.

Server Core’is ei ole:

```text
tavalist desktopi
Start-menüüd
Server Manageri GUI akent
tavalist graafilist haldusliidest
```

Server Core’is kasutatakse:

```text
Command Prompt
PowerShell
sconfig
kaughaldust teisest serverist
```

Server Core’is saab PowerShelli minna käsuga:

```cmd
powershell
```

Põhiseadistuste jaoks saab kasutada:

```cmd
sconfig
```

`sconfig` kaudu saab teha näiteks:

```text
arvuti nime muutmine
IP-aadressi seadistamine
DNS serveri määramine
domeeniga liitmine
remote management seadistamine
restart
```

---

## DNS

DNS muudab nimed IP-aadressideks.

Näide:

```text
dc1.oige.local → 10.0.x.10
siseportaal.oige.local → dc1.oige.local
```

Active Directory vajab DNS-i korrektseks tööks.

Kui DNS on valesti seadistatud, võivad ebaõnnestuda:

```text
domeeniga liitumine
sisselogimine
GPO rakendumine
domeenikontrollerite leidmine
```

---

## DHCP

DHCP jagab klientidele automaatselt võrguseadistusi.

Näiteks:

```text
IP-aadress
subnet mask
gateway
DNS serverid
domeeninimi
```

Selles piletis peab DHCP jagama DNS serveritena välja mõlemad domeenikontrollerid.

---

## DHCP failover

DHCP failover tähendab, et kaks DHCP serverit töötavad koos.

Selles piletis:

```text
DC1 DHCP server
DC2 DHCP failover partner
```

Kui üks DHCP server ei tööta, saab teine klientidele aadresse edasi jagada.

---

## OU ehk Organizational Unit

OU on Active Directory konteiner, kuhu pannakse kasutajad või arvutid.

Piletis on otseselt nõutud:

```text
Kasutajad
Arvutid
```

Lisaks on loogiline luua:

```text
Personal
Toimetajad
```

Põhjus:

- `Personal` OU on vajalik GPO `Teade` jaoks.
- `Toimetajad` OU on vajalik siseportaali toimetajate jaoks.

Soovituslik OU struktuur:

```text
oige.local
├── Kasutajad
│   ├── Personal
│   └── Toimetajad
└── Arvutid
```

---

## GPO

GPO ehk Group Policy Object on domeenipoliitika.

Selles piletis tuleb luua GPO:

```text
Teade
```

See peab OU `Personal` kasutajatele sisselogimisel kuvama teate:

```text
Arvuti kasutamisel tuleb järgida ettevõtte turvapoliitikat ja arvuti kasutamise eeskirja.
```

Selles juhendis tehakse see **kasutaja logon-scriptina**, sest OU `Personal` sisaldab kasutajaid.

---

## IIS

IIS ehk Internet Information Services on Microsofti veebiserver.

Selle abil saab Windows Serveris majutada veebilehte või veebirakendust.

---

## CNAME kirje

CNAME on DNS alias.

Näiteks:

```text
siseportaal.oige.local → dc1.oige.local
siseveeb.oige.local → dc1.oige.local
```

See tähendab, et mitu nime võivad viidata samale serverile.

---

# 3. Näidis IP-plaan

`x` tuleb asendada enda Proxmoxi vmbr numbriga.

| Masin | Roll | IP |
|---|---|---|
| DC1 | Esimene domeenikontroller, DNS, DHCP, IIS | `10.0.x.10` |
| DC2 | Teine domeenikontroller, DNS, DHCP failover | `10.0.x.11` |
| Windows 11 klient | Domeeni klient | DHCP reservation |
| UbuntuServer | Linux server | `10.0.x.20` |
| AlmaServer | Linux server | `10.0.x.21` |
| DebianServer | Linux server | `10.0.x.22` |
| Gateway | Ruuter / võrgulüüs | `10.0.x.1` |

Näidis domeeni info:

```text
Domeen: oige.local
DC1 FQDN: dc1.oige.local
DC2 FQDN: dc2.oige.local
Siseportaal: siseportaal.oige.local
Siseveeb: siseveeb.oige.local
```

---

# 4. Soovituslik tööjärjekord

1. Pane paika IP-plaan.
2. Seadista DC1 staatilise IP-ga.
3. Muuda DC1 nimeks `DC1`.
4. Paigalda DC1 peale AD DS ja DNS roll.
5. Loo uus domeen `oige.local`.
6. Kontrolli DNS-i ja domeeni tööd.
7. Seadista DC2 staatilise IP-ga.
8. Muuda DC2 nimeks `DC2`.
9. Liida DC2 domeeniga.
10. Paigalda DC2 peale AD DS ja DNS roll.
11. Tõsta DC2 teiseks domeenikontrolleriks.
12. Kontrolli replikatsiooni.
13. Paigalda DHCP roll DC1 ja DC2 peale.
14. Loo DHCP scope DC1 serveris.
15. Seadista DHCP lease time 4 tundi.
16. Seadista DHCP DNS serveriteks DC1 ja DC2.
17. Autoriseeri DHCP serverid domeenis.
18. Seadista DHCP failover DC1 ja DC2 vahel.
19. Lisa staatilised DHCP rendid klientarvutitele.
20. Loo OU-d.
21. Loo kasutaja `Haldur`.
22. Lisa `Haldur` gruppi `Domain Admins`.
23. Liida Windows 11 klient domeeniga.
24. Tõsta Windows 11 arvuti OU-sse `Arvutid`.
25. Lisa DNS A-kirjed Linuxi serveritele.
26. Loo või ekspordi CSV fail `kasutajad.csv`.
27. Loo PowerShelli skript AD kasutajate importimiseks.
28. Loo GPO `Teade`.
29. Seo GPO OU-ga `Personal`.
30. Loo logon-script, mis kuvab Personal OU kasutajatele teate.
31. Paigalda IIS.
32. Loo siseportaali veebileht / lihtne sisuhalduslahendus.
33. Loo toimetajate OU ja AD grupp.
34. Seadista toimetajatele AD autentimine ja õigused.
35. Loo SSL sertifikaat.
36. Seo sertifikaat IIS saidiga.
37. Lisa DNS CNAME kirjed `siseportaal` ja `siseveeb`.
38. Kontrolli kogu lahendus.
39. Dokumenteeri kogu protsess.

---

# 5. DC1 võrgu seadistamine

DC1 peab kasutama staatilist IP-aadressi.

Näide:

```text
IP: 10.0.x.10
Mask: 255.255.255.0
Gateway: 10.0.x.1
DNS: 10.0.x.10
```

Alguses võib DC1 DNS serverina kasutada iseennast.

PowerShellis:

```powershell
Get-NetAdapter
```

Näide IP seadistamiseks:

```powershell
New-NetIPAddress `
-InterfaceAlias "Ethernet" `
-IPAddress 10.0.x.10 `
-PrefixLength 24 `
-DefaultGateway 10.0.x.1
```

DNS seadistamine:

```powershell
Set-DnsClientServerAddress `
-InterfaceAlias "Ethernet" `
-ServerAddresses 10.0.x.10
```

Kontroll:

```powershell
ipconfig /all
```

---

# 6. DC1 nime muutmine

```powershell
Rename-Computer -NewName "DC1" -Restart
```

Pärast restarti kontrolli:

```powershell
hostname
```

Oodatav:

```text
DC1
```

---

# 7. AD DS ja DNS rolli paigaldamine DC1 serverisse

PowerShellis:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

Kontroll:

```powershell
Get-WindowsFeature AD-Domain-Services,DNS
```

---

# 8. Uue domeeni loomine `oige.local`

DC1 serveris:

```powershell
Install-ADDSForest `
-DomainName "oige.local" `
-DomainNetbiosName "OIGE" `
-InstallDNS `
-SafeModeAdministratorPassword (Read-Host -AsSecureString "Sisesta DSRM parool") `
-Force
```

Server teeb pärast seda restarti.

Pärast restarti logi sisse domeeni administraatorina:

```text
OIGE\Administrator
```

Kontroll:

```powershell
Get-ADDomain
Get-ADForest
```

---

# 9. DNS kontroll DC1 serveris

Kontrolli, kas domeen lahendub:

```powershell
nslookup oige.local
nslookup dc1.oige.local
```

Kontrolli DNS tsooni:

```powershell
Get-DnsServerZone
```

---

# 10. DC2 seadistamine Server Core’is

NB! **DC2 on Windows Server Core. Seal ei ole tavalist graafilist liidest.**  
Kõik tehakse `sconfig`, Command Prompti või PowerShelliga.

Kui näed ainult käsurida, on see normaalne.

PowerShelli avamiseks kirjuta:

```cmd
powershell
```

Põhiseadistuste menüü avamiseks kirjuta:

```cmd
sconfig
```

---

# 11. DC2 võrgu seadistamine

Näidis IP:

```text
IP: 10.0.x.11
Mask: 255.255.255.0
Gateway: 10.0.x.1
DNS: 10.0.x.10
```

DC2 peab enne domeeniga liitmist kasutama DNS serverina DC1 aadressi.

## Variant A: sconfig

Server Core’is käivita:

```cmd
sconfig
```

Sealt seadista:

```text
Network Settings
Computer Name
Domain/Workgroup
```

## Variant B: PowerShell

Kontrolli võrgukaardi nime:

```powershell
Get-NetAdapter
```

Seadista IP:

```powershell
New-NetIPAddress `
-InterfaceAlias "Ethernet" `
-IPAddress 10.0.x.11 `
-PrefixLength 24 `
-DefaultGateway 10.0.x.1
```

Seadista DNS DC1 peale:

```powershell
Set-DnsClientServerAddress `
-InterfaceAlias "Ethernet" `
-ServerAddresses 10.0.x.10
```

Kontroll:

```powershell
ipconfig /all
```

---

# 12. DC2 nime muutmine

Server Core’is PowerShelliga:

```powershell
Rename-Computer -NewName "DC2" -Restart
```

Pärast restarti kontrolli:

```powershell
hostname
```

Oodatav:

```text
DC2
```

---

# 13. DC2 domeeniga liitmine

DC2 Server Core’is:

```powershell
Add-Computer `
-DomainName "oige.local" `
-Credential "OIGE\Administrator" `
-Restart
```

Pärast restarti logi sisse domeeni kasutajaga.

Kontroll:

```powershell
whoami
```

Oodatav näiteks:

```text
oige\administrator
```

Kontrolli, et DC2 näeb domeeni:

```powershell
nslookup oige.local
nslookup dc1.oige.local
```

---

# 14. AD DS ja DNS rolli paigaldamine DC2 Server Core’i

DC2 Server Core’is:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

Kontroll:

```powershell
Get-WindowsFeature AD-Domain-Services,DNS
```

---

# 15. DC2 teiseks domeenikontrolleriks tõstmine

DC2 Server Core’is:

```powershell
Install-ADDSDomainController `
-DomainName "oige.local" `
-InstallDns `
-Credential (Get-Credential "OIGE\Administrator") `
-SafeModeAdministratorPassword (Read-Host -AsSecureString "Sisesta DSRM parool") `
-Force
```

Server teeb restarti.

Pärast restarti saab DC2 kontrollida DC2 käsurealt või DC1 pealt graafiliste haldustööriistadega.

Kontroll DC1 või DC2 serveris:

```powershell
Get-ADDomainController -Filter *
```

Peaksid nägema:

```text
DC1
DC2
```

Kontrolli replikatsiooni:

```powershell
repadmin /replsummary
```

Kui vigu ei ole, on replikatsioon korras.

---

# 16. DNS seadistamine mõlemas DC-s

Pärast DC2 lisamist võiks DNS serverid olla nii:

DC1:

```text
Preferred DNS: 10.0.x.10
Alternate DNS: 10.0.x.11
```

DC2:

```text
Preferred DNS: 10.0.x.11
Alternate DNS: 10.0.x.10
```

Näide DC1 PowerShellis:

```powershell
Set-DnsClientServerAddress `
-InterfaceAlias "Ethernet" `
-ServerAddresses 10.0.x.10,10.0.x.11
```

Näide DC2 Server Core’is PowerShelliga:

```powershell
Set-DnsClientServerAddress `
-InterfaceAlias "Ethernet" `
-ServerAddresses 10.0.x.11,10.0.x.10
```

Kontroll:

```powershell
ipconfig /all
nslookup dc1.oige.local
nslookup dc2.oige.local
```

---

# 17. DHCP rolli paigaldamine DC1 ja DC2 serverisse

DC1 serveris:

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

DC2 Server Core’is:

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

DHCP autoriseerimine domeenis.

DC1:

```powershell
Add-DhcpServerInDC `
-DnsName "dc1.oige.local" `
-IPAddress 10.0.x.10
```

DC2:

```powershell
Add-DhcpServerInDC `
-DnsName "dc2.oige.local" `
-IPAddress 10.0.x.11
```

Kontroll:

```powershell
Get-DhcpServerInDC
```

---

# 18. DHCP scope loomine DC1 serveris

Näide scope:

```text
Võrk: 10.0.x.0/24
DHCP vahemik: 10.0.x.100 - 10.0.x.200
Mask: 255.255.255.0
Gateway: 10.0.x.1
DNS: 10.0.x.10 ja 10.0.x.11
Lease time: 4 tundi
```

DC1 serveris:

```powershell
Add-DhcpServerv4Scope `
-ComputerName "dc1.oige.local" `
-Name "OIGE LAN" `
-StartRange 10.0.x.100 `
-EndRange 10.0.x.200 `
-SubnetMask 255.255.255.0 `
-State Active
```

Lease time 4 tundi:

```powershell
Set-DhcpServerv4Scope `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-LeaseDuration 04:00:00
```

Gateway:

```powershell
Set-DhcpServerv4OptionValue `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-Router 10.0.x.1
```

DNS serverid ja domeeninimi:

```powershell
Set-DhcpServerv4OptionValue `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-DnsServer 10.0.x.10,10.0.x.11 `
-DnsDomain "oige.local"
```

Kontroll:

```powershell
Get-DhcpServerv4Scope -ComputerName "dc1.oige.local"
Get-DhcpServerv4OptionValue -ComputerName "dc1.oige.local" -ScopeId 10.0.x.0
```

---

# 19. DHCP failover seadistamine

DHCP failover seadistatakse DC1 pealt ja partneriks määratakse DC2.

DC1 serveris:

```powershell
Add-DhcpServerv4Failover `
-ComputerName "dc1.oige.local" `
-Name "OIGE-DHCP-Failover" `
-PartnerServer "dc2.oige.local" `
-ScopeId 10.0.x.0 `
-SharedSecret "TugevSaladus123!" `
-LoadBalancePercent 50 `
-AutoStateTransition $true `
-StateSwitchInterval 00:30:00
```

Kontroll DC1 serveris:

```powershell
Get-DhcpServerv4Failover -ComputerName "dc1.oige.local"
```

Kontroll DC2 Server Core’is:

```powershell
Get-DhcpServerv4Scope -ComputerName "dc2.oige.local"
```

Kui failover töötab, peaks scope olema nähtav ka DC2 poolel.

---

# 20. Staatilised DHCP rendid klientarvutitele

Staatiline rent ehk reservation seob kliendi MAC-aadressi kindla IP-ga.

Windows 11 kliendi MAC-aadressi leiab kliendis:

```powershell
ipconfig /all
```

Linuxi kliendi MAC-aadressi leiab kliendis:

```bash
ip a
```

Näide Windows 11 kliendile:

```powershell
Add-DhcpServerv4Reservation `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-IPAddress 10.0.x.101 `
-ClientId "AA-BB-CC-DD-EE-FF" `
-Description "Windows11 klient"
```

Näide Linuxi kliendile:

```powershell
Add-DhcpServerv4Reservation `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-IPAddress 10.0.x.102 `
-ClientId "11-22-33-44-55-66" `
-Description "Ubuntu klient"
```

Kontroll:

```powershell
Get-DhcpServerv4Reservation `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0
```

NB! Kui reservationid ei ilmu DC2 poolel kohe nähtavale, kontrolli failover replikatsiooni või oota veidi.

---

# 21. OU-de loomine

Nõutud OU-d:

```text
Kasutajad
Arvutid
```

Lisaks luuakse:

```text
Personal
Toimetajad
```

PowerShell DC1 serveris:

```powershell
New-ADOrganizationalUnit `
-Name "Kasutajad" `
-Path "DC=oige,DC=local"
```

```powershell
New-ADOrganizationalUnit `
-Name "Arvutid" `
-Path "DC=oige,DC=local"
```

```powershell
New-ADOrganizationalUnit `
-Name "Personal" `
-Path "OU=Kasutajad,DC=oige,DC=local"
```

```powershell
New-ADOrganizationalUnit `
-Name "Toimetajad" `
-Path "OU=Kasutajad,DC=oige,DC=local"
```

Kontroll:

```powershell
Get-ADOrganizationalUnit -Filter * |
Select-Object Name,DistinguishedName
```

---

# 22. Kasutaja `Haldur` loomine ja Domain Admins gruppi lisamine

```powershell
New-ADUser `
-Name "Haldur" `
-SamAccountName "haldur" `
-UserPrincipalName "haldur@oige.local" `
-Path "OU=Kasutajad,DC=oige,DC=local" `
-AccountPassword (Read-Host -AsSecureString "Sisesta Haldur parool") `
-Enabled $true
```

Lisa kasutaja Domain Admins gruppi:

```powershell
Add-ADGroupMember `
-Identity "Domain Admins" `
-Members "haldur"
```

Kontroll:

```powershell
Get-ADGroupMember "Domain Admins"
```

---

# 23. Windows 11 kliendi domeeniga liitmine

Windows 11 kliendis peab DNS olema domeenikontrollerite peale suunatud.

Kontroll Windows 11 kliendis:

```powershell
ipconfig /all
```

DNS peab olema näiteks:

```text
10.0.x.10
10.0.x.11
```

Domeeniga liitmine GUI kaudu:

```text
Settings
System
About
Domain or workgroup
Join domain
oige.local
```

Või PowerShelliga administraatorina:

```powershell
Add-Computer `
-DomainName "oige.local" `
-Credential "OIGE\Haldur" `
-Restart
```

Pärast restarti logi sisse domeeni kasutajaga:

```text
OIGE\Haldur
```

---

# 24. Windows 11 arvuti liigutamine OU-sse `Arvutid`

DC1 serveris:

```powershell
Get-ADComputer -Filter * |
Select-Object Name,DistinguishedName
```

Leia Windows 11 arvuti nimi.

Näide, kui arvuti nimi on `WIN11`:

```powershell
Move-ADObject `
-Identity "CN=WIN11,CN=Computers,DC=oige,DC=local" `
-TargetPath "OU=Arvutid,DC=oige,DC=local"
```

NB! Asenda `WIN11` päris arvutinimega.

Kontroll:

```powershell
Get-ADComputer WIN11 -Properties DistinguishedName |
Select-Object Name,DistinguishedName
```

---

# 25. DNS A-kirjed Linuxi serveritele

DNS serveris tuleb teha A-kirjed kõigi pileti Linuxi serverite jaoks.

Näide:

```powershell
Add-DnsServerResourceRecordA `
-ZoneName "oige.local" `
-Name "ubuntu" `
-IPv4Address "10.0.x.20"
```

```powershell
Add-DnsServerResourceRecordA `
-ZoneName "oige.local" `
-Name "alma" `
-IPv4Address "10.0.x.21"
```

```powershell
Add-DnsServerResourceRecordA `
-ZoneName "oige.local" `
-Name "debian" `
-IPv4Address "10.0.x.22"
```

Kontroll:

```powershell
Resolve-DnsName ubuntu.oige.local
Resolve-DnsName alma.oige.local
Resolve-DnsName debian.oige.local
```

Või:

```powershell
nslookup ubuntu.oige.local
nslookup alma.oige.local
nslookup debian.oige.local
```

---

# 26. CSV faili näidis `kasutajad.csv`

CSV fail võiks olla näiteks sellise kujuga:

```csv
Eesnimi,Perenimi,Kasutajanimi,Parool,OU
Mari,Tamm,mari.tamm,Parool123!,Personal
Jaan,Kask,jaan.kask,Parool123!,Personal
Tiina,Sepp,tiina.sepp,Parool123!,Toimetajad
Karl,Saar,karl.saar,Parool123!,Toimetajad
```

Faili nimi:

```text
kasutajad.csv
```

Näiteks asukoht:

```text
C:\Scripts\kasutajad.csv
```

---

# 27. PowerShelli skript kasutajate importimiseks CSV failist

Loo kaust:

```powershell
mkdir C:\Scripts
```

Loo fail:

```powershell
notepad C:\Scripts\impordi_kasutajad.ps1
```

Skripti sisu:

```powershell
Import-Module ActiveDirectory

$csvPath = "C:\Scripts\kasutajad.csv"
$domainPath = "DC=oige,DC=local"
$baseOU = "OU=Kasutajad,$domainPath"

$users = Import-Csv -Path $csvPath

foreach ($user in $users) {

    $ouName = $user.OU
    $targetOU = "OU=$ouName,$baseOU"

    # Kontrollib, kas OU on olemas. Kui ei ole, loob selle.
    if (-not (Get-ADOrganizationalUnit -LDAPFilter "(ou=$ouName)" -SearchBase $baseOU -ErrorAction SilentlyContinue)) {
        New-ADOrganizationalUnit -Name $ouName -Path $baseOU
    }

    $fullName = "$($user.Eesnimi) $($user.Perenimi)"
    $sam = $user.Kasutajanimi
    $upn = "$sam@oige.local"

    # Kontrollib, kas kasutaja on juba olemas.
    if (-not (Get-ADUser -Filter "SamAccountName -eq '$sam'" -ErrorAction SilentlyContinue)) {

        New-ADUser `
            -Name $fullName `
            -GivenName $user.Eesnimi `
            -Surname $user.Perenimi `
            -SamAccountName $sam `
            -UserPrincipalName $upn `
            -Path $targetOU `
            -AccountPassword (ConvertTo-SecureString $user.Parool -AsPlainText -Force) `
            -Enabled $true `
            -ChangePasswordAtLogon $true

        Write-Host "Loodi kasutaja: $sam"
    }
    else {
        Write-Host "Kasutaja on juba olemas: $sam"
    }
}
```

Käivita PowerShell administraatorina.

Kui skriptide käivitamine on keelatud:

```powershell
Set-ExecutionPolicy RemoteSigned -Scope Process
```

Käivita skript:

```powershell
C:\Scripts\impordi_kasutajad.ps1
```

Kontroll:

```powershell
Get-ADUser `
-Filter * `
-SearchBase "OU=Kasutajad,DC=oige,DC=local" |
Select-Object Name,SamAccountName
```

---

# 28. GPO `Teade` loomine

GPO eesmärk:

OU `Personal` kasutajatele kuvatakse sisselogimisel teade:

```text
Arvuti kasutamisel tuleb järgida ettevõtte turvapoliitikat ja arvuti kasutamise eeskirja.
```

Kuna `Personal` on kasutajate OU, on kõige loogilisem teha see kasutaja logon-scriptina.

Loo GPO:

```powershell
New-GPO -Name "Teade"
```

Seo GPO OU-ga `Personal`:

```powershell
New-GPLink `
-Name "Teade" `
-Target "OU=Personal,OU=Kasutajad,DC=oige,DC=local"
```

---

# 29. Logon-script teate kuvamiseks

Loo skriptikaust SYSVOL-is:

```powershell
mkdir "\\oige.local\SYSVOL\oige.local\scripts"
```

Loo PowerShelli logon-script:

```powershell
notepad "\\oige.local\SYSVOL\oige.local\scripts\teade.ps1"
```

Sisu:

```powershell
Add-Type -AssemblyName PresentationFramework

[System.Windows.MessageBox]::Show(
    "Arvuti kasutamisel tuleb järgida ettevõtte turvapoliitikat ja arvuti kasutamise eeskirja.",
    "AS Oige teade",
    "OK",
    "Information"
)
```

Loo wrapper `.bat` fail, sest GPO logon scripti kaudu on lihtne käivitada BAT faili:

```powershell
notepad "\\oige.local\SYSVOL\oige.local\scripts\teade.bat"
```

Sisu:

```bat
powershell.exe -ExecutionPolicy Bypass -WindowStyle Hidden -File "\\oige.local\SYSVOL\oige.local\scripts\teade.ps1"
```

## GPO-sse lisamine GUI kaudu DC1 serveris

Ava DC1 serveris:

```text
Group Policy Management
```

Seejärel:

```text
Forest: oige.local
Domains
oige.local
Kasutajad
Personal
Teade
Edit
User Configuration
Policies
Windows Settings
Scripts (Logon/Logoff)
Logon
Add
```

Lisa script:

```text
\\oige.local\SYSVOL\oige.local\scripts\teade.bat
```

Kontroll kliendis Personal OU kasutajaga:

```powershell
gpupdate /force
```

Logi välja ja sisse tagasi.

Kontroll:

```powershell
gpresult /r
```

GPO `Teade` peab olema rakendunud kasutaja poolel.

---

# 30. IIS rolli paigaldamine Windows Server 2025 peale

Eeldus: IIS paigaldatakse DC1 serverisse, kuna pilet ütleb, et kasutatakse olemasolevat Windows Server 2025 masinat.

NB! Tootmiskeskkonnas ei ole soovitatav veebiserverit domeenikontrolleris jooksutada, aga eksamiülesandes kasutatakse olemasolevaid ressursse ja pilet nõuab Windows Server 2025 serverit.

DC1 serveris:

```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools
```

Lisaks AD autentimise jaoks:

```powershell
Install-WindowsFeature Web-Windows-Auth
```

IIS PowerShelli haldamiseks:

```powershell
Import-Module WebAdministration
```

Kontroll:

```powershell
Get-WindowsFeature Web-Server,Web-Windows-Auth
```

Test brauseris:

```text
http://dc1.oige.local
```

Või PowerShellis:

```powershell
Invoke-WebRequest http://localhost
```

---

# 31. Siseportaali / sisuhalduslahenduse valik

Eksami jaoks on kõige stabiilsem ja kaitstav lahendus:

```text
IIS veebileht + AD autentimine + toimetajate AD grupp + NTFS muutmisõigused
```

See tähendab:

- siseportaal jookseb IIS-is;
- kasutajad autenditakse AD kaudu;
- toimetajad asuvad eraldi OU-s;
- toimetajate grupp saab veebilehe sisu muuta;
- tavakasutajatele saab anda ainult lugemisõiguse.

Kui hindaja nõuab tingimata eraldi valmis CMS tarkvara, võib kasutada näiteks:

```text
Orchard Core
Umbraco
Piranha CMS
```

Kuid need vajavad rohkem eeldusi ja võtavad eksamil rohkem aega.

Selles juhendis kasutatakse lihtsat siseportaali, kus sisu hallatakse IIS-i kaustas AD õiguste kaudu.

---

# 32. Toimetajate OU ja grupp

Loo OU `Toimetajad`, kui seda pole veel tehtud:

```powershell
New-ADOrganizationalUnit `
-Name "Toimetajad" `
-Path "OU=Kasutajad,DC=oige,DC=local"
```

Loo grupp:

```powershell
New-ADGroup `
-Name "Siseportaali_Toimetajad" `
-SamAccountName "Siseportaali_Toimetajad" `
-GroupScope Global `
-GroupCategory Security `
-Path "OU=Toimetajad,OU=Kasutajad,DC=oige,DC=local"
```

Lisa toimetaja kasutaja gruppi:

```powershell
Add-ADGroupMember `
-Identity "Siseportaali_Toimetajad" `
-Members "tiina.sepp"
```

Kontroll:

```powershell
Get-ADGroupMember "Siseportaali_Toimetajad"
```

---

# 33. Veebilehe kausta loomine

Loo kaust:

```powershell
mkdir C:\Sites\Siseportaal
```

Loo testleht:

```powershell
notepad C:\Sites\Siseportaal\index.html
```

Näidis sisu:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>AS Oige siseportaal</title>
</head>
<body>
    <h1>AS Oige siseportaal</h1>
    <p>Tere tulemast ettevõtte siseportaali.</p>
</body>
</html>
```

---

# 34. IIS saidi loomine

Laadi IIS PowerShell moodul:

```powershell
Import-Module WebAdministration
```

Peata Default Web Site, kui see kasutab porti 80:

```powershell
Stop-Website -Name "Default Web Site"
```

Loo uus sait:

```powershell
New-Website `
-Name "Siseportaal" `
-PhysicalPath "C:\Sites\Siseportaal" `
-Port 80 `
-HostHeader "siseportaal.oige.local"
```

Lisa teine HTTP binding `siseveeb.oige.local` jaoks:

```powershell
New-WebBinding `
-Name "Siseportaal" `
-Protocol "http" `
-Port 80 `
-HostHeader "siseveeb.oige.local"
```

Kontroll:

```powershell
Get-Website
Get-WebBinding -Name "Siseportaal"
```

---

# 35. AD autentimise seadistamine IIS-is

Lülita saidil Anonymous Authentication välja ja Windows Authentication sisse.

```powershell
Set-WebConfigurationProperty `
-Filter "/system.webServer/security/authentication/anonymousAuthentication" `
-Name enabled `
-Value false `
-PSPath "IIS:\Sites\Siseportaal"
```

```powershell
Set-WebConfigurationProperty `
-Filter "/system.webServer/security/authentication/windowsAuthentication" `
-Name enabled `
-Value true `
-PSPath "IIS:\Sites\Siseportaal"
```

Kontroll:

```powershell
Get-WebConfigurationProperty `
-Filter "/system.webServer/security/authentication/anonymousAuthentication" `
-Name enabled `
-PSPath "IIS:\Sites\Siseportaal"
```

```powershell
Get-WebConfigurationProperty `
-Filter "/system.webServer/security/authentication/windowsAuthentication" `
-Name enabled `
-PSPath "IIS:\Sites\Siseportaal"
```

---

# 36. NTFS õigused siseportaali kaustale

Tavakasutajatele võib anda lugemisõiguse:

```powershell
icacls "C:\Sites\Siseportaal" /grant "OIGE\Domain Users:(OI)(CI)RX"
```

Toimetajate grupile anna muutmisõigus:

```powershell
icacls "C:\Sites\Siseportaal" /grant "OIGE\Siseportaali_Toimetajad:(OI)(CI)M"
```

Selgitus:

```text
RX = Read and Execute
M = Modify
OI = Object Inherit
CI = Container Inherit
```

Kontroll:

```powershell
icacls "C:\Sites\Siseportaal"
```

---

# 37. SSL sertifikaadi loomine

Loo self-signed sertifikaat:

```powershell
$cert = New-SelfSignedCertificate `
-DnsName "siseportaal.oige.local","siseveeb.oige.local" `
-CertStoreLocation "cert:\LocalMachine\My" `
-FriendlyName "AS Oige Siseportaal"
```

Kontroll:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
Where-Object {$_.FriendlyName -eq "AS Oige Siseportaal"}
```

---

# 38. HTTPS binding IIS saidile

Lisa HTTPS binding `siseportaal.oige.local` jaoks:

```powershell
New-WebBinding `
-Name "Siseportaal" `
-Protocol "https" `
-Port 443 `
-HostHeader "siseportaal.oige.local" `
-SslFlags 1
```

Seo sertifikaat bindinguga:

```powershell
New-Item `
-Path "IIS:\SslBindings\0.0.0.0!443!siseportaal.oige.local" `
-Thumbprint $cert.Thumbprint `
-SSLFlags 1
```

Lisa HTTPS binding `siseveeb.oige.local` jaoks:

```powershell
New-WebBinding `
-Name "Siseportaal" `
-Protocol "https" `
-Port 443 `
-HostHeader "siseveeb.oige.local" `
-SslFlags 1
```

Seo sertifikaat bindinguga:

```powershell
New-Item `
-Path "IIS:\SslBindings\0.0.0.0!443!siseveeb.oige.local" `
-Thumbprint $cert.Thumbprint `
-SSLFlags 1
```

Kontroll:

```powershell
Get-WebBinding -Name "Siseportaal"
```

---

# 39. DNS CNAME kirjed siseportaali jaoks

Kuna veebileht asub DC1 serveris, võib teha CNAME kirjed DC1 peale.

```powershell
Add-DnsServerResourceRecordCName `
-ZoneName "oige.local" `
-Name "siseportaal" `
-HostNameAlias "dc1.oige.local"
```

```powershell
Add-DnsServerResourceRecordCName `
-ZoneName "oige.local" `
-Name "siseveeb" `
-HostNameAlias "dc1.oige.local"
```

Kontroll:

```powershell
nslookup siseportaal.oige.local
nslookup siseveeb.oige.local
```

---

# 40. Windows Firewall IIS jaoks

Kui vaja, luba HTTP ja HTTPS:

```powershell
New-NetFirewallRule `
-DisplayName "Allow HTTP" `
-Direction Inbound `
-Protocol TCP `
-LocalPort 80 `
-Action Allow
```

```powershell
New-NetFirewallRule `
-DisplayName "Allow HTTPS" `
-Direction Inbound `
-Protocol TCP `
-LocalPort 443 `
-Action Allow
```

Kontroll:

```powershell
Get-NetFirewallRule -DisplayName "Allow HTTP","Allow HTTPS"
```

---

# 41. Kontrollnimekiri

## Domeen

```powershell
Get-ADDomain
Get-ADForest
```

Oodatav domeen:

```text
oige.local
```

---

## Domeenikontrollerid

```powershell
Get-ADDomainController -Filter *
```

Peaksid olema:

```text
DC1
DC2
```

---

## Replikatsioon

```powershell
repadmin /replsummary
```

Vigade arv peaks olema 0 või tuleb vead lahendada.

---

## DNS

```powershell
nslookup dc1.oige.local
nslookup dc2.oige.local
nslookup ubuntu.oige.local
nslookup alma.oige.local
nslookup debian.oige.local
nslookup siseportaal.oige.local
nslookup siseveeb.oige.local
```

---

## DHCP

```powershell
Get-DhcpServerInDC
Get-DhcpServerv4Scope -ComputerName "dc1.oige.local"
Get-DhcpServerv4OptionValue -ComputerName "dc1.oige.local" -ScopeId 10.0.x.0
Get-DhcpServerv4Failover -ComputerName "dc1.oige.local"
Get-DhcpServerv4Reservation -ComputerName "dc1.oige.local" -ScopeId 10.0.x.0
```

Kontrolli, et:

```text
lease time = 4 tundi
DNS serverid = DC1 ja DC2
failover partner = DC2
reservationid olemas klientidele
```

---

## OU-d

```powershell
Get-ADOrganizationalUnit -Filter * |
Select-Object Name,DistinguishedName
```

Peavad olemas olema:

```text
Kasutajad
Arvutid
Personal
Toimetajad
```

---

## Haldur kasutaja

```powershell
Get-ADUser haldur
Get-ADGroupMember "Domain Admins"
```

---

## Windows 11 klient domeenis

Windows 11 kliendis:

```powershell
whoami
systeminfo | findstr /B /C:"Domain"
```

Või:

```powershell
Get-ComputerInfo | Select-Object CsDomain
```

---

## GPO

Kliendis Personal OU kasutajaga:

```powershell
gpupdate /force
gpresult /r
```

Kontrolli, et GPO `Teade` rakendub kasutaja poolel.

Seejärel logi välja ja sisse tagasi.  
Sisselogimisel peab ilmuma teade.

---

## IIS

DC1 serveris:

```powershell
Get-Website
Get-WebBinding -Name "Siseportaal"
```

Kliendist:

```text
http://siseportaal.oige.local
https://siseportaal.oige.local
https://siseveeb.oige.local
```

---

## Sertifikaat

```powershell
Get-ChildItem Cert:\LocalMachine\My |
Where-Object {$_.FriendlyName -eq "AS Oige Siseportaal"}
```

---

# 42. Dokumentatsiooni struktuur

Dokumentatsiooni võiks teha selliste peatükkidega:

## 1. IP-plaan

Kirjuta välja kõik masinad ja IP-d.

Näide:

```text
DC1: 10.0.x.10
DC2: 10.0.x.11
UbuntuServer: 10.0.x.20
AlmaServer: 10.0.x.21
DebianServer: 10.0.x.22
Gateway: 10.0.x.1
```

---

## 2. Active Directory

Kirjuta:

```text
Domeeniks loodi oige.local.
Esimeseks domeenikontrolleriks seadistati DC1.
Teiseks domeenikontrolleriks seadistati DC2 Server Core.
Mõlemale domeenikontrollerile paigaldati DNS roll.
DC2 seadistati PowerShelli ja sconfig tööriista abil.
```

Lisa kontrollkäsud:

```powershell
Get-ADDomain
Get-ADDomainController -Filter *
repadmin /replsummary
```

---

## 3. DHCP

Kirjuta:

```text
DHCP scope loodi võrgule 10.0.x.0/24.
Rendi kestuseks määrati 4 tundi.
DNS serveritena jagatakse klientidele DC1 ja DC2 aadressid.
DHCP failover seadistati DC1 ja DC2 vahel.
Klientarvutitele lisati staatilised DHCP rendid.
```

Lisa kontrollkäsud:

```powershell
Get-DhcpServerv4Scope -ComputerName "dc1.oige.local"
Get-DhcpServerv4Failover -ComputerName "dc1.oige.local"
Get-DhcpServerv4Reservation -ComputerName "dc1.oige.local" -ScopeId 10.0.x.0
```

---

## 4. OU-d ja kasutajad

Kirjuta:

```text
Loodi OU-d Kasutajad ja Arvutid.
Lisaks loodi Personal ja Toimetajad OU-d.
Loodi kasutaja Haldur ja lisati Domain Admins gruppi.
```

---

## 5. Windows 11 klient

Kirjuta:

```text
Windows 11 klient liideti domeeniga oige.local ja arvutiobjekt liigutati OU-sse Arvutid.
```

---

## 6. DNS

Kirjuta:

```text
DNS serverisse lisati A-kirjed Linuxi serveritele.
Lisaks loodi CNAME kirjed siseportaal.oige.local ja siseveeb.oige.local.
```

---

## 7. CSV import

Kirjuta:

```text
Kasutajate andmed eksporditi faili kasutajad.csv.
PowerShelli skript impordi_kasutajad.ps1 loob CSV põhjal OU struktuuri ja kasutajad.
```

Lisa skript või oluline osa sellest dokumentatsiooni.

---

## 8. GPO

Kirjuta:

```text
Loodi GPO nimega Teade.
GPO seoti OU-ga Personal.
GPO käivitab kasutaja sisselogimisel logon-scripti, mis kuvab teate ettevõtte turvapoliitika ja arvuti kasutamise eeskirja järgimise kohta.
```

---

## 9. IIS ja siseportaal

Kirjuta:

```text
DC1 serverisse paigaldati IIS roll.
Loodi veebileht Siseportaal.
Veebilehele seadistati Windows Authentication.
Toimetajate õigused anti AD grupile Siseportaali_Toimetajad.
```

---

## 10. SSL

Kirjuta:

```text
Veebilehele loodi self-signed SSL sertifikaat nimedele siseportaal.oige.local ja siseveeb.oige.local.
Sertifikaat seoti IIS HTTPS bindingutega.
```

---

# 43. Tüüpilised probleemid ja lahendused

## Probleem: DC2-s ei ole graafilist liidest

See on normaalne, sest DC2 on Server Core.

Kasuta:

```cmd
sconfig
```

või:

```cmd
powershell
```

DC2 saab hiljem hallata ka DC1 pealt graafiliste tööriistadega.

---

## Probleem: Windows 11 klient ei saa domeeniga liituda

Kontrolli kliendis DNS-i:

```powershell
ipconfig /all
```

DNS peab olema:

```text
10.0.x.10
10.0.x.11
```

Kontrolli nime lahendamist:

```powershell
nslookup oige.local
nslookup dc1.oige.local
```

Kui DNS ei tööta, ei tööta ka domeeniga liitumine.

---

## Probleem: DC2 ei saa domeenikontrolleriks

Kontrolli DC2 Server Core’is:

```powershell
nslookup oige.local
nslookup dc1.oige.local
ping dc1.oige.local
```

DC2 peab kasutama DNS serverina DC1 aadressi enne domeeni liitmist.

---

## Probleem: DHCP ei jaga aadresse

Kontrolli, kas DHCP on autoriseeritud:

```powershell
Get-DhcpServerInDC
```

Kontrolli scope:

```powershell
Get-DhcpServerv4Scope -ComputerName "dc1.oige.local"
```

Kontrolli, kas scope on aktiivne:

```powershell
Set-DhcpServerv4Scope `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-State Active
```

---

## Probleem: DHCP failover ei tööta

Kontrolli:

```powershell
Get-DhcpServerv4Failover -ComputerName "dc1.oige.local"
```

Kontrolli, kas DC1 ja DC2 näevad üksteist:

```powershell
ping dc2.oige.local
ping dc1.oige.local
```

Kontrolli tulemüüri ja DNS-i.

---

## Probleem: GPO ei rakendu

Kliendis:

```powershell
gpupdate /force
gpresult /r
```

Kontrolli:

```text
kas kasutaja on õiges OU-s
kas GPO on lingitud OU Personal külge
kas klient kasutab õiget DNS serverit
kas kasutaja logis uuesti sisse
kas script asub SYSVOL kaustas
kas scripti tee on õigesti lisatud
```

---

## Probleem: DNS CNAME ei tööta

Kontrolli:

```powershell
nslookup siseportaal.oige.local
nslookup siseveeb.oige.local
```

Kui ei tööta, kontrolli DNS kirjeid:

```powershell
Get-DnsServerResourceRecord -ZoneName "oige.local"
```

---

## Probleem: IIS leht ei avane

Kontrolli, kas sait töötab:

```powershell
Get-Website
```

Kontrolli bindinguid:

```powershell
Get-WebBinding -Name "Siseportaal"
```

Kontrolli tulemüüri:

```powershell
Get-NetFirewallRule -DisplayName "Allow HTTP","Allow HTTPS"
```

Testi serveris lokaalselt:

```powershell
Invoke-WebRequest http://localhost
```

---

## Probleem: HTTPS annab sertifikaadi hoiatuse

Self-signed sertifikaadiga on see testkeskkonnas normaalne.

Kui tahad, et klient usaldaks sertifikaati, tuleb sertifikaat lisada kliendi Trusted Root Certification Authorities alla.

---

## Probleem: Windows Authentication ei tööta

Kontrolli, kas roll on paigaldatud:

```powershell
Get-WindowsFeature Web-Windows-Auth
```

Kontrolli IIS seadistust:

```powershell
Get-WebConfigurationProperty `
-Filter "/system.webServer/security/authentication/windowsAuthentication" `
-Name enabled `
-PSPath "IIS:\Sites\Siseportaal"
```

Kontrolli, et Anonymous Authentication oleks välja lülitatud.

---

## Probleem: HTTPS binding annab vea

Kontrolli, kas sertifikaat on olemas:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
Where-Object {$_.FriendlyName -eq "AS Oige Siseportaal"}
```

Kontrolli, kas binding on olemas:

```powershell
Get-WebBinding -Name "Siseportaal"
```

Kui binding on valesti loodud, eemalda vale binding ja loo uuesti.

---

# 44. Väga lühike kokkuvõte

Selle pileti lahendus kõige lihtsamalt:

```text
1. Teen DC1 serverist esimese domeenikontrolleri domeenile oige.local.
2. Paigaldan DNS rolli.
3. Seadistan DC2 Server Core’is käsurealt.
4. Liidan DC2 domeeniga.
5. Teen DC2 serverist teise domeenikontrolleri.
6. Paigaldan DHCP rolli DC1 ja DC2 serverisse.
7. Loon DHCP scope'i ja seadistan lease time 4 tunniks.
8. Seadistan DHCP failoveri DC1 ja DC2 vahel.
9. Lisan staatilised DHCP rendid klientidele.
10. Loon OU-d Kasutajad, Arvutid, Personal ja Toimetajad.
11. Loon kasutaja Haldur ja lisan Domain Admins gruppi.
12. Liidan Windows 11 kliendi domeeniga ja liigutan OU-sse Arvutid.
13. Lisan DNS A-kirjed Linuxi serveritele.
14. Teen CSV faili ja PowerShelli skripti kasutajate importimiseks.
15. Loon GPO Teade ja seon selle Personal OU-ga.
16. Loon logon-scripti, mis kuvab Personal OU kasutajatele teate.
17. Paigaldan IIS rolli.
18. Loon siseportaali veebilehe.
19. Seadistan Windows Authenticationi.
20. Annan toimetajatele õigused AD grupi kaudu.
21. Loon SSL sertifikaadi.
22. Seon sertifikaadi IIS saidiga.
23. Lisan DNS CNAME kirjed siseportaal ja siseveeb.
24. Kontrollin DNS-i, DHCP-d, AD-d, GPO-d ja IIS-i.
25. Dokumenteerin kogu protsessi.
```
# Windows. Pilet 2 — Active Directory, DHCP, DNS, DFS, FSRM ja GPO võrgukettad

## Märksõnad

- Active Directory kataloogiteenused
- Windows Server Core
- DNS serveri haldus
- DHCP serveri haldus
- DHCP failover
- OU struktuur
- AD kasutajate import CSV failist
- Group Policy Object ehk GPO
- DFS Namespaces
- DFS Replication
- File Server Resource Manager ehk FSRM
- Võrgukettad
- Failitüüpide piirangud
- Kvoodid

---

# 1. Ülesande lühikokkuvõte

AS Oige vajab Windowsi-põhist terviklahendust.

Tuleb luua Active Directory domeen:

```text
oige.local
```

Domeenikontrollerid:

```text
DC1 = Windows Server 2025 Desktop Experience ehk graafilise liidesega server
DC2 = Windows Server Core 2025 ehk käsureapõhine server
```

NB! **DC2 Server Core’il ei ole tavalist graafilist kasutajaliidest ega Server Manageri akent.**  
DC2 seadistamine tehakse:

```text
sconfig
PowerShell
kaugelt DC1 haldustööriistade kaudu
```

Lisaks tuleb seadistada:

- DNS roll mõlemale domeenikontrollerile
- DHCP roll DC1 ja DC2 serverile
- DHCP failover DC1 ja DC2 vahel
- staatilised DHCP rendid klientarvutitele
- DHCP rendi kestus 4 tundi
- DHCP jagab DNS serveritena välja DC1 ja DC2
- OU-d `Kasutajad` ja `Arvutid`
- kasutaja `Haldur`, kes kuulub gruppi `Domain Admins`
- Windows 11 klient domeeni ja OU-sse `Arvutid`
- DNS A-kirjed Linuxi serveritele
- CSV fail `kasutajad.csv`
- PowerShelli skript kasutajate importimiseks
- GPO `Teade`, mis kuvab OU `Personal` kasutajatele sisselogimisel teate

Pileti teine osa:

- Paigaldada DC1 ja DC2 serverisse:
  - DFS Namespaces
  - DFS Replication
  - File Server Resource Manager
- Luua DFS nimeruum:

```text
\\oige.local\Jagatud
```

- Luua DFS nimeruumi kaust:

```text
\\oige.local\Jagatud\Kogukond
```

- Luua sellele replikatsioon DC1 ja DC2 vahel
- Jagada `Kogukond` kõigile domeeni kasutajatele võrgukettana:

```text
Y:
```

- Määrata `Kogukond` ressursile FSRM abil mahupiirang:

```text
10 GB
```

- Keelata sinna programmifailide kopeerimine:

```text
.msi
.exe
.bat
.ps1
```

- Luua DFS nimeruumi kaust:

```text
\\oige.local\Jagatud\Isiklik
```

- Luua sellele replikatsioon DC1 ja DC2 vahel
- Luua igale kasutajale tema kasutajanimega kaust
- Jagada ainult tema enda kaust võrgukettana:

```text
Z:
```

- Määrata isiklikele kasutajakaustadele FSRM abil mahupiirang:

```text
1 GB kasutaja kohta
```

---

# 2. Olulised mõisted

## Active Directory ehk AD

Active Directory on Windowsi keskne kataloogiteenus.  
Selle kaudu hallatakse domeeni kasutajaid, arvuteid, gruppe, õiguseid ja poliitikaid.

---

## Windows Server Core

Windows Server Core on Windows Serveri käsureapõhine paigaldus.

Server Core’is ei ole:

```text
tavalist desktopi
Start-menüüd
Server Manageri GUI akent
tavalist graafilist haldusliidest
```

Server Core’is kasutatakse:

```text
Command Prompt
PowerShell
sconfig
kaughaldust teisest serverist
```

PowerShelli avamiseks:

```cmd
powershell
```

Põhiseadistuste jaoks:

```cmd
sconfig
```

---

## DFS Namespaces

DFS Namespace annab kasutajatele ühe loogilise tee, kuigi tegelikud kaustad võivad asuda mitmes serveris.

Näiteks kasutaja näeb:

```text
\\oige.local\Jagatud\Kogukond
```

Aga tegelikult võivad failid asuda:

```text
\\DC1\Kogukond$
\\DC2\Kogukond$
```

Kasutaja ei pea teadma, millises serveris kaust päriselt asub.

---

## DFS Replication

DFS Replication sünkroonib kaustade sisu mitme serveri vahel.

Näiteks:

```text
DC1 D:\DFSData\Kogukond
      ↕
DC2 D:\DFSData\Kogukond
```

Kui faili muudetakse ühes serveris, kopeeritakse muutus ka teise serverisse.

NB! DFS Namespace ja DFS Replication ei ole sama asi.

```text
DFS Namespace = ilus ühine tee kasutajatele
DFS Replication = failide sünkroonimine serverite vahel
```

---

## FSRM

FSRM ehk File Server Resource Manager võimaldab failiserveris kehtestada:

```text
mahupiiranguid
failitüüpide piiranguid
raporteid
failihaldusreegleid
```

Selles piletis kasutatakse FSRM-i:

```text
Kogukond kaustale 10 GB piirang
Isiklik kasutajakaustadele 1 GB kasutaja kohta
programmifailide blokeerimine Kogukond kaustas
```

---

## Võrguketas

Võrguketas on jagatud võrgukaust, mis kuvatakse kasutaja arvutis kettatähena.

Selles piletis:

```text
Y: = Kogukond
Z: = kasutaja isiklik kaust
```

---

# 3. Näidis IP-plaan

`x` tuleb asendada enda Proxmoxi vmbr numbriga.

| Masin | Roll | IP |
|---|---|---|
| DC1 | Esimene domeenikontroller, DNS, DHCP, DFS, FSRM | `10.0.x.10` |
| DC2 | Teine domeenikontroller, DNS, DHCP failover, DFS, FSRM | `10.0.x.11` |
| Windows 11 klient | Domeeni klient | DHCP reservation |
| UbuntuServer | Linux server | `10.0.x.20` |
| AlmaServer | Linux server | `10.0.x.21` |
| DebianServer | Linux server | `10.0.x.22` |
| Gateway | Ruuter / võrgulüüs | `10.0.x.1` |

Näidis domeeni info:

```text
Domeen: oige.local
DC1 FQDN: dc1.oige.local
DC2 FQDN: dc2.oige.local
DFS namespace: \\oige.local\Jagatud
Kogukond: \\oige.local\Jagatud\Kogukond
Isiklik: \\oige.local\Jagatud\Isiklik
```

---

# 4. Soovituslik tööjärjekord

1. Pane paika IP-plaan.
2. Seadista DC1 staatilise IP-ga.
3. Muuda DC1 nimeks `DC1`.
4. Paigalda DC1 peale AD DS ja DNS roll.
5. Loo uus domeen `oige.local`.
6. Kontrolli DNS-i ja domeeni tööd.
7. Seadista DC2 Server Core staatilise IP-ga.
8. Muuda DC2 nimeks `DC2`.
9. Liida DC2 domeeniga.
10. Paigalda DC2 peale AD DS ja DNS roll.
11. Tõsta DC2 teiseks domeenikontrolleriks.
12. Kontrolli replikatsiooni.
13. Paigalda DHCP roll DC1 ja DC2 peale.
14. Loo DHCP scope DC1 serveris.
15. Seadista DHCP lease time 4 tundi.
16. Seadista DHCP DNS serveriteks DC1 ja DC2.
17. Autoriseeri DHCP serverid domeenis.
18. Seadista DHCP failover DC1 ja DC2 vahel.
19. Lisa staatilised DHCP rendid klientarvutitele.
20. Loo OU-d.
21. Loo kasutaja `Haldur`.
22. Lisa `Haldur` gruppi `Domain Admins`.
23. Liida Windows 11 klient domeeniga.
24. Tõsta Windows 11 arvuti OU-sse `Arvutid`.
25. Lisa DNS A-kirjed Linuxi serveritele.
26. Loo või ekspordi CSV fail `kasutajad.csv`.
27. Loo PowerShelli skript AD kasutajate importimiseks.
28. Loo GPO `Teade`.
29. Seo GPO OU-ga `Personal`.
30. Loo logon-script, mis kuvab Personal OU kasutajatele teate.
31. Paigalda DC1 ja DC2 peale DFS ja FSRM rollid.
32. Loo DC1 ja DC2 peal DFS andmekaustad.
33. Loo DFS namespace `\\oige.local\Jagatud`.
34. Lisa namespace root target ka DC2 serverisse.
35. Loo DFS kaust `Kogukond`.
36. Loo DFS kaust `Isiklik`.
37. Loo DFS Replication grupid Kogukond ja Isiklik jaoks.
38. Seadista FSRM kvoodid.
39. Seadista failitüüpide blokeerimine.
40. Loo GPO `Kogukond`, mis jagab Y: ketta.
41. Loo GPO `Isiklik`, mis loob kasutajale kausta ja jagab Z: ketta.
42. Kontrolli kogu lahendus.
43. Dokumenteeri kogu protsess.

---

# 5. DC1 võrgu seadistamine

DC1 peab kasutama staatilist IP-aadressi.

Näide:

```text
IP: 10.0.x.10
Mask: 255.255.255.0
Gateway: 10.0.x.1
DNS: 10.0.x.10
```

PowerShellis:

```powershell
Get-NetAdapter
```

Näide IP seadistamiseks:

```powershell
New-NetIPAddress `
-InterfaceAlias "Ethernet" `
-IPAddress 10.0.x.10 `
-PrefixLength 24 `
-DefaultGateway 10.0.x.1
```

DNS seadistamine:

```powershell
Set-DnsClientServerAddress `
-InterfaceAlias "Ethernet" `
-ServerAddresses 10.0.x.10
```

Kontroll:

```powershell
ipconfig /all
```

---

# 6. DC1 nime muutmine

```powershell
Rename-Computer -NewName "DC1" -Restart
```

Pärast restarti kontrolli:

```powershell
hostname
```

Oodatav:

```text
DC1
```

---

# 7. AD DS ja DNS rolli paigaldamine DC1 serverisse

PowerShellis:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

Kontroll:

```powershell
Get-WindowsFeature AD-Domain-Services,DNS
```

---

# 8. Uue domeeni loomine `oige.local`

DC1 serveris:

```powershell
Install-ADDSForest `
-DomainName "oige.local" `
-DomainNetbiosName "OIGE" `
-InstallDNS `
-SafeModeAdministratorPassword (Read-Host -AsSecureString "Sisesta DSRM parool") `
-Force
```

Server teeb pärast seda restarti.

Pärast restarti logi sisse domeeni administraatorina:

```text
OIGE\Administrator
```

Kontroll:

```powershell
Get-ADDomain
Get-ADForest
```

---

# 9. DNS kontroll DC1 serveris

Kontrolli, kas domeen lahendub:

```powershell
nslookup oige.local
nslookup dc1.oige.local
```

Kontrolli DNS tsooni:

```powershell
Get-DnsServerZone
```

---

# 10. DC2 seadistamine Server Core’is

NB! **DC2 on Windows Server Core. Seal ei ole tavalist graafilist liidest.**

PowerShelli avamiseks:

```cmd
powershell
```

Põhiseadistuste menüü avamiseks:

```cmd
sconfig
```

---

# 11. DC2 võrgu seadistamine

Näidis IP:

```text
IP: 10.0.x.11
Mask: 255.255.255.0
Gateway: 10.0.x.1
DNS: 10.0.x.10
```

DC2 peab enne domeeniga liitmist kasutama DNS serverina DC1 aadressi.

Kontrolli võrgukaardi nime:

```powershell
Get-NetAdapter
```

Seadista IP:

```powershell
New-NetIPAddress `
-InterfaceAlias "Ethernet" `
-IPAddress 10.0.x.11 `
-PrefixLength 24 `
-DefaultGateway 10.0.x.1
```

Seadista DNS DC1 peale:

```powershell
Set-DnsClientServerAddress `
-InterfaceAlias "Ethernet" `
-ServerAddresses 10.0.x.10
```

Kontroll:

```powershell
ipconfig /all
```

---

# 12. DC2 nime muutmine

Server Core’is PowerShelliga:

```powershell
Rename-Computer -NewName "DC2" -Restart
```

Pärast restarti kontrolli:

```powershell
hostname
```

Oodatav:

```text
DC2
```

---

# 13. DC2 domeeniga liitmine

DC2 Server Core’is:

```powershell
Add-Computer `
-DomainName "oige.local" `
-Credential "OIGE\Administrator" `
-Restart
```

Pärast restarti logi sisse domeeni kasutajaga.

Kontroll:

```powershell
whoami
```

Oodatav näiteks:

```text
oige\administrator
```

Kontrolli, et DC2 näeb domeeni:

```powershell
nslookup oige.local
nslookup dc1.oige.local
```

---

# 14. AD DS ja DNS rolli paigaldamine DC2 Server Core’i

DC2 Server Core’is:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

Kontroll:

```powershell
Get-WindowsFeature AD-Domain-Services,DNS
```

---

# 15. DC2 teiseks domeenikontrolleriks tõstmine

DC2 Server Core’is:

```powershell
Install-ADDSDomainController `
-DomainName "oige.local" `
-InstallDns `
-Credential (Get-Credential "OIGE\Administrator") `
-SafeModeAdministratorPassword (Read-Host -AsSecureString "Sisesta DSRM parool") `
-Force
```

Server teeb restarti.

Kontroll DC1 või DC2 serveris:

```powershell
Get-ADDomainController -Filter *
```

Peaksid nägema:

```text
DC1
DC2
```

Kontrolli replikatsiooni:

```powershell
repadmin /replsummary
```

---

# 16. DNS seadistamine mõlemas DC-s

Pärast DC2 lisamist võiks DNS serverid olla nii:

DC1:

```text
Preferred DNS: 10.0.x.10
Alternate DNS: 10.0.x.11
```

DC2:

```text
Preferred DNS: 10.0.x.11
Alternate DNS: 10.0.x.10
```

DC1 PowerShellis:

```powershell
Set-DnsClientServerAddress `
-InterfaceAlias "Ethernet" `
-ServerAddresses 10.0.x.10,10.0.x.11
```

DC2 Server Core’is:

```powershell
Set-DnsClientServerAddress `
-InterfaceAlias "Ethernet" `
-ServerAddresses 10.0.x.11,10.0.x.10
```

Kontroll:

```powershell
ipconfig /all
nslookup dc1.oige.local
nslookup dc2.oige.local
```

---

# 17. DHCP rolli paigaldamine DC1 ja DC2 serverisse

DC1 serveris:

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

DC2 Server Core’is:

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

DHCP autoriseerimine domeenis.

DC1:

```powershell
Add-DhcpServerInDC `
-DnsName "dc1.oige.local" `
-IPAddress 10.0.x.10
```

DC2:

```powershell
Add-DhcpServerInDC `
-DnsName "dc2.oige.local" `
-IPAddress 10.0.x.11
```

Kontroll:

```powershell
Get-DhcpServerInDC
```

---

# 18. DHCP scope loomine DC1 serveris

Näide scope:

```text
Võrk: 10.0.x.0/24
DHCP vahemik: 10.0.x.100 - 10.0.x.200
Mask: 255.255.255.0
Gateway: 10.0.x.1
DNS: 10.0.x.10 ja 10.0.x.11
Lease time: 4 tundi
```

DC1 serveris:

```powershell
Add-DhcpServerv4Scope `
-ComputerName "dc1.oige.local" `
-Name "OIGE LAN" `
-StartRange 10.0.x.100 `
-EndRange 10.0.x.200 `
-SubnetMask 255.255.255.0 `
-State Active
```

Lease time 4 tundi:

```powershell
Set-DhcpServerv4Scope `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-LeaseDuration 04:00:00
```

Gateway:

```powershell
Set-DhcpServerv4OptionValue `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-Router 10.0.x.1
```

DNS serverid ja domeeninimi:

```powershell
Set-DhcpServerv4OptionValue `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-DnsServer 10.0.x.10,10.0.x.11 `
-DnsDomain "oige.local"
```

Kontroll:

```powershell
Get-DhcpServerv4Scope -ComputerName "dc1.oige.local"
Get-DhcpServerv4OptionValue -ComputerName "dc1.oige.local" -ScopeId 10.0.x.0
```

---

# 19. DHCP failover seadistamine

DHCP failover seadistatakse DC1 pealt ja partneriks määratakse DC2.

DC1 serveris:

```powershell
Add-DhcpServerv4Failover `
-ComputerName "dc1.oige.local" `
-Name "OIGE-DHCP-Failover" `
-PartnerServer "dc2.oige.local" `
-ScopeId 10.0.x.0 `
-SharedSecret "TugevSaladus123!" `
-LoadBalancePercent 50 `
-AutoStateTransition $true `
-StateSwitchInterval 00:30:00
```

Kontroll DC1 serveris:

```powershell
Get-DhcpServerv4Failover -ComputerName "dc1.oige.local"
```

Kontroll DC2 Server Core’is:

```powershell
Get-DhcpServerv4Scope -ComputerName "dc2.oige.local"
```

---

# 20. Staatilised DHCP rendid klientarvutitele

Staatiline rent ehk reservation seob kliendi MAC-aadressi kindla IP-ga.

Windows 11 kliendi MAC-aadressi leiab kliendis:

```powershell
ipconfig /all
```

Linuxi kliendi MAC-aadressi leiab kliendis:

```bash
ip a
```

Näide Windows 11 kliendile:

```powershell
Add-DhcpServerv4Reservation `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-IPAddress 10.0.x.101 `
-ClientId "AA-BB-CC-DD-EE-FF" `
-Description "Windows11 klient"
```

Näide Linuxi kliendile:

```powershell
Add-DhcpServerv4Reservation `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-IPAddress 10.0.x.102 `
-ClientId "11-22-33-44-55-66" `
-Description "Ubuntu klient"
```

Kontroll:

```powershell
Get-DhcpServerv4Reservation `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0
```

---

# 21. OU-de loomine

Nõutud OU-d:

```text
Kasutajad
Arvutid
```

Lisaks luuakse:

```text
Personal
Toimetajad
```

PowerShell DC1 serveris:

```powershell
New-ADOrganizationalUnit `
-Name "Kasutajad" `
-Path "DC=oige,DC=local"
```

```powershell
New-ADOrganizationalUnit `
-Name "Arvutid" `
-Path "DC=oige,DC=local"
```

```powershell
New-ADOrganizationalUnit `
-Name "Personal" `
-Path "OU=Kasutajad,DC=oige,DC=local"
```

```powershell
New-ADOrganizationalUnit `
-Name "Toimetajad" `
-Path "OU=Kasutajad,DC=oige,DC=local"
```

Kontroll:

```powershell
Get-ADOrganizationalUnit -Filter * |
Select-Object Name,DistinguishedName
```

---

# 22. Kasutaja `Haldur` loomine ja Domain Admins gruppi lisamine

```powershell
New-ADUser `
-Name "Haldur" `
-SamAccountName "haldur" `
-UserPrincipalName "haldur@oige.local" `
-Path "OU=Kasutajad,DC=oige,DC=local" `
-AccountPassword (Read-Host -AsSecureString "Sisesta Haldur parool") `
-Enabled $true
```

Lisa kasutaja Domain Admins gruppi:

```powershell
Add-ADGroupMember `
-Identity "Domain Admins" `
-Members "haldur"
```

Kontroll:

```powershell
Get-ADGroupMember "Domain Admins"
```

---

# 23. Windows 11 kliendi domeeniga liitmine

Windows 11 kliendis peab DNS olema domeenikontrollerite peale suunatud.

Kontroll Windows 11 kliendis:

```powershell
ipconfig /all
```

DNS peab olema näiteks:

```text
10.0.x.10
10.0.x.11
```

Domeeniga liitmine GUI kaudu:

```text
Settings
System
About
Domain or workgroup
Join domain
oige.local
```

Või PowerShelliga administraatorina:

```powershell
Add-Computer `
-DomainName "oige.local" `
-Credential "OIGE\Haldur" `
-Restart
```

Pärast restarti logi sisse domeeni kasutajaga:

```text
OIGE\Haldur
```

---

# 24. Windows 11 arvuti liigutamine OU-sse `Arvutid`

DC1 serveris:

```powershell
Get-ADComputer -Filter * |
Select-Object Name,DistinguishedName
```

Leia Windows 11 arvuti nimi.

Näide, kui arvuti nimi on `WIN11`:

```powershell
Move-ADObject `
-Identity "CN=WIN11,CN=Computers,DC=oige,DC=local" `
-TargetPath "OU=Arvutid,DC=oige,DC=local"
```

NB! Asenda `WIN11` päris arvutinimega.

Kontroll:

```powershell
Get-ADComputer WIN11 -Properties DistinguishedName |
Select-Object Name,DistinguishedName
```

---

# 25. DNS A-kirjed Linuxi serveritele

DNS serveris tuleb teha A-kirjed kõigi pileti Linuxi serverite jaoks.

Näide:

```powershell
Add-DnsServerResourceRecordA `
-ZoneName "oige.local" `
-Name "ubuntu" `
-IPv4Address "10.0.x.20"
```

```powershell
Add-DnsServerResourceRecordA `
-ZoneName "oige.local" `
-Name "alma" `
-IPv4Address "10.0.x.21"
```

```powershell
Add-DnsServerResourceRecordA `
-ZoneName "oige.local" `
-Name "debian" `
-IPv4Address "10.0.x.22"
```

Kontroll:

```powershell
Resolve-DnsName ubuntu.oige.local
Resolve-DnsName alma.oige.local
Resolve-DnsName debian.oige.local
```

Või:

```powershell
nslookup ubuntu.oige.local
nslookup alma.oige.local
nslookup debian.oige.local
```

---

# 26. CSV faili näidis `kasutajad.csv`

CSV fail võiks olla näiteks sellise kujuga:

```csv
Eesnimi,Perenimi,Kasutajanimi,Parool,OU
Mari,Tamm,mari.tamm,Parool123!,Personal
Jaan,Kask,jaan.kask,Parool123!,Personal
Tiina,Sepp,tiina.sepp,Parool123!,Toimetajad
Karl,Saar,karl.saar,Parool123!,Toimetajad
```

Faili asukoht:

```text
C:\Scripts\kasutajad.csv
```

---

# 27. PowerShelli skript kasutajate importimiseks CSV failist

Loo kaust:

```powershell
mkdir C:\Scripts
```

Loo fail:

```powershell
notepad C:\Scripts\impordi_kasutajad.ps1
```

Skripti sisu:

```powershell
Import-Module ActiveDirectory

$csvPath = "C:\Scripts\kasutajad.csv"
$domainPath = "DC=oige,DC=local"
$baseOU = "OU=Kasutajad,$domainPath"

$users = Import-Csv -Path $csvPath

foreach ($user in $users) {

    $ouName = $user.OU
    $targetOU = "OU=$ouName,$baseOU"

    if (-not (Get-ADOrganizationalUnit -LDAPFilter "(ou=$ouName)" -SearchBase $baseOU -ErrorAction SilentlyContinue)) {
        New-ADOrganizationalUnit -Name $ouName -Path $baseOU
    }

    $fullName = "$($user.Eesnimi) $($user.Perenimi)"
    $sam = $user.Kasutajanimi
    $upn = "$sam@oige.local"

    if (-not (Get-ADUser -Filter "SamAccountName -eq '$sam'" -ErrorAction SilentlyContinue)) {

        New-ADUser `
            -Name $fullName `
            -GivenName $user.Eesnimi `
            -Surname $user.Perenimi `
            -SamAccountName $sam `
            -UserPrincipalName $upn `
            -Path $targetOU `
            -AccountPassword (ConvertTo-SecureString $user.Parool -AsPlainText -Force) `
            -Enabled $true `
            -ChangePasswordAtLogon $true

        Write-Host "Loodi kasutaja: $sam"
    }
    else {
        Write-Host "Kasutaja on juba olemas: $sam"
    }
}
```

Kui skriptide käivitamine on keelatud:

```powershell
Set-ExecutionPolicy RemoteSigned -Scope Process
```

Käivita skript:

```powershell
C:\Scripts\impordi_kasutajad.ps1
```

Kontroll:

```powershell
Get-ADUser `
-Filter * `
-SearchBase "OU=Kasutajad,DC=oige,DC=local" |
Select-Object Name,SamAccountName
```

---

# 28. GPO `Teade` loomine

GPO eesmärk:

OU `Personal` kasutajatele kuvatakse sisselogimisel teade:

```text
Arvuti kasutamisel tuleb järgida ettevõtte turvapoliitikat ja arvuti kasutamise eeskirja.
```

Loo GPO:

```powershell
New-GPO -Name "Teade"
```

Seo GPO OU-ga `Personal`:

```powershell
New-GPLink `
-Name "Teade" `
-Target "OU=Personal,OU=Kasutajad,DC=oige,DC=local"
```

---

# 29. Logon-script teate kuvamiseks

Loo skriptikaust SYSVOL-is:

```powershell
mkdir "\\oige.local\SYSVOL\oige.local\scripts"
```

Loo PowerShelli logon-script:

```powershell
notepad "\\oige.local\SYSVOL\oige.local\scripts\teade.ps1"
```

Sisu:

```powershell
Add-Type -AssemblyName PresentationFramework

[System.Windows.MessageBox]::Show(
    "Arvuti kasutamisel tuleb järgida ettevõtte turvapoliitikat ja arvuti kasutamise eeskirja.",
    "AS Oige teade",
    "OK",
    "Information"
)
```

Loo wrapper `.bat` fail:

```powershell
notepad "\\oige.local\SYSVOL\oige.local\scripts\teade.bat"
```

Sisu:

```bat
powershell.exe -ExecutionPolicy Bypass -WindowStyle Hidden -File "\\oige.local\SYSVOL\oige.local\scripts\teade.ps1"
```

## GPO-sse lisamine GUI kaudu DC1 serveris

Ava DC1 serveris:

```text
Group Policy Management
```

Seejärel:

```text
Forest: oige.local
Domains
oige.local
Kasutajad
Personal
Teade
Edit
User Configuration
Policies
Windows Settings
Scripts (Logon/Logoff)
Logon
Add
```

Lisa script:

```text
\\oige.local\SYSVOL\oige.local\scripts\teade.bat
```

Kontroll kliendis Personal OU kasutajaga:

```powershell
gpupdate /force
```

Logi välja ja sisse tagasi.

Kontroll:

```powershell
gpresult /r
```

---

# 30. DFS ja FSRM rollide paigaldamine DC1 ja DC2 serverisse

Paigaldada tuleb:

```text
DFS Namespaces
DFS Replication
File Server Resource Manager
```

DC1 serveris:

```powershell
Install-WindowsFeature FS-DFS-Namespace,FS-DFS-Replication,FS-Resource-Manager,FS-FileServer -IncludeManagementTools
```

DC2 Server Core’is:

```powershell
Install-WindowsFeature FS-DFS-Namespace,FS-DFS-Replication,FS-Resource-Manager,FS-FileServer -IncludeManagementTools
```

Kontroll mõlemas serveris:

```powershell
Get-WindowsFeature FS-DFS-Namespace,FS-DFS-Replication,FS-Resource-Manager,FS-FileServer
```

---

# 31. Andmekaustade loomine DC1 ja DC2 serveris

Mõlemas serveris luuakse samad kaustad.

DC1 serveris:

```powershell
mkdir D:\DFSRoots\Jagatud
mkdir D:\DFSData\Kogukond
mkdir D:\DFSData\Isiklik
```

DC2 Server Core’is:

```powershell
mkdir D:\DFSRoots\Jagatud
mkdir D:\DFSData\Kogukond
mkdir D:\DFSData\Isiklik
```

NB! Kui `D:` ketast ei ole, kasuta näiteks `C:\DFSRoots` ja `C:\DFSData`.  
Dokumentatsioonis kirjuta, millist ketast kasutasid.

---

# 32. SMB jagamiste loomine DC1 ja DC2 serveris

DFS namespace root jaoks:

DC1:

```powershell
New-SmbShare `
-Name "JagatudRoot$" `
-Path "D:\DFSRoots\Jagatud" `
-FullAccess "OIGE\Domain Admins" `
-ChangeAccess "OIGE\Domain Users"
```

DC2:

```powershell
New-SmbShare `
-Name "JagatudRoot$" `
-Path "D:\DFSRoots\Jagatud" `
-FullAccess "OIGE\Domain Admins" `
-ChangeAccess "OIGE\Domain Users"
```

Kogukond jagamine:

DC1:

```powershell
New-SmbShare `
-Name "Kogukond$" `
-Path "D:\DFSData\Kogukond" `
-FullAccess "OIGE\Domain Admins" `
-ChangeAccess "OIGE\Domain Users"
```

DC2:

```powershell
New-SmbShare `
-Name "Kogukond$" `
-Path "D:\DFSData\Kogukond" `
-FullAccess "OIGE\Domain Admins" `
-ChangeAccess "OIGE\Domain Users"
```

Isiklik jagamine:

DC1:

```powershell
New-SmbShare `
-Name "Isiklik$" `
-Path "D:\DFSData\Isiklik" `
-FullAccess "OIGE\Domain Admins" `
-ChangeAccess "OIGE\Domain Users"
```

DC2:

```powershell
New-SmbShare `
-Name "Isiklik$" `
-Path "D:\DFSData\Isiklik" `
-FullAccess "OIGE\Domain Admins" `
-ChangeAccess "OIGE\Domain Users"
```

Kontroll mõlemas serveris:

```powershell
Get-SmbShare
```

---

# 33. DFS nimeruumi `Jagatud` loomine

DFS namespace tee peab olema:

```text
\\oige.local\Jagatud
```

DC1 serveris:

```powershell
New-DfsnRoot `
-Path "\\oige.local\Jagatud" `
-TargetPath "\\DC1\JagatudRoot$" `
-Type DomainV2
```

Lisa DC2 namespace root targetiks:

```powershell
New-DfsnRootTarget `
-Path "\\oige.local\Jagatud" `
-TargetPath "\\DC2\JagatudRoot$"
```

Kontroll:

```powershell
Get-DfsnRoot
Get-DfsnRootTarget -Path "\\oige.local\Jagatud"
```

---

# 34. DFS kausta `Kogukond` loomine

Loo DFS kaust `Kogukond` ja lisa DC1 target:

```powershell
New-DfsnFolder `
-Path "\\oige.local\Jagatud\Kogukond" `
-TargetPath "\\DC1\Kogukond$"
```

Lisa DC2 target:

```powershell
New-DfsnFolderTarget `
-Path "\\oige.local\Jagatud\Kogukond" `
-TargetPath "\\DC2\Kogukond$"
```

Kontroll:

```powershell
Get-DfsnFolder -Path "\\oige.local\Jagatud\Kogukond"
Get-DfsnFolderTarget -Path "\\oige.local\Jagatud\Kogukond"
```

---

# 35. DFS kausta `Isiklik` loomine

Loo DFS kaust `Isiklik` ja lisa DC1 target:

```powershell
New-DfsnFolder `
-Path "\\oige.local\Jagatud\Isiklik" `
-TargetPath "\\DC1\Isiklik$"
```

Lisa DC2 target:

```powershell
New-DfsnFolderTarget `
-Path "\\oige.local\Jagatud\Isiklik" `
-TargetPath "\\DC2\Isiklik$"
```

Kontroll:

```powershell
Get-DfsnFolder -Path "\\oige.local\Jagatud\Isiklik"
Get-DfsnFolderTarget -Path "\\oige.local\Jagatud\Isiklik"
```

---

# 36. DFS Replication grupp `RG-Kogukond`

DFS Replication seadistatakse DC1 serveris.

Loo replikatsioonigrupp:

```powershell
New-DfsReplicationGroup `
-GroupName "RG-Kogukond"
```

Lisa liikmed:

```powershell
Add-DfsrMember `
-GroupName "RG-Kogukond" `
-ComputerName "DC1","DC2"
```

Loo replicated folder:

```powershell
New-DfsReplicatedFolder `
-GroupName "RG-Kogukond" `
-FolderName "Kogukond" `
-DfsnPath "\\oige.local\Jagatud\Kogukond"
```

Määra DC1 liikmel kausta asukoht ja tee see esmaseks:

```powershell
Set-DfsrMembership `
-GroupName "RG-Kogukond" `
-FolderName "Kogukond" `
-ComputerName "DC1" `
-ContentPath "D:\DFSData\Kogukond" `
-PrimaryMember $true `
-Force
```

Määra DC2 liikmel kausta asukoht:

```powershell
Set-DfsrMembership `
-GroupName "RG-Kogukond" `
-FolderName "Kogukond" `
-ComputerName "DC2" `
-ContentPath "D:\DFSData\Kogukond" `
-Force
```

Loo ühendused mõlemas suunas:

```powershell
Add-DfsrConnection `
-GroupName "RG-Kogukond" `
-SourceComputerName "DC1" `
-DestinationComputerName "DC2"
```

```powershell
Add-DfsrConnection `
-GroupName "RG-Kogukond" `
-SourceComputerName "DC2" `
-DestinationComputerName "DC1"
```

Kontroll:

```powershell
Get-DfsReplicationGroup
Get-DfsrMember -GroupName "RG-Kogukond"
Get-DfsReplicatedFolder -GroupName "RG-Kogukond"
Get-DfsrMembership -GroupName "RG-Kogukond"
```

---

# 37. DFS Replication grupp `RG-Isiklik`

Loo replikatsioonigrupp:

```powershell
New-DfsReplicationGroup `
-GroupName "RG-Isiklik"
```

Lisa liikmed:

```powershell
Add-DfsrMember `
-GroupName "RG-Isiklik" `
-ComputerName "DC1","DC2"
```

Loo replicated folder:

```powershell
New-DfsReplicatedFolder `
-GroupName "RG-Isiklik" `
-FolderName "Isiklik" `
-DfsnPath "\\oige.local\Jagatud\Isiklik"
```

Määra DC1 liikmel kausta asukoht ja tee see esmaseks:

```powershell
Set-DfsrMembership `
-GroupName "RG-Isiklik" `
-FolderName "Isiklik" `
-ComputerName "DC1" `
-ContentPath "D:\DFSData\Isiklik" `
-PrimaryMember $true `
-Force
```

Määra DC2 liikmel kausta asukoht:

```powershell
Set-DfsrMembership `
-GroupName "RG-Isiklik" `
-FolderName "Isiklik" `
-ComputerName "DC2" `
-ContentPath "D:\DFSData\Isiklik" `
-Force
```

Loo ühendused mõlemas suunas:

```powershell
Add-DfsrConnection `
-GroupName "RG-Isiklik" `
-SourceComputerName "DC1" `
-DestinationComputerName "DC2"
```

```powershell
Add-DfsrConnection `
-GroupName "RG-Isiklik" `
-SourceComputerName "DC2" `
-DestinationComputerName "DC1"
```

Kontroll:

```powershell
Get-DfsReplicationGroup
Get-DfsrMember -GroupName "RG-Isiklik"
Get-DfsReplicatedFolder -GroupName "RG-Isiklik"
Get-DfsrMembership -GroupName "RG-Isiklik"
```

---

# 38. DFS replikatsiooni testimine

DC1 serveris loo testfail:

```powershell
"DFS test Kogukond" | Out-File "D:\DFSData\Kogukond\test-kogukond.txt"
```

Oota veidi ja kontrolli DC2 serveris:

```powershell
dir D:\DFSData\Kogukond
```

DC1 serveris loo Isiklik testfail:

```powershell
"DFS test Isiklik" | Out-File "D:\DFSData\Isiklik\test-isiklik.txt"
```

Oota veidi ja kontrolli DC2 serveris:

```powershell
dir D:\DFSData\Isiklik
```

Kui failid jõuavad DC2 serverisse, töötab DFS Replication.

---

# 39. Kogukond kausta õigused

Kogukond peab olema kõigile domeeni kasutajatele ligipääsetav.

DC1 serveris:

```powershell
icacls "D:\DFSData\Kogukond" /grant "OIGE\Domain Users:(OI)(CI)M"
icacls "D:\DFSData\Kogukond" /grant "OIGE\Domain Admins:(OI)(CI)F"
```

DC2 serveris:

```powershell
icacls "D:\DFSData\Kogukond" /grant "OIGE\Domain Users:(OI)(CI)M"
icacls "D:\DFSData\Kogukond" /grant "OIGE\Domain Admins:(OI)(CI)F"
```

Selgitus:

```text
M = Modify
F = Full Control
OI = Object Inherit
CI = Container Inherit
```

Kontroll:

```powershell
icacls "D:\DFSData\Kogukond"
```

---

# 40. Isiklik kausta õiguste loogika

Isiklik kausta puhul on oluline, et kasutaja ei pääseks teiste kasutajate kaustadesse.

Soovituslik loogika:

```text
D:\DFSData\Isiklik
├── mari.tamm
├── jaan.kask
├── tiina.sepp
└── karl.saar
```

Igal kasutajal peab olema õigus ainult enda kaustale.

Praktiline ja eksamis kindel lahendus:

1. Kasutajakaustad luuakse administraatori skriptiga.
2. Iga kasutaja kaustale määratakse õigused ainult sellele kasutajale.
3. GPO kaardistab kasutajale ainult tema enda kausta Z: kettana.

---

# 41. Isiklike kaustade loomise skript

Loo DC1 serveris fail:

```powershell
notepad C:\Scripts\loo_isiklikud_kaustad.ps1
```

Sisu:

```powershell
Import-Module ActiveDirectory

$root = "D:\DFSData\Isiklik"
$users = Get-ADUser -Filter * -SearchBase "OU=Kasutajad,DC=oige,DC=local" | Select-Object -ExpandProperty SamAccountName

foreach ($user in $users) {

    $path = Join-Path $root $user

    if (-not (Test-Path $path)) {
        New-Item -Path $path -ItemType Directory
    }

    # Eemaldab päranduvad õigused ja kopeerib olemasolevad õigused
    icacls $path /inheritance:r

    # Annab adminidele ja süsteemile täisõigused
    icacls $path /grant "OIGE\Domain Admins:(OI)(CI)F"
    icacls $path /grant "SYSTEM:(OI)(CI)F"

    # Annab kasutajale õigused ainult tema enda kaustale
    icacls $path /grant "OIGE\$user:(OI)(CI)M"
}
```

Käivita:

```powershell
C:\Scripts\loo_isiklikud_kaustad.ps1
```

Kontroll:

```powershell
dir D:\DFSData\Isiklik
icacls D:\DFSData\Isiklik\mari.tamm
```

NB! Kui kasutajad lisanduvad hiljem, tuleb skripti uuesti käivitada või täiendada seda ajastatud tööna.

---

# 42. Isiklik kausta õigused DC2 serveris

DFS replikatsioon kopeerib failid, kuid õiguseid tuleb vajadusel kontrollida ka DC2 poolel.

DC2 Server Core’is kontroll:

```powershell
dir D:\DFSData\Isiklik
icacls D:\DFSData\Isiklik
```

Kui õigused pole korrektsed, käivita sama skript ka DC2 serveris või kasuta DC1 pealt PowerShell Remotingut.

---

# 43. FSRM kvoot Kogukond kaustale

Kogukond kaustale tuleb määrata 10 GB piirang.

NB! Kui DFS targeteid on kaks, on mõistlik FSRM kvoot seadistada **mõlemas serveris**, sest kasutaja võib sattuda DC1 või DC2 targeti peale.

DC1 serveris:

```powershell
New-FsrmQuota `
-Path "D:\DFSData\Kogukond" `
-Size 10GB `
-Description "Kogukond 10 GB piirang"
```

DC2 Server Core’is:

```powershell
New-FsrmQuota `
-Path "D:\DFSData\Kogukond" `
-Size 10GB `
-Description "Kogukond 10 GB piirang"
```

Kontroll mõlemas serveris:

```powershell
Get-FsrmQuota -Path "D:\DFSData\Kogukond"
```

---

# 44. Programmifailide blokeerimine Kogukond kaustas

Keelata tuleb failitüübid:

```text
*.msi
*.exe
*.bat
*.ps1
```

Tee seda mõlemas serveris.

DC1 serveris:

```powershell
New-FsrmFileGroup `
-Name "Keelatud programmifailid" `
-IncludePattern @("*.msi","*.exe","*.bat","*.ps1")
```

```powershell
New-FsrmFileScreen `
-Path "D:\DFSData\Kogukond" `
-IncludeGroup "Keelatud programmifailid" `
-Active `
-Description "Keelab programmifailid Kogukond kaustas"
```

DC2 Server Core’is:

```powershell
New-FsrmFileGroup `
-Name "Keelatud programmifailid" `
-IncludePattern @("*.msi","*.exe","*.bat","*.ps1")
```

```powershell
New-FsrmFileScreen `
-Path "D:\DFSData\Kogukond" `
-IncludeGroup "Keelatud programmifailid" `
-Active `
-Description "Keelab programmifailid Kogukond kaustas"
```

Kontroll mõlemas serveris:

```powershell
Get-FsrmFileGroup
Get-FsrmFileScreen
```

Test kliendist:

```text
Proovi kopeerida .exe või .bat fail Y: kettale.
See peab ebaõnnestuma.
```

---

# 45. FSRM 1 GB kvoot Isiklik kasutajakaustadele

Isiklik kaustadele on kõige õigem kasutada **auto apply quota** lahendust.  
See tähendab, et igale alamkaustale rakendatakse automaatselt 1 GB piirang.

Tee seda mõlemas serveris.

DC1 serveris:

```powershell
New-FsrmQuotaTemplate `
-Name "Isiklik 1GB" `
-Size 1GB `
-Description "1 GB kasutaja isiklikule kaustale"
```

```powershell
New-FsrmAutoQuota `
-Path "D:\DFSData\Isiklik" `
-Template "Isiklik 1GB"
```

DC2 Server Core’is:

```powershell
New-FsrmQuotaTemplate `
-Name "Isiklik 1GB" `
-Size 1GB `
-Description "1 GB kasutaja isiklikule kaustale"
```

```powershell
New-FsrmAutoQuota `
-Path "D:\DFSData\Isiklik" `
-Template "Isiklik 1GB"
```

Kontroll mõlemas serveris:

```powershell
Get-FsrmAutoQuota
Get-FsrmQuota
```

---

# 46. GPO `Kogukond` loomine

GPO `Kogukond` peab jagama kõigile domeeni kasutajatele võrguketta:

```text
Y:
```

Tee GPO:

```powershell
New-GPO -Name "Kogukond"
```

Seo GPO kasutajate OU-ga:

```powershell
New-GPLink `
-Name "Kogukond" `
-Target "OU=Kasutajad,DC=oige,DC=local"
```

---

# 47. Y: võrguketta seadistamine GUI kaudu

Ava DC1 serveris:

```text
Group Policy Management
```

Mine:

```text
Forest: oige.local
Domains
oige.local
Kasutajad
Kogukond
Edit
User Configuration
Preferences
Windows Settings
Drive Maps
New
Mapped Drive
```

Seadista:

```text
Action: Update
Location: \\oige.local\Jagatud\Kogukond
Drive Letter: Y:
Reconnect: enabled
Label as: Kogukond
```

Kontroll kliendis:

```powershell
gpupdate /force
```

Logi kasutajaga välja ja sisse.

Kontroll:

```cmd
net use
```

Peab näitama:

```text
Y: \\oige.local\Jagatud\Kogukond
```

---

# 48. GPO `Isiklik` loomine

GPO `Isiklik` peab:

1. looma kasutajale tema kasutajanimega kausta;
2. jagama ainult selle kausta kasutajale võrgukettana `Z:`.

Tee GPO:

```powershell
New-GPO -Name "Isiklik"
```

Seo GPO kasutajate OU-ga:

```powershell
New-GPLink `
-Name "Isiklik" `
-Target "OU=Kasutajad,DC=oige,DC=local"
```

---

# 49. Z: võrguketta seadistamine

Kui isiklikud kaustad on juba administraatori skriptiga loodud, saab Z: ketta teha Drive Maps abil.

Ava DC1 serveris:

```text
Group Policy Management
```

Mine:

```text
Forest: oige.local
Domains
oige.local
Kasutajad
Isiklik
Edit
User Configuration
Preferences
Windows Settings
Drive Maps
New
Mapped Drive
```

Seadista:

```text
Action: Update
Location: \\oige.local\Jagatud\Isiklik\%USERNAME%
Drive Letter: Z:
Reconnect: enabled
Label as: Isiklik
```

Kontroll kliendis:

```powershell
gpupdate /force
```

Logi kasutajaga välja ja sisse.

Kontroll:

```cmd
net use
```

Peab näitama:

```text
Z: \\oige.local\Jagatud\Isiklik\<kasutajanimi>
```

---

# 50. Alternatiiv: Z: ketta loomine logon-scriptiga

Kui Drive Maps ei loo või ei leia kausta, võib kasutada logon-scripti.

Loo SYSVOL-i skript:

```powershell
notepad "\\oige.local\SYSVOL\oige.local\scripts\map_isiklik.ps1"
```

Sisu:

```powershell
$root = "\\oige.local\Jagatud\Isiklik"
$user = $env:USERNAME
$target = Join-Path $root $user

if (Test-Path $target) {
    net use Z: /delete /y
    net use Z: $target /persistent:yes
}
```

Loo `.bat` wrapper:

```powershell
notepad "\\oige.local\SYSVOL\oige.local\scripts\map_isiklik.bat"
```

Sisu:

```bat
powershell.exe -ExecutionPolicy Bypass -WindowStyle Hidden -File "\\oige.local\SYSVOL\oige.local\scripts\map_isiklik.ps1"
```

Lisa see GPO `Isiklik` alla:

```text
User Configuration
Policies
Windows Settings
Scripts (Logon/Logoff)
Logon
Add
```

---

# 51. DFS ja võrguketaste kontroll kliendis

Windows 11 kliendis:

```powershell
gpupdate /force
```

Logi välja ja sisse.

Kontrolli võrgukettaid:

```cmd
net use
```

Oodatav:

```text
Y: \\oige.local\Jagatud\Kogukond
Z: \\oige.local\Jagatud\Isiklik\<kasutajanimi>
```

Kontrolli ligipääsu:

```cmd
Y:
dir
```

```cmd
Z:
dir
```

Testi Kogukond kaustas keelatud failitüüpe:

```text
Proovi salvestada test.exe või test.bat Y: kettale.
See peab olema blokeeritud.
```

Testi isiklikku kausta:

```text
Kasutaja peab nägema ja kasutama ainult enda Z: kausta.
```

---

# 52. Kontrollnimekiri

## Domeen

```powershell
Get-ADDomain
Get-ADForest
```

Oodatav domeen:

```text
oige.local
```

---

## Domeenikontrollerid

```powershell
Get-ADDomainController -Filter *
```

Peaksid olema:

```text
DC1
DC2
```

---

## Replikatsioon

```powershell
repadmin /replsummary
```

Vigade arv peaks olema 0 või tuleb vead lahendada.

---

## DNS

```powershell
nslookup dc1.oige.local
nslookup dc2.oige.local
nslookup ubuntu.oige.local
nslookup alma.oige.local
nslookup debian.oige.local
```

---

## DHCP

```powershell
Get-DhcpServerInDC
Get-DhcpServerv4Scope -ComputerName "dc1.oige.local"
Get-DhcpServerv4OptionValue -ComputerName "dc1.oige.local" -ScopeId 10.0.x.0
Get-DhcpServerv4Failover -ComputerName "dc1.oige.local"
Get-DhcpServerv4Reservation -ComputerName "dc1.oige.local" -ScopeId 10.0.x.0
```

Kontrolli, et:

```text
lease time = 4 tundi
DNS serverid = DC1 ja DC2
failover partner = DC2
reservationid olemas klientidele
```

---

## OU-d

```powershell
Get-ADOrganizationalUnit -Filter * |
Select-Object Name,DistinguishedName
```

Peavad olemas olema:

```text
Kasutajad
Arvutid
Personal
Toimetajad
```

---

## Haldur kasutaja

```powershell
Get-ADUser haldur
Get-ADGroupMember "Domain Admins"
```

---

## Windows 11 klient domeenis

Windows 11 kliendis:

```powershell
whoami
systeminfo | findstr /B /C:"Domain"
```

Või:

```powershell
Get-ComputerInfo | Select-Object CsDomain
```

---

## DFS Namespace

```powershell
Get-DfsnRoot
Get-DfsnRootTarget -Path "\\oige.local\Jagatud"
Get-DfsnFolder -Path "\\oige.local\Jagatud\Kogukond"
Get-DfsnFolder -Path "\\oige.local\Jagatud\Isiklik"
```

---

## DFS Replication

```powershell
Get-DfsReplicationGroup
Get-DfsrMember -GroupName "RG-Kogukond"
Get-DfsrMember -GroupName "RG-Isiklik"
Get-DfsrMembership -GroupName "RG-Kogukond"
Get-DfsrMembership -GroupName "RG-Isiklik"
```

---

## FSRM

Kogukond kvoot:

```powershell
Get-FsrmQuota -Path "D:\DFSData\Kogukond"
```

Isiklik auto quota:

```powershell
Get-FsrmAutoQuota
Get-FsrmQuota
```

Failitüüpide piirang:

```powershell
Get-FsrmFileGroup
Get-FsrmFileScreen
```

---

## GPO-d

Kliendis:

```powershell
gpupdate /force
gpresult /r
```

Peavad rakenduma:

```text
Teade
Kogukond
Isiklik
```

---

## Võrgukettad

Kliendis:

```cmd
net use
```

Oodatav:

```text
Y: \\oige.local\Jagatud\Kogukond
Z: \\oige.local\Jagatud\Isiklik\<kasutajanimi>
```

---

# 53. Dokumentatsiooni struktuur

Dokumentatsioonis võiks olla järgmised peatükid.

## 1. IP-plaan

Näide:

```text
DC1: 10.0.x.10
DC2: 10.0.x.11
UbuntuServer: 10.0.x.20
AlmaServer: 10.0.x.21
DebianServer: 10.0.x.22
Gateway: 10.0.x.1
```

---

## 2. Active Directory

Kirjuta:

```text
Domeeniks loodi oige.local.
Esimeseks domeenikontrolleriks seadistati DC1.
Teiseks domeenikontrolleriks seadistati DC2 Server Core.
Mõlemale domeenikontrollerile paigaldati DNS roll.
DC2 seadistati PowerShelli ja sconfig tööriista abil.
```

Lisa kontrollkäsud:

```powershell
Get-ADDomain
Get-ADDomainController -Filter *
repadmin /replsummary
```

---

## 3. DHCP

Kirjuta:

```text
DHCP scope loodi võrgule 10.0.x.0/24.
Rendi kestuseks määrati 4 tundi.
DNS serveritena jagatakse klientidele DC1 ja DC2 aadressid.
DHCP failover seadistati DC1 ja DC2 vahel.
Klientarvutitele lisati staatilised DHCP rendid.
```

---

## 4. OU-d ja kasutajad

Kirjuta:

```text
Loodi OU-d Kasutajad ja Arvutid.
Lisaks loodi Personal ja Toimetajad OU-d.
Loodi kasutaja Haldur ja lisati Domain Admins gruppi.
```

---

## 5. Windows 11 klient

Kirjuta:

```text
Windows 11 klient liideti domeeniga oige.local ja arvutiobjekt liigutati OU-sse Arvutid.
```

---

## 6. DNS

Kirjuta:

```text
DNS serverisse lisati A-kirjed Linuxi serveritele.
```

---

## 7. CSV import

Kirjuta:

```text
Kasutajate andmed eksporditi faili kasutajad.csv.
PowerShelli skript impordi_kasutajad.ps1 loob CSV põhjal OU struktuuri ja kasutajad.
```

---

## 8. GPO Teade

Kirjuta:

```text
Loodi GPO nimega Teade.
GPO seoti OU-ga Personal.
GPO käivitab kasutaja sisselogimisel logon-scripti, mis kuvab teate ettevõtte turvapoliitika ja arvuti kasutamise eeskirja järgimise kohta.
```

---

## 9. DFS

Kirjuta:

```text
DC1 ja DC2 serveritesse paigaldati DFS Namespaces ja DFS Replication rollid.
Loodi domeenipõhine DFS namespace \\oige.local\Jagatud.
Namespace targetiteks lisati DC1 ja DC2.
Loodi DFS kaustad Kogukond ja Isiklik.
Mõlemale kaustale loodi DFS Replication grupid, et kaustade sisu oleks olemas mõlemas serveris.
```

---

## 10. FSRM

Kirjuta:

```text
DC1 ja DC2 serveritesse paigaldati File Server Resource Manager.
Kogukond kaustale seadistati 10 GB kvoot.
Kogukond kaustas keelati .msi, .exe, .bat ja .ps1 failide salvestamine.
Isiklik kausta alamkaustadele seadistati auto apply quota, mis annab igale kasutajakaustale 1 GB piirangu.
```

---

## 11. GPO Kogukond ja Isiklik

Kirjuta:

```text
Loodi GPO Kogukond, millega kaardistatakse kõigile domeeni kasutajatele \\oige.local\Jagatud\Kogukond võrgukettana Y:.
Loodi GPO Isiklik, millega kaardistatakse kasutajale tema enda kaust \\oige.local\Jagatud\Isiklik\%USERNAME% võrgukettana Z:.
```

---

# 54. Tüüpilised probleemid ja lahendused

## Probleem: DC2-s ei ole graafilist liidest

See on normaalne, sest DC2 on Server Core.

Kasuta:

```cmd
sconfig
```

või:

```cmd
powershell
```

DC2 saab hiljem hallata ka DC1 pealt graafiliste tööriistadega.

---

## Probleem: Windows 11 klient ei saa domeeniga liituda

Kontrolli kliendis DNS-i:

```powershell
ipconfig /all
```

DNS peab olema:

```text
10.0.x.10
10.0.x.11
```

Kontrolli nime lahendamist:

```powershell
nslookup oige.local
nslookup dc1.oige.local
```

---

## Probleem: DHCP ei jaga aadresse

Kontrolli, kas DHCP on autoriseeritud:

```powershell
Get-DhcpServerInDC
```

Kontrolli scope:

```powershell
Get-DhcpServerv4Scope -ComputerName "dc1.oige.local"
```

Kontrolli, kas scope on aktiivne:

```powershell
Set-DhcpServerv4Scope `
-ComputerName "dc1.oige.local" `
-ScopeId 10.0.x.0 `
-State Active
```

---

## Probleem: DFS namespace ei avane

Kontrolli:

```powershell
Get-DfsnRoot
Get-DfsnRootTarget -Path "\\oige.local\Jagatud"
```

Kliendis kontrolli:

```cmd
dir \\oige.local\Jagatud
```

Kui ei avane, kontrolli:

```text
DNS töötab
SMB share on olemas
kasutajal on õigused
DFS Namespace teenus töötab
```

---

## Probleem: DFS Replication ei tööta

Kontrolli:

```powershell
Get-DfsReplicationGroup
Get-DfsrMember -GroupName "RG-Kogukond"
Get-DfsrMembership -GroupName "RG-Kogukond"
```

Kontrolli teenust mõlemas serveris:

```powershell
Get-Service DFSR
```

Kui vaja, taaskäivita:

```powershell
Restart-Service DFSR
```

NB! DFS replikatsioon ei pruugi olla täiesti kohene. Oota veidi ja testi uuesti.

---

## Probleem: Y: või Z: ketas ei ilmu

Kliendis:

```powershell
gpupdate /force
gpresult /r
```

Kontrolli:

```text
kas GPO on lingitud õige OU külge
kas kasutaja asub õiges OU-s
kas kasutaja logis uuesti sisse
kas DFS path töötab käsitsi
```

Kontrolli käsitsi:

```cmd
dir \\oige.local\Jagatud\Kogukond
dir \\oige.local\Jagatud\Isiklik
```

---

## Probleem: kasutaja näeb teiste isiklikke kaustu

See tähendab, et NTFS õigused on valed.

Kontrolli:

```powershell
icacls D:\DFSData\Isiklik
icacls D:\DFSData\Isiklik\kasutajanimi
```

Õige loogika:

```text
kasutajal on õigused ainult tema enda alamkaustale
Domain Admins ja SYSTEM omavad täisõiguseid
```

---

## Probleem: .exe või .bat fail saab ikka Kogukond kausta kopeerida

Kontrolli FSRM file screeni:

```powershell
Get-FsrmFileGroup
Get-FsrmFileScreen
```

Kontrolli, et failigrupp sisaldab:

```text
*.msi
*.exe
*.bat
*.ps1
```

Kontrolli, et file screen on Active.

---

## Probleem: kvoot ei rakendu

Kontrolli Kogukond kvooti:

```powershell
Get-FsrmQuota -Path "D:\DFSData\Kogukond"
```

Kontrolli Isiklik auto quota:

```powershell
Get-FsrmAutoQuota
Get-FsrmQuota
```

Kui isiklikud kasutajakaustad loodi enne auto quota seadistamist, kontrolli, kas FSRM genereeris neile eraldi kvoodid.

---

# 55. Väga lühike kokkuvõte

Selle pileti lahendus kõige lihtsamalt:

```text
1. Teen DC1 serverist esimese domeenikontrolleri domeenile oige.local.
2. Seadistan DC2 Server Core’is käsurealt.
3. Teen DC2 serverist teise domeenikontrolleri.
4. Paigaldan ja seadistan DHCP failoveri DC1 ja DC2 vahel.
5. Loon OU-d, Haldur kasutaja, CSV impordi ja GPO Teade.
6. Liidan Windows 11 kliendi domeeniga.
7. Lisan DNS A-kirjed Linuxi serveritele.
8. Paigaldan DC1 ja DC2 serverisse DFS Namespaces, DFS Replication ja FSRM rollid.
9. Loon DFS namespace \\oige.local\Jagatud.
10. Loon DFS kaustad Kogukond ja Isiklik.
11. Loon mõlemale DFS Replication grupid DC1 ja DC2 vahel.
12. Määran Kogukond kaustale 10 GB kvoodi.
13. Blokeerin Kogukond kaustas .msi, .exe, .bat ja .ps1 failid.
14. Loon isiklikud kasutajakaustad.
15. Määran isiklikele kaustadele 1 GB kvoodi kasutaja kohta.
16. Loon GPO Kogukond, mis jagab Y: ketta.
17. Loon GPO Isiklik, mis jagab Z: ketta.
18. Kontrollin DNS-i, AD-d, DHCP-d, DFS-i, FSRM-i ja GPO-sid.
19. Dokumenteerin kogu protsessi.
```
