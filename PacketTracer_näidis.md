# Võrgud. Pilet 1 — ruuterid, switchid, HSRP, EtherChannel, SSH ja staatiline marsruutimine

<img width="1098" height="477" alt="image" src="https://github.com/user-attachments/assets/f03c022d-ca46-4557-929d-15def630e412" />


## Märksõnad

- Cisco Packet Tracer
- Ruuterite algseadistus
- Switchide algseadistus
- IP-aadresside seadistamine
- Staatiline marsruutimine
- HSRP
- EtherChannel
- SSH seadistamine
- Paroolid ja turvaseaded
- Konfiguratsiooni salvestamine

---

# 1. Ülesande lühikokkuvõte

Selles piletis tuleb Packet Traceris seadistada võrk, kus on:

```text
3 ruuterit: R0, R1, R2
2 switchi: S1, S2
2 klientarvutit: PC0, PC1
```

Põhieesmärk on seadistada:

- seadmete hostinimed
- IP-aadressid
- paroolid
- SSH ligipääs
- staatiline marsruutimine
- HSRP
- EtherChannel
- switchide haldus-IP-d
- konfiguratsiooni salvestamine

Lõpptulemusena peab võrk töötama nii, et:

```text
PC0 ja PC1 saavad omavahel suhelda
PC-d saavad pingida gateway aadressi
ruuterid näevad üksteist
staatilised marsruudid töötavad
HSRP annab varugateway lahenduse
EtherChannel töötab S1 ja S2 vahel
SSH ligipääs töötab ruuteritesse ja switchidesse
kõik konfiguratsioonid on salvestatud
```

---

# 2. Baasteadmised

## Ruuter

Ruuter ühendab erinevaid võrke omavahel.

Näiteks:

```text
192.168.1.0/24 LAN võrk
10.1.1.0/30 R1-R0 serial võrk
10.2.2.0/30 R2-R0 serial võrk
25.200.225.0/27 loopback võrk
```

Ruuter otsustab, kuhu pakett edasi saata.

---

## Switch

Switch ühendab sama kohaliku võrgu seadmeid.

Näiteks:

```text
PC0
PC1
R1
R2
```

Switch töötab tavaliselt Layer 2 tasemel ehk MAC-aadresside järgi.

---

## VLAN

VLAN jagab switchi loogilisteks võrkudeks.

Kui ülesandes ei nõuta eraldi VLAN-e, kasutatakse tavaliselt vaikimisi VLAN-i:

```text
VLAN 1
```

Switchi haldamiseks pannakse IP tavaliselt VLAN interface’i peale:

```cisco
interface vlan 1
ip address 192.168.1.4 255.255.255.0
no shutdown
```

---

## Default gateway

Default gateway on aadress, kuhu arvuti saadab liikluse siis, kui sihtkoht ei ole samas võrgus.

Näiteks:

```text
PC0 IP: 192.168.1.31/24
PC0 gateway: 192.168.1.254
```

Kui kasutatakse HSRP-d, peab PC-de gateway olema HSRP virtuaalne IP.

---

## Staatiline marsruut

Staatiline marsruut ütleb ruuterile käsitsi, kuhu mingi võrk asub.

Näide:

```cisco
ip route 192.168.1.0 255.255.255.0 10.1.1.1
```

See tähendab:

```text
Kui tahad minna võrku 192.168.1.0/24, saada liiklus järgmisele ruuterile 10.1.1.1.
```

---

## HSRP

HSRP on Cisco protokoll, millega kaks ruuterit jagavad ühte virtuaalset gateway aadressi.

Näide:

```text
R1 füüsiline IP: 192.168.1.1
R2 füüsiline IP: 192.168.1.3
HSRP virtuaalne IP: 192.168.1.254
```

Klientide default gateway:

```text
192.168.1.254
```

Kui aktiivne ruuter läheb katki, võtab teine ruuter gateway rolli üle.

---

## EtherChannel

EtherChannel ühendab mitu füüsilist linki üheks loogiliseks lingiks.

Näiteks kui S1 ja S2 vahel on kaks kaablit:

```text
Fa0/1
Fa0/2
```

siis tehakse neist üks loogiline ühendus:

```text
Port-channel 1
```

EtherChannel annab:

```text
rohkem läbilaskevõimet
varunduse ühe lingi katkemisel
vähem spanning-tree probleeme
```

---

## SSH

SSH on turvaline viis võrguseadmesse kaugelt sisse logida.

SSH jaoks on vaja:

```text
hostname
domain-name
kasutaja
parool
RSA võti
VTY liinidel login local
transport input ssh
```

---

# 3. Näidis aadressiplaan

Pildilt loetav ligikaudne aadressiplaan.

NB! Kui sinu Packet Traceri failis on mõni aadress erinev, kasuta sealset aadressitabelit.

| Seade | Liides | IP-aadress |
|---|---|---|
| R1 | G0/1 | `192.168.1.1/24` |
| R1 | S0/1/1 | `10.1.1.1/30` |
| R2 | G0/1 | `192.168.1.3/24` |
| R2 | S0/1/0 | `10.2.2.1/30` |
| R0 | S0/1/0 | `10.1.1.2/30` |
| R0 | S0/1/1 | `10.2.2.2/30` |
| R0 | Loopback 1 | `25.200.225.1/27` |
| S1 | VLAN 1 | `192.168.1.4/24` |
| S2 | VLAN 1 | `192.168.1.10/24` |
| PC0 | NIC | `192.168.1.31/24` |
| PC1 | NIC | `192.168.1.32/24` |
| HSRP | Virtual IP | `192.168.1.254/24` |

PC-de default gateway:

```text
192.168.1.254
```

---

# 4. Soovituslik tööjärjekord

1. Pane seadmete nimed paika.
2. Seadista paroolid ja turvaseaded.
3. Seadista IP-aadressid ruuterite liidestele.
4. Lülita ruuterite liidesed sisse.
5. Seadista PC-de IP, mask ja gateway.
6. Seadista switchide VLAN1 IP-aadressid.
7. Seadista switchide default gateway.
8. Kontrolli sama võrgu ühendust pingiga.
9. Seadista staatilised marsruudid.
10. Kontrolli ühendust eri võrkude vahel.
11. Seadista HSRP R1 ja R2 vahel.
12. Kontrolli, et PC-de gateway on HSRP virtuaalne IP.
13. Seadista EtherChannel S1 ja S2 vahel.
14. Seadista SSH ruuteritel ja switchidel.
15. Salvesta konfiguratsioonid.
16. Tee lõplik kontroll.

---

# 5. Ruuterite algseadistus

## R0

```cisco
enable
configure terminal
hostname R0
no ip domain-lookup
enable secret class
service password-encryption
banner motd #Lubamatu ligipääs keelatud!#
```

## R1

```cisco
enable
configure terminal
hostname R1
no ip domain-lookup
enable secret class
service password-encryption
banner motd #Lubamatu ligipääs keelatud!#
```

## R2

```cisco
enable
configure terminal
hostname R2
no ip domain-lookup
enable secret class
service password-encryption
banner motd #Lubamatu ligipääs keelatud!#
```

---

# 6. Konsooli ja VTY paroolid ruuterites

Tee igal ruuteril:

```cisco
line console 0
password cisco
login
logging synchronous
exit

line vty 0 4
password cisco
login
logging synchronous
exit
```

Hiljem SSH seadistamisel muudetakse VTY read turvalisemaks:

```cisco
login local
transport input ssh
```

---

# 7. R1 IP-aadressid

R1:

```cisco
configure terminal

interface g0/1
description LAN R1-S2
ip address 192.168.1.1 255.255.255.0
no shutdown
exit

interface s0/1/1
description Link R1-R0
ip address 10.1.1.1 255.255.255.252
no shutdown
exit
```

Kontroll:

```cisco
show ip interface brief
```

---

# 8. R2 IP-aadressid

R2:

```cisco
configure terminal

interface g0/1
description LAN R2-S1
ip address 192.168.1.3 255.255.255.0
no shutdown
exit

interface s0/1/0
description Link R2-R0
ip address 10.2.2.1 255.255.255.252
no shutdown
exit
```

Kontroll:

```cisco
show ip interface brief
```

---

# 9. R0 IP-aadressid

R0:

```cisco
configure terminal

interface s0/1/0
description Link R0-R1
ip address 10.1.1.2 255.255.255.252
no shutdown
exit

interface s0/1/1
description Link R0-R2
ip address 10.2.2.2 255.255.255.252
no shutdown
exit

interface loopback 1
description Test loopback network
ip address 25.200.225.1 255.255.255.224
exit
```

Kui serial ühenduse DCE pool on R0 peal ja link ei tõuse üles, lisa DCE poolele clock rate:

```cisco
interface s0/1/0
clock rate 64000
exit

interface s0/1/1
clock rate 64000
exit
```

Kontroll:

```cisco
show ip interface brief
show controllers serial 0/1/0
show controllers serial 0/1/1
```

---

# 10. PC-de IP seadistamine

Packet Traceris:

```text
PC → Desktop → IP Configuration
```

## PC0

```text
IP Address: 192.168.1.31
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.1.254
```

## PC1

```text
IP Address: 192.168.1.32
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.1.254
```

NB! Gateway on HSRP virtuaalne IP.

---

# 11. Switchide algseadistus

## S1

```cisco
enable
configure terminal
hostname S1
no ip domain-lookup
enable secret class
service password-encryption
banner motd #Lubamatu ligipääs keelatud!#
```

## S2

```cisco
enable
configure terminal
hostname S2
no ip domain-lookup
enable secret class
service password-encryption
banner motd #Lubamatu ligipääs keelatud!#
```

---

# 12. Konsooli ja VTY paroolid switchides

Tee mõlemal switchil:

```cisco
line console 0
password cisco
login
logging synchronous
exit

line vty 0 15
password cisco
login
logging synchronous
exit
```

---

# 13. Switchide haldus-IP VLAN1 peale

## S1

```cisco
configure terminal

interface vlan 1
ip address 192.168.1.4 255.255.255.0
no shutdown
exit

ip default-gateway 192.168.1.254
```

## S2

```cisco
configure terminal

interface vlan 1
ip address 192.168.1.10 255.255.255.0
no shutdown
exit

ip default-gateway 192.168.1.254
```

Kontroll:

```cisco
show ip interface brief
```

Kui VLAN1 on `down`, kontrolli, kas switchil on mõni aktiivne port samas VLAN-is ühendatud.

---

# 14. Esmane pingikontroll

## PC0-st

```text
ping 192.168.1.1
ping 192.168.1.3
ping 192.168.1.4
ping 192.168.1.10
```

## R1-st

```cisco
ping 10.1.1.2
```

## R2-st

```cisco
ping 10.2.2.2
```

## R0-st

```cisco
ping 10.1.1.1
ping 10.2.2.1
```

Kui need ei tööta, ära mine edasi HSRP ega EtherChanneli juurde.  
Paranda kõigepealt IP-d, kaablid ja `no shutdown`.

---

# 15. Staatilised marsruudid

## R1

R1-le võib lisada default route R0 suunas:

```cisco
configure terminal
ip route 0.0.0.0 0.0.0.0 10.1.1.2
```

## R2

R2-le võib lisada default route R0 suunas:

```cisco
configure terminal
ip route 0.0.0.0 0.0.0.0 10.2.2.2
```

## R0

R0 peab teadma, kus asub LAN `192.168.1.0/24`.

```cisco
configure terminal
ip route 192.168.1.0 255.255.255.0 10.1.1.1
ip route 192.168.1.0 255.255.255.0 10.2.2.1
```

Kontroll:

```cisco
show ip route
```

---

# 16. HSRP seadistamine R1 ja R2 vahel

HSRP tehakse R1 ja R2 LAN liidestele.

Virtuaalne gateway:

```text
192.168.1.254
```

## R1 HSRP

