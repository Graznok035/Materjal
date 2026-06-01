# Windows pilet 2 lahenduskäik

See juhend kirjeldab ainult **Windows pilet 2 eriosa**.

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

---

## 1. Windows pilet 2 eesmärk

Windows pilet 2 põhiteemad on:

| Teema | Mida tuleb teha | Milleks seda vaja on |
|---|---|---|
| DFS Namespaces | Luua domeenipõhine DFS nimeruum | Et jagatud kaustad oleksid kättesaadavad ühe domeeninime kaudu |
| DFS Replication | Seadistada replikatsioon DC1 ja DC2 vahel | Et jagatud kaustad oleksid mõlemas serveris olemas |
| File Server Resource Manager | Seadistada mahupiirangud ja failitüüpide piirangud | Et kontrollida kettaruumi ja keelata programmifailid |
| Kogukond kaust | Luua ühine jagatud kaust kõigile domeeni kasutajatele | Et ettevõtte kasutajad saaksid jagatud faile kasutada |
| Isiklikud kaustad | Luua kasutajatele isiklikud kaustad | Et igal kasutajal oleks oma võrgukaust |
| GPO võrgukettad | Ühendada jagatud kaustad kasutajatele automaatselt | Et kasutaja näeks kaustu Windowsis kettatähtedena |
| Õigused | Piirata ligipääs õigete AD gruppidega | Et andmed oleksid turvaliselt hallatud |
| Testimine | Kontrollida DFS-i, replikatsiooni, GPO-d ja FSRM-i | Et tõendada lahenduse toimimist |

---

# 2. Vajalikud rollid ja tööriistad

Windows pilet 2 jaoks on vaja lisaks üldosale:

| Roll / tööriist | Kus paigaldada | Milleks vajalik |
|---|---|---|
| DFS Namespaces | DC1 ja DC2 | DFS nimeruumi loomiseks ja haldamiseks |
| DFS Replication | DC1 ja DC2 | Kaustade replikatsiooniks serverite vahel |
| File Server Resource Manager | DC1 ja DC2 | Quota ja file screen seadistamiseks |
| Group Policy Management | DC1 | Võrguketaste GPO loomiseks |
| Active Directory Users and Computers | DC1 | Gruppide ja OU-de haldamiseks |
| File and Storage Services | DC1 ja DC2 | Jagatud kaustade haldamiseks |

---

## 2.1 Paigalda rollid DC1-s graafiliselt

DC1-s:

```text
Server Manager
→ Manage
→ Add Roles and Features
→ Role-based or feature-based installation
→ Select server DC1
→ File and Storage Services
→ File and iSCSI Services
```

Vali:

```text
DFS Namespaces
DFS Replication
File Server Resource Manager
```

Seejärel:

```text
Install
```

PowerShelli alternatiiv DC1-s:

```powershell
Install-WindowsFeature FS-DFS-Namespace,FS-DFS-Replication,FS-Resource-Manager -IncludeManagementTools
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab DFS Namespace, DFS Replication ja FSRM rollid | Rollid paigaldatakse edukalt |

Kontroll:

```powershell
Get-WindowsFeature FS-DFS-Namespace,FS-DFS-Replication,FS-Resource-Manager
```

Oodatav:

```text
Install State = Installed
```

---

## 2.2 Paigalda rollid DC2-s

DC2 Server Core PowerShellis:

```powershell
Install-WindowsFeature FS-DFS-Namespace,FS-DFS-Replication,FS-Resource-Manager
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab vajalikud failiserveri rollid DC2 masinasse | DC2 saab DFS nimeruumi ja replikatsiooni partneriks |

Kontroll DC2-s:

```powershell
Get-WindowsFeature FS-DFS-Namespace,FS-DFS-Replication,FS-Resource-Manager
```

Oodatav:

```text
Install State = Installed
```

---

# 3. Soovitatav struktuur

Näidis domeen:

```text
sinuNimi.local
```

DFS nimeruum:

```text
\\sinuNimi.local\Jagatud
```

DFS kaustad:

```text
\\sinuNimi.local\Jagatud\Kogukond
\\sinuNimi.local\Jagatud\Isiklik
```

Serverite lokaalsed kaustad:

