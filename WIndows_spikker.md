# Windowsi piletite baasosa ja lisateenused

See tabel annab kiire ülevaate, mida on vaja teha iga Windowsi pileti puhul.

Kõik Windowsi piletid algavad sama baasülesandega: tuleb luua AS Õige Windows Serveri põhine IT terviklahendus.  
Erinevus tuleb pileti lõpus olevast lisateenusest või eraldi praktilisest ülesandest.

---

## Windowsi piletite ühine baasosa

| Osa | Mida tuleb teha | Vajalik roll / teenus | Selgitus |
|---|---|---|---|
| AD domeen | Loo domeen `sinuNimi.local` | Active Directory Domain Services ehk AD DS | Windows Server GUI masinast saab esimene domeenikontroller `DC1` |
| Teine domeenikontroller | Lisa Windows Server Core domeenikontrolleriks | AD DS | Server Core masinast saab teine domeenikontroller `DC2` |
| DNS | DNS peab olema paigaldatud | DNS Server | DNS on vajalik domeeni toimimiseks ja nimelahenduseks |
| DHCP | Seadista DHCP server | DHCP Server | Jagab klientidele IP-aadresse |
| DHCP failover | Seadista DHCP failover DC1 ja DC2 vahel | DHCP Server | Kui üks DHCP server ei tööta, saab teine edasi IP-aadresse jagada |
| Staatilised rendid | Lisa klientidele DHCP reservation’id | DHCP Server | Kõigile Windowsi ja Linuxi klientidele kindlad IP-d |
| DHCP rendiaeg | Määra DHCP lease time 4 tundi | DHCP Server | IP-aadressi rendi kehtivusaeg |
| DHCP DNS seaded | DHCP peab jagama DNS serveritena mõlema domeenikontrolleri IP-d | DHCP Options | Klient peab kasutama domeeni DNS-servereid |
| OU struktuur | Loo domeeni OU-d `Kasutajad` ja `Arvutid` | Active Directory Users and Computers | Kasutajad ja arvutid hoitakse eraldi OU-des |
| Admin kasutaja | Loo kasutaja `Haldur` ja lisa `Domain Admins` gruppi | ADUC | Halduskasutaja domeeni administreerimiseks |
| Windows 11 klient | Lisa Windows 11 klient domeeni | System Properties / Settings | Klient peab olema domeenis |
| Arvuti OU | Paiguta Windows 11 klient OU-sse `Arvutid` | ADUC | GPO rakendub õigele arvuti OU-le |
| DNS kirjed | Lisa vajalikud DNS kirjed | DNS Manager | Piletis nõutud teenuste nimelahendus |
| Kasutajate import | Impordi kasutajad CSV failist | PowerShell | Kasutajad luuakse automaatselt OU struktuuri järgi |
| Paroolipoliitika | Loo või seadista GPO `ParooliKehtivus` | Group Policy Management | Parooli maksimaalne eluiga 30 päeva |
| Kontolukustus | Loo GPO `KontodeLukustamine` | Group Policy Management | Konto lukustub 15 minutiks pärast 5 valet parooli |
| USB piirang | Loo GPO `KeelaUSB` | Group Policy Management | Keelab OU `Arvutid` arvutites USB andmekandjate lugemise ja kirjutamise |

---

## Windowsi piletite lisateenused ja eriosad

| Pilet | Lisateenus / eriosa | Vajalik roll, funktsioon või tarkvara | Kus tehakse | Lühiselgitus |
|---|---|---|---|---|
| Windows pilet 1 | IIS veebiserver | `Web Server (IIS)` | Windows Server GUI | Paigaldatakse veebiserver ja seadistatakse veebileht |
| Windows pilet 1 | Autentimine veebilehele | IIS + AD kasutajad | IIS Manager / ADUC | Veebilehe sisselogimine seotakse Active Directory kasutajatega |
| Windows pilet 1 | Sertifikaadid | `Active Directory Certificate Services` ehk AD CS | Windows Server GUI | Paigaldatakse Certificate Authority ja luuakse SSL sertifikaat |
| Windows pilet 1 | HTTPS veebileht | IIS + AD CS sertifikaat | IIS Manager | Veebileht peab töötama HTTPS-iga |
| Windows pilet 1 | DNS CNAME | DNS Server | DNS Manager | Veebilehele luuakse DNS alias ehk CNAME kirje |