```cisco
configure terminal

interface g0/1
standby 1 ip 192.168.1.254
standby 1 priority 110
standby 1 preempt
exit
```

## R2 HSRP

```cisco
configure terminal

interface g0/1
standby 1 ip 192.168.1.254
standby 1 priority 100
standby 1 preempt
exit
```

Selgitus:

```text
standby 1 ip = HSRP virtuaalne IP
priority 110 = R1 on eelistatud aktiivne ruuter
priority 100 = R2 on varuruuter
preempt = kui R1 tuleb tagasi, võtab ta aktiivse rolli tagasi
```

Kontroll:

```cisco
show standby brief
```

Oodatav tulemus:

```text
R1 = Active
R2 = Standby
Virtual IP = 192.168.1.254
```

---

# 17. EtherChannel S1 ja S2 vahel

Pildil on S1 ja S2 vahel mitu paralleelset linki. Need tuleb panna EtherChannelisse.

NB! Kontrolli Packet Traceris, millised pordid päriselt S1 ja S2 vahel ühendatud on.

Näites kasutatakse porte:

```text
Fa0/1
Fa0/2
```

## S1 EtherChannel

```cisco
configure terminal

interface range fa0/1 - 2
description EtherChannel to S2
switchport mode trunk
channel-group 1 mode active
no shutdown
exit

interface port-channel 1
description EtherChannel to S2
switchport mode trunk
exit
```

## S2 EtherChannel

```cisco
configure terminal

interface range fa0/1 - 2
description EtherChannel to S1
switchport mode trunk
channel-group 1 mode active
no shutdown
exit

interface port-channel 1
description EtherChannel to S1
switchport mode trunk
exit
```

Siin kasutatakse LACP varianti:

```text
mode active
```

Kontroll:

```cisco
show etherchannel summary
```

Oodatav:

```text
Po1(SU)
Fa0/1(P)
Fa0/2(P)
```

Tähendused:

```text
S = Layer 2
U = in use
P = port on korrektselt port-channelis
```

---

# 18. SSH seadistamine ruuterites

Tee igas ruuteris.

Näide R1 kohta:

```cisco
configure terminal

ip domain-name oige.local
username admin secret cisco123
crypto key generate rsa
```

Kui küsib key size, sisesta:

```text
1024
```

Seejärel:

```cisco
ip ssh version 2

line vty 0 4
login local
transport input ssh
exit
```

Sama tee R0 ja R2 peal.

Kontroll:

```cisco
show ip ssh
```

Test Packet Traceris PC-st:

```text
ssh -l admin 192.168.1.1
```

või:

```text
ssh -l admin 192.168.1.3
```

---

# 19. SSH seadistamine switchides

Tee mõlemal switchil.

Näide S1 kohta:

```cisco
configure terminal

ip domain-name oige.local
username admin secret cisco123
crypto key generate rsa
```

Kui küsib key size, sisesta:

```text
1024
```

Seejärel:

```cisco
ip ssh version 2

line vty 0 15
login local
transport input ssh
exit
```

Sama tee S2 peal.

Test PC-st:

```text
ssh -l admin 192.168.1.4
ssh -l admin 192.168.1.10
```

---

# 20. Konfiguratsiooni salvestamine

Igal ruuteril ja switchil:

```cisco
copy running-config startup-config
```

Või lühidalt:

```cisco
wr
```

See on kohustuslik, sest muidu kaob seadistus pärast restarti.

---

# 21. Lõplik kontrollnimekiri

## Ruuterite liidesed

Igal ruuteril:

```cisco
show ip interface brief
```

Kõik vajalikud liidesed peavad olema:

```text
up/up
```

---

## Switchide liidesed

Igal switchil:

```cisco
show ip interface brief
show interfaces status
```

VLAN1 peab olema üleval.

---

## Routing

Igal ruuteril:

```cisco
show ip route
```

Kontrolli, et vajalikud võrgud on teada.

---

## Pingid PC0-st

```text
ping 192.168.1.32
ping 192.168.1.254
ping 25.200.225.1
```

---

## Pingid PC1-st