| Server | Kaust | Milleks |
|---|---|---|
| DC1 | `D:\DFSRoots\Jagatud` | DFS nimeruumi root |
| DC2 | `D:\DFSRoots\Jagatud` | DFS nimeruumi root |
| DC1 | `D:\Shares\Kogukond` | Kogukond kausta andmed |
| DC2 | `D:\Shares\Kogukond` | Kogukond koopia |
| DC1 | `D:\Shares\Isiklik` | Kasutajate isiklikud kaustad |
| DC2 | `D:\Shares\Isiklik` | Isiklik kaustade koopia |

Kui D: ketast pole, võib kasutada C: ketast:

```text
C:\DFSRoots\Jagatud
C:\Shares\Kogukond
C:\Shares\Isiklik
```

Eksamil on D: parem, kui eraldi andmeketas on olemas.

---

# 4. Loo AD grupid

## 4.1 Loo grupid ADUC-is

Ava DC1-s:

```text
Server Manager
→ Tools
→ Active Directory Users and Computers
```

Loo näiteks OU `Kasutajad` alla või eraldi OU `Grupid` alla järgmised turvagrupid:

| Grupp | Tüüp | Milleks |
|---|---|---|
| `GG_Kogukond_RW` | Global Security | Kogukond kausta lugemis- ja kirjutamisõigus |
| `GG_Isiklik_Users` | Global Security | Isiklike kaustade kasutajad |

Kui ülesandes on kirjas, et kõigil domeeni kasutajatel peab olema ligipääs `Kogukond` kaustale, lisa gruppi `GG_Kogukond_RW` kõik vajalikud kasutajad või kasuta `Domain Users` gruppi.

---

## 4.2 PowerShelli alternatiiv

```powershell
New-ADGroup -Name "GG_Kogukond_RW" -GroupScope Global -GroupCategory Security -Path "OU=Kasutajad,DC=sinuNimi,DC=local"
New-ADGroup -Name "GG_Isiklik_Users" -GroupScope Global -GroupCategory Security -Path "OU=Kasutajad,DC=sinuNimi,DC=local"
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob AD turvagrupid | Grupid on ADUC-is nähtavad |

Lisa kasutajad gruppi:

```powershell
Add-ADGroupMember -Identity "GG_Kogukond_RW" -Members "Domain Users"
Add-ADGroupMember -Identity "GG_Isiklik_Users" -Members "Domain Users"
```

Kontroll:

```powershell
Get-ADGroupMember GG_Kogukond_RW
```

Oodatav:

```text
Näha on kasutajad või Domain Users grupp
```

---

# 5. Loo kaustad DC1 ja DC2 serverites

## 5.1 Loo kaustad DC1-s

DC1 PowerShellis administraatorina:

```powershell
New-Item -ItemType Directory -Path "C:\DFSRoots\Jagatud" -Force
New-Item -ItemType Directory -Path "C:\Shares\Kogukond" -Force
New-Item -ItemType Directory -Path "C:\Shares\Isiklik" -Force
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob DFS root ja jagatavad kaustad DC1-s | Kaustad on olemas |

Kontroll:

```powershell
Test-Path "C:\DFSRoots\Jagatud"
Test-Path "C:\Shares\Kogukond"
Test-Path "C:\Shares\Isiklik"
```

Oodatav:

```text
True
True
True
```

---

## 5.2 Loo kaustad DC2-s

DC2 PowerShellis:

```powershell
New-Item -ItemType Directory -Path "C:\DFSRoots\Jagatud" -Force
New-Item -ItemType Directory -Path "C:\Shares\Kogukond" -Force
New-Item -ItemType Directory -Path "C:\Shares\Isiklik" -Force
```

Kontroll:

```powershell
Test-Path "C:\DFSRoots\Jagatud"
Test-Path "C:\Shares\Kogukond"
Test-Path "C:\Shares\Isiklik"
```

Oodatav:

```text
True
True
True
```

---

# 6. Loo SMB jagamised

## 6.1 Loo jagamised DC1-s

DC1 PowerShellis:

