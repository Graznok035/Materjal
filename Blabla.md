# Windows Pilet 2 – Erinevuste juhend koos troubleshootiga

## Ülevaade

See juhend katab ainult need osad, mis erinevad tavalisest AD/DNS/DHCP piletist.

Eeldus:

- Domeen `sinunimi.local` on loodud
- DC1 ja DC2 töötavad
- DNS töötab
- DHCP ja DHCP Failover töötavad
- Windows 11 klient on domeenis
- OU-d on loodud
- Kasutajad on imporditud

---

# 1. Tühja kõvaketta vormindamine ja ühendamine F: tähisega

## Eesmärk

Serverisse lisatud tühi kõvaketas tuleb vormindada ja ühendada draivitähega `F:`.

---

## GUI kaudu

Ava:

```text
Server Manager
→ Tools
→ Computer Management
→ Disk Management
```

Kui ketas on uus, kuvatakse see tavaliselt kujul:

```text
Unknown
Not Initialized
Unallocated
```

Paremklõps kettal:

```text
Initialize Disk
```

Vali:

```text
GPT
```

Seejärel paremklõps tühjal alal:

```text
New Simple Volume
```

Vali:

```text
Use entire disk
```

Draivitäheks määra:

```text
F:
```

Failisüsteem:

```text
NTFS
```

Volume label:

```text
Andmed
```

---

## Kontroll

PowerShellis:

```powershell
Get-Volume
```

Peab olema näha:

```text
DriveLetter : F
FileSystem  : NTFS
```

---

## Troubleshoot

### Probleem: ketast ei ole Disk Managementis näha

Kontrolli:

```powershell
Get-Disk
```

Kui ketast ei kuvata, kontrolli VM seadetes, kas uus kõvaketas on üldse lisatud.

---

### Probleem: F: täht on juba kasutusel

Kontrolli:

```powershell
Get-Volume
```

Kui `F:` on hõivatud, muuda olemasoleva draivi tähte või kasuta ajutiselt muud tähte. Ülesandes nõutakse siiski F:, seega lõpuks peab uus ketas olema `F:`.

---

### Probleem: ketas on Offline

PowerShell:

```powershell
Get-Disk
Set-Disk -Number 1 -IsOffline $false
```

Vajadusel:

```powershell
Set-Disk -Number 1 -IsReadOnly $false
```

---

# 2. GPO_ParooliKehtivus

## Eesmärk

Kasutajate paroolide maksimaalne eluiga peab olema 30 päeva.

---

## GPO loomine

Ava:

```text
Group Policy Management
```

Domeenil `sinunimi.local` paremklõps:

```text
Create a GPO in this domain, and Link it here
```

Nimi:

```text
GPO_ParooliKehtivus
```

---

## Seadistus

Mine:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Account Policies
                └── Password Policy
```

Muuda:

```text
Maximum password age
```

väärtuseks:

```text
30 days
```

---

## Kontroll

DC1 PowerShellis:

```powershell
Get-ADDefaultDomainPasswordPolicy
```

Kontrolli rida:

```text
MaxPasswordAge : 30.00:00:00
```

---

## Troubleshoot

### Probleem: poliitika ei rakendu

Paroolipoliitika töötab domeenitasemel. See GPO peab olema lingitud domeeni juurele, mitte ainult mõne OU külge.

Õige koht:

```text
sinunimi.local
```

Mitte ainult:

```text
OU=Kasutajad
```

---

### Probleem: kasutajal ei muutu parooli aegumine

Kontrolli kasutaja seadistust ADUC-is:

```text
User properties
→ Account
```

Ei tohi olla märgitud:

```text
Password never expires
```

---

# 3. GPO_KontodeLukustamine

## Eesmärk

Kui parool sisestatakse 5 korda valesti, lukustatakse konto 15 minutiks.

---

## GPO loomine

Domeeni juurele loo GPO:

```text
GPO_KontodeLukustamine
```

---

## Seadistus

Mine:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Account Policies
                └── Account Lockout Policy
```

Määra:

```text
Account lockout threshold: 5 invalid logon attempts
Account lockout duration: 15 minutes
Reset account lockout counter after: 15 minutes
```

---

## Kontroll

Windows 11 kliendis proovi domeenikasutajaga 5 korda vale parooli sisestada.

Konto peab lukustuma.

Kontroll DC1-s:

```powershell
Search-ADAccount -LockedOut
```

---

## Troubleshoot

### Probleem: konto ei lukustu

Kontrolli, et GPO oleks lingitud domeeni juurele:

```text
sinunimi.local
```

Seejärel kliendis:

