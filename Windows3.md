# Windows pilet 3 lahenduskäik

See juhend kirjeldab ainult **Windows pilet 3 eriosa**.

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

## 1. Windows pilet 3 eesmärk

Windows pilet 3 põhiteemad on PowerShell skriptid.

| Teema | Mida tuleb teha | Milleks seda vaja on |
|---|---|---|
| AD kontode skript | Luua skript, mis koostab raporti AD kontodest | Et tuvastada probleemsed või kasutamata kontod |
| Mitte kunagi loginud kontod | Leida kontod, mis pole kunagi AD domeeni sisse loginud | Turvalisuse ja korrastamise jaoks |
| Keelatud kontod | Leida disabled kasutajakontod | Et admin näeks mitteaktiivseid kontosid |
| Aegunud kontod | Leida aegunud kasutajakontod | Et tuvastada kontod, mida ei tohiks enam kasutada |
| Lukustatud kontod | Testimiseks lukustada mõned kontod ja kuvada need raportis | Konto lukustuspoliitika kontrolliks |
| DHCP raport | Luua skript DHCP serveri kohta | Võrgu aadressihalduse kontrolliks |
| DHCP scope’id | Kuvada aktiivsed DHCP scope’id | Et näha, millised IP-vahemikud töötavad |
| DHCP lease’id | Kuvada lease’id koos MAC-aadressidega | Et näha, millised seadmed on IP saanud |
| Vabad IP-d | Kuvada DHCP scope’ides vabad IP-aadressid | Et näha, kui palju aadresse on veel saadaval |
| Dokumentatsioon | Kirjeldada skriptid, käsud ja tulemused | Et tõendada lahenduse toimimist |

---

# 2. Vajalikud moodulid ja tööriistad

## 2.1 Vajalikud PowerShell moodulid

Skriptid käivita DC1 serveris administraatoriõigustega PowerShellis.

Vajalikud moodulid:

| Moodul | Milleks vajalik |
|---|---|
| `ActiveDirectory` | AD kasutajate ja kontode info pärimiseks |
| `DhcpServer` | DHCP scope’ide, lease’ide ja vabade IP-de pärimiseks |

Kontrolli mooduleid:

```powershell
Get-Module -ListAvailable ActiveDirectory
Get-Module -ListAvailable DhcpServer
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Mõlemad moodulid kuvatakse | Kui moodul puudub, kontrolli, et AD DS ja DHCP haldustööriistad on paigaldatud |

Vajadusel impordi moodulid:

```powershell
Import-Module ActiveDirectory
Import-Module DhcpServer
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Laeb AD ja DHCP PowerShell moodulid aktiivsesse sessiooni | Käsud `Get-ADUser` ja `Get-DhcpServerv4Scope` töötavad |

---

## 2.2 Kontrolli, et põhikäsud töötavad

```powershell
Get-ADDomain
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse domeeni info | Kui tuleb error, ei tööta AD moodul või server pole DC/domeeniga seotud |

```powershell
Get-ADUser -Filter * -ResultSetSize 5
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse mõned AD kasutajad | Kui kasutajaid ei kuvata, kontrolli AD struktuuri ja õiguseid |

```powershell
Get-DhcpServerv4Scope
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse DHCP scope’id | Kui tuleb error, kontrolli DHCP rolli ja õiguseid |

---

# 3. Loo skriptide kaust

## 3.1 Loo kaust

DC1 PowerShellis administraatorina:

```powershell
New-Item -ItemType Directory -Path "C:\Scripts" -Force
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob skriptide jaoks kausta | Kaust `C:\Scripts` on olemas |

Kontroll:

```powershell
Test-Path "C:\Scripts"
```

Oodatav:

```text
True
```

---

## 3.2 Loo raportite kaust

```powershell
New-Item -ItemType Directory -Path "C:\Scripts\Reports" -Force
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Loob raportite salvestamise kausta | Kaust `C:\Scripts\Reports` on olemas |

Kontroll:

```powershell
Test-Path "C:\Scripts\Reports"
```

Oodatav:

```text
True
```

---

# 4. Testimiseks lukusta mõned AD kontod

Piletis on märgitud, et testimiseks võib ise mõned kontod lukustada.

## 4.1 Miks seda teha?

Kui kontolukustuse poliitika on üldosas seadistatud, siis pärast vale parooli korduvat sisestamist lukustub kasutaja konto.

Skript saab lukustatud kontosid kuvada ainult siis, kui mõni konto on päriselt lukus.

---

## 4.2 Lukusta konto testimiseks

Lihtsaim GUI meetod:

```text
Windows 11 klient
→ proovi domeeni kasutajaga mitu korda vale parooliga sisse logida
→ konto lukustub pärast 5 vale katset
```

Seejärel DC1-s kontrolli:

```powershell
Search-ADAccount -LockedOut -UsersOnly
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse lukustatud kontod | Kui midagi ei kuvata, pole konto lukus või GPO pole rakendunud |

