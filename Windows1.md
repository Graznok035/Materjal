# Windows pilet 1 lahenduskäik

See juhend kirjeldab ainult **Windows pilet 1 eriosa**.

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

## 1. Windows pilet 1 eesmärk

Windows pilet 1 põhiteemad on:

| Teema | Mida tuleb teha | Milleks seda vaja on |
|---|---|---|
| IIS veebiserver | Paigaldada Windows Serverisse IIS roll | Et server saaks majutada veebirakendust |
| CMS / sisuhaldussüsteem | Paigaldada IIS peale vabalt valitud sisuhaldussüsteem | Näiteks WordPress või Drupal |
| AD autentimine | Veebilehe sisuhaldajate autentimine peab olema seotud Active Directoryga | Sisuhaldajad tulevad AD kasutajatest/gruppidest |
| AD grupp | Luua sisuhaldajate jaoks turvagrupp | Et õiguseid hallata grupipõhiselt |
| Edge avalehe GPO | Määrata OU `Personal` kasutajatele Edge avaleht ja uue vahekaardi leht siseportaali aadressile | Kasutajad ei pea aadressi käsitsi sisestama |
| AD CS | Paigaldada domeenikontrollerile Certification Authority roll | Sertifikaatide väljastamiseks |
| Enterprise Root CA | Seadistada sisemine Enterprise Root CA | Domeenimasinad usaldavad CA-d automaatselt |
| SSL sertifikaat | Väljastada sertifikaat CMS veebilehele | HTTPS jaoks |
| DNS CNAME kirjed | Luua DNS nimed veebilehele | Sama veebileht peab avanema mitme nimega |
| Testimine | Kontrollida DNS, HTTP/HTTPS, CMS ja AD autentimine | Tõestab, et lahendus töötab |

---

# 2. Vajalikud rollid ja tarkvara

## 2.1 Windows Serveri rollid

| Roll / funktsioon | Kus paigaldada | Milleks vajalik |
|---|---|---|
| Web Server IIS | Veebiserveris või DC1-s, kui eraldi veebiserverit pole | CMS-i majutamiseks |
| CGI / PHP tugi | IIS-is | Kui CMS on WordPress/Drupal PHP peal |
| IIS Management Console | Veebiserveris | IIS haldamiseks GUI kaudu |
| Active Directory Certificate Services | Domeenikontrolleris | Sertifikaatide väljastamiseks |
| Certification Authority | AD CS rolliteenus | Enterprise Root CA loomiseks |
| Group Policy Management | DC1-s | Edge avalehe GPO seadistamiseks |
| DNS Manager | DC1-s | CNAME ja A-kirjete loomiseks |

---

## 2.2 Väline tarkvara

Kui valid WordPressi, on vaja:

| Tarkvara | Milleks |
|---|---|
| PHP | WordPressi käivitamiseks |
| MySQL / MariaDB või muu toetatud andmebaas | WordPressi andmebaasiks |
| WordPress | Sisuhaldussüsteem |
| URL Rewrite või sobiv IIS lisamoodul | WordPressi ilusamate URL-ide jaoks, kui vaja |

Lihtsaim eksamiloogika:

```text
IIS + PHP + WordPress + andmebaas
```

Kui WordPressi paigaldus muutub liiga ajamahukaks, võib dokumenteerida, mida jõudsid teha:

```text
IIS roll paigaldatud
PHP tugi seadistatud
CMS failid paigaldatud
Andmebaasi seadistus pooleli / kontrollitud
```

---

# 3. Soovitatav nimelahendus

Näide domeeniga:

```text
sinuNimi.local
```

Siseportaali põhinimi:

```text
siseportaal.sinuNimi.local
```

Lisanimed CNAME kirjetena:

```text
www.sinuNimi.local
portaal.sinuNimi.local
```

Soovitatav DNS loogika:

| DNS nimi | Kirje tüüp | Osutab kuhu |
|---|---|---|
| `siseportaal.sinuNimi.local` | A | IIS serveri IP |
| `www.sinuNimi.local` | CNAME | `siseportaal.sinuNimi.local` |
| `portaal.sinuNimi.local` | CNAME | `siseportaal.sinuNimi.local` |

Oluline:

```text
Kui veebileht peab avanema mitme DNS nimega HTTPS kaudu,
peab sertifikaadil olema need nimed olemas CN või SAN väljal.
```

---

# 4. IIS rolli paigaldamine

## 4.1 Paigalda IIS Server Manageriga

Ava serveris:

```text
Server Manager
→ Manage
→ Add Roles and Features
→ Role-based or feature-based installation
→ Select server
→ Web Server (IIS)
→ Add Features
```

Vali kindlasti:

```text
Web Server
Management Tools
IIS Management Console
```

Kui kasutad PHP-põhist CMS-i, vali vajadusel ka:

```text
Application Development
CGI
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab IIS veebiserveri | IIS Manager on Server Manager → Tools all nähtav |

PowerShelli alternatiiv:

```powershell
Install-WindowsFeature Web-Server,Web-Mgmt-Console,Web-CGI -IncludeManagementTools
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab IIS-i, halduskonsooli ja CGI toe | Käsk lõpeb `Success: True` tulemusega |

---

## 4.2 Kontrolli IIS teenust

```powershell
Get-Service W3SVC
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Teenus `W3SVC` on `Running` | Kui teenus ei tööta, käivita `Start-Service W3SVC` |

Käivita vajadusel:

```powershell
Start-Service W3SVC
```

Testi brauseris serverist:

```text
http://localhost
```

Oodatav tulemus:

```text
Avaneb IIS vaikimisi leht
```

---

# 5. CMS-i jaoks veebikausta loomine

## 5.1 Loo veebikaust

```powershell
New-Item -ItemType Directory -Path "C:\inetpub\siseportaal" -Force
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob CMS-i failide jaoks kausta | Kaust `C:\inetpub\siseportaal` on olemas |

Kontroll:

```powershell
Test-Path "C:\inetpub\siseportaal"
```

Oodatav:

```text
True
```

---

## 5.2 Ajutine testleht enne CMS-i

Enne CMS-i paigaldust on mõistlik teha lihtne testleht.

