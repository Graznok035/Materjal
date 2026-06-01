# Windowsi piletite baasosa ja lisateenused

See dokument annab ülevaate, mida tuleb teha kõikides Windowsi piletites ning millised lisateenused on iga konkreetse pileti puhul vajalikud.

Kõik Windowsi piletid sisaldavad ühist baasosa:

- Active Directory domeen
- DNS server
- DHCP server
- DHCP failover
- OU struktuur
- kasutajate import CSV failist
- Windows 11 kliendi lisamine domeeni
- GPO poliitikad

Lisaks on igal piletil oma eraldi lisateenus või praktiline eriosa.

---

## 1. Kõikide Windowsi piletite baasosa

| Teema | Mida tuleb teha | Vajalik roll / tööriist | Milleks seda vaja on |
|---|---|---|---|
| Domeen | Loo AD domeen `sinuNimi.local` | Active Directory Domain Services | Domeeni, kasutajate ja arvutite keskseks haldamiseks |
| DC1 | Windows Server GUI masin seadista esimeseks domeenikontrolleriks | AD DS + DNS | Peamine domeenikontroller |
| DC2 | Windows Server Core masin seadista teiseks domeenikontrolleriks | AD DS + DNS | Varudomeenikontroller |
| DNS | Paigalda ja seadista DNS roll | DNS Server | Domeeni ja serverite nimelahenduseks |
| DHCP | Paigalda ja seadista DHCP | DHCP Server | Klientidele IP-aadresside jagamiseks |
| DHCP failover | Seadista DHCP failover DC1 ja DC2 vahel | DHCP Server | DHCP töökindluse tagamiseks |
| Staatilised rendid | Lisa reservation’id klientarvutitele | DHCP Manager | Klientidele kindlate IP-aadresside määramiseks |
| DHCP lease time | Määra rendi kehtivusajaks 4 tundi | DHCP Scope Properties | IP rendi aja määramiseks |
| DHCP DNS option | DHCP peab jagama mõlema DC DNS-aadresse | DHCP Options | Klient peab kasutama domeeni DNS-servereid |
| OU struktuur | Loo OU-d `Kasutajad` ja `Arvutid` | Active Directory Users and Computers | Kasutajate ja arvutite loogiliseks eraldamiseks |
| Haldur kasutaja | Loo kasutaja `Haldur` ja lisa `Domain Admins` gruppi | ADUC | Administraatori õigustega domeenikasutaja |
| Windows 11 klient | Lisa Windows 11 klient domeeni | Windows Settings / System Properties | Et klient saaks kasutada domeeni kasutajaid ja GPO-sid |
| Arvuti OU | Tõsta Windows 11 klient OU-sse `Arvutid` | ADUC | Et arvutile rakenduksid õiged GPO-d |
| Kasutajate import | Impordi kasutajad failist `kasutajad.csv` | PowerShell | Kasutajate automaatseks loomiseks |
| Paroolipoliitika | Loo GPO `ParooliKehtivus` | Group Policy Management | Parooli maksimaalne eluiga 30 päeva |
| Kontolukustus | Loo GPO `KontodeLukustamine` | Group Policy Management | Konto lukustub 15 minutiks pärast 5 vale parooli |
| USB piirang | Loo GPO `KeelaUSB` | Group Policy Management | Keelab USB andmekandjate lugemise ja kirjutamise |

---

## 2. Piletite lisateenused