Märkus:

```text
Ära testi lukustamist Haldur või Administrator kontoga.
Kasuta tavalist testkasutajat.
```

---

# 5. AD kontode raporti skript

## 5.1 Mida skript peab tegema?

Skript peab koostama raporti AD kontodest:

| Kontroll | Selgitus |
|---|---|
| Kontod, mis pole kunagi AD domeeni loginud | `LastLogonDate` puudub |
| Keelatud kontod | `Enabled = False` |
| Aegunud kontod | `AccountExpirationDate` on minevikus |
| Lukustatud kontod | Konto on locked out |
| Kokkuvõte | Mitu kontot igas kategoorias oli |
| CSV väljund | Raport salvestatakse failina |

---

## 5.2 Loo skript

Ava Notepad või PowerShell ISE administraatorina.

Fail:

```text
C:\Scripts\AD-Kontode-Raport.ps1
```

PowerShelliga:

```powershell
notepad C:\Scripts\AD-Kontode-Raport.ps1
```

Lisa skripti sisu:

```powershell
# AD-Kontode-Raport.ps1
# Skript koostab raporti AD kasutajakontodest:
# - kontod, mis pole kunagi sisse loginud
# - keelatud kontod
# - aegunud kontod
# - lukustatud kontod

Import-Module ActiveDirectory

$ReportPath = "C:\Scripts\Reports"
$Date = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

New-Item -ItemType Directory -Path $ReportPath -Force | Out-Null

Write-Host "Koostan AD kontode raportit..." -ForegroundColor Cyan

# Kõik AD kasutajad vajalike omadustega
$AllUsers = Get-ADUser -Filter * -Properties `
    SamAccountName, `
    Name, `
    Enabled, `
    LastLogonDate, `
    AccountExpirationDate, `
    LockedOut, `
    DistinguishedName

# Kontod, mis pole kunagi sisse loginud
$NeverLoggedIn = $AllUsers | Where-Object {
    $null -eq $_.LastLogonDate
} | Select-Object Name, SamAccountName, Enabled, LastLogonDate, DistinguishedName

# Keelatud kontod
$DisabledUsers = $AllUsers | Where-Object {
    $_.Enabled -eq $false
} | Select-Object Name, SamAccountName, Enabled, DistinguishedName

# Aegunud kontod
$ExpiredUsers = $AllUsers | Where-Object {
    $_.AccountExpirationDate -ne $null -and $_.AccountExpirationDate -lt (Get-Date)
} | Select-Object Name, SamAccountName, Enabled, AccountExpirationDate, DistinguishedName

# Lukustatud kontod
$LockedOutUsers = Search-ADAccount -LockedOut -UsersOnly | Select-Object Name, SamAccountName, DistinguishedName

# CSV raportid
$NeverLoggedIn | Export-Csv "$ReportPath\AD_MitteKunagiLoginud_$Date.csv" -NoTypeInformation -Encoding UTF8
$DisabledUsers | Export-Csv "$ReportPath\AD_KeelatudKontod_$Date.csv" -NoTypeInformation -Encoding UTF8
$ExpiredUsers | Export-Csv "$ReportPath\AD_AegunudKontod_$Date.csv" -NoTypeInformation -Encoding UTF8
$LockedOutUsers | Export-Csv "$ReportPath\AD_LukustatudKontod_$Date.csv" -NoTypeInformation -Encoding UTF8

# Ekraaniraport
Write-Host ""
Write-Host "AD kontode raport valmis." -ForegroundColor Green
Write-Host "Kokku kasutajaid: $($AllUsers.Count)"
Write-Host "Mitte kunagi loginud kontosid: $($NeverLoggedIn.Count)"
Write-Host "Keelatud kontosid: $($DisabledUsers.Count)"
Write-Host "Aegunud kontosid: $($ExpiredUsers.Count)"
Write-Host "Lukustatud kontosid: $($LockedOutUsers.Count)"
Write-Host ""
Write-Host "Raportid salvestati kausta: $ReportPath" -ForegroundColor Yellow