```text
ping 192.168.1.31
ping 192.168.1.254
ping 25.200.225.1
```

---

## Pingid R1-st

```cisco
ping 10.1.1.2
ping 10.2.2.1
ping 25.200.225.1
```

---

## Pingid R2-st

```cisco
ping 10.2.2.2
ping 10.1.1.1
ping 25.200.225.1
```

---

## HSRP

R1 ja R2 peal:

```cisco
show standby brief
```

Oodatav:

```text
üks ruuter Active
teine ruuter Standby
virtuaalne IP olemas
```

---

## EtherChannel

S1 ja S2 peal:

```cisco
show etherchannel summary
```

Oodatav:

```text
Po1(SU)
füüsilised pordid tähisega (P)
```

---

## SSH

Ruuterites ja switchides:

```cisco
show ip ssh
```

PC-st:

```text
ssh -l admin 192.168.1.1
ssh -l admin 192.168.1.3
ssh -l admin 192.168.1.4
ssh -l admin 192.168.1.10
```

---

## Salvestatud konfiguratsioon

Igal seadmel:

```cisco
show startup-config
```

Või salvesta uuesti:

```cisco
copy running-config startup-config
```

---

# 22. Dokumentatsiooni struktuur

Dokumentatsioonis võiks olla järgmised osad.

## 1. Topoloogia

Kirjelda:

```text
Võrgus on kolm ruuterit R0, R1 ja R2.
R1 ja R2 ühendavad LAN võrku.
R0 ühendab R1 ja R2 serial linkidega.
S1 ja S2 ühendavad klientarvuteid ja ruutereid.
S1 ja S2 vahel on EtherChannel.
```

---

## 2. IP-aadressiplaan

Lisa tabel:

| Seade | Liides | IP |
|---|---|---|
| R1 | G0/1 | `192.168.1.1/24` |
| R1 | S0/1/1 | `10.1.1.1/30` |
| R2 | G0/1 | `192.168.1.3/24` |
| R2 | S0/1/0 | `10.2.2.1/30` |
| R0 | S0/1/0 | `10.1.1.2/30` |
| R0 | S0/1/1 | `10.2.2.2/30` |
| R0 | Loopback 1 | `25.200.225.1/27` |
| S1 | VLAN1 | `192.168.1.4/24` |
| S2 | VLAN1 | `192.168.1.10/24` |
| PC0 | NIC | `192.168.1.31/24` |
| PC1 | NIC | `192.168.1.32/24` |
| HSRP | Virtual IP | `192.168.1.254` |

---

## 3. Ruuterite seadistus

Kirjuta:

```text
Ruuteritele määrati hostinimed, paroolid, banner, liideste IP-aadressid ja staatilised marsruudid.
```

Lisa kontrollkäsud:

```cisco
show ip interface brief
show ip route
```

---

## 4. Switchide seadistus

Kirjuta:

```text
Switchidele määrati hostinimed, haldus-IP aadressid VLAN1 liidesele, default gateway ja SSH ligipääs.
```

Lisa kontrollkäsud:

```cisco
show ip interface brief
show interfaces status
```

---

## 5. HSRP

Kirjuta:

```text
R1 ja R2 vahel seadistati HSRP.
Virtuaalseks gateway aadressiks määrati 192.168.1.254.
R1 seadistati kõrgema prioriteediga aktiivseks ruuteriks.
R2 jäi standby ruuteriks.
```

Kontroll:

```cisco
show standby brief
```

---

## 6. EtherChannel

Kirjuta:

```text
S1 ja S2 vahelised füüsilised lingid ühendati LACP abil EtherChannelisse.
Loodi Port-channel 1.
```

Kontroll:

```cisco
show etherchannel summary
```

---

## 7. SSH

Kirjuta:

```text
Kõikidel ruuteritel ja switchidel seadistati SSH ligipääs.
Loodi kohalik admin kasutaja ja lubati VTY liinidel ainult SSH.
```

Kontroll:

```cisco
show ip ssh
```

---

## 8. Testimine

Lisa pingide tulemused:

```text
PC0 → PC1
PC0 → HSRP gateway
PC1 → HSRP gateway
PC0 → R0 loopback
PC1 → R0 loopback
```

---

# 23. Tüüpilised probleemid ja lahendused

## Probleem: Serial link on down/down

Kontrolli:

```cisco
show ip interface brief
```

Võimalikud põhjused:

```text
vale kaabel
vale liides
no shutdown tegemata
```

Lahendus:

```cisco
interface s0/1/0
no shutdown
```

---

## Probleem: Serial link on up/down

Võimalikud põhjused:

```text
DCE poolel puudub clock rate
vale IP-aadress või mask
teisel poolel liides down
```

Kontrolli DCE poolt:

```cisco
show controllers serial 0/1/0
```

Kui liides on DCE pool:

```cisco
interface s0/1/0
clock rate 64000
```

---

## Probleem: PC ei saa gatewayd pingida

Kontrolli PC-s:

```text
IP address
Subnet mask
Default gateway
```

Kontrolli ruuteris:

```cisco
show ip interface brief
show standby brief
```

PC gateway peab olema:

```text
192.168.1.254
```

---

## Probleem: HSRP ei tööta

Kontroll:

```cisco
show standby brief
```

Võimalikud põhjused:

```text
R1 ja R2 LAN liidesed ei ole samas võrgus
vale virtuaalne IP
liides on shutdown
HSRP group number erinev
```

Õige näide:

```cisco
interface g0/1
standby 1 ip 192.168.1.254
```

Mõlemal ruuteril peab group number olema sama:

```text
standby 1
```

---

## Probleem: EtherChannel ei moodustu

Kontroll:

```cisco
show etherchannel summary
```

Võimalikud põhjused:

```text
ühel pool vale port
üks pool trunk, teine access
üks pool LACP active, teine vale mode
kiirused/duplex erinevad
```

Paranda mõlemal switchil samamoodi:

```cisco
interface range fa0/1 - 2
switchport mode trunk
channel-group 1 mode active
```

---

## Probleem: SSH ei tööta

Kontroll:

```cisco
show ip ssh
show running-config | section line vty
```

SSH jaoks peavad olemas olema:

```text
hostname
ip domain-name
username
RSA key
ip ssh version 2
line vty login local
transport input ssh
```

Näide:

```cisco
ip domain-name oige.local
username admin secret cisco123
crypto key generate rsa
ip ssh version 2

line vty 0 4
login local
transport input ssh
```

---

## Probleem: Switchi VLAN1 on down

Kontroll:

```cisco
show ip interface brief
show interfaces status
```

Võimalikud põhjused:

```text
ühtki aktiivset porti VLAN1-s ei ole
kaabel puudub
port shutdown
```

Lahendus:

```cisco
interface vlan 1
no shutdown
```

ja veendu, et vähemalt üks VLAN1 port on aktiivne.

---

## Probleem: Pärast restarti seadistus kadus

Põhjus:

```text
running-config jäi startup-configi salvestamata
```

Lahendus igal seadmel:

```cisco
copy running-config startup-config
```

või:

```cisco
wr
```

---

# 24. Väga lühike kokkuvõte

Selle pileti lahendus kõige lihtsamalt:

```text
1. Panen ruuteritele ja switchidele nimed.
2. Seadistan paroolid, banneri ja no ip domain-lookup.
3. Panen ruuterite liidestele IP-aadressid.
4. Lülitan liidesed sisse käsuga no shutdown.
5. Panen PC-dele IP-d ja gatewayks HSRP virtuaalse IP.
6. Panen switchidele VLAN1 haldus-IP-d.
7. Seadistan staatilised marsruudid.
8. Seadistan R1 ja R2 vahel HSRP.
9. Seadistan S1 ja S2 vahel EtherChanneli.
10. Seadistan SSH ruuteritel ja switchidel.
11. Testin pingidega ühendust.
12. Kontrollin HSRP ja EtherChanneli käsuga show.
13. Salvestan kõik konfiguratsioonid.
14. Dokumenteerin kogu protsessi.
```