```powershell
New-SmbShare -Name "DFSRoot$" -Path "C:\DFSRoots\Jagatud" -FullAccess "Domain Admins" -ChangeAccess "Authenticated Users"
New-SmbShare -Name "Kogukond$" -Path "C:\Shares\Kogukond" -FullAccess "Domain Admins" -ChangeAccess "GG_Kogukond_RW"
New-SmbShare -Name "Isiklik$" -Path "C:\Shares\Isiklik" -FullAccess "Domain Admins" -ChangeAccess "GG_Isiklik_Users"
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob peidetud SMB jagamised DC1-s | Jagamised on nähtavad käsuga `Get-SmbShare` |

Kontroll:

```powershell
Get-SmbShare
```

Oodatav:

```text
DFSRoot$
Kogukond$
Isiklik$
```

---

## 6.2 Loo jagamised DC2-s

DC2 PowerShellis:

```powershell
New-SmbShare -Name "DFSRoot$" -Path "C:\DFSRoots\Jagatud" -FullAccess "Domain Admins" -ChangeAccess "Authenticated Users"
New-SmbShare -Name "Kogukond$" -Path "C:\Shares\Kogukond" -FullAccess "Domain Admins" -ChangeAccess "GG_Kogukond_RW"
New-SmbShare -Name "Isiklik$" -Path "C:\Shares\Isiklik" -FullAccess "Domain Admins" -ChangeAccess "GG_Isiklik_Users"
```

Kontroll:

```powershell
Get-SmbShare
```

Oodatav:

```text
DFSRoot$
Kogukond$
Isiklik$
```

---

# 7. Seadista NTFS õigused

## 7.1 Kogukond kausta õigused DC1-s

DC1-s:

```powershell
icacls "C:\Shares\Kogukond" /inheritance:r
icacls "C:\Shares\Kogukond" /grant "Domain Admins:(OI)(CI)F"
icacls "C:\Shares\Kogukond" /grant "GG_Kogukond_RW:(OI)(CI)M"
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Eemaldab päriluse ja annab õigused adminidele ning Kogukond grupile | Kasutajad saavad kausta kirjutada, kui nad on grupis |

Kontroll:

```powershell
icacls "C:\Shares\Kogukond"
```

Oodatav:

```text
Domain Admins = Full
GG_Kogukond_RW = Modify
```

---

## 7.2 Kogukond kausta õigused DC2-s

DC2-s:

```powershell
icacls "C:\Shares\Kogukond" /inheritance:r
icacls "C:\Shares\Kogukond" /grant "Domain Admins:(OI)(CI)F"
icacls "C:\Shares\Kogukond" /grant "GG_Kogukond_RW:(OI)(CI)M"
```

Kontroll:

```powershell
icacls "C:\Shares\Kogukond"
```

---

## 7.3 Isiklik kausta üldõigused

Isiklik kausta puhul on loogika:

```text
Kasutaja näeb ja kasutab enda kausta.
Admin saab hallata kõiki kaustu.
```

DC1-s:

```powershell
icacls "C:\Shares\Isiklik" /inheritance:r
icacls "C:\Shares\Isiklik" /grant "Domain Admins:(OI)(CI)F"
icacls "C:\Shares\Isiklik" /grant "CREATOR OWNER:(OI)(CI)F"
icacls "C:\Shares\Isiklik" /grant "GG_Isiklik_Users:(RX)"
```

DC2-s sama:

```powershell
icacls "C:\Shares\Isiklik" /inheritance:r
icacls "C:\Shares\Isiklik" /grant "Domain Admins:(OI)(CI)F"
icacls "C:\Shares\Isiklik" /grant "CREATOR OWNER:(OI)(CI)F"
icacls "C:\Shares\Isiklik" /grant "GG_Isiklik_Users:(RX)"
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Seadistab isiklike kaustade baastaseme õigused | Kasutajad saavad oma kaustu kasutada, adminid saavad kõike hallata |

---

# 8. Loo DFS nimeruum

## 8.1 Ava DFS Management

DC1-s:

```text
Server Manager
→ Tools
→ DFS Management
```

Kui tööriista pole näha, kontrolli, et DFS Management Tools oleks paigaldatud.

---

## 8.2 Loo uus namespace

DFS Managementis:

```text
Namespaces
→ New Namespace
```

Vali namespace server:

```text
DC1
```

Namespace name:

```text
Jagatud
```

Vali:

```text
Domain-based namespace
```

Tulemus:

```text
\\sinuNimi.local\Jagatud
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob domeenipõhise DFS nimeruumi | Kasutajad saavad kasutada teed `\\sinuNimi.local\Jagatud` |

---

## 8.3 Lisa DC2 namespace serveriks

DFS Managementis:

```text
Namespaces
→ \\sinuNimi.local\Jagatud
→ Namespace Servers
→ Add Namespace Server
```