# Kuva tabelina ka ekraanile
Write-Host "`n--- Mitte kunagi loginud kontod ---" -ForegroundColor Cyan
$NeverLoggedIn | Format-Table Name, SamAccountName, Enabled -AutoSize

Write-Host "`n--- Keelatud kontod ---" -ForegroundColor Cyan
$DisabledUsers | Format-Table Name, SamAccountName, Enabled -AutoSize

Write-Host "`n--- Aegunud kontod ---" -ForegroundColor Cyan
$ExpiredUsers | Format-Table Name, SamAccountName, AccountExpirationDate -AutoSize

Write-Host "`n--- Lukustatud kontod ---" -ForegroundColor Cyan
$LockedOutUsers | Format-Table Name, SamAccountName -AutoSize
```

---

## 5.3 Käivita AD raporti skript

PowerShell administraatorina:

```powershell
Set-ExecutionPolicy RemoteSigned -Scope Process
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Lubab skripti käivitamise ainult selles PowerShelli aknas | Käsk ei muuda püsivalt kogu serveri poliitikat |

Käivita skript:

```powershell
C:\Scripts\AD-Kontode-Raport.ps1
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ekraanile kuvatakse kokkuvõte ja tabelid | Kui tuleb execution policy viga, käivita `Set-ExecutionPolicy RemoteSigned -Scope Process` |

---

## 5.4 Kontrolli AD raportifaile

```powershell
Get-ChildItem C:\Scripts\Reports
```

Oodatav:

```text
AD_MitteKunagiLoginud_*.csv
AD_KeelatudKontod_*.csv
AD_AegunudKontod_*.csv
AD_LukustatudKontod_*.csv
```

Ava üks raport:

```powershell
Import-Csv (Get-ChildItem C:\Scripts\Reports\AD_KeelatudKontod_*.csv | Select-Object -Last 1).FullName
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| CSV sisu kuvatakse PowerShellis | Kui fail puudub, skript ei salvestanud raportit õigesse kausta |

---

# 6. DHCP raporti skript

## 6.1 Mida skript peab tegema?

Skript peab koostama raporti DHCP serverist:

| Kontroll | Selgitus |
|---|---|
| Aktiivsed scope’id | DHCP IPv4 scope’id |
| Lease’id | IP-aadressid, kliendi nimed ja MAC-aadressid |
| Reservation’id | Staatilised DHCP rendid |
| Vabad IP-d | DHCP serveri vabad aadressid scope’ide kaupa |
| Kokkuvõte | Mitu lease’i, reservation’it ja vaba IP-d on |
| CSV väljund | Raport salvestatakse failidena |

---

## 6.2 Kontrolli DHCP serveri nime

Kui skript käivitatakse DC1-s ja DHCP on DC1-s, võib kasutada:

```powershell
$DhcpServer = $env:COMPUTERNAME
```

Kontroll:

```powershell
Get-DhcpServerv4Scope -ComputerName $env:COMPUTERNAME
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse DHCP scope’id | Kui error ütleb, et DHCP serverit ei leita, kasuta serveri FQDN-i või kontrolli DHCP rolli |

Näide FQDN-iga:

```powershell
Get-DhcpServerv4Scope -ComputerName "DC1.sinuNimi.local"
```

---

## 6.3 Loo DHCP raporti skript

Fail:

```text
C:\Scripts\DHCP-Raport.ps1
```

Ava:

```powershell
notepad C:\Scripts\DHCP-Raport.ps1
```

Lisa skripti sisu:

```powershell
# DHCP-Raport.ps1
# Skript koostab raporti DHCP serveri IPv4 scope'idest,
# lease'idest, reservation'itest ja vabadest IP-aadressidest.

Import-Module DhcpServer

$DhcpServer = $env:COMPUTERNAME
$ReportPath = "C:\Scripts\Reports"
$Date = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

New-Item -ItemType Directory -Path $ReportPath -Force | Out-Null

Write-Host "Koostan DHCP raportit serverist $DhcpServer ..." -ForegroundColor Cyan

# DHCP scope'id
$Scopes = Get-DhcpServerv4Scope -ComputerName $DhcpServer