```cmd
gpupdate /force
```

---

### Probleem: kasutaja jääb lukku ja sisse ei saa

Ava ADUC:

```text
User properties
→ Account
→ Unlock account
```

Või PowerShell:

```powershell
Unlock-ADAccount -Identity kasutajanimi
```

---

# 4. Edge_Siseportaal

## Eesmärk

OU `Personal` kasutajatel peab Microsoft Edge:

- avama siseportaali avalehena
- avama siseportaali uue vahekaardi lehena
- keelama kasutajal avalehe muutmise

Aadress:

```text
https://siseportaal.sinunimi.local
```

---

## Edge ADMX mallide lisamine

Laadi alla Microsoft Edge Policy Templates.
otsene allalaadimise link: https://msedge.sf.dl.delivery.mp.microsoft.com/filestreamingservice/files/67fbb399-543d-49d5-b2ad-69f11af0d5e5/MicrosoftEdgePolicyTemplates.cab
või leheküljelt: https://www.microsoft.com/et-ee/edge/business/download?form=MA13FJ

Kopeeri ADMX fail:

~\MicrosoftEdgePolicyTemplates\windows\admx\msedge.admx
lehekülje allotsas

```text
msedge.admx
```

kausta:

```text
C:\Windows\PolicyDefinitions
```

Kopeeri ADML fail, näiteks inglise keele puhul:

~\MicrosoftEdgePolicyTemplates\windows\adm\en-US\msedge.adml

```text
msedge.adml
```

kausta:

```text
C:\Windows\PolicyDefinitions\en-US
```

---

## GPO loomine

Loo GPO:

```text
Edge_Siseportaal
```

Lingi see OU-le:

```text
Personal
```

---

## Seadistus

Mine:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── Microsoft Edge
            └── Startup, home page and new tab page
```

Seadista:

```text
Configure the home page URL: Enabled
Home page URL: https://siseportaal.sinunimi.local
```

```text
Action to take on startup: Enabled
Open a list of URLs
```

```text
Sites to open when the browser starts:
https://siseportaal.sinunimi.local
```

```text
Configure the new tab page URL:
https://siseportaal.sinunimi.local
```

Kui olemas:

```text
Configure whether users can set homepage: Disabled / või lukustatud seadistus vastavalt mallile
```

Martin: ei leidnud. võib olla kuskil sügavamal...

---

## Kontroll

Personal OU kasutajaga Windows 11 kliendis:

```cmd
gpupdate /force
```

Logi välja ja sisse.

Ava Edge.

Peab avanema:

```text
https://siseportaal.sinunimi.local
```

---

## Troubleshoot

### Probleem: Microsoft Edge seadeid ei ole GPO-s näha

ADMX mallid pole õigesti lisatud.

Kontrolli, et failid oleksid siin:

```text
C:\Windows\PolicyDefinitions\msedge.admx
C:\Windows\PolicyDefinitions\en-US\msedge.adml
```

Kui kasutad Central Store’i, siis kopeeri need hoopis sinna:

```text
\\sinunimi.local\SYSVOL\sinunimi.local\Policies\PolicyDefinitions
```

---

### Probleem: GPO ei rakendu Personal kasutajale

Kontrolli:

- kasutaja asub OU-s `Personal`
- GPO on lingitud OU-le `Personal`
- GPO on User Configuration poliitika
- kasutaja tegi välja/sisse logimise

Kliendis:

```cmd
gpresult /r
```

---

### Probleem: siseportaal ei avane

Kontrolli DNS:

```cmd
nslookup siseportaal.sinunimi.local
```

Kui kirjet ei leita, lisa DNS-i A-kirje.

---

# 5. GPO_TurvalineSisselogimine

## Eesmärk

Klientarvutitel peab olema turvalisem sisselogimine:

- viimase kasutaja nime ei kuvata
- Guest konto on keelatud
- lokaalsete tavakasutajate sisselogimine on keelatud

---

## GPO loomine

Loo GPO:

```text
GPO_TurvalineSisselogimine
```

Soovituslik link:

```text
OU=Arvutid
```

---

## Viimase kasutaja peitmine

Mine:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Local Policies
                └── Security Options
```

Muuda:

```text
Interactive logon: Do not display last signed-in
```

väärtuseks:

```text
Enabled
```

Vajadusel ka:

```text
Interactive logon: Do not display username at sign-in
```

väärtuseks:

```text
Enabled
```

---

## Guest konto keelamine

Samas kohas:

```text
Accounts: Guest account status
```

väärtuseks:

```text
Disabled
```

---

## Lokaalsete tavakasutajate sisselogimise keelamine