| Windows pilet 2 | DFS teenus | `DFS Namespaces` ja `DFS Replication` | Windows Server GUI + Core | Jagatud kaustad tehakse kättesaadavaks DFS nimeruumi kaudu |
| Windows pilet 2 | Kaustade replikatsioon | DFS Replication | DC1 ja DC2 | Jagatud kaustad replikeeritakse kahe serveri vahel |
| Windows pilet 2 | Failide piirang | File Server Resource Manager ehk FSRM | Windows Server GUI | Keelatakse teatud failitüübid, näiteks `.exe`, `.bat`, `.ps1` |
| Windows pilet 2 | Kasutajapõhine mahupiirang | FSRM Quota | Windows Server GUI | Kasutajate kaustadele määratakse 1 GB piirang |

| Windows pilet 3 | PowerShell skriptid | PowerShell | Windows Server | Tuleb koostada skriptid AD kontode ja DHCP info kontrollimiseks |
| Windows pilet 3 | AD kontode kontroll | AD PowerShell moodul | DC1 | Skript peab näitama AD kontosid, sh lukustatud ja aegunud kontosid |
| Windows pilet 3 | DHCP raport | DHCP PowerShell moodul | DHCP server | Skript peab andma ülevaate DHCP skoobist, lease’idest, reservation’idest ja vabadest IP-dest |

| Windows pilet 4 | Tarkvara paigaldus domeeni klientidele | GPO Software Installation või muu keskne paigaldusviis | Group Policy Management | Tuleb paigaldada tarkvara domeeni arvutitele |
| Windows pilet 4 | LibreOffice | LibreOffice MSI installer | Klientarvutid / GPO | Paigaldada eesti keelne LibreOffice |
| Windows pilet 4 | PuTTY | PuTTY installer / MSI | Klientarvutid / GPO | Paigaldada PuTTY |
| Windows pilet 4 | Google Chrome | Chrome Enterprise MSI | Klientarvutid / GPO | Paigaldada Chrome ja määrata avaleht |
| Windows pilet 4 | Taustapilt | GPO Desktop Wallpaper | Group Policy Management | Kõigile klientidele määratakse ettevõtte taustapilt |
| Windows pilet 4 | Logimise piirang | GPO / User Rights Assignment | Group Policy Management | Ainult AD kontod tohivad sisse logida |
| Windows pilet 4 | Õiguste piirang | GPO | Group Policy Management | Kasutajad ei tohi muuta taustapilti |
| Windows pilet 4 | Keelatud tegevused | GPO | Group Policy Management | Keelatakse näiteks Run, Control Panel, Task Manager, CMD käivitamine |

| Windows pilet 5 | WDS teenus | `Windows Deployment Services` | Windows Server GUI | Windowsi paigaldamine üle võrgu |
| Windows pilet 5 | Windows 10 image | Windows 10 Enterprise ISO | WDS server | Lisatakse Windows 10 paigaldusmeedia |
| Windows pilet 5 | Windows 11 image | Windows 11 ISO | WDS server | Lisatakse Windows 11 paigaldusmeedia |
| Windows pilet 5 | DHCP seadistus PXE jaoks | DHCP Server options | DHCP Manager | DHCP peab toetama WDS/PXE bootimist |
| Windows pilet 5 | Testmasina paigaldus üle võrgu | WDS + PXE boot | Test VM | Testmasin käivitatakse võrgust ja sinna paigaldatakse Windows 11 |

---

## Lühike ülevaade pileti kaupa

| Pilet | Baasosa | Peamine lisateenus |
|---|---|---|
| Windows pilet 1 | AD DS, DNS, DHCP, GPO, OU-d, kasutajad, domeeniklient | IIS veebiserver, AD autentimine, AD CS sertifikaadid, HTTPS |
| Windows pilet 2 | AD DS, DNS, DHCP, GPO, OU-d, kasutajad, domeeniklient | DFS Namespaces, DFS Replication, FSRM |
| Windows pilet 3 | AD DS, DNS, DHCP, GPO, OU-d, kasutajad, domeeniklient | PowerShell skriptid AD ja DHCP info kontrollimiseks |
| Windows pilet 4 | AD DS, DNS, DHCP, GPO, OU-d, kasutajad, domeeniklient | Tarkvara keskne paigaldus ja kasutajapiirangud GPO-ga |
| Windows pilet 5 | AD DS, DNS, DHCP, GPO, OU-d, kasutajad, domeeniklient | WDS ehk Windows Deployment Services |