# Salvestame scope'id CSV-sse
$Scopes | Select-Object `
    ScopeId, `
    Name, `
    State, `
    StartRange, `
    EndRange, `
    SubnetMask, `
    LeaseDuration |
Export-Csv "$ReportPath\DHCP_Scopeid_$Date.csv" -NoTypeInformation -Encoding UTF8

# Tühjad kogumid raportite jaoks
$AllLeases = @()
$AllReservations = @()
$AllFreeIPs = @()
$ScopeStats = @()

foreach ($Scope in $Scopes) {
    $ScopeId = $Scope.ScopeId

    Write-Host "Töötlen scope'i $ScopeId ..." -ForegroundColor Yellow

    # Lease'id
    $Leases = Get-DhcpServerv4Lease -ComputerName $DhcpServer -ScopeId $ScopeId -ErrorAction SilentlyContinue

    foreach ($Lease in $Leases) {
        $AllLeases += [PSCustomObject]@{
            ScopeId       = $ScopeId
            IPAddress     = $Lease.IPAddress
            HostName      = $Lease.HostName
            ClientId      = $Lease.ClientId
            AddressState  = $Lease.AddressState
            LeaseExpiry   = $Lease.LeaseExpiryTime
        }
    }

    # Reservation'id
    $Reservations = Get-DhcpServerv4Reservation -ComputerName $DhcpServer -ScopeId $ScopeId -ErrorAction SilentlyContinue

    foreach ($Reservation in $Reservations) {
        $AllReservations += [PSCustomObject]@{
            ScopeId      = $ScopeId
            IPAddress    = $Reservation.IPAddress
            Name         = $Reservation.Name
            ClientId     = $Reservation.ClientId
            Description  = $Reservation.Description
        }
    }

    # Vabad IP-aadressid
    $FreeIPs = Get-DhcpServerv4FreeIPAddress -ComputerName $DhcpServer -ScopeId $ScopeId -NumAddress 20 -ErrorAction SilentlyContinue

    foreach ($FreeIP in $FreeIPs) {
        $AllFreeIPs += [PSCustomObject]@{
            ScopeId   = $ScopeId
            FreeIP    = $FreeIP
        }
    }

    # Scope statistika
    $Stats = Get-DhcpServerv4ScopeStatistics -ComputerName $DhcpServer -ScopeId $ScopeId -ErrorAction SilentlyContinue

    $ScopeStats += [PSCustomObject]@{
        ScopeId             = $ScopeId
        Name                = $Scope.Name
        State               = $Scope.State
        InUse               = $Stats.InUse
        Free                = $Stats.Free
        PercentageInUse     = $Stats.PercentageInUse
        Reserved            = $Reservations.Count
        ActiveLeases        = $Leases.Count
    }
}

# Ekspordi CSV failid
$AllLeases | Export-Csv "$ReportPath\DHCP_Leaseid_$Date.csv" -NoTypeInformation -Encoding UTF8
$AllReservations | Export-Csv "$ReportPath\DHCP_Reservationid_$Date.csv" -NoTypeInformation -Encoding UTF8
$AllFreeIPs | Export-Csv "$ReportPath\DHCP_Vabad_IPd_$Date.csv" -NoTypeInformation -Encoding UTF8
$ScopeStats | Export-Csv "$ReportPath\DHCP_Statistika_$Date.csv" -NoTypeInformation -Encoding UTF8

# Ekraanile kokkuvõte
Write-Host ""
Write-Host "DHCP raport valmis." -ForegroundColor Green
Write-Host "DHCP server: $DhcpServer"
Write-Host "Scope'e kokku: $($Scopes.Count)"
Write-Host "Lease'e kokku: $($AllLeases.Count)"
Write-Host "Reservation'e kokku: $($AllReservations.Count)"
Write-Host "Raportis kuvatud vabu IP-sid: $($AllFreeIPs.Count)"
Write-Host "Raportid salvestati kausta: $ReportPath" -ForegroundColor Yellow

Write-Host "`n--- DHCP scope'id ---" -ForegroundColor Cyan
$Scopes | Format-Table ScopeId, Name, State, StartRange, EndRange, LeaseDuration -AutoSize

Write-Host "`n--- DHCP statistika ---" -ForegroundColor Cyan
$ScopeStats | Format-Table ScopeId, Name, InUse, Free, PercentageInUse, Reserved, ActiveLeases -AutoSize

Write-Host "`n--- DHCP lease'id koos MAC-aadressidega ---" -ForegroundColor Cyan
$AllLeases | Format-Table ScopeId, IPAddress, HostName, ClientId, AddressState -AutoSize

