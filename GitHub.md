# GitHubi kasutamise spikker seadmes või VM-is

See juhend näitab, kuidas paigaldada Git, seadistada Git uues seadmes või virtuaalmasinas ning kuidas oma projekt GitHubi üles laadida.

---

## 1. Laadi alla ja paigalda Git

Git on programm, mille abil saab faile versioonihallata ja GitHubi üles laadida.

### Windows

Mine Git ametlikule lehele:

```text
https://git-scm.com/downloads
```

Vali Windowsi versioon ja laadi installer alla.

Paigaldamisel võib enamasti vajutada `Next`, `Next`, `Next`, kuni paigaldus on valmis.

Pärast paigaldamist ava:

```text
Git Bash
```

või kasuta Windows PowerShelli.

---

### Linux / Ubuntu

Ubuntu või Debian põhises Linuxis kasuta:

```bash
sudo apt update
sudo apt install git -y
```

---

### macOS

macOS-is saab Giti paigaldada näiteks Homebrew abil:

```bash
brew install git
```

Kui Homebrew puudub, võib Git olla paigaldatav ka käsuga:

```bash
git --version
```

Kui Git puudub, pakub macOS sageli võimalust paigaldada vajalikud Command Line Tools tööriistad.

---

## 2. Kontrolli, kas Git on olemas

Pärast paigaldamist kontrolli:

```bash
git --version
```

Kui näitab versiooni, näiteks:

```bash
git version 2.43.0
```

siis Git on olemas ja töötab.

Kui versiooni ei kuvata, tuleb Git uuesti paigaldada või terminal uuesti avada.

---

## 3. Seadista Git kasutaja selles seadmes / VM-is

Igas uues seadmes või VM-is tuleb Gitile öelda, kes sa oled.

```bash
git config --global user.name "Sinu Nimi"
git config --global user.email "sinu.email@example.com"
```

Näide:

```bash
git config --global user.name "Valter Iliste"
git config --global user.email "valter@example.com"
```

Kontrollimiseks:

```bash
git config --global --list
```

---

## 4. Mine oma projekti kausta

Näide Windowsis:

```bash
cd Desktop
cd minu-projekt
```

Näide Linuxis:

```bash
cd /home/kasutaja/minu-projekt
```

Kontrolli, kas oled õiges kohas:

```bash
ls
```

Windows PowerShellis võib kasutada ka:

```powershell
dir
```

---

## 5. Kui kaust EI OLE veel Git repo

Kui projektikaustas ei ole veel Git seadistatud, tee:

```bash
git init
```

Seejärel lisa GitHubi repo aadress.

GitHubis loo enne tühi repository, näiteks:

```text
https://github.com/kasutajanimi/reponimi.git
```

Lisa see oma projektile:

```bash
git remote add origin https://github.com/kasutajanimi/reponimi.git
```

Kontrolli:

```bash
git remote -v
```

---

## 6. Kui GitHubi repo on juba olemas ja tahad selle VM-i tõmmata

Kasuta käsku `git clone`.

```bash
git clone https://github.com/kasutajanimi/reponimi.git
```

Seejärel mine repo kausta:

```bash
cd reponimi
```

---

## 7. Tavaline töövoog: lisa, salvesta ja pushi

Kui oled faile muutnud, siis kasuta neid käske.

### Vaata muudatusi

```bash
git status
```

### Lisa kõik muudatused

```bash
git add .
```

### Tee commit

```bash
git commit -m "Uuendasin dokumentatsiooni"
```

### Pushi GitHubi

Kui branch on `main`:

```bash
git push origin main
```

Kui branch on `master`:

```bash
git push origin master
```

---

## 8. Kuidas teada, kas branch on main või master?

```bash
git branch
```

Näide:

```bash
* main
```

Siis kasuta:

```bash
git push origin main
```

Kui näitab:

```bash
* master
```

siis kasuta:

```bash
git push origin master
```

---

## 9. Esimene push uues repos

Kui oled repo alles loonud, võib vaja minna:

```bash
git branch -M main
git push -u origin main
```

