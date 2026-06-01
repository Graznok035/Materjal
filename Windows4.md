# Windows pilet 4 lahenduskäik

See juhend kirjeldab ainult **Windows pilet 4 eriosa**.

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

## 1. Windows pilet 4 eesmärk

Windows pilet 4 keskendub tarkvara paigaldamisele ja turvapiirangutele Group Policy abil.

| Teema | Mida tuleb teha | Milleks seda vaja on |
|---|---|---|
| LibreOffice | Paigaldada LibreOffice klientarvutitesse | Kontoritarkvara kasutajatele |
| FastStone | Paigaldada FastStone tarkvara klientarvutitesse | Pildihalduse / ekraanipiltide tööriist |
| Google Chrome | Paigaldada Chrome veebilehitseja | Ühtne veebilehitseja |
| Chrome avaleht | Määrata Chrome avaleht | Et kasutajatel avaneks õige sise- või ettevõtte veeb |
| Ettevõtte taustapilt | Seadistada kõigile kasutajatele ettevõtte logo/taustapilt | Ühtne tööjaama kujundus |
| Lokaalsete kontode piiramine | Keelata kohalike kontodega sisselogimine | Et masinatesse logitaks ainult AD kontodega |
| Viimase kasutaja peitmine | Peita viimati sisse loginud kasutaja nimi | Turvalisuse suurendamiseks |
| Kasutajapiirangud | Keelata CMD, PowerShell, juhtpaneel, Task Manager ja Registry Editor teatud OU kasutajatele | Et tavakasutajad ei saaks süsteemi muuta |
| GPO testimine | Kontrollida GPO rakendumist Windows 11 kliendis | Et tõendada lahenduse toimimist |

---

# 2. Vajalikud failid ja tööriistad

## 2.1 Vajalikud installerid

Tarkvara paigaldamiseks GPO kaudu on kõige parem kasutada `.msi` paigalduspakette.

| Tarkvara | Soovitatav paigaldusfail | Märkus |
|---|---|---|
| LibreOffice | `.msi` | Sobib GPO Software Installation jaoks |
| Google Chrome Enterprise | `.msi` | Kasuta Enterprise MSI installerit |
| FastStone | `.msi`, kui olemas | Kui ainult `.exe`, kasuta startup scripti |

Oluline:

```text
GPO Software Installation töötab kõige lihtsamalt MSI failidega.
Kui tarkvara on ainult EXE kujul, tuleb kasutada startup scripti või teisendada paigaldus loogiliseks käsuks.
```

---

## 2.2 Tarkvarajagamise kaust

Soovitatav luua DC1-s eraldi kaust:

```text
C:\Software
```

Jagatud võrgutee:

```text
\\DC1\Software$
```

Sinna pane:

```text
LibreOffice.msi
ChromeEnterprise.msi
FastStone.msi või FastStone.exe
taustapilt.jpg
```

---

# 3. Loo tarkvara jagamise kaust

## 3.1 Loo kaust DC1-s

PowerShell administraatorina:

```powershell
New-Item -ItemType Directory -Path "C:\Software" -Force
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob tarkvara paigaldusfailide kausta | Kaust `C:\Software` on olemas |

Kontroll:

```powershell
Test-Path "C:\Software"
```

Oodatav:

```text
True
```

---

## 3.2 Loo SMB share

```powershell
New-SmbShare -Name "Software$" -Path "C:\Software" -ReadAccess "Domain Computers","Domain Users" -FullAccess "Domain Admins"
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Jagab kausta võrku peidetud share’ina `Software$` | Kliendid saavad paigaldusfaile lugeda |

Kontroll:

```powershell
Get-SmbShare Software$
```

Oodatav:

```text
Software$ share on olemas
```

Kontroll võrguteega:

```powershell
Test-Path "\\DC1\Software$"
```

Oodatav:

```text
True
```

---

## 3.3 Kopeeri installerid kausta

Kopeeri installerid kausta:

```text
C:\Software
```

Näiteks:

```text
C:\Software\LibreOffice.msi
C:\Software\ChromeEnterprise.msi
C:\Software\FastStone.msi
C:\Software\taustapilt.jpg
```

Kontroll:

```powershell
Get-ChildItem "C:\Software"
```

Oodatav:

```text
Näed tarkvara paigaldusfaile ja taustapildi faili
```