Write-Host "`n--- DHCP reservation'id ---" -ForegroundColor Cyan
$AllReservations | Format-Table ScopeId, IPAddress, Name, ClientId -AutoSize

Write-Host "`n--- Näidis vabadest IP-aadressidest ---" -ForegroundColor Cyan
$AllFreeIPs | Format-Table ScopeId, FreeIP -AutoSize
```

---

## 6.4 Käivita DHCP raporti skript

```powershell
Set-ExecutionPolicy RemoteSigned -Scope Process
```

Käivita:

```powershell
C:\Scripts\DHCP-Raport.ps1
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Ekraanile kuvatakse DHCP scope’id, lease’id, reservation’id ja vabad IP-d | Kui tuleb õiguste viga, käivita PowerShell administraatorina |

---

## 6.5 Kontrolli DHCP raportifaile

```powershell
Get-ChildItem C:\Scripts\Reports\DHCP_*.csv
```

Oodatav:

```text
DHCP_Scopeid_*.csv
DHCP_Leaseid_*.csv
DHCP_Reservationid_*.csv
DHCP_Vabad_IPd_*.csv
DHCP_Statistika_*.csv
```

Ava lease’i raport:

```powershell
Import-Csv (Get-ChildItem C:\Scripts\Reports\DHCP_Leaseid_*.csv | Select-Object -Last 1).FullName
```

| Oodatav tulemus | Kui tulemus on teine |
|---|---|
| Kuvatakse lease’id koos IP ja MAC-aadressiga | Kui raport on tühi, pole klientidel aktiivseid DHCP lease’e |

---

# 7. Ühisskript mõlema raporti käivitamiseks

Soovi korral loo üks skript, mis käivitab mõlemad raportid järjest.

Fail:

```text
C:\Scripts\Koosta-Koik-Raportid.ps1
```

Ava:

```powershell
notepad C:\Scripts\Koosta-Koik-Raportid.ps1
```

Lisa:

```powershell
# Koosta-Koik-Raportid.ps1
# Käivitab AD kontode raporti ja DHCP raporti.

Write-Host "Käivitan AD kontode raporti..." -ForegroundColor Cyan
& "C:\Scripts\AD-Kontode-Raport.ps1"

Write-Host "`nKäivitan DHCP raporti..." -ForegroundColor Cyan
& "C:\Scripts\DHCP-Raport.ps1"

Write-Host "`nKõik raportid on valmis." -ForegroundColor Green
Write-Host "Raportid asuvad kaustas C:\Scripts\Reports"
```

Käivita:

```powershell
C:\Scripts\Koosta-Koik-Raportid.ps1
```

| Mida see teeb | Oodatav tulemus |
|---|---|
| Käivitab mõlemad skriptid järjest | C:\Scripts\Reports kausta tekivad AD ja DHCP raportid |

---

# 8. Skriptide tulemuste kontroll

## 8.1 AD raporti kontroll

```powershell
Get-ChildItem C:\Scripts\Reports\AD_*.csv
```

Oodatav:

```text
AD raportite CSV failid on olemas
```

Kontrolli lukustatud kontosid:

```powershell
Search-ADAccount -LockedOut -UsersOnly
```

Oodatav:

```text
Kui testkonto on lukustatud, kuvatakse see
```

Kontrolli disabled kontosid:

```powershell
Get-ADUser -Filter {Enabled -eq $false}
```

Oodatav:

```text
Kuvatakse keelatud kontod
```

---

## 8.2 DHCP raporti kontroll

```powershell
Get-DhcpServerv4Scope
```

Oodatav:

```text
Aktiivsed scope’id kuvatakse
```

```powershell
Get-DhcpServerv4Lease -ScopeId SCOPE_ID
```

Näide:

```powershell
Get-DhcpServerv4Lease -ScopeId 10.0.0.0
```

Oodatav:

```text
Kuvatakse lease’id koos ClientId ehk MAC-aadressiga
```

```powershell
Get-DhcpServerv4FreeIPAddress -ScopeId SCOPE_ID -NumAddress 10
```

Näide:

```powershell
Get-DhcpServerv4FreeIPAddress -ScopeId 10.0.0.0 -NumAddress 10
```

Oodatav:

```text
Kuvatakse vabad IP-aadressid
```

---

# 9. Lõppkontroll

| Kontroll | Käsk | Oodatav tulemus |
|---|---|---|
| AD moodul | `Get-Module -ListAvailable ActiveDirectory` | Moodul on olemas |
| DHCP moodul | `Get-Module -ListAvailable DhcpServer` | Moodul on olemas |
| AD kasutajad | `Get-ADUser -Filter *` | Kasutajad kuvatakse |
| Lukustatud kontod | `Search-ADAccount -LockedOut -UsersOnly` | Testlukustatud kontod kuvatakse |
| DHCP scope’id | `Get-DhcpServerv4Scope` | Scope’id kuvatakse |
| DHCP lease’id | `Get-DhcpServerv4Lease -ScopeId ...` | Lease’id ja MAC-id kuvatakse |
| Vabad IP-d | `Get-DhcpServerv4FreeIPAddress -ScopeId ...` | Vabad IP-d kuvatakse |
| AD skript | `C:\Scripts\AD-Kontode-Raport.ps1` | AD raportid tekivad |
| DHCP skript | `C:\Scripts\DHCP-Raport.ps1` | DHCP raportid tekivad |
| Raportid | `Get-ChildItem C:\Scripts\Reports` | CSV failid on olemas |

---

# 10. Dokumentatsiooni näidis

```markdown
## Windows pilet 3 dokumentatsioon