Mine:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Local Policies
                └── User Rights Assignment
```

Muuda:

```text
Deny log on locally
```

Lisa näiteks:

```text
Guests
Local account
```

või vastavalt eksami olukorrale lokaalne kasutajagrupp.

Oluline: ära lisa siia `Domain Users`, muidu ei saa domeenikasutajad sisse logida.

---

## Kontroll

Kliendis:

```cmd
gpupdate /force
```

Pärast restarti ei tohiks viimast kasutajat kuvada.

---

## Troubleshoot

### Probleem: domeenikasutaja ei saa enam sisse logida

Tõenäoliselt lisasid keelatud gruppi vale grupi.

Kontrolli:

```text
Deny log on locally
```

Ära keela:

```text
Domain Users
Authenticated Users
Administrators
```

---

### Probleem: viimane kasutaja kuvatakse ikka

Tee kliendis:

```cmd
gpupdate /force
shutdown /r /t 0
```

Kontrolli:

```cmd
gpresult /r
```

Kas GPO rakendus arvutile?

---

# 6. GPO_KeelaUSB

## Eesmärk

OU `Arvutid` masinatel keelatakse väliste USB-andmekandjate lugemis- ja kirjutamisõigused.

---

## GPO loomine

Loo GPO:

```text
GPO_KeelaUSB
```

Lingi see OU-le:

```text
Arvutid
```

---

## Seadistus

Mine:

```text
Computer Configuration
└── Policies
    └── Administrative Templates
        └── System
            └── Removable Storage Access
```

Lülita sisse:

```text
Removable Disks: Deny read access
Removable Disks: Deny write access
```

Või tugevam variant:

```text
All Removable Storage classes: Deny all access
```

väärtuseks:

```text
Enabled
```

---

## Kontroll

Kliendis:

```cmd
gpupdate /force
```

Restart.

Sisesta USB mälupulk.

Tulemus peaks olema:

```text
Access denied
```

---

## Troubleshoot

### Probleem: USB töötab ikka

Kontrolli, kas arvuti objekt on OU-s `Arvutid`.

ADUC:

```text
sinunimi.local
└── Arvutid
    └── WIN11
```

Kliendis:

```cmd
gpresult /r
```

GPO peab olema arvuti poliitikate all.

---

### Probleem: GPO on kasutaja all, aga ei mõju

USB keeld peab olema:

```text
Computer Configuration
```

mitte ainult:

```text
User Configuration
```

---

# 7. GPO_TöölauaTaustapilt

## Eesmärk

Kõigile kasutajatele määratakse kohustuslik ettevõtte taustapilt.

Nõue: pilt peab töötama ka ilma AD võrguühenduseta.

Selleks kopeerime pildi kliendi lokaalsele kettale ja määrame taustapildi sealt.

---

## Jagatud kausta loomine

DC1-s loo kaust:

```text
C:\Wallpaper
```

Pane sinna fail:

```text
logo.jpg
```

Jaga kaust nimega:

```text
Wallpaper
```

Jagatud tee:

```text
\\DC1\Wallpaper\logo.jpg
```

Õigused:

```text
Domain Users: Read
```

---

## GPO loomine

Loo GPO:

```text
GPO_TöölauaTaustapilt
```

Lingi see kasutajate OU-dele või domeenile, sõltuvalt ülesandest.

---

## Faili kopeerimine kliendile

Mine:

```text
User Configuration
└── Preferences
    └── Windows Settings
        └── Files
```

Paremklõps:

```text
New → File
```

Seadista:

```text
Action: Replace
Source file: \\DC1\Wallpaper\logo.jpg
Destination file: C:\Windows\Web\Wallpaper\logo.jpg
```

---

## Taustapildi määramine

Mine:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── Desktop
            └── Desktop
```

Muuda:

```text
Desktop Wallpaper
```

väärtuseks:

```text
Enabled
```

Wallpaper name:

```text
C:\Windows\Web\Wallpaper\logo.jpg
```

Wallpaper style:

```text
Fill
```

---

## Kontroll

Kliendis:

```cmd
gpupdate /force
```

Logi välja ja sisse.

Kontrolli, kas fail on olemas:

```cmd
dir C:\Windows\Web\Wallpaper\
```

---

## Troubleshoot

### Probleem: taustapilt ei ilmu

Kontrolli, kas fail kopeeriti:

```cmd
dir C:\Windows\Web\Wallpaper\logo.jpg
```

Kui faili ei ole, kontrolli jagatud kausta õiguseid.

---

### Probleem: võrgutaustapilt kaob ilma domeenita

Taustapilt ei tohi viidata ainult võrguteele:

```text
\\DC1\Wallpaper\logo.jpg
```

Õige lõplik tee peab olema lokaalne:

```text
C:\Windows\Web\Wallpaper\logo.jpg
```

---

### Probleem: kasutaja saab taustapilti muuta

Lisa poliitika:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── Control Panel
            └── Personalization
```

Muuda:

```text
Prevent changing desktop background
```

väärtuseks:

```text
Enabled
```

---

# 8. GPO_KasutajaPiirangud

## Eesmärk

Tavakasutajatel keelatakse:

- CMD
- PowerShell
- Juhtpaneel
- Tegumihaldur
- Regedit

GPO lingitakse OU-dele:

```text
Personal
Myyk
Juhtkond
Haldus
Toimetajad
```

---

## GPO loomine

Loo GPO:

```text
GPO_KasutajaPiirangud
```

Lingi see OU-dele:

```text
Personal
Myyk
Juhtkond
Haldus
Toimetajad
```

---

## CMD keelamine

Mine:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── System
```

Muuda:

```text
Prevent access to the command prompt
```

väärtuseks:

```text
Enabled
```

Soovi korral:

```text
Disable the command prompt script processing also?
Yes
```

---

## Regedit keelamine

Samas kohas:

```text
Prevent access to registry editing tools
```

väärtuseks:

```text
Enabled
```

---

## Juhtpaneeli keelamine

Mine:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── Control Panel
```

Muuda:

```text
Prohibit access to Control Panel and PC settings
```

väärtuseks:

```text
Enabled
```

---

## Tegumihalduri keelamine

Mine:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── System
            └── Ctrl+Alt+Del Options
```

Muuda:

```text
Remove Task Manager
```

väärtuseks:

```text
Enabled
```

---

## PowerShell keelamine Software Restriction Policy abil

Mine:

```text
User Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Software Restriction Policies
```

Kui tühi:

```text
New Software Restriction Policies
```

Seejärel:

```text
Additional Rules
→ New Path Rule
```

Lisa:

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

Kui PowerShell 7 on olemas:

```text
C:\Program Files\PowerShell\7\pwsh.exe
```

---

## Kontroll

Tavakasutajaga proovi avada:

```text
cmd
powershell
control
taskmgr
regedit
```

Need ei tohi avaneda.

---

## Troubleshoot

### Probleem: CMD on keelatud, aga PowerShell avaneb

CMD ja PowerShell on eraldi programmid. CMD keeld ei keela PowerShelli.

Lisa Software Restriction Policy või AppLocker reegel.

---

### Probleem: administraatoril ka keelatakse tööriistad

Kui GPO on lingitud kasutaja OU-le, mõjutab see kõiki selles OU-s olevaid kasutajaid.

Lahendus:

- ära pane administraatoreid nendesse OU-desse
- või kasuta Security Filteringut
- või lisa administraatoritele Deny Apply Group Policy

---

### Probleem: GPO ei rakendu ühele osakonnale

Kontrolli, kas GPO on lingitud kõigile nõutud OU-dele:

```text
Personal
Myyk
Juhtkond
Haldus
Toimetajad
```

---

# 9. GPO_TurvapoliitikaTeade

## Eesmärk

OU `Personal` kasutajatele kuvatakse enne sisselogimist interaktiivne turvapoliitika teade.

---

## GPO loomine

Loo GPO:

```text
GPO_TurvapoliitikaTeade
```

Lingi see OU-le:

```text
Personal
```

---

## Seadistus

Mine:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Local Policies
                └── Security Options
```

Muuda:

```text
Interactive logon: Message title for users attempting to log on
```

Näide:

```text
AS Oige Turvapoliitika
```

Muuda:

```text
Interactive logon: Message text for users attempting to log on
```

Näide:

```text
Käesolev arvuti on AS Oige omand.

Arvuti kasutamine on lubatud ainult volitatud töötajatele.

Kõik tegevused võivad olla logitud.

Süsteemi kuritarvitamine võib kaasa tuua distsiplinaar- või õiguslikud meetmed.

