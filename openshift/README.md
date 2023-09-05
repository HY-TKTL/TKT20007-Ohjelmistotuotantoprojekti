# OpenShift

Yliopistolla on käytössä OpenShift-konttialusta, jolla voi ajaa projektin testi- sekä tuotantoversioita. Konttialustan wiki-sivut löytyvät [täältä](https://wiki.helsinki.fi/display/SO/Tiken+konttialusta). Ohtuprojektin osallistujilla on käyttöoikeudet konttialustan testipuolen projektiin `ohtuprojekti-staging` osoitteessa [console-openshift-console.apps.ocp-test-0.k8s.it.helsinki.fi](https://console-openshift-console.apps.ocp-test-0.k8s.it.helsinki.fi/).

## Esimerkkiprojektit

OpenShiftiin on konfiguroitu kaksi yliopiston kirjautumista käyttävää esimerkkiprojektia ks. [github.com/UniversityOfHelsinkiCS/shibboleth-postgres-example](https://github.com/UniversityOfHelsinkiCS/shibboleth-postgres-example) sekä [github.com/UniversityOfHelsinkiCS/openid-mongo-example](https://github.com/UniversityOfHelsinkiCS/openid-mongo-example/).

Jos on tarve yliopiston kirjautumiselle ks. [kirjautuminen.md](kirjautuminen.md).

## Tietokannat

- [Konttialustan ohjeet](https://wiki.helsinki.fi/display/SO/Tietokannat)

Tietokantaa voi suorittaa konttialustalla, mutta tätä ei yleensä suositella. Parempi ratkaisu on tilata Postgres-tietokanta yliopiston yhteiskäyttöklusterille ks. [wiki.helsinki.fi/display/SO/PostgreSQL](https://wiki.helsinki.fi/display/SO/PostgreSQL).

Konttialustalla toimivan tietokannan pysyväistallentaminen vaati persistent volume claimin käyttöä ks. [ohjeet](https://wiki.helsinki.fi/pages/viewpage.action?pageId=350278065). Projektin `storageClassName` on `pomppa25`. Mallia voi katsoa esimerkkisovelluksista, joille on konfiguroitu pysyväistallennetut Postgres- ja Mongo-tietokannat.

### Tietokantaan yhdistäminen

Konttialustan suuntaan avattuihin yliopiston yhteiskäyttökantoihin ei voi yhdistää OpenShiftin ulkopuolelta. Vastaavasti konttialustan sisällä ajettaville tietokannoille ei ole tarvetta luoda klusterin ulkopuolisen yhteyden mahdollistavaa `route`-resurssia.

Tietokantoihin on siis yhdistettävä klusterin sisältä käsin. `ohtuprojekti-staging`-projektille on tätä varten konfiguroitu `db-tools` podi, joka sisältää manuaalisten tietokantayhteyksien kannalta tarvittavat työkalut, kuten `psql` ja `mongosh`. Podin sisälle pääsee käyttöliittymän kautta Topology-näkymästä painamalla podia `db-tools` -> View logs -> Terminal.

Tietokantoihin voi yhdistää podin kautta myös komentorivityökalun `oc` avulla esim.
```bash
oc exec -it $(oc get pods -l deployment=db-tools -o jsonpath='{.items[0].metadata.name}') -- psql postgres://kayttaja:salasana@possu-test.it.helsinki.fi:5432/tietokanta
```

## Sovelluksen julkaisu

OpenShiftiin voi julkaista sovelluksia monella tavalla. Suositeltu ratkaisu on julkaista sovellus konttina johonkin konttirekisteriin, kuten [quay.io](https://quay.io/) tai [Docker Hub](https://hub.docker.com/). Uuden version sovelluksesta voi puskea rekisteriin automaattisesti esimerkiksi GitHub Actions:in avulla.

OpenShiftin verkkokonsolin avulla voi helposti lisätä konttirekisteristä löytyvän kontin. Developer-näkymässä +Add-tabista löytyy kohta Container images. Täyttämällä vaaditut kentät OpenShift luo sovellukselle resurssit: `deployment`, `service`, `route` ja `imageStream`.

Administrator-näkymässä -> Builds -> ImageStreams, muokkaa luotua `imageStream`-resurssia asettamalla `importPolicy` arvoksi `scheduled: true`. Tämän jälkeen OpenShift hakee 15 minuutin välein konttirekisteristä uudet julkaisut.

Esimerkkisovelluksille on konfiguroitu Github Actions ja quay.io pohjainen julkaisuputki konttialustalle ks. [esimerkki](https://github.com/UniversityOfHelsinkiCS/shibboleth-postgres-example/blob/main/.github/workflows/staging.yaml).