### Eesmärk

Eesmärk oli luua PowerShell skriptid, mis koostavad raportid Active Directory kasutajakontode ja DHCP serveri kohta. AD raport kuvab kontod, mis pole kunagi sisse loginud, keelatud kontod, aegunud kontod ja lukustatud kontod. DHCP raport kuvab aktiivsed scope’id, DHCP lease’id koos MAC-aadressidega, reservation’id ja vabad IP-aadressid.

### Kasutatud serverid

| Server | Roll |
|---|---|
| DC1 | AD DS, DNS, DHCP, PowerShell skriptide käivitamine |
| DC2 | Teine domeenikontroller ja DHCP failover partner |
| Windows 11 klient | Testkonto lukustamise ja DHCP lease’i testimiseks |

### Loodud skriptid

| Skript | Asukoht | Milleks |
|---|---|---|
| `AD-Kontode-Raport.ps1` | `C:\Scripts` | AD kontode raport |
| `DHCP-Raport.ps1` | `C:\Scripts` | DHCP serveri raport |
| `Koosta-Koik-Raportid.ps1` | `C:\Scripts` | Käivitab mõlemad raportid |

### Tehtud tegevused

- Kontrollisin ActiveDirectory ja DhcpServer PowerShell moodulite olemasolu.
- Lõin kausta `C:\Scripts`.
- Lõin raportite kausta `C:\Scripts\Reports`.
- Lõin AD kontode raporti skripti.
- Lisasin AD skripti kontrollid mitte kunagi loginud, keelatud, aegunud ja lukustatud kontode jaoks.
- Lukustasin testimiseks ühe tavalise kasutajakonto.
- Lõin DHCP raporti skripti.
- Lisasin DHCP skripti kontrollid scope’ide, lease’ide, reservation’ite ja vabade IP-de jaoks.
- Käivitasin mõlemad skriptid administraatoriõigustega PowerShellis.
- Kontrollisin, et CSV raportid tekkisid.

### Kontrollid

| Kontroll | Tulemus |
|---|---|
| `Get-ADUser -Filter *` | AD kasutajad kuvatakse |
| `Search-ADAccount -LockedOut -UsersOnly` | Lukustatud testkonto kuvatakse |
| `Get-DhcpServerv4Scope` | DHCP scope’id kuvatakse |
| `Get-DhcpServerv4Lease` | Lease’id ja MAC-aadressid kuvatakse |
| `Get-DhcpServerv4FreeIPAddress` | Vabad IP-d kuvatakse |
| `C:\Scripts\AD-Kontode-Raport.ps1` | AD CSV raportid tekkisid |
| `C:\Scripts\DHCP-Raport.ps1` | DHCP CSV raportid tekkisid |

### Kokkuvõte