---

# 4. Loo GPO tarkvara paigaldamiseks

## 4.1 Ava Group Policy Management

DC1-s:

```text
Server Manager
→ Tools
→ Group Policy Management
```

Loo uus GPO:

```text
GPO_Tarkvara_Paigaldus
```

Lingi see OU-le:

```text
Arvutid
```

Miks `Arvutid` OU?

```text
Tarkvara paigaldamine toimub tavaliselt Computer Configuration alt.
Seetõttu peab GPO olema lingitud arvutite OU-le.
```

---

# 5. LibreOffice paigaldamine GPO kaudu

## 5.1 Lisa MSI pakett

GPO-s:

```text
Computer Configuration
→ Policies
→ Software Settings
→ Software installation
→ New
→ Package
```

Vali fail kindlasti **UNC võrguteega**, mitte lokaalse C: teega:

```text
\\DC1\Software$\LibreOffice.msi
```

Vali:

```text
Assigned
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Määrab LibreOffice’i arvutitele automaatselt paigaldatavaks | Tarkvara paigaldub kliendi järgmise käivituse ajal |

Oluline:

```text
Ära vali faili teega C:\Software\LibreOffice.msi.
Klientarvuti ei näe DC1 lokaalset C: ketast.
Kasuta alati \\DC1\Software$\fail.msi teed.
```

---

## 5.2 Kontroll kliendis

Windows 11 kliendis:

```cmd
gpupdate /force
```

Tee restart:

```cmd
shutdown /r /t 0
```

Pärast restarti kontrolli:

```text
Start Menu → LibreOffice
```

või PowerShellis:

```powershell
Get-Package *LibreOffice*
```

Oodatav:

```text
LibreOffice on paigaldatud
```

Kui ei ole:

| Põhjus | Lahendus |
|---|---|
| GPO pole rakendunud | Kontrolli `gpresult /r` |
| MSI tee vale | Peab olema `\\DC1\Software$\LibreOffice.msi` |
| Arvuti pole OU-s Arvutid | Tõsta arvuti õigesse OU-sse |
| Failiõigused puuduvad | Anna `Domain Computers` lugemisõigus |

---

# 6. Google Chrome paigaldamine GPO kaudu

## 6.1 Lisa Chrome Enterprise MSI

GPO-s:

```text
Computer Configuration
→ Policies
→ Software Settings
→ Software installation
→ New
→ Package
```

Vali:

```text
\\DC1\Software$\ChromeEnterprise.msi
```

Vali:

```text
Assigned
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab Google Chrome’i domeeni arvutitesse | Chrome ilmub pärast restarti klientarvutisse |

Kontroll kliendis:

```cmd
gpupdate /force
shutdown /r /t 0
```

Pärast restarti:

```text
Start Menu → Google Chrome
```

---

# 7. FastStone paigaldamine

## Variant A: kui FastStone on MSI failina

GPO-s:

```text
Computer Configuration
→ Policies
→ Software Settings
→ Software installation
→ New
→ Package
```

Vali:

```text
\\DC1\Software$\FastStone.msi
```

Vali:

```text
Assigned
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab FastStone tarkvara klientarvutitesse | FastStone on pärast restarti olemas |

---

## Variant B: kui FastStone on ainult EXE failina

Kui `.msi` pole, tee startup script.

Loo skript DC1-s:

```powershell
notepad C:\Software\install-faststone.cmd
```

Lisa näiteks:

```cmd
@echo off
if exist "C:\Program Files (x86)\FastStone Capture\FSCapture.exe" exit /b 0
\\DC1\Software$\FastStone.exe /S
exit /b 0
```

Märkus:

```text
Vaikne paigaldusparameeter võib sõltuda FastStone installerist.
Levinud variandid on /S, /silent või /verysilent.
Kui üks ei tööta, kontrolli installerit käsitsi.
```

GPO-s:

```text
Computer Configuration
→ Policies
→ Windows Settings
→ Scripts (Startup/Shutdown)
→ Startup
→ Add
```

Script name:

```text
\\DC1\Software$\install-faststone.cmd
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Käivitab FastStone paigalduse arvuti käivitumisel | Tarkvara paigaldub startup ajal |

Kontroll pärast restarti:

```text
Start Menu → FastStone
```

---

