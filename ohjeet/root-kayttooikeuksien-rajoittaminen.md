# Root-käyttöoikeuksien rajoittaminen Debianissa

Tämä ohje kuvaa, miten Debian-järjestelmässä estetään suora root-käyttö. Ylläpito tehdään tavallisella käyttäjällä `sudo`-komennon kautta, ja root-tilille ei pääse kirjautumaan SSH:n kautta eikä salasanalla.

**Testattu:** Debian 12 ja 13. Ohjeen `sshd_config.d`-hakemistoa käyttävä tapa toimii Debian 11:stä alkaen.

## Miksi?

- Root on jokaisessa Linux-järjestelmässä tunnettu käyttäjänimi, joten automaattiset hyökkäysbotit kokeilevat sitä ensimmäisenä.
- `sudo` kirjaa ylläpitotoimet lokiin käyttäjäkohtaisesti (`/var/log/auth.log` tai `journalctl`).
- Ylläpito-oikeudet ovat käytössä vain silloin, kun niitä pyydetään erikseen.

## Ennen kuin aloitat

> **Varoitus:** Tee vaiheet tässä järjestyksessä. Jos lukitset root-tilin ennen kuin sudo toimii, lukitset itsesi ulos ylläpito-oikeuksista.

Ohjeessa `<käyttäjä>` tarkoittaa omaa käyttäjänimeäsi. Sen näkee komentokehotteesta ennen @-merkkiä (`<käyttäjä>@<kone>:~$`) tai komennolla `whoami`.

Jos teet muutoksia etäyhteydellä, pidä yksi istunto auki koko ajan, kunnes olet testannut uuden kirjautumisen toisessa istunnossa.

## 1. Varmista sudo-oikeudet

```bash
sudo whoami
```

Jos tulos on `root`, siirry vaiheeseen 2.

Jos tulee ilmoitus `<käyttäjä> is not in the sudoers file` tai `sudo: komentoa ei löydy`, lisää käyttäjä sudo-ryhmään root-käyttäjänä:

```bash
su -                         # anna root-salasana; viiva on tärkeä
apt install sudo
usermod -aG sudo <käyttäjä>
exit
```

Kirjaudu kokonaan ulos ja takaisin sisään, jotta ryhmämuutos tulee voimaan, ja testaa `sudo whoami` uudelleen.

> **Huom:** Jos Debian asennettiin jättämällä root-salasana tyhjäksi, asennusohjelma on jo antanut ensimmäiselle käyttäjälle sudo-oikeudet, eikä root-tilillä ole salasanaa. Tällöin `su -` antaa virheen `Tunnistautumisvirhe`, mikä on odotettua.

## 2. Estä root-kirjautuminen SSH:n kautta

Tarkista ensin, onko SSH-palvelin asennettu:

```bash
systemctl status ssh
```

Jos palvelua ei löydy (`Unit ssh.service could not be found`), ohita tämä vaihe.

Muussa tapauksessa luo oma asetustiedosto. OpenSSH käyttää kunkin asetuksen ensimmäistä löytämäänsä arvoa, ja `sshd_config.d`-hakemiston tiedostot luetaan aakkosjärjestyksessä ennen pääasetustiedostoa, joten etuliite `00-` varmistaa, että tämä asetus on voimassa.

```bash
echo "PermitRootLogin no" | sudo tee /etc/ssh/sshd_config.d/00-disable-root.conf
sudo sshd -t                                  # syntaksin tarkistus, ei tulostetta = ok
sudo sshd -T | grep -i permitrootlogin        # odotettu tulos: permitrootlogin no
sudo systemctl restart ssh
```

Testaa toiselta koneelta: `ssh root@<kone>` pitäisi hylätä, ja `ssh <käyttäjä>@<kone>` pitäisi toimia.

**Palautus:**

```bash
sudo rm /etc/ssh/sshd_config.d/00-disable-root.conf
sudo systemctl restart ssh
```

## 3. Lukitse root-tili

```bash
sudo passwd -l root
sudo passwd -S root          # odotettu tulos: root L ...
```