```powershell
@"
<!doctype html>
<html lang="et">
<head>
    <meta charset="utf-8">
    <title>AS Õige siseportaal</title>
</head>
<body>
    <h1>AS Õige siseportaal</h1>
    <p>IIS veebiserver töötab.</p>
</body>
</html>
"@ | Out-File -FilePath "C:\inetpub\siseportaal\index.html" -Encoding utf8
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob ajutise avalehe | Veebiserveri toimimist saab enne CMS-i testida |

---

# 6. IIS veebilehe loomine

## 6.1 Ava IIS Manager

```text
Server Manager
→ Tools
→ Internet Information Services (IIS) Manager
```

Liigu:

```text
Serveri nimi
→ Sites
→ Add Website
```

Täida:

| Väli | Väärtus |
|---|---|
| Site name | `Siseportaal` |
| Physical path | `C:\inetpub\siseportaal` |
| Binding type | `http` |
| IP address | serveri IP või `All Unassigned` |
| Port | `80` |
| Host name | `siseportaal.sinuNimi.local` |

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob IIS-is veebisaidi | Sait `Siseportaal` on IIS Sites all nähtav |

---

## 6.2 Kontrolli HTTP bindingut

PowerShellis:

```powershell
Get-WebBinding -Name "Siseportaal"
```

Oodatav:

```text
Näha on http binding port 80 ja host name siseportaal.sinuNimi.local
```

---

# 7. DNS kirjete loomine

## 7.1 Loo põhiline A-kirje

DC1-s ava:

```text
Server Manager
→ Tools
→ DNS
→ Forward Lookup Zones
→ sinuNimi.local
```

Paremklõps:

```text
New Host (A or AAAA)
```

Täida:

| Väli | Väärtus |
|---|---|
| Name | `siseportaal` |
| IP address | IIS serveri IP |

Tulemus:

```text
siseportaal.sinuNimi.local → IIS serveri IP
```

---

## 7.2 Loo CNAME kirjed

DNS Manageris:

```text
Forward Lookup Zones
→ sinuNimi.local
→ New Alias (CNAME)
```

Näited:

| Alias | Fully qualified domain name target |
|---|---|
| `www` | `siseportaal.sinuNimi.local` |
| `portaal` | `siseportaal.sinuNimi.local` |

Tulemus:

```text
www.sinuNimi.local → siseportaal.sinuNimi.local
portaal.sinuNimi.local → siseportaal.sinuNimi.local
```

---

## 7.3 Kontrolli DNS-i

```powershell
nslookup siseportaal.sinuNimi.local
```

Oodatav:

```text
Nimi lahendub IIS serveri IP-ks
```

```powershell
nslookup www.sinuNimi.local
```

Oodatav:

```text
CNAME viitab siseportaal.sinuNimi.local nimele
```

```powershell
nslookup portaal.sinuNimi.local
```

Oodatav:

```text
CNAME viitab siseportaal.sinuNimi.local nimele
```

Kui nimi ei lahendu:

```cmd
ipconfig /flushdns
```

ja kontrolli, et klient kasutab DNS serverina DC1/DC2 IP-aadresse.

---

# 8. Edge avalehe GPO Personal OU kasutajatele

## 8.1 Eeldus

Üldosas on olemas OU:

```text
Kasutajad
```

Pilet 1 puhul peab olema kasutajate OU struktuuris vähemalt OU:

```text
Personal
```

Näide:

```text
Kasutajad
└── Personal
```

Kui OU puudub, loo see ADUC-is:

```text
Active Directory Users and Computers
→ Kasutajad
→ New
→ Organizational Unit
→ Personal
```

---

## 8.2 Loo GPO Edge avalehe jaoks

Ava:

```text
Server Manager
→ Tools
→ Group Policy Management
```

Loo uus GPO:

```text
GPO_Edge_Siseportaal
```

Lingi see OU-le:

```text
OU=Personal
```

---

## 8.3 Seadista Edge homepage ja New Tab Page

GPO-s:

```text
User Configuration
→ Policies
→ Administrative Templates
→ Microsoft Edge
```

Seadista näiteks:

| Seade | Väärtus |
|---|---|
| Configure the home page URL | `https://siseportaal.sinuNimi.local` |
| Configure the new tab page URL | `https://siseportaal.sinuNimi.local` |
| Show Home button on toolbar | Enabled |
| Action to take on startup | Open a list of URLs |
| Sites to open when the browser starts | `https://siseportaal.sinuNimi.local` |

Kui Microsoft Edge Administrative Templates puuduvad, võib kasutada ADMX malle või dokumenteerida, et GPO loodi, kuid Edge mallid tuleb lisada.

| Mida see teeb | Oodatav tulemus |
|---|---|
| Määrab Personal OU kasutajatele Edge avalehe siseportaali peale | Personal kasutajatel avaneb Edge’is siseportaal |

---

## 8.4 Kontroll kliendis

Windows 11 kliendis logi sisse Personal OU kasutajana.

Käivita:

```cmd
gpupdate /force
```

Kontrolli:

```cmd
gpresult /r
```

Oodatav:

```text
GPO_Edge_Siseportaal on rakendunud kasutajale
```

Ava Microsoft Edge.

Oodatav:

```text
Avaleht või käivitusleht on siseportaal.sinuNimi.local
```

---

# 9. AD grupp sisuhaldajatele

## 9.1 Loo sisuhaldajate grupp

ADUC-is loo turvagrupp:

```text
Toimetajad
```

Soovitatav asukoht:

```text
OU=Personal või OU=Kasutajad
```

Grupi tüüp:

| Väli | Väärtus |
|---|---|
| Group scope | Global |
| Group type | Security |

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob grupi sisuhaldajate haldamiseks | AD-s on grupp `Toimetajad` |

---

## 9.2 Lisa sisuhaldajad gruppi

ADUC-is:

```text
Toimetajad
→ Members
→ Add
```

Lisa kasutajad, kes peavad CMS-i haldama.

PowerShelli alternatiiv:

```powershell
Add-ADGroupMember -Identity "Toimetajad" -Members "kasutaja1"
```

Kontroll:

```powershell
Get-ADGroupMember Toimetajad
```

Oodatav:

```text
Näha on sisuhaldajad
```

---

# 10. CMS-i AD autentimise loogika

## 10.1 Oluline põhimõte

Piletis on nõutud, et:

```text
veebilehe sisuhaldajate autentimine peab olema seotud Active Directoryga
```

See tähendab, et CMS-i halduskasutajad ei tohiks olla lihtsalt täiesti eraldi lokaalsed kontod, vaid autentimine peab olema seotud AD kasutajate või AD grupiga.

Võimalikud lahendused:

| Lahendus | Selgitus |
|---|---|
| CMS LDAP / Active Directory plugin | Parim variant WordPressi/Drupaliga |
| IIS Windows Authentication admin kaustale | Lihtsam variant, kui CMS plugina seadistus ei õnnestu |
| AD grupi põhine ligipääs | Kasutatakse gruppi `Toimetajad` |

---

## 10.2 WordPressi puhul

Kui valid WordPressi, otsi ja paigalda LDAP/AD autentimise plugin.

Näiteks loogika:

```text
WordPress admin login → AD/LDAP plugin → domeeni kasutaja → Toimetajad grupp
```

Seadistuses on tavaliselt vaja:

| Väli | Väärtus |
|---|---|
| Domain Controller / LDAP server | DC1 IP või FQDN |
| Base DN | `DC=sinuNimi,DC=local` |
| Bind user | domeeni kasutaja, kellel on õigus AD-st lugeda |
| User group | `Toimetajad` |
| Login attribute | `sAMAccountName` |

Dokumentatsiooni kirjuta:

```text
CMS sisuhaldajate ligipääs seoti AD kasutajatega. Sisuhaldajate õiguseid hallatakse AD turvagrupi Toimetajad kaudu.
```

---

## 10.3 Kui LDAP plugin ei õnnestu

Kui CMS-i pluginaga läheb liiga kaua, tee vähemalt IIS tasemel kaitse CMS admin osale.

Näide WordPressi admin kaust:

```text
/wp-admin
```

IIS-is saab piirata ligipääsu:

```text
Authentication
Anonymous Authentication = Disabled
Windows Authentication = Enabled
```

ja NTFS õigustes anda lugemisõigus ainult grupile:

```text
sinuNimi\Toimetajad
```

Dokumenteeri ausalt:

```text
CMS-i täielik LDAP login jäi pooleli, kuid sisuhaldusosa ligipääs piirati IIS Windows Authentication ja AD grupi Toimetajad abil.
```

---

# 11. AD CS paigaldamine

## 11.1 Paigalda AD CS roll

DC1-s:

```text
Server Manager
→ Manage
→ Add Roles and Features
→ Active Directory Certificate Services
→ Certification Authority
→ Install
```

PowerShelli alternatiiv:

```powershell
Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Paigaldab sertifikaaditeenuse | AD CS roll on olemas |

---

## 11.2 Seadista Enterprise Root CA

Server Manageris vali:

```text
Configure Active Directory Certificate Services
```

Vali:

```text
Certification Authority
```

Seadistused:

| Samm | Valik |
|---|---|
| Setup type | Enterprise CA |
| CA type | Root CA |
| Private key | Create a new private key |
| Cryptography | Vaikeseaded sobivad |
| CA name | `sinuNimi-RootCA` |
| Validity | Vaikeseaded sobivad |
| Database locations | Vaikeseaded sobivad |

Oluline:

```text
Piletis on rõhk Certification Authority rollil.
Web Enrollment ei ole vajalik, kui seda pole eraldi nõutud.
```

Kontroll:

```powershell
Get-Service CertSvc
```

Oodatav:

```text
Running
```

---

# 12. Sertifikaadi loomine veebilehele

## 12.1 Sertifikaadi nimed

Kui veebileht avaneb mitme DNS nimega, peaks sertifikaat katma need nimed.

Näide:

```text
siseportaal.sinuNimi.local
www.sinuNimi.local
portaal.sinuNimi.local
```

Soovitatav:

```text
CN = siseportaal.sinuNimi.local
SAN = siseportaal.sinuNimi.local, www.sinuNimi.local, portaal.sinuNimi.local
```

---

## 12.2 Lihtsam variant: Create Domain Certificate IIS-is

IIS Manageris:

```text
Serveri nimi
→ Server Certificates
→ Create Domain Certificate
```

Täida:

| Väli | Näide |
|---|---|
| Common name | `siseportaal.sinuNimi.local` |
| Organization | `AS Õige` |
| Organizational unit | `IT` |
| City/locality | `Haapsalu` |
| State/province | `Läänemaa` |
| Country/region | `EE` |

CA:

```text
sinuNimi-RootCA
```

Friendly name:

```text
siseportaal.sinuNimi.local
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Taotleb sertifikaadi domeeni CA-lt | Sertifikaat ilmub IIS Server Certificates nimekirja |

