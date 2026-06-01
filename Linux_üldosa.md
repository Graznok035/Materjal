# Linux eksam – üldosa ja lahendamise strateegia

## Mis kordub kõikides Linuxi piletites?

- UbuntuServer masinas Ansible paigaldamine
- Ansible ligipääs AlmaServer ja DebianServer masinatesse
- kasutaja hkh loomine
- sudo õiguste andmine
- SSH võtmega ligipääs
- root/parooliga SSH keelamine
- tulemüüri lubamine ainult vajalikele teenustele
- serverite nimelahenduse kontroll
- dokumenteerimine

## Soovitatav tööjärjekord eksamil

1. Kontrolli masinad ja IP-aadressid
2. Sea hostnamed
3. Lisa `/etc/hosts` kirjed
4. Loo kasutaja `hkh`
5. Anna sudo õigused
6. Sea SSH võtmega ligipääs
7. Keela root/parooliga SSH
8. Paigalda Ansible UbuntuServerisse
9. Tee Ansible inventory
10. Testi `ansible all -m ping`
11. Alles siis alusta pileti eriteenusega

## Kiirkontroll

```bash
hostnamectl
ip a
ip route
cat /etc/hosts
whoami
groups hkh
sudo -l
systemctl status ssh



### `01_ansible_baasjuhend.md`

See on kriitiline, sest kõik Linuxi piletid algavad Ansiblega.

Sisu peaks olema:
```markdown
# Ansible baasjuhend

## Eesmärk

UbuntuServer on juhtmasin. Selle kaudu hallatakse AlmaServer ja DebianServer masinaid.

## Paigaldamine UbuntuServeris

```bash
sudo apt update
sudo apt install ansible -y
ansible --version