# 8. Chrome ADMX mallide lisamine

Chrome avalehe haldamiseks on vaja Chrome Group Policy malle.

## 8.1 Lisa Chrome ADMX mallid Central Store’i

Kontrolli, kas Central Store on olemas:

```powershell
Test-Path "\\sinuNimi.local\SYSVOL\sinuNimi.local\Policies\PolicyDefinitions"
```

Kui ei ole, loo:

```powershell
New-Item -ItemType Directory -Path "\\sinuNimi.local\SYSVOL\sinuNimi.local\Policies\PolicyDefinitions" -Force
```

Kopeeri Chrome’i mallid:

| Fail | Kuhu |
|---|---|
| `chrome.admx` | `\\sinuNimi.local\SYSVOL\sinuNimi.local\Policies\PolicyDefinitions` |
| `chrome.adml` | `\\sinuNimi.local\SYSVOL\sinuNimi.local\Policies\PolicyDefinitions\en-US` või vastav keelekaust |

Kontroll:

```powershell
Get-ChildItem "\\sinuNimi.local\SYSVOL\sinuNimi.local\Policies\PolicyDefinitions" | Where-Object Name -like "*chrome*"
```

Oodatav:

```text
chrome.admx on olemas
```

Kui Chrome poliitikaid GPO Editoris ei näe, sulge Group Policy Management Editor ja ava uuesti.

---

# 9. Chrome avalehe GPO

## 9.1 Loo GPO

Loo uus GPO:

```text
GPO_Chrome_Avaleht
```

Lingi see kasutajate OU-le, näiteks:

```text
Kasutajad
```

või konkreetsetele osakondade OU-dele.

---

## 9.2 Seadista Chrome avaleht

GPO-s:

```text
User Configuration
→ Policies
→ Administrative Templates
→ Google
→ Google Chrome
```

Seadista:

| Seade | Väärtus |
|---|---|
| Show Home button on toolbar | Enabled |
| Configure the home page URL | `https://siseportaal.sinuNimi.local` |
| Action on startup | Open a list of URLs |
| URLs to open on startup | `https://siseportaal.sinuNimi.local` |
| New Tab Page Location | soovi korral sama siseportaali URL |

Kui piletis on konkreetne veebiaadress, kasuta seda.

| Mida see teeb | Oodatav tulemus |
|---|---|
| Määrab Chrome’i avalehe ja käivituslehe | Kasutajatel avaneb Chrome’is määratud aadress |

---

## 9.3 Kontroll kliendis

Windows 11 kliendis domeeni kasutajana:

```cmd
gpupdate /force
```

Kontroll:

```cmd
gpresult /r
```

Oodatav:

```text
GPO_Chrome_Avaleht rakendub kasutajale
```

Ava Chrome.

Oodatav:

```text
Chrome avaneb määratud avalehega
```

---

# 10. Ettevõtte taustapildi GPO

## 10.1 Paiguta taustapilt võrgujagamisse

Pane taustapilt kausta:

```text
C:\Software\taustapilt.jpg
```

Võrgutee:

```text
\\DC1\Software$\taustapilt.jpg
```

Kontroll kliendis:

```cmd
dir \\DC1\Software$\taustapilt.jpg
```

Oodatav:

```text
Fail on nähtav
```

---

## 10.2 Loo GPO taustapildi jaoks

Loo uus GPO:

```text
GPO_Taustapilt
```

Lingi kasutajate OU-le:

```text
Kasutajad
```

või domeeni sobivale tasemele.

GPO-s:

```text
User Configuration
→ Policies
→ Administrative Templates
→ Desktop
→ Desktop
→ Desktop Wallpaper
```

Seadista:

| Seade | Väärtus |
|---|---|
| Desktop Wallpaper | Enabled |
| Wallpaper Name | `\\DC1\Software$\taustapilt.jpg` |
| Wallpaper Style | Fill või Stretch |

| Mida see teeb | Oodatav tulemus |
|---|---|
| Määrab kasutajatele ettevõtte taustapildi | Kasutaja töölaual kuvatakse määratud pilt |

---

## 10.3 Kontroll kliendis

```cmd
gpupdate /force
```

Logi välja ja sisse tagasi.

Oodatav:

```text
Taustapilt muutub ettevõtte pildiks
```

Kui ei muutu:

| Põhjus | Lahendus |
|---|---|
| Failitee pole võrgutee | Kasuta `\\DC1\Software$\taustapilt.jpg` |
| Kasutajal pole lugemisõigust | Anna `Domain Users` Read õigus |
| GPO pole rakendunud | Kontrolli `gpresult /r` |
| Pilt on vales formaadis | Kasuta `.jpg` või `.bmp` |

---

# 11. Viimase kasutaja nime peitmine sisselogimisel

## 11.1 Loo turva-GPO

Loo GPO:

```text
GPO_Turva_Logon
```

Lingi see OU-le:

```text
Arvutid
```

Miks `Arvutid`?

```text
See on arvutipõhine turvasäte, seega rakendub Computer Configuration kaudu.
```

---

## 11.2 Seadista viimasena loginud kasutaja peitmine

GPO-s:

```text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Local Policies
→ Security Options
```

Luba:

```text
Interactive logon: Don't display last signed-in
```

Vanemates Windowsites võib nimi olla:

```text
Interactive logon: Do not display last user name
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Peidab sisselogimisekraanilt viimase kasutaja nime | Kasutaja peab sisestama kasutajanime ja parooli |

---

# 12. Keela lokaalsete kontodega sisselogimine

## 12.1 Mõte

Piletis on vaja, et klientarvutitesse saaksid sisse logida ainult AD kontod.

Praktiline lahendus:

```text
Deny log on locally kohalikule Users grupile
```

Samas tuleb olla ettevaatlik, et domeeni kasutajaid kogemata ära ei blokeeriks.

---

## 12.2 Seadista GPO

Kasuta sama GPO-d:

```text
GPO_Turva_Logon
```

GPO-s:

```text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Local Policies
→ User Rights Assignment
```

Ava:

```text
Deny log on locally
```

Lisa:

```text
.\Users
```

või konkreetsete arvutite lokaalsed kasutajad, kui neid on.

Alternatiiv turvalisem loogika:

```text
Deny log on locally = kohalik testkasutaja või lokaalne grupp, mida ei kasutata AD kontodeks.
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Takistab kohalikel kontodel arvutisse interaktiivselt sisse logida | AD kontodega saab sisse, lokaalse kontoga ei saa |

Oluline hoiatus:

```text
Ära lisa siia Domain Users gruppi.
Muidu ei saa tavalised domeeni kasutajad enam sisse logida.
```

---

## 12.3 Kontroll kliendis

Windows 11 kliendis:

```cmd
gpupdate /force
```

Tee restart.

Testi:

| Konto | Oodatav tulemus |
|---|---|
| Domeeni kasutaja `sinuNimi\kasutaja` | Saab sisse |
| Lokaalne kasutaja `.\kasutaja` | Ei saa sisse |

Kui ka domeeni kasutaja ei saa sisse, eemalda vale grupp `Deny log on locally` seadest.

---

# 13. Kasutajapiirangute GPO

## 13.1 Kellele piirang rakendub?

Piletis on vaja piirata kasutajaid, kes kuuluvad näiteks OU-desse:

```text
Personal
Müük
Juhtkond
Haldus
Tootmine
```

Piirangud:

```text
CMD keelamine
PowerShelli keelamine
Juhtpaneeli keelamine
Task Manageri keelamine
Registry Editori keelamine
```

Soovitatav:

```text
Loo üks GPO nimega GPO_Kasutaja_Piirangud
Lingi see vajalikele kasutajate OU-dele
Ära lingi seda administraatorite OU-le
```

---

## 13.2 Loo GPO

Group Policy Managementis loo:

```text
GPO_Kasutaja_Piirangud
```

Lingi see OU-dele:

```text
OU=Personal
OU=Müük
OU=Juhtkond
OU=Haldus
OU=Tootmine
```

Kui OU nimed on teised, kasuta tegelikke OU nimesid.

---

# 14. Keela CMD

GPO-s:

```text
User Configuration
→ Policies
→ Administrative Templates
→ System
→ Prevent access to the command prompt
```

Seadista:

```text
Enabled
```

Valik:

```text
Disable the command prompt script processing also?
Yes
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Keelab kasutajal CMD avamise | `cmd.exe` ei käivitu |

---

# 15. Keela Registry Editor

GPO-s:

```text
User Configuration
→ Policies
→ Administrative Templates
→ System
→ Prevent access to registry editing tools
```

Seadista:

```text
Enabled
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Keelab `regedit.exe` kasutamise | Kasutaja ei saa Registry Editori avada |