Lukittuun tiliin ei voi kirjautua salasanalla, eikä `su -` toimi, mutta `sudo` toimii normaalisti.

**Palautus:** `sudo passwd -u root`

## 4. (Valinnainen) Estä root-kuori

```bash
sudo usermod -s /usr/sbin/nologin root
```

Tämä on tiukempi toimenpide, joka voi hankaloittaa vianetsintää: esimerkiksi palautustilassa (rescue mode) root-kuori ei avaudu. Useimmille järjestelmille vaiheet 1–3 riittävät.

**Palautus:** `sudo usermod -s /bin/bash root`

## Lopputulos

| Asia | Tila |
|---|---|
| Root-kirjautuminen SSH:n kautta | Estetty |
| Root-tilin salasanakirjautuminen | Lukittu |
| Ylläpito | `sudo` |

## Mitä tämä ei vielä suojaa

Tavallinen käyttäjä voi edelleen kirjautua SSH:n kautta salasanalla, ja sillä on sudo-oikeudet. Järjestelmän suojaus SSH:ta vastaan on siis yhtä vahva kuin tämän käyttäjän salasana.

Riski riippuu siitä, näkyykö kone internetiin. Kotiverkossa reitittimen takana, kun porttia 22 ei ole ohjattu koneelle, ulkopuolinen ei yleensä pääse yhdistämään SSH:hon. Julkisessa verkossa olevaan koneeseen kohdistuu automaattisia kirjautumisyrityksiä jatkuvasti.

Suositellut jatkotoimet:

- Jos SSH:ta ei käytetä, sammuta se: `sudo systemctl disable --now ssh`
- SSH-avaimet käyttöön ja salasanakirjautumisen esto (`PasswordAuthentication no`)
- Palomuuri (`ufw`) ja SSH:n salliminen vain luotetuista verkoista
- `fail2ban` toistuvien kirjautumisyritysten estoon
- Säännölliset päivitykset: `sudo apt update && sudo apt upgrade`

## Vianetsintä

**`Lupa evätty` tai `Unable to acquire the dpkg frontend lock ... are you root?`**
Komento vaatii ylläpito-oikeudet. Lisää alkuun `sudo` tai aja se root-käyttäjänä.

**`usermod: komentoa ei löydy`**
`usermod` on hakemistossa `/usr/sbin`, joka ei ole tavallisen käyttäjän polussa. Käytä `sudo usermod ...` tai vaihda rootiksi komennolla `su -` (viivan kanssa; pelkkä `su` ei päivitä polkua).

**`su: Tunnistautumisvirhe`**
Root-salasana on väärä tai root-tilillä ei ole salasanaa. Kokeile `sudo whoami`. Jos sudo toimii, root-salasanaa ei tarvita.

**Root-salasana unohtunut eikä sudo toimi**
Root-oikeudet saa palautettua käynnistysvalikon kautta, kun koneeseen on fyysinen pääsy:

1. Käynnistä kone uudelleen. Kun GRUB-valikko näkyy (pidä tarvittaessa Shift tai paina Esc käynnistyksen alussa), valitse Debian-rivi mutta älä paina Enteriä.
2. Paina `e`, etsi `linux`-alkuinen rivi ja lisää sen loppuun `init=/bin/bash`.
3. Käynnistä painamalla Ctrl+X tai F10.
4. Aja:
   ```bash
   mount -o remount,rw /
   usermod -aG sudo <käyttäjä>
   passwd root              # valinnainen: aseta uusi root-salasana
   sync
   reboot -f
   ```

Muutos GRUB-riviin on väliaikainen. Jos `sudo`-pakettia ei ole asennettu, kirjaudu uudelleenkäynnistyksen jälkeen rootiksi uudella salasanalla ja aja `apt install sudo`.

## Lähteet

- `man sshd_config` (asetus `PermitRootLogin`)
- `man passwd`
- [Debian Wiki: sudo](https://wiki.debian.org/sudo)
- [Debian Wiki: SSH](https://wiki.debian.org/SSH)