Pärast seda piisab edaspidi enamasti ainult:

```bash
git push
```

---

## 10. Kui GitHub küsib parooli

GitHub ei luba enam tavalist konto parooli kasutada käsurealt pushimiseks.

Sul on kaks lihtsat varianti:

---

### Variant A: kasuta GitHubi tokenit

Kui küsib:

```text
Username:
Password:
```

siis sisesta:

```text
Username: sinu GitHubi kasutajanimi
Password: GitHub Personal Access Token
```

Oluline: parooliks ei ole GitHubi tavaline konto parool, vaid Personal Access Token.

---

### Variant B: kasuta SSH võtit

See on mugavam, kui kasutad sama VM-i mitu korda.

#### Loo SSH võti

```bash
ssh-keygen -t ed25519 -C "sinu.email@example.com"
```

Vajuta mitu korda `Enter`.

#### Kuva public key

Linuxis/macOS-is:

```bash
cat ~/.ssh/id_ed25519.pub
```

Windows PowerShellis:

```powershell
cat ~/.ssh/id_ed25519.pub
```

Kopeeri kogu kuvatud tekst.

#### Lisa võti GitHubi

GitHubis mine:

```text
Settings → SSH and GPG keys → New SSH key
```

Lisa sinna kopeeritud public key.

#### Testi ühendust

```bash
ssh -T git@github.com
```

Kui kõik on korras, tuleb umbes selline tekst:

```text
Hi kasutajanimi! You've successfully authenticated.
```

#### Kasuta SSH remote aadressi

```bash
git remote set-url origin git@github.com:kasutajanimi/reponimi.git
```

Kontrolli:

```bash
git remote -v
```

---

## 11. Kui tahad enne pushimist GitHubist uuendused alla tõmmata

Kui sama repo kallal on tehtud muudatusi teises arvutis või VM-is:

```bash
git pull
```

Kui tekib konflikt, siis Git ütleb, millised failid vajavad käsitsi parandamist.

---

## 12. Kõige lühem igapäevane pushimise spikker

Kui repo on juba seadistatud:

```bash
git status
git add .
git commit -m "Muudatused"
git push
```

Kui `git push` ei tööta, proovi:

```bash
git push origin main
```

või:

```bash
git push origin master
```

---

## 13. Tüüpilised vead ja lahendused

### Viga: fatal: not a git repository

See tähendab, et sa ei ole Git repo kaustas.

Lahendus:

```bash
cd õige-kausta-nimi
```

või loo repo:

```bash
git init
```

---

### Viga: remote origin already exists

See tähendab, et remote on juba olemas.

Kontrolli:

```bash
git remote -v
```

Kui vaja muuta aadressi:

```bash
git remote set-url origin https://github.com/kasutajanimi/reponimi.git
```

või SSH-ga:

```bash
git remote set-url origin git@github.com:kasutajanimi/reponimi.git
```

---

### Viga: src refspec main does not match any

See tähendab tavaliselt, et sul ei ole veel ühtegi commiti või branchi nimi pole `main`.

Lahendus:

```bash
git add .
git commit -m "Esimene commit"
git branch
```

Kui branch on `master`, pushi nii:

```bash
git push origin master
```

Või muuda branch `main`-iks:

```bash
git branch -M main
git push -u origin main
```

---

### Viga: Authentication failed

See tähendab, et GitHubi tavaline parool ei tööta.

Kasuta kas:

```text
Personal Access Token
```

või seadista SSH võti.

---

## 14. Näidis algusest lõpuni

Kui sul on uus VM ja projektikaust olemas:

```bash
git --version

git config --global user.name "Valter Iliste"
git config --global user.email "valter@example.com"

cd minu-projekt

git init
git remote add origin https://github.com/kasutajanimi/reponimi.git

git add .
git commit -m "Esimene üleslaadimine"

git branch -M main
git push -u origin main
```

---

## 15. Kõige olulisem meelde jätta

```bash
git add .
git commit -m "Kirjeldus"
git push
```

Need kolm käsku on põhiline igapäevane GitHubi üleslaadimise töövoog.