Märkus:

```text
Kui on vaja SAN nimesid, võib Create Domain Certificate olla piiratud.
Sellisel juhul loo sertifikaaditaotlus või kasuta certreq/PowerShell meetodit.
```

---

## 12.3 Varuvariant: self-signed sertifikaat

Kui AD CS sertifikaadiga jääd kinni, loo ajutiselt self-signed sertifikaat:

```text
IIS Manager
→ Server Certificates
→ Create Self-Signed Certificate
```

Friendly name:

```text
siseportaal.sinuNimi.local
```

Dokumentatsiooni kirjuta:

```text
AD CS roll paigaldati, kuid veebiserveri sertifikaadi väljastamine jäi pooleli. HTTPS testimiseks kasutati ajutiselt self-signed sertifikaati.
```

---

# 13. HTTPS binding IIS-is

## 13.1 Lisa HTTPS binding

IIS Manageris:

```text
Sites
→ Siseportaal
→ Bindings
→ Add
```

Täida:

| Väli | Väärtus |
|---|---|
| Type | `https` |
| IP address | serveri IP või All Unassigned |
| Port | `443` |
| Host name | `siseportaal.sinuNimi.local` |
| SSL certificate | vali loodud sertifikaat |

Lisa vajadusel ka teistele nimedele HTTPS bindingud:

```text
www.sinuNimi.local
portaal.sinuNimi.local
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Seob veebilehe HTTPS-iga | Leht avaneb `https://` aadressiga |

---

## 13.2 Kontrolli bindinguid

```powershell
Get-WebBinding -Name "Siseportaal"
```

Oodatav:

```text
Olemas on HTTP 80 ja HTTPS 443 bindingud
```

---

# 14. Windows Firewall

## 14.1 Luba HTTP ja HTTPS

PowerShell administraatorina:

```powershell
New-NetFirewallRule -DisplayName "Allow HTTP 80" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
New-NetFirewallRule -DisplayName "Allow HTTPS 443" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lubab klientidel veebiserveri poole pöörduda | HTTP ja HTTPS ühendused töötavad |

Kontroll:

```powershell
Get-NetFirewallRule -DisplayName "Allow HTTP 80","Allow HTTPS 443"
```

---

# 15. Testimine

## 15.1 DNS kontroll

```powershell
nslookup siseportaal.sinuNimi.local
```

Oodatav:

```text
Nimi lahendub IIS serveri IP-ks
```

```powershell
nslookup www.sinuNimi.local
```

Oodatav:

```text
CNAME viitab siseportaal.sinuNimi.local nimele
```

---

## 15.2 HTTP kontroll

```powershell
Invoke-WebRequest http://siseportaal.sinuNimi.local
```

Oodatav:

```text
Veebiserver vastab
```

---

## 15.3 HTTPS kontroll

```powershell
Invoke-WebRequest https://siseportaal.sinuNimi.local
```

Oodatav:

```text
HTTPS veebiserver vastab
```

Kui tuleb sertifikaadi usalduse viga:

| Põhjus | Lahendus |
|---|---|
| Sertifikaat on self-signed | Dokumenteeri või usalda sertifikaat käsitsi |
| Klient ei usalda CA-d | Kontrolli, kas klient on domeenis |
| Sertifikaadi nimi ei klapi | Kontrolli CN/SAN nimesid |

---

## 15.4 CMS kontroll

Brauseris:

```text
https://siseportaal.sinuNimi.local
```

Oodatav:

```text
CMS-i avaleht avaneb
```

Admin / sisuhaldus:

```text
https://siseportaal.sinuNimi.local/wp-admin
```

või CMS-i vastav admin aadress.

Oodatav:

```text
Sisuhaldaja autentimine on seotud AD kasutaja või AD grupiga
```

---

## 15.5 GPO kontroll

Windows 11 kliendis Personal OU kasutajaga:

```cmd
gpupdate /force
```

```cmd
gpresult /r
```

Oodatav:

```text
GPO_Edge_Siseportaal rakendub kasutajale
```

Ava Edge.

Oodatav:

```text
Avaleht / uus vahekaart viitab siseportaalile
```

---

# 16. Lõppkontroll

| Kontroll | Käsk või koht | Oodatav tulemus |
|---|---|---|
| IIS teenus | `Get-Service W3SVC` | `Running` |
| DNS A-kirje | `nslookup siseportaal.sinuNimi.local` | Lahendub IIS serveri IP-ks |
| DNS CNAME | `nslookup www.sinuNimi.local` | Viitab siseportaalile |
| HTTP | `http://siseportaal.sinuNimi.local` | Veebileht avaneb |
| HTTPS | `https://siseportaal.sinuNimi.local` | Veebileht avaneb |
| AD CS | `Get-Service CertSvc` | `Running` |
| Sertifikaat | IIS → Server Certificates | Sertifikaat olemas |
| HTTPS binding | IIS → Bindings | Port 443 olemas |
| Toimetajad grupp | `Get-ADGroupMember Toimetajad` | Sisuhaldajad olemas |
| Edge GPO | `gpresult /r` | GPO rakendub Personal OU kasutajale |