Lisa:

```text
DC2
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lisab DC2 nimeruumi teiseks serveriks | DFS namespace on kättesaadav mõlema DC kaudu |

Kontroll:

```powershell
Get-DfsnRoot
```

Oodatav:

```text
\\sinuNimi.local\Jagatud
```

---

# 9. Lisa DFS kaust Kogukond

## 9.1 Loo DFS folder

DFS Managementis:

```text
\\sinuNimi.local\Jagatud
→ New Folder
```

Folder name:

```text
Kogukond
```

Lisa folder targetid:

```text
\\DC1\Kogukond$
\\DC2\Kogukond$
```

Tulemus:

```text
\\sinuNimi.local\Jagatud\Kogukond
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob DFS tee Kogukond kaustale | Kasutaja saab avada `\\sinuNimi.local\Jagatud\Kogukond` |

Kontroll kliendis:

```cmd
dir \\sinuNimi.local\Jagatud\Kogukond
```

Oodatav:

```text
Kaust avaneb
```

---

# 10. Lisa DFS kaust Isiklik

## 10.1 Loo DFS folder

DFS Managementis:

```text
\\sinuNimi.local\Jagatud
→ New Folder
```

Folder name:

```text
Isiklik
```

Lisa folder targetid:

```text
\\DC1\Isiklik$
\\DC2\Isiklik$
```

Tulemus:

```text
\\sinuNimi.local\Jagatud\Isiklik
```

Kontroll kliendis:

```cmd
dir \\sinuNimi.local\Jagatud\Isiklik
```

Oodatav:

```text
Kaust avaneb
```

---

# 11. Seadista DFS Replication

## 11.1 Replikeeri Kogukond kaust

DFS Managementis:

```text
Replication
→ New Replication Group
```

Vali:

```text
Multipurpose replication group
```

Replication group name:

```text
RG_Kogukond
```

Members:

```text
DC1
DC2
```

Topology:

```text
Full mesh
```

Primary member:

```text
DC1
```

Folders to replicate:

```text
C:\Shares\Kogukond
```

DC2 local path:

```text
C:\Shares\Kogukond
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob replikatsiooni Kogukond kaustale DC1 ja DC2 vahel | Failid kopeeruvad mõlema serveri vahel |

---

## 11.2 Replikeeri Isiklik kaust

Sama loogika:

Replication group name:

```text
RG_Isiklik
```

Members:

```text
DC1
DC2
```

Primary member:

```text
DC1
```

Folders to replicate:

```text
C:\Shares\Isiklik
```

DC2 local path:

```text
C:\Shares\Isiklik
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob replikatsiooni isiklikele kaustadele | Kasutajate kaustad replikeeruvad DC1 ja DC2 vahel |

---

## 11.3 Kontrolli DFS replikatsiooni teenust

DC1 ja DC2 PowerShellis:

```powershell
Get-Service DFSR
```

Oodatav:

```text
Running
```

Kui ei tööta:

```powershell
Start-Service DFSR
```

---

## 11.4 Testi replikatsiooni

DC1-s loo testfail:

```powershell
"DFS test" | Out-File "C:\Shares\Kogukond\test-dfs.txt"
```

Oota natuke ja kontrolli DC2-s:

```powershell
Get-ChildItem "C:\Shares\Kogukond"
```

Oodatav:

```text
test-dfs.txt on DC2-s olemas
```

Kui fail ei ilmu kohe:

```text
DFS Replication võib võtta aega. Oota mõni minut ja kontrolli uuesti.
```

Kontrolli DFS Replication event logi:

```powershell
Get-WinEvent -LogName "DFS Replication" -MaxEvents 20
```

---

# 12. Loo isiklikud kasutajakaustad

## 12.1 Loo kaustad kasutajate jaoks

DC1-s saab luua isiklikud kaustad AD kasutajate põhjal.

Näidis:

```powershell
$users = Get-ADUser -Filter * -SearchBase "OU=Kasutajad,DC=sinuNimi,DC=local"

foreach ($u in $users) {
    $path = "C:\Shares\Isiklik\$($u.SamAccountName)"
    New-Item -ItemType Directory -Path $path -Force

    icacls $path /inheritance:r
    icacls $path /grant "Domain Admins:(OI)(CI)F"
    icacls $path /grant "$($u.SamAccountName):(OI)(CI)M"
}
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob igale kasutajale eraldi kausta ja annab õigused | Kasutajal on oma isiklik kaust |

Kui domeeninime on vaja lisada õiguste juures, kasuta:

```powershell
icacls $path /grant "sinuNimi\$($u.SamAccountName):(OI)(CI)M"
```

Kontroll:

```powershell
Get-ChildItem "C:\Shares\Isiklik"
```

Oodatav:

```text
Näha on kasutajanimedega kaustad
```

---

# 13. Seadista FSRM

## 13.1 Ava File Server Resource Manager

DC1-s:

```text
Server Manager
→ Tools
→ File Server Resource Manager
```

Kui tööriist puudub, kontrolli rolli:

```powershell
Get-WindowsFeature FS-Resource-Manager
```

---

# 14. Kogukond kausta 10 GB quota

## 14.1 Loo quota

FSRM-is:

```text
Quota Management
→ Quotas
→ Create Quota
```

Quota path:

```text
C:\Shares\Kogukond
```

Vali:

```text
Create quota on path
```

Custom properties:

```text
Limit = 10 GB
Hard quota
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Piirab Kogukond kausta mahu 10 GB peale | Kaust ei saa üle 10 GB kasvada |

PowerShelli alternatiiv:

```powershell
New-FsrmQuota -Path "C:\Shares\Kogukond" -Size 10GB
```

Kontroll:

```powershell
Get-FsrmQuota -Path "C:\Shares\Kogukond"
```

Oodatav:

```text
Size = 10 GB
```

---

# 15. Isiklike kaustade 1 GB quota kasutaja kohta

## 15.1 Loo auto quota

FSRM-is:

```text
Quota Management
→ Quotas
→ Create Quota
```

Quota path:

```text
C:\Shares\Isiklik
```

Vali:

```text
Auto apply template and create quotas on existing and new subfolders
```

Custom properties:

```text
Limit = 1 GB
Hard quota
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob igale alamkaustale automaatselt 1 GB piirangu | Igal kasutajal on 1 GB isiklik ruum |

PowerShelli alternatiiv:

```powershell
New-FsrmAutoQuota -Path "C:\Shares\Isiklik" -Size 1GB
```

Kontroll:

```powershell
Get-FsrmAutoQuota
```

Oodatav:

```text
C:\Shares\Isiklik auto quota on olemas
```

---

# 16. Keela programmifailide kopeerimine Kogukond kausta

## 16.1 Loo File Group

FSRM-is:

```text
File Screening Management
→ File Groups
→ Create File Group
```

Name:

```text
KeelatudProgrammid
```

Files to include:

```text
*.exe
*.bat
*.ps1
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob failitüüpide grupi programmifailide jaoks | FSRM tunneb keelatud failitüübid ära |

PowerShelli alternatiiv:

```powershell
New-FsrmFileGroup -Name "KeelatudProgrammid" -IncludePattern @("*.exe","*.bat","*.ps1")
```

---

## 16.2 Loo File Screen Kogukond kaustale

FSRM-is:

```text
File Screening Management
→ File Screens
→ Create File Screen
```

File screen path:

```text
C:\Shares\Kogukond
```

Vali:

```text
Active screening
```

Keelatud file group:

```text
KeelatudProgrammid
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Keelab `.exe`, `.bat` ja `.ps1` failide salvestamise Kogukond kausta | Programmifailide kopeerimine sinna ebaõnnestub |

PowerShelli alternatiiv:

```powershell
New-FsrmFileScreen -Path "C:\Shares\Kogukond" -IncludeGroup "KeelatudProgrammid" -Active
```

Kontroll:

```powershell
Get-FsrmFileScreen -Path "C:\Shares\Kogukond"
```

Oodatav:

```text
File screen on aktiivne ja kasutab gruppi KeelatudProgrammid
```

---

# 17. Tee GPO võrguketaste ühendamiseks

## 17.1 Ava Group Policy Management

DC1-s:

```text
Server Manager
→ Tools
→ Group Policy Management
```

Loo uus GPO:

```text
GPO_DFS_Kettad
```

Lingi see kasutajate OU-le, näiteks:

```text
OU=Kasutajad
```

Kui piletis on eraldi OU-d, lingi sinna, kus kasutajad asuvad.

---

## 17.2 Lisa Kogukond võrgukettaks

GPO-s:

```text
User Configuration
→ Preferences
→ Windows Settings
→ Drive Maps
→ New
→ Mapped Drive
```

Täida:

| Väli | Väärtus |
|---|---|
| Action | Update |
| Location | `\\sinuNimi.local\Jagatud\Kogukond` |
| Drive Letter | näiteks `K:` |
| Label as | `Kogukond` |
| Reconnect | Enabled |

| Mida see teeb | Oodatav tulemus |
|---|---|
| Ühendab Kogukond kausta kasutajale võrgukettana | Kasutaja näeb K: ketast |

---

## 17.3 Lisa Isiklik võrgukettaks

Isikliku kausta puhul kasuta kasutajanime muutujat:

```text
\\sinuNimi.local\Jagatud\Isiklik\%USERNAME%
```

GPO Drive Map:

| Väli | Väärtus |
|---|---|
| Action | Update |
| Location | `\\sinuNimi.local\Jagatud\Isiklik\%USERNAME%` |
| Drive Letter | näiteks `I:` |
| Label as | `Isiklik` |
| Reconnect | Enabled |

| Mida see teeb | Oodatav tulemus |
|---|---|
| Ühendab kasutaja isikliku kausta võrgukettana | Kasutaja näeb ainult enda isiklikku kausta |

---

# 18. Testi GPO-d Windows 11 kliendis

Logi Windows 11 klienti domeeni kasutajana.

Käivita:

```cmd
gpupdate /force
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Rakendab GPO-d kohe | Policy update completed successfully |

Kontrolli GPO-d:

```cmd
gpresult /r
```

Oodatav:

```text
GPO_DFS_Kettad on rakendunud
```

Kontrolli kettad:

```cmd
net use
```

Oodatav:

```text
K: -> \\sinuNimi.local\Jagatud\Kogukond
I: -> \\sinuNimi.local\Jagatud\Isiklik\kasutajanimi
```

File Exploreris peaksid olema nähtavad:

```text
K: Kogukond
I: Isiklik
```

---

# 19. Testi FSRM piiranguid

## 19.1 Testi keelatud failitüüpe

Windows 11 kliendis proovi Kogukond kettale kopeerida:

```text
test.exe
test.bat
test.ps1
```

Oodatav:

```text
Kopeerimine keelatakse
```

Kui fail läheb läbi:

| Kontroll | Lahendus |
|---|---|
| Kas File Screen on C:\Shares\Kogukond peal? | Kontrolli FSRM-is |
| Kas File Group sisaldab *.exe, *.bat, *.ps1? | Lisa mustrid |
| Kas testid õiget DFS kausta? | Kontrolli tee `\\sinuNimi.local\Jagatud\Kogukond` |
| Kas fail läheb DC2 koopiasse, kus FSRM pole seadistatud? | Seadista FSRM mõlemas serveris või suuna test DC1-le |

Soovitus:

```text
FSRM piirangud seadista mõlemas failiserveris, sest DFS võib suunata kasutaja DC1 või DC2 targetile.
```

---

## 19.2 Testi Kogukond 10 GB quota

FSRM-is kontrolli quota olemasolu:

```powershell
Get-FsrmQuota -Path "C:\Shares\Kogukond"
```

Oodatav:

```text
Limit = 10 GB
```

Eksamil ei pea päriselt 10 GB täis kirjutama, piisab seadistuse näitamisest.

---

## 19.3 Testi Isiklik 1 GB quota

Kontroll:

```powershell
Get-FsrmAutoQuota
```

Oodatav:

```text
C:\Shares\Isiklik auto quota on olemas
```

Kui kasutajakaust on loodud, kontrolli konkreetset quota’t:

```powershell
Get-FsrmQuota
```

Oodatav:

```text
Kasutaja alamkaustadele on tekkinud 1 GB quota
```

---

# 20. Lõppkontroll