Windows pilet 3 tulemusena valmisid PowerShell skriptid AD kontode ja DHCP serveri raportite koostamiseks. AD skript aitab tuvastada mitteaktiivseid, keelatud, aegunud ja lukustatud kontosid. DHCP skript kuvab aktiivsed scope’id, lease’id koos MAC-aadressidega, reservation’id ja vabad IP-aadressid. Raportid salvestatakse CSV failidena kausta `C:\Scripts\Reports`.
```

---

# 11. Troubleshooting

## ActiveDirectory moodul puudub

Kontrolli:

```powershell
Get-Module -ListAvailable ActiveDirectory
```

Kui moodulit pole, paigalda RSAT / AD tööriistad:

```powershell
Install-WindowsFeature RSAT-AD-PowerShell
```

Seejärel:

```powershell
Import-Module ActiveDirectory
```

---

## DhcpServer moodul puudub

Kontrolli:

```powershell
Get-Module -ListAvailable DhcpServer
```

Kui moodulit pole, paigalda DHCP haldustööriistad:

```powershell
Install-WindowsFeature RSAT-DHCP
```

või DHCP serveri roll koos tööriistadega:

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

---

## Skripti ei lubata käivitada

Kui tuleb execution policy error, käivita:

```powershell
Set-ExecutionPolicy RemoteSigned -Scope Process
```

Seejärel käivita skript uuesti:

```powershell
C:\Scripts\AD-Kontode-Raport.ps1
```

---

## DHCP käsk annab errori

Kontrolli, kas DHCP server töötab:

```powershell
Get-Service DHCPServer
```

Kui teenus ei tööta:

```powershell
Start-Service DHCPServer
```

Kontrolli scope’i:

```powershell
Get-DhcpServerv4Scope
```

Kui DHCP server on teises masinas, määra skriptis:

```powershell
$DhcpServer = "DC1.sinuNimi.local"
```

---

## DHCP lease’i raport on tühi

Põhjused:

| Põhjus | Lahendus |
|---|---|
| Klient pole DHCP-st IP-d küsinud | Tee kliendis `ipconfig /release` ja `ipconfig /renew` |
| Scope pole aktiivne | Kontrolli DHCP Manageris või `Get-DhcpServerv4Scope` |
| Vale scope ID | Kontrolli scope ID-d käsuga `Get-DhcpServerv4Scope` |
| DHCP töötab teises serveris | Muuda `$DhcpServer` väärtust skriptis |

---

## Lukustatud kontosid ei kuvata

Kontroll:

```powershell
Search-ADAccount -LockedOut -UsersOnly
```

Kui midagi ei kuvata:

| Põhjus | Lahendus |
|---|---|
| Ühtegi kontot pole lukus | Lukusta testkasutaja vale parooliga |
| GPO pole rakendunud | Kontrolli kontolukustuse poliitikat |
| Testid admin kontoga | Kasuta tavalist testkasutajat |

---

## Aegunud kontosid ei kuvata

Kui sul pole aegunud kontosid, saad testimiseks määrata ühele testkontole aegumise:

```powershell
Set-ADAccountExpiration -Identity "testkasutaja" -DateTime (Get-Date).AddDays(-1)
```

Kontroll:

```powershell
Get-ADUser testkasutaja -Properties AccountExpirationDate
```

Kui tahad hiljem aegumise eemaldada:

```powershell
Clear-ADAccountExpiration -Identity "testkasutaja"
```

---

## Keelatud kontosid ei kuvata

Testimiseks keela üks testkonto:

```powershell
Disable-ADAccount -Identity "testkasutaja"
```

Kontroll:

```powershell
Get-ADUser testkasutaja -Properties Enabled
```

Kui soovid konto tagasi lubada:

```powershell
Enable-ADAccount -Identity "testkasutaja"
```

---

# 12. Kõige lühem spikker

```text
1. Ava DC1 PowerShell administraatorina
2. Kontrolli mooduleid ActiveDirectory ja DhcpServer
3. Loo C:\Scripts ja C:\Scripts\Reports
4. Lukusta testimiseks üks tavaline AD kasutaja
5. Loo AD-Kontode-Raport.ps1
6. AD skript kuvab:
   - mitte kunagi loginud kontod
   - keelatud kontod
   - aegunud kontod
   - lukustatud kontod
7. Käivita AD skript
8. Kontrolli AD CSV raporteid
9. Loo DHCP-Raport.ps1
10. DHCP skript kuvab:
    - aktiivsed scope’id
    - lease’id koos MAC-aadressidega
    - reservation’id
    - vabad IP-aadressid
11. Käivita DHCP skript
12. Kontrolli DHCP CSV raporteid
13. Vajadusel loo Koosta-Koik-Raportid.ps1
14. Dokumenteeri skriptid ja kontrollide tulemused
```