---

# 17. Dokumentatsiooni näidis

```markdown
## Windows pilet 1 dokumentatsioon

### Eesmärk

Eesmärk oli paigaldada Windows Serverisse IIS veebiserver ja sellele sisuhaldussüsteem, siduda sisuhaldajate autentimine Active Directoryga, seadistada siseportaali DNS nimed, luua Edge avalehe grupipoliitika, paigaldada AD CS Enterprise Root CA ning seadistada veebileht HTTPS kaudu töötama.

### Kasutatud serverid

| Server | Roll |
|---|---|
| DC1 | AD DS, DNS, DHCP, AD CS |
| DC2 | Teine domeenikontroller ja DHCP failover partner |
| IIS server | IIS ja CMS |
| Windows 11 klient | Veebilehe, GPO ja autentimise testimiseks |

### Tehtud seadistused

- Paigaldasin IIS rolli.
- Lisasin vajalikud IIS komponendid CMS-i jaoks.
- Lõin veebikausta `C:\inetpub\siseportaal`.
- Lõin IIS-is saidi `Siseportaal`.
- Lisasin DNS A-kirje `siseportaal.sinuNimi.local`.
- Lisasin CNAME kirjed `www` ja `portaal`.
- Lõin AD grupi `Toimetajad`.
- Lisasin sisuhaldajad gruppi `Toimetajad`.
- Seadistasin CMS-i sisuhaldajate autentimise AD-ga või piirasin admin osa AD grupiga.
- Lõin GPO `GPO_Edge_Siseportaal`.
- Linkisin GPO OU-le `Personal`.
- Seadistasin Edge avalehe siseportaali aadressile.
- Paigaldasin AD CS rolli.
- Seadistasin Enterprise Root CA.
- Lõin veebilehele sertifikaadi.
- Lisasin IIS-i HTTPS bindingu.
- Lubasin tulemüüris pordid 80 ja 443.
- Testisin HTTP, HTTPS, DNS, CMS-i ja GPO-d.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `Get-Service W3SVC` | IIS töötab |
| `nslookup siseportaal.sinuNimi.local` | DNS A-kirje töötab |
| `nslookup www.sinuNimi.local` | CNAME töötab |
| `Get-Service CertSvc` | AD CS töötab |
| IIS Bindings | HTTP ja HTTPS bindingud olemas |
| Windows 11 brauser | HTTPS siseportaal avaneb |
| `gpresult /r` | Edge GPO rakendub |
| AD grupp `Toimetajad` | Sisuhaldajad kuuluvad gruppi |

### Kokkuvõte

Windows pilet 1 tulemusena töötab domeenikeskkonnas IIS veebiserver koos sisuhaldussüsteemiga. Siseportaal avaneb DNS nimedega, kasutab HTTPS ühendust AD CS sertifikaadiga ning sisuhaldajate ligipääs on seotud Active Directory kasutajate või grupiga. Personal OU kasutajatele rakendub GPO, mis määrab Microsoft Edge avalehe siseportaali aadressile.
```