| Kontroll | Käsk või koht | Oodatav tulemus |
|---|---|---|
| Rollid DC1 | `Get-WindowsFeature FS-DFS-Namespace,FS-DFS-Replication,FS-Resource-Manager` | Installed |
| Rollid DC2 | sama käsk DC2-s | Installed |
| DFS namespace | DFS Management | `\\sinuNimi.local\Jagatud` olemas |
| DFS Kogukond | `dir \\sinuNimi.local\Jagatud\Kogukond` | Kaust avaneb |
| DFS Isiklik | `dir \\sinuNimi.local\Jagatud\Isiklik` | Kaust avaneb |
| Replikatsioon | testfail DC1 → DC2 | Fail ilmub teises serveris |
| GPO | `gpresult /r` | `GPO_DFS_Kettad` rakendub |
| Võrgukettad | `net use` | K: ja I: olemas |
| Kogukond quota | `Get-FsrmQuota -Path "C:\Shares\Kogukond"` | 10 GB |
| Isiklik quota | `Get-FsrmAutoQuota` | 1 GB per alamkaust |
| File screen | `Get-FsrmFileScreen -Path "C:\Shares\Kogukond"` | aktiivne |
| Keelatud fail | kopeeri `.exe` K: kettale | kopeerimine keelatakse |

---

# 21. Dokumentatsiooni näidis

```markdown
## Windows pilet 2 dokumentatsioon

### Eesmärk

Eesmärk oli seadistada domeenikeskkonnas jagatud faililahendus DFS-i abil. Lahenduses tuli luua domeenipõhine DFS nimeruum, seadistada jagatud kaustad `Kogukond` ja `Isiklik`, seadistada replikatsioon DC1 ja DC2 vahel, ühendada kaustad kasutajatele GPO abil võrgukettana ning piirata kettakasutust ja failitüüpe File Server Resource Manageriga.

### Kasutatud serverid

| Server | Roll |
|---|---|
| DC1 | AD DS, DNS, DHCP, DFS Namespace, DFS Replication, FSRM |
| DC2 | Teine domeenikontroller, DFS Namespace, DFS Replication, FSRM |
| Windows 11 klient | GPO ja võrguketaste testimine |

### Tehtud seadistused

- Paigaldasin DC1 ja DC2 serveritesse DFS Namespaces rolli.
- Paigaldasin DC1 ja DC2 serveritesse DFS Replication rolli.
- Paigaldasin DC1 ja DC2 serveritesse File Server Resource Manager rolli.
- Lõin kaustad `C:\Shares\Kogukond` ja `C:\Shares\Isiklik`.
- Lõin SMB jagamised `Kogukond$` ja `Isiklik$`.
- Seadistasin NTFS õigused AD gruppidega.
- Lõin domeenipõhise DFS nimeruumi `\\sinuNimi.local\Jagatud`.
- Lisasin DFS kaustad `Kogukond` ja `Isiklik`.
- Seadistasin DFS replikatsiooni DC1 ja DC2 vahel.
- Lõin AD grupid `GG_Kogukond_RW` ja `GG_Isiklik_Users`.
- Lõin kasutajatele isiklikud kaustad.
- Seadistasin FSRM-is `Kogukond` kaustale 10 GB quota.
- Seadistasin isiklikele kaustadele 1 GB auto quota.
- Lõin FSRM File Screen reegli, mis keelab `.exe`, `.bat` ja `.ps1` failid.
- Lõin GPO `GPO_DFS_Kettad`.
- Ühendasin GPO-ga `Kogukond` kausta K: kettana.
- Ühendasin GPO-ga kasutaja isikliku kausta I: kettana.
- Testisin GPO rakendumist Windows 11 kliendis.
- Testisin DFS ligipääsu, replikatsiooni ja FSRM piiranguid.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| DFS nimeruum | `\\sinuNimi.local\Jagatud` avaneb |
| Kogukond kaust | `\\sinuNimi.local\Jagatud\Kogukond` avaneb |
| Isiklik kaust | `\\sinuNimi.local\Jagatud\Isiklik` avaneb |
| GPO | `gpresult /r` näitab `GPO_DFS_Kettad` |
| Võrgukettad | `net use` näitab K: ja I: kettaid |
| Replikatsioon | DC1 loodud fail tekib DC2-s |
| FSRM quota | Kogukond kaustal 10 GB piirang |
| FSRM auto quota | Isiklik alamkaustadel 1 GB piirang |
| File screen | `.exe`, `.bat`, `.ps1` failide kopeerimine on keelatud |

### Kokkuvõte

Windows pilet 2 tulemusena töötab domeenipõhine failijagamise lahendus DFS-i abil. Kasutajad saavad GPO kaudu automaatselt võrgukettad `Kogukond` ja `Isiklik`. Jagatud kaustad replikeeruvad DC1 ja DC2 vahel ning FSRM piirab kettakasutust ja keelab programmifailide salvestamise ühiskausta.
```

---

# 22. Troubleshooting

