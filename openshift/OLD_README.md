# OpenShift

> ÄLÄ LUE TÄTÄ OHJETTA

Yliopistolla on käytössä OpenShift-konttialusta, jolla voi ajaa projektin testi- sekä tuotantoversioita. Konttialustan wiki-sivut löytyvät [täältä](https://wiki.helsinki.fi/xwiki/bin/view/SO/Sovelluskehitt%C3%A4j%C3%A4n%20ohjeet/Alustat/Tiken%20konttialusta/). Ohtuprojektin osallistujilla on käyttöoikeudet (sen jälkeen kun ne lisätään, eli jos tarvitsette OpenShiftiä, pyytäkää pääsyä ohjaajaltanne) konttialustan testipuolen projektiin `ohtuprojekti-staging` osoitteessa [console-openshift-console.apps.ocp-test-0.k8s.it.helsinki.fi](https://console-openshift-console.apps.ocp-test-0.k8s.it.helsinki.fi/).

Usein kysyttyjä kysymyksiä https://github.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/blob/master/openshift/faq.md

Lue https://devops.pages.helsinki.fi/guides/tike-container-platform/instructions/all-in-one.html

## Esimerkkiprojektit

OpenShiftiin on konfiguroitu kaksi yliopiston kirjautumista käyttävää esimerkkiprojektia ks. [github.com/UniversityOfHelsinkiCS/shibboleth-postgres-example](https://github.com/UniversityOfHelsinkiCS/shibboleth-postgres-example) sekä [github.com/UniversityOfHelsinkiCS/openid-mongo-example](https://github.com/UniversityOfHelsinkiCS/openid-mongo-example/).

Jos on tarve yliopiston kirjautumiselle ks. [kirjautuminen.md](kirjautuminen.md).

## Tietokannat

- [Konttialustan ohjeet](https://wiki.helsinki.fi/xwiki/bin/view/SO/Sovelluskehitt%C3%A4j%C3%A4n%20ohjeet/Alustat/Tiken%20konttialusta/3%20-%20Ohjeet/Tietokannat/)

Tietokantaa voi suorittaa konttialustalla, mutta tätä ei yleensä suositella. Parempi ratkaisu on tilata Postgres-tietokanta yliopiston yhteiskäyttöklusterille ks. [wiki.helsinki.fi/xwiki/bin/view/SO/Sovelluskehittäjän ohjeet/Alustat/Yhteiskäyttöiset tietokannat/PostgreSQL/](https://wiki.helsinki.fi/xwiki/bin/view/SO/Sovelluskehitt%C3%A4j%C3%A4n%20ohjeet/Alustat/Yhteisk%C3%A4ytt%C3%B6iset%20tietokannat/PostgreSQL/).

Konttialustalla toimivan tietokannan pysyväistallentaminen vaati persistent volume claimin käyttöä ks. [ohjeet]([https://wiki.helsinki.fi/pages/viewpage.action?pageId=350278065](https://wiki.helsinki.fi/xwiki/bin/view/SO/Sovelluskehitt%C3%A4j%C3%A4n%20ohjeet/Alustat/Tiken%20konttialusta/3%20-%20Ohjeet/3.8%20Levyn%20k%C3%A4ytt%C3%B6%20Tiken%20OpenShiftiss%C3%A4/)). Projektin `storageClassName` on `pomppa25`. Mallia voi katsoa esimerkkisovelluksista, joille on konfiguroitu pysyväistallennetut Postgres- ja Mongo-tietokannat.

### Tietokantaan yhdistäminen

Konttialustan suuntaan avattuihin yliopiston yhteiskäyttökantoihin ei voi yhdistää OpenShiftin ulkopuolelta. Vastaavasti konttialustan sisällä ajettaville tietokannoille ei ole tarvetta luoda klusterin ulkopuolisen yhteyden mahdollistavaa `route`-resurssia.

Tietokantoihin on siis yhdistettävä klusterin sisältä käsin. `ohtuprojekti-staging`-projektille on tätä varten konfiguroitu `db-tools` podi, joka sisältää manuaalisten tietokantayhteyksien kannalta tarvittavat työkalut, kuten `psql` ja `mongosh`. Podin sisälle pääsee käyttöliittymän kautta Topology-näkymästä painamalla podia `db-tools` -> View logs -> Terminal.

Tietokantoihin voi yhdistää podin kautta myös komentorivityökalun `oc` avulla esim.

```bash
oc exec -it $(oc get pods -l deployment=db-tools -o jsonpath='{.items[0].metadata.name}') -- psql postgres://kayttaja:salasana@possu-test.it.helsinki.fi:5432/tietokanta
```

Tämä edellyttää, että olet Eduroamissa ja kirjautunut klusteriin, ks lisää Tiken [konttialustaohjeesta](https://wiki.helsinki.fi/xwiki/bin/view/SO/Sovelluskehitt%C3%A4j%C3%A4n%20ohjeet/Alustat/Tiken%20konttialusta/).

## Sovelluksen julkaisu

OpenShiftiin voi julkaista sovelluksia monella tavalla. Suositeltu ratkaisu on julkaista sovellus konttina johonkin konttirekisteriin, kuten [quay.io](https://quay.io/) tai [Docker Hub](https://hub.docker.com/). Uuden version sovelluksesta voi puskea rekisteriin automaattisesti esimerkiksi GitHub Actions:in avulla.

OpenShiftin verkkokonsolin avulla voi helposti lisätä konttirekisteristä löytyvän kontin. Developer-näkymässä +Add-tabista löytyy kohta Container images. Täyttämällä vaaditut kentät OpenShift luo sovellukselle resurssit: `deployment`, `service`, `route` ja `imageStream`.

Administrator-näkymässä -> Builds -> ImageStreams, muokkaa luotua `imageStream`-resurssia asettamalla `importPolicy` arvoksi `scheduled: true`. Tämän jälkeen OpenShift hakee 15 minuutin välein konttirekisteristä uudet julkaisut.

Esimerkkisovelluksille on konfiguroitu Github Actions ja quay.io pohjainen julkaisuputki konttialustalle ks. [esimerkki](https://github.com/UniversityOfHelsinkiCS/shibboleth-postgres-example/blob/main/.github/workflows/staging.yaml).

### Resurssirajat

Kun sovellus on lisätty OpenShiftiin aseta resurssirajat Topology-näkymässä painamalla sovelluksen podin kohdalla hiiren oikealla näppäimellä -> Edit resource limits. Podin Observe-tabista näkee nykyisen CPU:n ja RAM:in käytön, joita kannattaa käyttää hyödyksi rajoja asettaessa.

Klusteri priorisoi sovelluksia, joille on asetettu resurssirajat. Jos resursseista on pulaa, niin ilman rajojen asettamista sovellus ei välttämättä edes käynnisty.


### Käyttöoikeudet

OpenShift ei salli tietoturvasyistä konttien ajoa root-oikeuksilla. Suoritusaikaisen kontin käyttäjän UID on sattumanvarainen ja käyttäjä kuuluu aina root-ryhmään. Näin ollen tiedostoon voi antaa oikeudet esim.

```bash
chgrp root tiedosto && chmod 660 tiedosto
```