---

# 18. Troubleshooting

## DNS nimi ei lahendu

Kontrolli:

```powershell
nslookup siseportaal.sinuNimi.local
```

Kui ei tööta:

```cmd
ipconfig /flushdns
```

Kontrolli kliendis:

```cmd
ipconfig /all
```

DNS serverid peavad olema:

```text
DC1 IP
DC2 IP
```

mitte:

```text
1.1.1.1
8.8.8.8
```

---

## IIS leht ei avane

Kontrolli:

```powershell
Get-Service W3SVC
```

Kui teenus ei tööta:

```powershell
Start-Service W3SVC
```

Kontrolli bindinguid:

```powershell
Get-WebBinding -Name "Siseportaal"
```

Kontrolli tulemüüri:

```powershell
Get-NetFirewallRule -DisplayName "Allow HTTP 80","Allow HTTPS 443"
```

---

## HTTPS ei tööta

Kontrolli IIS-is:

```text
Sites → Siseportaal → Bindings
```

Seal peab olema:

```text
https
port 443
sertifikaat valitud
```

Kontrolli sertifikaati:

```text
IIS Manager → Server Certificates
```

Kontrolli, et nimi klapib:

```text
siseportaal.sinuNimi.local
```

---

## Sertifikaat ei ole usaldatud

Põhjused:

| Põhjus | Lahendus |
|---|---|
| Kasutati self-signed sertifikaati | Dokumenteeri või usalda käsitsi |
| Klient ei ole domeenis | Lisa klient domeeni |
| AD CS pole Enterprise Root CA | Kontrolli CA seadistust |
| Sertifikaadi nimi ei klapi URL-iga | Loo uus sertifikaat õige CN/SAN nimega |

---

## Edge GPO ei rakendu

Kliendis:

```cmd
gpupdate /force
gpresult /r
```

Kui GPO puudub:

| Kontroll | Lahendus |
|---|---|
| Kas kasutaja on OU-s Personal? | Tõsta kasutaja õigesse OU-sse |
| Kas GPO on lingitud Personal OU-le? | Linki GPO õigele OU-le |
| Kas Edge ADMX mallid on olemas? | Lisa Microsoft Edge ADMX mallid |
| Kas testid õige kasutajaga? | Logi sisse Personal OU kasutajana |

---

## CMS-i AD autentimine ei tööta

Kontrolli:

| Kontroll | Mida vaadata |
|---|---|
| AD grupp | Kas `Toimetajad` grupp on olemas |
| Grupi liikmed | Kas sisuhaldajad on grupis |
| LDAP seaded | Base DN, DC aadress, bind user |
| IIS Authentication | Kas admin osa on kaitstud |
| DNS | Kas CMS server leiab DC1/DC2 |
| Aeg | Kontrolli, et serverite kellaaeg ei erineks liiga palju |

---

# 19. Kõige lühem spikker

```text
1. Paigalda IIS
2. Lisa PHP/CMS jaoks vajalikud komponendid
3. Loo C:\inetpub\siseportaal
4. Loo IIS sait Siseportaal
5. Lisa HTTP binding siseportaal.sinuNimi.local
6. Lisa DNS A-kirje siseportaal → IIS serveri IP
7. Lisa CNAME kirjed www ja portaal
8. Paigalda CMS, näiteks WordPress või Drupal
9. Loo AD grupp Toimetajad
10. Lisa sisuhaldajad gruppi
11. Seo CMS admin ligipääs AD-ga või piira admin osa AD grupiga
12. Loo GPO_Edge_Siseportaal
13. Linki GPO OU-le Personal
14. Määra Edge avaleht siseportaali aadressile
15. Paigalda AD CS
16. Seadista Enterprise Root CA
17. Loo sertifikaat siseportaal.sinuNimi.local jaoks
18. Lisa IIS HTTPS binding
19. Luba tulemüüris 80 ja 443
20. Testi DNS, HTTP, HTTPS, CMS, AD autentimine ja GPO
21. Dokumenteeri
```