| Windowsi pilet | Pileti lisateenus / eriosa | Vajalik roll, teenus või tarkvara | Lühiselgitus |
|---|---|---|---|
| Windows pilet 1 | Veebiserver | Web Server IIS | Tuleb paigaldada ja seadistada IIS veebiserver |
| Windows pilet 1 | Veebilehe AD autentimine | IIS + Active Directory | Veebilehele sisselogimine peab olema seotud AD kasutajatega |
| Windows pilet 1 | Sertifikaadid | Active Directory Certificate Services | Tuleb seadistada CA ja luua sertifikaat |
| Windows pilet 1 | HTTPS | IIS + SSL sertifikaat | Veebileht peab töötama turvaliselt HTTPS kaudu |
| Windows pilet 1 | DNS kirje veebilehele | DNS Manager | Veebilehele tuleb luua sobiv DNS kirje või CNAME |
| Windows pilet 2 | DFS nimeruum | DFS Namespaces | Jagatud kaustade keskseks esitamiseks ühe võrgunime kaudu |
| Windows pilet 2 | DFS replikatsioon | DFS Replication | Jagatud kaustade sisu sünkroonimiseks serverite vahel |
| Windows pilet 2 | Failiserveri piirangud | File Server Resource Manager | Failitüüpide ja kettakasutuse piiramiseks |
| Windows pilet 2 | Kvoodid | FSRM Quota Management | Kasutajatele kettamahu piirangu seadmiseks |
| Windows pilet 3 | AD kontrollskript | PowerShell + AD moodul | Skript peab kontrollima AD kontode infot |
| Windows pilet 3 | DHCP kontrollskript | PowerShell + DHCP moodul | Skript peab näitama DHCP skoobi ja rendi infot |
| Windows pilet 3 | Raporti koostamine | PowerShell | Tuleb koostada raport AD ja DHCP seisu kohta |
| Windows pilet 4 | Tarkvara paigaldus klientidele | GPO / tarkvara paigaldus | Domeeni arvutitele tuleb paigaldada vajalik tarkvara |
| Windows pilet 4 | LibreOffice | LibreOffice MSI installer | LibreOffice tuleb paigaldada eesti keeles |
| Windows pilet 4 | PuTTY | PuTTY installer / MSI | PuTTY tuleb paigaldada klientarvutitesse |
| Windows pilet 4 | Google Chrome | Chrome Enterprise MSI | Chrome tuleb paigaldada ja seadistada avaleht |
| Windows pilet 4 | Taustapilt | GPO Desktop Wallpaper | Klientidele määratakse ettevõtte taustapilt |
| Windows pilet 4 | Kasutajapiirangud | Group Policy | Keelatakse näiteks taustapildi muutmine, CMD, Run, Control Panel või Task Manager |
| Windows pilet 5 | Windowsi paigaldus üle võrgu | Windows Deployment Services | Tuleb seadistada WDS server |
| Windows pilet 5 | Windows 10 image | Windows 10 Enterprise ISO | WDS-i lisatakse Windows 10 paigaldusmeedia |
| Windows pilet 5 | Windows 11 image | Windows 11 ISO | WDS-i lisatakse Windows 11 paigaldusmeedia |
| Windows pilet 5 | PXE boot | WDS + DHCP | Testmasin peab saama käivituda võrgu kaudu |
| Windows pilet 5 | Testpaigaldus | WDS | Windows 11 paigaldatakse testmasinasse üle võrgu |

---

## 3. Rollid, mida Server Managerist lisada

| Roll / funktsioon | Millistes piletites vajalik | Milleks kasutatakse |
|---|---|---|
| Active Directory Domain Services | Kõik piletid | Domeeni ja domeenikontrollerite loomiseks |
| DNS Server | Kõik piletid | Nimelahenduseks domeenis |
| DHCP Server | Kõik piletid | IP-aadresside automaatseks jagamiseks |
| Group Policy Management | Kõik piletid | GPO-de loomiseks ja haldamiseks |
| Web Server IIS | Windows pilet 1 | Veebiserveri loomiseks |
| Active Directory Certificate Services | Windows pilet 1 | Sertifikaatide ja HTTPS jaoks |
| DFS Namespaces | Windows pilet 2 | DFS jagatud nimeruumi loomiseks |
| DFS Replication | Windows pilet 2 | Kaustade sünkroonimiseks serverite vahel |
| File Server Resource Manager | Windows pilet 2 | Kvootide ja failipiirangute seadistamiseks |
| Windows Deployment Services | Windows pilet 5 | Windowsi paigaldamiseks üle võrgu |

---

## 4. Välised failid ja tarkvara