---

## Millised rollid tuleb Server Managerist lisada?

| Roll / funktsioon | Vajalik millistes piletites? | Milleks kasutatakse? |
|---|---|---|
| Active Directory Domain Services | Kõik Windowsi piletid | Domeeni ja domeenikontrollerite loomiseks |
| DNS Server | Kõik Windowsi piletid | Domeeni ja teenuste nimelahenduseks |
| DHCP Server | Kõik Windowsi piletid | Klientidele IP-aadresside jagamiseks |
| Group Policy Management | Kõik Windowsi piletid | GPO-de loomiseks ja haldamiseks |
| Web Server (IIS) | Windows pilet 1 | Veebilehe majutamiseks |
| Active Directory Certificate Services | Windows pilet 1 | Sertifikaatide ja HTTPS jaoks |
| DFS Namespaces | Windows pilet 2 | Ühtse jagatud kaustade nimeruumi loomiseks |
| DFS Replication | Windows pilet 2 | Kaustade replikatsiooniks serverite vahel |
| File Server Resource Manager | Windows pilet 2 | Failitüüpide ja kettamahu piiramiseks |
| Windows Deployment Services | Windows pilet 5 | Windowsi paigaldamiseks üle võrgu |

---

## Millist välist tarkvara võib vaja minna?

| Tarkvara / fail | Vajalik millises piletis? | Milleks kasutatakse? |
|---|---|---|
| `kasutajad.csv` | Kõik Windowsi piletid | Kasutajate importimiseks AD-sse PowerShelli skriptiga |
| Windows 10 Enterprise ISO | Windows pilet 5 | WDS paigaldusimage’i lisamiseks |
| Windows 11 ISO | Windows pilet 5 | WDS paigaldusimage’i lisamiseks |
| LibreOffice MSI | Windows pilet 4 | Tarkvara paigaldamiseks domeeni klientidele |
| PuTTY installer / MSI | Windows pilet 4 | Tarkvara paigaldamiseks domeeni klientidele |
| Google Chrome Enterprise MSI | Windows pilet 4 | Chrome’i keskseks paigaldamiseks |
| Ettevõtte taustapilt | Windows pilet 4 | GPO kaudu klientidele taustapildiks |
| Veebilehe failid | Windows pilet 1 | IIS veebilehe sisuks |
| SSL sertifikaat | Windows pilet 1 | HTTPS veebilehe jaoks, luuakse AD CS kaudu |

---

## Soovitatav tööjärjekord iga Windowsi pileti puhul

| Järk | Tegevus |
|---|---|
| 1 | Kontrolli VM-id ja võrguliidesed |
| 2 | Määra serveritele staatilised IP-aadressid |
| 3 | Muuda serverite nimed: näiteks `DC1` ja `DC2` |
| 4 | Paigalda `DC1` masinasse AD DS ja DNS roll |
| 5 | Loo uus domeen `sinuNimi.local` |
| 6 | Lisa `DC2` domeeni |
| 7 | Paigalda `DC2` masinasse AD DS ja DNS |
| 8 | Tee `DC2` teiseks domeenikontrolleriks |
| 9 | Paigalda ja seadista DHCP |
| 10 | Tee DHCP failover DC1 ja DC2 vahel |
| 11 | Loo OU-d `Kasutajad` ja `Arvutid` |
| 12 | Loo kasutaja `Haldur` ja lisa `Domain Admins` gruppi |
| 13 | Impordi kasutajad CSV failist |
| 14 | Lisa Windows 11 klient domeeni |
| 15 | Tõsta Windows 11 arvuti OU-sse `Arvutid` |
| 16 | Loo vajalikud GPO-d |
| 17 | Testi kliendist, kas GPO rakendub |
| 18 | Tee konkreetse pileti lisateenus |
| 19 | Testi kogu lahendus läbi |
| 20 | Dokumenteeri seadistused, testid ja tulemused |

---

## Kõige lühem spikker

| Pilet | Mida lisaks baasosale kindlasti õppida? |
|---|---|
| Windows pilet 1 | IIS, HTTPS, AD CS, veebilehe autentimine |
| Windows pilet 2 | DFS, DFS replikatsioon, FSRM |
| Windows pilet 3 | PowerShell skriptid AD ja DHCP kohta |
| Windows pilet 4 | GPO tarkvara paigaldus ja kasutajapiirangud |
| Windows pilet 5 | WDS, PXE boot, Windows image’id |