---

# 16. Keela Control Panel ja Settings

GPO-s:

```text
User Configuration
→ Policies
→ Administrative Templates
→ Control Panel
→ Prohibit access to Control Panel and PC settings
```

Seadista:

```text
Enabled
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Keelab juhtpaneeli ja Windows Settings avamise | Kasutaja ei saa süsteemiseadeid muuta |

---

# 17. Keela Task Manager

GPO-s:

```text
User Configuration
→ Policies
→ Administrative Templates
→ System
→ Ctrl+Alt+Del Options
→ Remove Task Manager
```

Seadista:

```text
Enabled
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Eemaldab Task Manageri kasutamise võimaluse | Kasutaja ei saa Task Manageri avada |

---

# 18. Keela PowerShell

PowerShelli saab piirata mitmel viisil. Kõige lihtsam eksamil on kasutada **Software Restriction Policies** või **AppLocker**. Kui AppLocker pole ülesandes eraldi nõutud, on SRP lihtsam.

## Variant A: Software Restriction Policies

GPO-s:

```text
User Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Software Restriction Policies
```

Kui näed, et poliitikat pole:

```text
Right-click Software Restriction Policies
→ New Software Restriction Policies
```

Lisa uus Path Rule:

```text
Additional Rules
→ New Path Rule
```

Lisa keelamiseks:

```text
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

Security level:

```text
Disallowed
```

Lisa ka:

```text
C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe
```

Vajadusel PowerShell 7 puhul:

```text
C:\Program Files\PowerShell\7\pwsh.exe
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Keelab PowerShelli käivitamise kasutajatele, kellele GPO rakendub | PowerShell ei avane |

---

## Variant B: AppLocker

Kui kasutad AppLockerit, siis:

```text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Application Control Policies
→ AppLocker
```

AppLocker vajab teenust:

```text
Application Identity
```

Kliendis peab töötama:

```powershell
Get-Service AppIDSvc
```

Käivita vajadusel:

```powershell
Start-Service AppIDSvc
```

Eksami jaoks on Software Restriction Policies tavaliselt lihtsam.

---

# 19. GPO testimine Windows 11 kliendis

## 19.1 Uuenda poliitikad

Windows 11 kliendis domeeni kasutajana:

```cmd
gpupdate /force
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Rakendab GPO-d kohe | Policy update completed successfully |

Kontrolli rakendunud GPO-sid:

```cmd
gpresult /r
```

Oodatav:

```text
Näha on GPO_Tarkvara_Paigaldus, GPO_Chrome_Avaleht, GPO_Taustapilt, GPO_Turva_Logon ja GPO_Kasutaja_Piirangud vastavalt kasutajale/arvutile
```

Täpsem raport:

```cmd
gpresult /h C:\gpresult.html
```

Ava:

```text
C:\gpresult.html
```

---

## 19.2 Testi tarkvara paigaldust

Pärast `gpupdate /force` tee restart:

```cmd
shutdown /r /t 0
```

Pärast sisselogimist kontrolli:

| Tarkvara | Oodatav tulemus |
|---|---|
| LibreOffice | Start menüüs olemas |
| Google Chrome | Start menüüs olemas |
| FastStone | Start menüüs olemas |

Kui tarkvara ei paigaldu:

| Põhjus | Lahendus |
|---|---|
| MSI pole UNC teega lisatud | Kasuta `\\DC1\Software$\fail.msi` |
| Arvuti pole OU-s Arvutid | Tõsta arvuti õigesse OU-sse |
| Restart tegemata | Tarkvara GPO paigaldub tihti boot ajal |
| Domain Computers pole õigustes | Anna share ja NTFS Read õigused |
| EXE installer ei toeta vaikset paigaldust | Kontrolli silent switchi |

---

## 19.3 Testi kasutajapiiranguid

Logi sisse kasutajana, kelle OU-le on `GPO_Kasutaja_Piirangud` lingitud.

Testi:

| Tegevus | Oodatav tulemus |
|---|---|
| Ava `cmd.exe` | Keelatud |
| Ava PowerShell | Keelatud |
| Ava Control Panel / Settings | Keelatud |
| Ava Task Manager | Keelatud |
| Ava `regedit.exe` | Keelatud |

Kui piirang ei tööta:

| Põhjus | Lahendus |
|---|---|
| Kasutaja pole õiges OU-s | Tõsta kasutaja vastavasse OU-sse |
| GPO pole lingitud | Linki GPO õigele OU-le |
| User Configuration ei rakendu | Kontrolli `gpresult /r` |
| Testid administraatoriga | Testi tavalise kasutajaga |

---

# 20. Lõppkontroll

| Kontroll | Käsk / koht | Oodatav tulemus |
|---|---|---|
| Tarkvara share | `Test-Path \\DC1\Software$` | `True` |
| GPO tarkvara | Group Policy Management | `GPO_Tarkvara_Paigaldus` olemas |
| LibreOffice | Windows 11 Start Menu | Paigaldatud |
| Chrome | Windows 11 Start Menu | Paigaldatud |
| FastStone | Windows 11 Start Menu | Paigaldatud |
| Chrome avaleht | Ava Chrome | Avaneb määratud aadress |
| Taustapilt | Kasutaja töölaud | Ettevõtte pilt |
| Viimase kasutaja peitmine | Logon screen | Viimast kasutajat ei kuvata |
| Lokaalne konto | Proovi lokaalselt sisse logida | Ei saa sisse |
| Domeeni konto | Logi domeenikasutajana | Saab sisse |
| CMD | Ava `cmd.exe` | Keelatud |
| PowerShell | Ava PowerShell | Keelatud |
| Control Panel | Ava Control Panel | Keelatud |
| Task Manager | Ava Task Manager | Keelatud |
| Regedit | Ava `regedit.exe` | Keelatud |
| GPO raport | `gpresult /r` | Õiged GPO-d rakenduvad |

---

# 21. Dokumentatsiooni näidis

```markdown
## Windows pilet 4 dokumentatsioon

### Eesmärk

Eesmärk oli paigaldada klientarvutitesse vajalik tarkvara Group Policy abil, seadistada Chrome avaleht, määrata ettevõtte taustapilt ning rakendada kasutajatele ja arvutitele turvapiirangud. Lisaks tuli keelata lokaalsete kontodega sisselogimine ja piirata tavakasutajatel süsteemitööriistade kasutamist.

### Kasutatud serverid ja kliendid

| Seade | Roll |
|---|---|
| DC1 | AD DS, DNS, DHCP, GPO haldus, tarkvara jagamine |
| DC2 | Teine domeenikontroller |
| Windows 11 klient | GPO ja tarkvara paigalduse testimine |

### Tehtud seadistused

- Lõin kausta `C:\Software`.
- Jagasin kausta võrgus nimega `Software$`.
- Kopeerisin sinna LibreOffice, Chrome ja FastStone paigaldusfailid.
- Lõin GPO `GPO_Tarkvara_Paigaldus`.
- Linkisin tarkvara GPO OU-le `Arvutid`.
- Lisasin LibreOffice MSI paketi GPO Software Installation alla.
- Lisasin Chrome Enterprise MSI paketi GPO Software Installation alla.
- Lisasin FastStone paigalduse MSI või startup scripti kaudu.
- Lisasin Chrome ADMX mallid Central Store’i.
- Lõin GPO `GPO_Chrome_Avaleht`.
- Seadistasin Chrome avalehe ja käivituslehe.
- Lõin GPO `GPO_Taustapilt`.
- Määrasin kasutajatele ettevõtte taustapildi.
- Lõin GPO `GPO_Turva_Logon`.
- Peitsin viimati sisse loginud kasutaja nime.
- Keelasin lokaalsete kontodega sisselogimise.
- Lõin GPO `GPO_Kasutaja_Piirangud`.
- Keelasin tavakasutajatele CMD, PowerShelli, Control Paneli, Task Manageri ja Registry Editori.
- Testisin GPO rakendumist Windows 11 kliendis.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `gpresult /r` | Vajalikud GPO-d rakenduvad |
| LibreOffice | Paigaldatud |
| Chrome | Paigaldatud |
| FastStone | Paigaldatud |
| Chrome avaleht | Avaneb määratud URL |
| Taustapilt | Kasutaja töölaual kuvatakse ettevõtte pilt |
| Lokaalne konto | Sisselogimine keelatud |
| Domeeni konto | Sisselogimine lubatud |
| CMD | Keelatud |
| PowerShell | Keelatud |
| Control Panel | Keelatud |
| Task Manager | Keelatud |
| Registry Editor | Keelatud |