Jätkates kinnitad, et nõustud ettevõtte turvapoliitika ja arvuti kasutamise eeskirjadega.
```

---

## Kontroll

Kliendis:

```cmd
gpupdate /force
shutdown /r /t 0
```

Enne sisselogimist peab ilmuma turvateade.

---

## Troubleshoot

### Probleem: teadet ei kuvata

See on Computer Configuration poliitika.

Kontrolli:

- kas arvuti objekt asub OU-s, millele GPO rakendub
- kui GPO on lingitud `Personal` OU-le, aga seal on ainult kasutajad, ei pruugi Computer Configuration rakenduda

Praktiline lahendus:

- lingi GPO OU-le `Arvutid`
- või domeeni tasemele
- või kasuta loopback processingut, kui ülesanne nõuab just Personal kasutajatele

---

### Probleem: Personal OU on kasutajate OU, aga turvateade on arvutipoliitika

Turvateade on arvutipõhine seadistus. Kui see peab kehtima Personal kasutajatele, on kaks võimalust:

Variant 1: linkida GPO arvutite OU-le, kus Personal kasutajad sisse logivad.

Variant 2: kasutada loopback processingut.

Loopback asub:

```text
Computer Configuration
└── Policies
    └── Administrative Templates
        └── System
            └── Group Policy
```

Muuda:

```text
Configure user Group Policy loopback processing mode
```

väärtuseks:

```text
Enabled
Mode: Merge
```

---

# 10. Üldine GPO rakendamise kontroll

## Kliendis poliitikate uuendamine

```cmd
gpupdate /force
```

---

## Arvuti restart

```cmd
shutdown /r /t 0
```

---

## Kasutaja GPO-de kontroll

```cmd
gpresult /r
```

---

## Detailne HTML raport

```cmd
gpresult /h C:\gpresult.html
```

Ava fail:

```text
C:\gpresult.html
```

---

## Kontrolli domeenikontrollerist

PowerShell:

```powershell
Get-GPO -All
```

---

# 11. Levinud üldvead

## Viga: GPO on olemas, aga ei rakendu

Kontrolli:

- kas GPO on lingitud õige OU külge
- kas kasutaja või arvuti asub selles OU-s
- kas seadistus on User Configuration või Computer Configuration all
- kas klient sai uued poliitikad kätte
- kas DNS töötab
- kas klient on domeenis

Kliendis:

```cmd
whoami
gpupdate /force
gpresult /r
```

---

## Viga: User Configuration ei rakendu

Kontrolli, kas kasutaja asub õiges OU-s.

Näiteks `Edge_Siseportaal` peab rakenduma kasutajale, mitte arvutile.

---

## Viga: Computer Configuration ei rakendu

Kontrolli, kas arvuti asub õiges OU-s.

Näiteks `GPO_KeelaUSB` peab rakenduma arvutile, mitte kasutajale.

---

## Viga: DNS ei tööta

Kliendis:

```cmd
ipconfig /all
```

DNS serverid peavad olema:

```text
DC1 IP
DC2 IP
```

Kontroll:

```cmd
nslookup sinunimi.local
nslookup dc1.sinunimi.local
```

---

## Viga: klient ei saa domeeni GPO-sid

Kontrolli ühendust:

```cmd
ping dc1.sinunimi.local
ping sinunimi.local
```

Kontrolli aega:

```cmd
w32tm /query /status
```

Kui aeg on väga vale, võib domeeniga suhtlus ebaõnnestuda.

---

# 12. Lõppkontroll

- [ ] F: ketas on loodud
- [ ] F: ketas on NTFS failisüsteemiga
- [ ] `GPO_ParooliKehtivus` määrab parooli elueaks 30 päeva
- [ ] `GPO_KontodeLukustamine` lukustab konto 5 vale parooli järel
- [ ] Lukustus kestab 15 minutit
- [ ] `Edge_Siseportaal` rakendub Personal kasutajatele
- [ ] Edge avaleht on `https://siseportaal.sinunimi.local`
- [ ] Edge uue vahekaardi leht on siseportaal
- [ ] Kasutaja ei saa Edge avalehte muuta
- [ ] `GPO_TurvalineSisselogimine` peidab viimase kasutaja nime
- [ ] Guest konto on keelatud
- [ ] Lokaalsete tavakasutajate sisselogimine on piiratud
- [ ] `GPO_KeelaUSB` keelab USB lugemise ja kirjutamise
- [ ] `GPO_TöölauaTaustapilt` määrab taustapildi
- [ ] Taustapilt töötab ka ilma AD ühenduseta
- [ ] `GPO_KasutajaPiirangud` keelab cmd
- [ ] `GPO_KasutajaPiirangud` keelab PowerShelli
- [ ] `GPO_KasutajaPiirangud` keelab Juhtpaneeli
- [ ] `GPO_KasutajaPiirangud` keelab Tegumihalduri
- [ ] `GPO_KasutajaPiirangud` keelab regedit.exe
- [ ] `GPO_TurvapoliitikaTeade` kuvab sisselogimisel teate
- [ ] `gpupdate /force` töötab
- [ ] `gpresult /r` näitab vajalikke GPO-sid