| Fail / tarkvara | Millises piletis vajalik | Milleks kasutatakse |
|---|---|---|
| `kasutajad.csv` | Kõik piletid | Kasutajate importimiseks AD-sse |
| Windows 10 Enterprise ISO | Windows pilet 5 | WDS paigaldusimage’i lisamiseks |
| Windows 11 ISO | Windows pilet 5 | WDS paigaldusimage’i lisamiseks |
| LibreOffice MSI | Windows pilet 4 | LibreOffice’i keskseks paigaldamiseks |
| PuTTY installer / MSI | Windows pilet 4 | PuTTY paigaldamiseks klientarvutitesse |
| Chrome Enterprise MSI | Windows pilet 4 | Google Chrome’i keskseks paigaldamiseks |
| Ettevõtte taustapilt | Windows pilet 4 | GPO kaudu töölaua taustapildiks |
| Veebilehe failid | Windows pilet 1 | IIS veebilehe sisuks |
| SSL sertifikaat | Windows pilet 1 | HTTPS ühenduse jaoks |

---

## 5. Kiire ülevaade pileti kaupa

| Pilet | Baasosa | Lisateenus |
|---|---|---|
| Windows pilet 1 | AD DS, DNS, DHCP, OU-d, GPO-d, kasutajad, Windows 11 domeenis | IIS, AD autentimine, AD CS, HTTPS |
| Windows pilet 2 | AD DS, DNS, DHCP, OU-d, GPO-d, kasutajad, Windows 11 domeenis | DFS, DFS replikatsioon, FSRM |
| Windows pilet 3 | AD DS, DNS, DHCP, OU-d, GPO-d, kasutajad, Windows 11 domeenis | PowerShell skriptid AD ja DHCP kontrolliks |
| Windows pilet 4 | AD DS, DNS, DHCP, OU-d, GPO-d, kasutajad, Windows 11 domeenis | Tarkvara paigaldus ja kasutajapiirangud GPO-ga |
| Windows pilet 5 | AD DS, DNS, DHCP, OU-d, GPO-d, kasutajad, Windows 11 domeenis | WDS ja Windowsi paigaldus üle võrgu |

---

## 6. Soovitatav tööjärjekord

| Järk | Tegevus |
|---|---|
| 1 | Kontrolli, et vajalikud VM-id töötavad |
| 2 | Määra serveritele staatilised IP-aadressid |
| 3 | Muuda serverite nimed, näiteks `DC1` ja `DC2` |
| 4 | Paigalda `DC1` masinasse AD DS ja DNS |
| 5 | Loo domeen `sinuNimi.local` |
| 6 | Lisa `DC2` domeeni |
| 7 | Tee `DC2` teisese domeenikontrollerina tööle |
| 8 | Paigalda ja seadista DHCP |
| 9 | Seadista DHCP failover |
| 10 | Lisa DHCP reservation’id klientidele |
| 11 | Loo OU-d `Kasutajad` ja `Arvutid` |
| 12 | Loo kasutaja `Haldur` ja lisa `Domain Admins` gruppi |
| 13 | Impordi kasutajad CSV failist |
| 14 | Lisa Windows 11 klient domeeni |
| 15 | Tõsta Windows 11 klient OU-sse `Arvutid` |
| 16 | Loo vajalikud GPO-d |
| 17 | Kontrolli kliendis käsuga `gpupdate /force` ja `gpresult /r` |
| 18 | Tee konkreetse pileti lisateenus |
| 19 | Testi kogu lahendus läbi |
| 20 | Dokumenteeri tehtud seadistused ja kontrollid |

---

## 7. Kõige lühem meelespea

| Pilet | Õpi kindlasti juurde |
|---|---|
| Windows pilet 1 | IIS, HTTPS, AD CS, veebilehe autentimine |
| Windows pilet 2 | DFS, DFS Replication, FSRM |
| Windows pilet 3 | PowerShell, AD moodul, DHCP moodul |
| Windows pilet 4 | GPO tarkvarapaigaldus, kasutajapiirangud |
| Windows pilet 5 | WDS, PXE boot, Windows image’id |

---

## Kokkuvõte

Kõikide Windowsi piletite põhituum on sama:

```text
AD DS + DNS + DHCP + OU-d + kasutajad + GPO + Windows 11 domeenis
```

Pärast baasosa lisandub igale piletile oma eriosa:

```text
Pilet 1 = IIS + AD CS + HTTPS
Pilet 2 = DFS + FSRM
Pilet 3 = PowerShell skriptid
Pilet 4 = GPO tarkvarahaldus ja piirangud
Pilet 5 = WDS ja võrgu kaudu Windowsi paigaldus
```