### Kokkuvõte

Windows pilet 4 tulemusena paigaldatakse domeeni klientarvutitesse vajalik tarkvara Group Policy abil. Kasutajatele määratakse Chrome avaleht ja ettevõtte taustapilt. Klientarvutites on keelatud kohalike kontodega sisselogimine ning tavakasutajatel on piiratud CMD, PowerShelli, juhtpaneeli, Task Manageri ja Registry Editori kasutamine.
```

---

# 22. Troubleshooting

## Tarkvara ei paigaldu

Kontrolli:

```cmd
gpresult /r
```

Kui GPO puudub:

```text
Kontrolli, kas arvuti on OU-s Arvutid ja GPO on sinna lingitud.
```

Kontrolli võrguteed kliendis:

```cmd
dir \\DC1\Software$
```

Kui ei avane:

```text
Kontrolli share õiguseid ja DNS-i.
```

Oluline:

```text
Software Installation GPO peab kasutama UNC teed:
\\DC1\Software$\fail.msi
Mitte:
C:\Software\fail.msi
```

---

## Chrome poliitikaid ei näe GPO-s

Põhjus:

```text
Chrome ADMX mallid puuduvad.
```

Lahendus:

```text
Lisa chrome.admx ja chrome.adml Central Store’i.
Sulge Group Policy Management Editor ja ava uuesti.
```

---

## Chrome avaleht ei muutu

Kontrolli kliendis:

```cmd
gpupdate /force
gpresult /r
```

Kui GPO on olemas, ava Chrome poliitikad:

```text
chrome://policy
```

Oodatav:

```text
Näed rakendunud Chrome poliitikaid
```

---

## Taustapilt ei rakendu

Kontrolli:

```cmd
dir \\DC1\Software$\taustapilt.jpg
gpresult /r
```

Kui fail pole nähtav, kontrolli õiguseid.

Kui GPO pole rakendunud, kontrolli GPO linki ja kasutaja OU-d.

---

## CMD või PowerShell ei ole keelatud

Kontrolli:

```cmd
gpresult /r
```

Kui `GPO_Kasutaja_Piirangud` puudub:

```text
Kasutaja ei ole õiges OU-s või GPO pole lingitud.
```

PowerShelli puhul kontrolli, kas SRP reegel on õige tee peal:

```text
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe
```

---

## Lokaalse kontoga saab ikka sisse

Kontrolli GPO-s:

```text
Computer Configuration
→ Windows Settings
→ Security Settings
→ Local Policies
→ User Rights Assignment
→ Deny log on locally
```

Veendu, et sinna ei ole lisatud vale gruppi.

Kontrolli kliendis:

```cmd
gpupdate /force
shutdown /r /t 0
```

Kui ikka saab sisse, kontrolli, kas arvuti on OU-s `Arvutid`.

---

# 23. Kõige lühem spikker

```text
1. Loo C:\Software
2. Jaga see \\DC1\Software$
3. Kopeeri installerid ja taustapilt sinna
4. Loo GPO_Tarkvara_Paigaldus
5. Linki see OU-le Arvutid
6. Lisa LibreOffice MSI Software Installation alla
7. Lisa Chrome Enterprise MSI Software Installation alla
8. Lisa FastStone MSI või startup scriptiga
9. Lisa Chrome ADMX mallid Central Store’i
10. Loo GPO_Chrome_Avaleht
11. Määra Chrome avaleht ja startup URL
12. Loo GPO_Taustapilt
13. Määra ettevõtte taustapilt võrguteega
14. Loo GPO_Turva_Logon
15. Peida viimati loginud kasutaja nimi
16. Keela lokaalsete kontodega sisselogimine
17. Loo GPO_Kasutaja_Piirangud
18. Keela CMD
19. Keela PowerShell
20. Keela Control Panel ja Settings
21. Keela Task Manager
22. Keela Registry Editor
23. Tee kliendis gpupdate /force
24. Tee restart
25. Testi tarkvara, avalehte, taustapilti ja piiranguid
26. Dokumenteeri
```