## DFS tee ei avane

Kontrolli kliendis:

```cmd
dir \\sinuNimi.local\Jagatud
```

Kui ei avane:

| Põhjus | Lahendus |
|---|---|
| DNS probleem | Kontrolli, et klient kasutab DC1/DC2 DNS-i |
| DFS namespace puudub | Kontrolli DFS Managementis |
| Jagamine puudub | Kontrolli `Get-SmbShare` DC1/DC2-s |
| Õigused puuduvad | Kontrolli NTFS ja share õiguseid |

---

## Võrgukettad ei ilmu

Kliendis:

```cmd
gpupdate /force
gpresult /r
net use
```

Kui kettaid pole:

| Põhjus | Lahendus |
|---|---|
| GPO pole rakendunud | Kontrolli GPO linki õigele OU-le |
| Kasutaja pole õiges OU-s | Tõsta kasutaja õigesse OU-sse |
| Drive Map tee vale | Kontrolli `\\sinuNimi.local\Jagatud\Kogukond` |
| DNS probleem | Kontrolli `nslookup sinuNimi.local` |

---

## Replikatsioon ei tööta

Kontrolli teenust:

```powershell
Get-Service DFSR
```

Vaata sündmuste logi:

```powershell
Get-WinEvent -LogName "DFS Replication" -MaxEvents 20
```

Kui fail ei ilmu kohe:

```text
DFS Replication ei pruugi olla sekunditega nähtav.
Oota mõni minut ja kontrolli uuesti.
```

Kontrolli, kas mõlemad serverid on replication group liikmed.

---

## Kasutaja ei saa Kogukond kausta kirjutada

Kontrolli:

```powershell
icacls "C:\Shares\Kogukond"
Get-SmbShareAccess Kogukond$
Get-ADGroupMember GG_Kogukond_RW
```

Kui kasutaja lisati gruppi hiljuti:

```text
Logi kasutaja välja ja sisse tagasi.
```

---

## File Screen ei keela .exe faile

Kontrolli:

```powershell
Get-FsrmFileGroup
Get-FsrmFileScreen -Path "C:\Shares\Kogukond"
```

Kui DFS suunab kasutaja DC2 peale, aga FSRM on ainult DC1-s, võib piirang mitte rakenduda.

Parandus:

```text
Seadista sama FSRM file screen ka DC2-s.
```

---

## Isiklik ketas ei avane

Kontrolli, kas kasutaja kaust eksisteerib:

```powershell
Test-Path "C:\Shares\Isiklik\kasutajanimi"
```

Kontrolli õiguseid:

```powershell
icacls "C:\Shares\Isiklik\kasutajanimi"
```

Kontrolli GPO teed:

```text
\\sinuNimi.local\Jagatud\Isiklik\%USERNAME%
```

Kui kasutajakausta pole, loo see või kasuta skripti kasutajakaustade loomiseks.

---

# 23. Kõige lühem spikker

```text
1. Paigalda DC1 ja DC2 serveritesse DFS Namespaces
2. Paigalda DC1 ja DC2 serveritesse DFS Replication
3. Paigalda DC1 ja DC2 serveritesse FSRM
4. Loo AD grupid GG_Kogukond_RW ja GG_Isiklik_Users
5. Loo kaustad C:\Shares\Kogukond ja C:\Shares\Isiklik mõlemas serveris
6. Loo SMB share’id Kogukond$ ja Isiklik$
7. Seadista NTFS õigused
8. Loo DFS namespace \\sinuNimi.local\Jagatud
9. Lisa DC2 namespace serveriks
10. Lisa DFS kaust Kogukond
11. Lisa DFS kaust Isiklik
12. Loo DFS Replication group Kogukond jaoks
13. Loo DFS Replication group Isiklik jaoks
14. Loo kasutajate isiklikud kaustad
15. Seadista Kogukond kaustale FSRM 10 GB quota
16. Seadista Isiklik kaustadele FSRM 1 GB auto quota
17. Loo File Group *.exe, *.bat, *.ps1
18. Loo File Screen Kogukond kaustale
19. Loo GPO_DFS_Kettad
20. Ühenda Kogukond K: kettaks
21. Ühenda Isiklik I: kettaks
22. Testi gpupdate /force, gpresult /r ja net use
23. Testi DFS replikatsiooni
24. Testi FSRM piiranguid
25. Dokumenteeri
```
