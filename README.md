# Securing-Linux-Debian-13

Ajankohtaisten suojamekanismien listaaminen Debianin 13. versiossa (codename "Trixie"), niiden arviointi ja käyttöönotto-ohjeistus.

## Kenelle?

Ohjeet on kirjoitettu kotikäyttäjille ja opiskelijoille, jotka käyttävät Debian 13:a omalla koneellaan. Komentorivin perusteet riittävät: jokainen ohje kertoo, mitä komento tekee, miten tuloksen voi tarkistaa ja miten muutoksen saa perutuksi.

## Miten ohjeita käytetään?

- Etene ohjeissa alla olevan taulukon järjestyksessä. Tärkeimmät ja riskittömimmät toimet ovat ensin.
- Lue ohje kokonaan ennen kuin aloitat, ja tee vaiheet siinä järjestyksessä kuin ne on kirjoitettu.
- Jokaisessa ohjeessa on **palautusohje**. Jos jokin menee pieleen, palauta muutos ennen kuin jatkat.
- Ota varmuuskopio tärkeistä tiedostoista ennen suurempia muutoksia, kuten levynsalausta.

> **Huom:** Ohjeet on testattu Debian 13:lla. Ne toimivat usein myös Debian 12:lla ja Debian-pohjaisissa jakeluissa, mutta poikkeamat on merkitty ohjeisiin vain silloin, kun ne on testattu.

## Ohjeet tärkeysjärjestyksessä

| # | Aihe | Miksi tärkeä | Tila |
|---|---|---|---|
| 1 | [Automaattiset tietoturvapäivitykset](ohjeet/automaattiset-tietoturvapaivitykset.md) (`unattended-upgrades`) | Useimmat hyökkäykset käyttävät haavoittuvuuksia, joihin on jo korjaus | Valmis |
| 2 | Palomuuri (`nftables` / `ufw`) | Estää yhteydet palveluihin, joita ei ole tarkoitettu ulkoverkolle | Tulossa |
| 3 | [Root-käyttöoikeuksien rajoittaminen](ohjeet/root-kayttooikeuksien-rajoittaminen.md) | Ylläpito `sudo`n kautta, root-kirjautuminen estetty | Valmis |
| 4 | SSH:n suojaaminen | Avainkirjautuminen ja salasanakirjautumisen esto, jos SSH on käytössä | Tulossa |
| 5 | AppArmor | Rajoittaa ohjelmien pääsyä tiedostoihin ja resursseihin | Tulossa |
| 6 | Levynsalaus (LUKS) | Suojaa tiedot, jos kone tai levy katoaa | Tulossa |
| 7 | Lokitus ja valvonta (`journalctl`, `auditd`) | Poikkeamien havaitseminen jälkikäteen | Tulossa |
| 8 | Selain ja yksityisyys | Seuranta, laajennukset ja telemetria | Tulossa |

Ohjeiden rinnalle tulee englanninkielinen [tarkistusskripti](scripts/), joka raportoi koneen tilan muuttamatta mitään.

## Seurattavat tietolähteet

Repoon tuodaan automaattisesti (GitHub Actions) ajankohtaista tietoa, joka koskee Debian 13:a ja Linuxia. Lähteet tärkeysjärjestyksessä:

| # | Lähde | Mitä seurataan |
|---|---|---|
| 1 | [Debian Security Tracker](https://security-tracker.debian.org/tracker/) ja [Debianin tietoturvatiedotteet (DSA)](https://www.debian.org/security/) | Trixieä koskevat korjaukset ja avoimet haavoittuvuudet |
| 2 | [Kyberturvallisuuskeskus: Kybersää](https://kyberturvallisuuskeskus.fi/fi/kybersaa) ja haavoittuvuustiedotteet | Linuxia koskevat suomenkieliset tilannekuvat ja varoitukset |
| 3 | [CVE](https://www.cve.org/) ja [CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) | Aktiivisesti hyödynnetyt haavoittuvuudet Linux-ohjelmistoissa |
| 4 | [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks) ja [ANSSI:n Linux-suositukset](https://cyber.gouv.fr/en/publications/configuration-recommendations-gnulinux-system) | Koventamissuositusten muutokset (kuukausittainen tarkistus) |

Tavoitteena on viikkokooste ja erillinen issue kriittisistä haavoittuvuuksista.

## Rakenne

```
ohjeet/     suomenkieliset käyttöönotto-ohjeet
scripts/    englanninkieliset apuskriptit
```

## Lisenssi

Ohjeet: [CC BY 4.0](LICENSE). Saat jakaa ja muokata niitä, kunhan mainitset lähteen.
Skriptit: [MIT](scripts/LICENSE).

Ohjeet tarjotaan sellaisenaan ilman takuuta. Kokeile muutoksia ensin virtuaalikoneessa, jos mahdollista.
