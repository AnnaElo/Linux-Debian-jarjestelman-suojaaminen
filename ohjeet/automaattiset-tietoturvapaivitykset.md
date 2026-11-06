# Automaattiset tietoturvapäivitykset Debianissa

Tämä ohje kuvaa, miten Debian 13 asentaa tietoturvapäivitykset automaattisesti `unattended-upgrades`-paketin avulla. Kone hakee päivitykset kerran päivässä ja asentaa ne taustalla, eikä sinun tarvitse muistaa ajaa `apt upgrade` -komentoa.

**Testattu:** Debian 13. Sama tapa toimii myös Debian 12:ssa.

## Miksi?

- Suurin osa hyökkäyksistä käyttää haavoittuvuuksia, joihin on jo julkaistu korjaus. Päivitetty kone on suojassa niiltä.
- Debianin tietoturvatiimi julkaisee korjauksia usein viikoittain. Käsin päivittäessä niitä jää helposti asentamatta.
- Automaattiset päivitykset ovat tämän repon ohjeista tärkein ja samalla riskittömin: Debianin vakaan version päivitykset ovat pieniä korjauksia, eivät uusia ohjelmaversioita.

## Ennen kuin aloitat

Tarvitset sudo-oikeudet. Jos `sudo whoami` ei tulosta `root`, katso ensin [root-käyttöoikeuksien ohjeen](root-kayttooikeuksien-rajoittaminen.md) vaihe 1.

Päivitä kone kerran käsin, jotta automatiikka aloittaa ajan tasalla olevasta tilasta:

```bash
sudo apt update && sudo apt upgrade
```

## 1. Tarkista, onko automaattinen päivitys jo käytössä

Debianin asennusohjelma on voinut ottaa sen käyttöön valmiiksi.

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Jos tulos on tämä, siirry vaiheeseen 3:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Jos tulee virhe `Tiedostoa tai hakemistoa ei ole` tai arvot ovat `"0"`, jatka vaiheeseen 2.

## 2. Asenna ja ota käyttöön

```bash
sudo apt install unattended-upgrades apt-listchanges
sudo dpkg-reconfigure -plow unattended-upgrades
```

Jälkimmäinen komento kysyy *Automatically download and install stable updates?* Valitse **Kyllä** (Yes). Komento kirjoittaa tiedoston `/etc/apt/apt.conf.d/20auto-upgrades`, jonka sisällön näet vaiheen 1 komennolla.

`apt-listchanges` näyttää päivitysten muutostiedot, kun päivität käsin. Se ei ole pakollinen, mutta siitä on apua, kun haluat tietää, mitä päivitys korjasi.

## 3. Tarkista, mitä päivitetään

Oletusasetukset ovat Debian 13:ssa järkevät, eikä niitä yleensä tarvitse muuttaa. Asetukset ovat tiedostossa `/etc/apt/apt.conf.d/50unattended-upgrades`:

```bash
grep -A6 "Origins-Pattern" /etc/apt/apt.conf.d/50unattended-upgrades
```

Rivit, joiden alussa ei ole `//`, ovat käytössä. Oletuksena niitä on kolme:

| Rivi | Mitä se tarkoittaa |
|---|---|
| `codename=${distro_codename},label=Debian` | Debianin pistejulkaisujen (esim. 13.1, 13.2) korjaukset |
| `codename=${distro_codename},label=Debian-Security` ja `codename=${distro_codename}-security,label=Debian-Security` | Tietoturvapäivitykset (`trixie-security`) |

Automaattisesti **ei** päivitetä:

- Kolmansien osapuolten ohjelmalähteitä, kuten Googlen, Microsoftin tai Spotifyn repoja
- `backports`-paketteja
- Flatpak- tai Snap-sovelluksia (katso [Mitä tämä ei vielä suojaa](#mitä-tämä-ei-vielä-suojaa))
- Debianin pääversiota: kone ei siirry itsestään Debian 14:ään

## 4. (Suositeltu) Omat lisäasetukset

Älä muokkaa tiedostoa `50unattended-upgrades` suoraan, sillä paketin päivitys kysyy silloin, kumpi versio säilytetään. Tee sen sijaan oma tiedosto, joka luetaan myöhemmin ja ohittaa oletukset:

```bash
sudo tee /etc/apt/apt.conf.d/52unattended-upgrades-local > /dev/null <<'EOF'
// Poista vanhat ytimet, jotta /boot ei täyty
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";

// Älä käynnistä konetta uudelleen kysymättä
Unattended-Upgrade::Automatic-Reboot "false";
EOF
```

Kannettavalla voit lisäksi estää päivitykset akkuvirralla, jotta asennus ei katkea akun loppuessa:

```bash
echo 'Unattended-Upgrade::OnlyOnACPower "true";' | sudo tee -a /etc/apt/apt.conf.d/52unattended-upgrades-local
```

Tarkista, että asetukset tulivat voimaan:

```bash
apt-config dump | grep -E "Periodic|Unattended-Upgrade::(Automatic-Reboot|Remove-Unused-Kernel|OnlyOnACPower)"
```

**Palautus:** `sudo rm /etc/apt/apt.conf.d/52unattended-upgrades-local`

## 5. Testaa

Kuivaharjoitus näyttää, mitä asennettaisiin, mutta ei asenna mitään:

```bash
sudo unattended-upgrade --dry-run --debug
```

Tulosteen alussa on rivi `Allowed origins are: ...`, jossa pitäisi näkyä `trixie-security`. Jos päivityksiä ei ole odottamassa, lopussa lukee `No packages found that can be upgraded unattended`, mikä on hyvä merkki.

Tarkista vielä ajastukset. Ensimmäinen hakee pakettilistat, toinen asentaa päivitykset:

```bash
systemctl list-timers 'apt-daily*'
```

Molemmilla pitäisi olla kellonaika sarakkeessa `NEXT`.

## 6. Seuraa päivityksiä

Loki kertoo, mitä on asennettu ja milloin:

```bash
sudo less /var/log/unattended-upgrades/unattended-upgrades.log
```

Osa päivityksistä, erityisesti ytimen (`linux-image`) ja `systemd`:n päivitykset, tulee voimaan vasta uudelleenkäynnistyksen jälkeen. `needrestart` kertoo, tarvitaanko sitä:

```bash
sudo apt install needrestart
sudo needrestart -r l       # l = vain listaa, ei käynnistä mitään uudelleen
```

Jos tulosteessa lukee `Pending kernel upgrade!` tai vastaava, käynnistä kone uudelleen sopivana hetkenä. Kotikoneelle riittää yleensä, että sammutat sen normaalisti päivän päätteeksi.

## Lopputulos

| Asia | Tila |
|---|---|
| Pakettilistojen haku | Päivittäin |
| Tietoturvapäivitysten asennus | Päivittäin, automaattisesti |
| Vanhojen ytimien poisto | Automaattinen |
| Uudelleenkäynnistys | Käsin, kun `needrestart` sitä ehdottaa |

## Mitä tämä ei vielä suojaa

- **Flatpak- ja Snap-sovellukset** päivittyvät erikseen. GNOME Ohjelmistot päivittää Flatpakit yleensä itse, mutta käsin se tapahtuu komennolla `flatpak update`.
- **Selainlaajennukset ja ohjelmien omat päivitykset**, kuten VS Coden laajennukset tai pip-paketit, eivät kuulu Debianin pakettienhallintaan.
- **Kolmansien osapuolten repot** pitää päivittää käsin (`sudo apt update && sudo apt upgrade`), ellei niitä lisää kohtaan `Origins-Pattern`. Lisää vain lähteitä, joihin luotat yhtä paljon kuin Debianiin.
- **Tuen päättyminen:** Debianin tietoturvatuki kestää noin kolme vuotta julkaisusta, ja sen jälkeen LTS-tiimi jatkaa tukea noin viiden vuoden ikään asti. Kun tuki päättyy, automaattisia korjauksia ei enää tule, ja kone pitää päivittää uuteen Debian-versioon.

## Vianetsintä

**`Could not get lock /var/lib/dpkg/lock-frontend`**
Automaattinen päivitys on juuri käynnissä. Odota muutama minuutti ja yritä uudelleen. Älä poista lukkotiedostoa käsin.

**Kuivaharjoitus kertoo `Package ... is kept back` tai ohittaa paketin**
Paketin päivitys vaatisi uuden riippuvuuden asentamista tai vanhan poistamista, eikä `unattended-upgrades` tee sitä itse. Aja `sudo apt upgrade` ja lue, mitä apt ehdottaa.

**`/boot` on täynnä**
Vanhoja ytimiä on kertynyt. Varmista, että vaiheen 4 asetus `Remove-Unused-Kernel-Packages` on käytössä, ja poista ylimääräiset käsin: `sudo apt autoremove --purge`.

**Lokissa lukee `dpkg was interrupted`**
Edellinen asennus katkesi, esimerkiksi koneen sammuessa. Korjaa tilanne: `sudo dpkg --configure -a`.

## Lähteet

- `man unattended-upgrade`
- Tiedoston `/etc/apt/apt.conf.d/50unattended-upgrades` kommentit
- [Debian Wiki: UnattendedUpgrades](https://wiki.debian.org/UnattendedUpgrades)
- [Debian Handbook: Automatic Upgrades](https://debian-handbook.info/browse/stable/sect.automatic-upgrades.html)
- [Debianin tietoturvatiedotteet](https://www.debian.org/security/)
- [Debian LTS](https://wiki.debian.org/LTS)
