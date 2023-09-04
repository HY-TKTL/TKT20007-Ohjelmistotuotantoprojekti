# Yliopiston kirjautuminen

Jos projekti vaatii yliopiston kirjautumista vaihtoehdot ovat käytännössä SAML-pohjainen Shibboleth kirjautuminen tai modernimpaan OAuth:iin ja OpenID Connect:iin (OIDC) perustuva ratkaisu.

## SP-rekisteri

Ohtuprojektien käyttöön on luotu yliopiston kirjautumisen testipuolella toimivat SAML- ja OIDC-providerit. Esim. uusia testikäyttäjiä voi luoda ja käyttäjän atribuutteja lisätä osoitteessa [sp-registry.it.helsinki.fi](https://sp-registry.it.helsinki.fi/).

## Shibboleth

- [Yliopiston Shibboleth ohjeet](https://wiki.helsinki.fi/display/IAMasioita/Ohjeet+Shibbolointiinhttps://wiki.helsinki.fi/display/IAMasioita/Ohjeet+Shibbolointiin)
- [Konttialustan Shibboleth ohjeet](https://wiki.helsinki.fi/display/SO/Shibboleth-kirjautuminen+sovelluksellesi)

Shibboleth-kirjautumiseen on mahdollista käyttää valmiiksi OpenShift:iin konfiguroitua instanssia. Riittää, että sovelluksen lisää tähän [Apache-konfiguraatiotiedostoon](https://console-openshift-console.apps.ocp-test-0.k8s.it.helsinki.fi/k8s/ns/ohtuprojekti-staging/configmaps/httpd-config) ja uudelleenkäynistää Shibbolethin. Tämän jälkeen sovellukseen voi tunnistautua osoitteessa [shibboleth.ext.ocp-test-0.k8s.it.helsinki.fi/osoite](https://shibboleth.ext.ocp-test-0.k8s.it.helsinki.fi/sovellus/). Sovellus sää käyttäjän atribuutit pyyntöjen headereissa.

Esimerkkitoteutus ks. [shibboleth-postgres-example](https://github.com/UniversityOfHelsinkiCS/shibboleth-postgres-example/blob/main/src/server/middleware/user.ts).

## OpenID Connect

- [Yliopiston OIDC ohjeet](https://wiki.helsinki.fi/display/IAMasioita/OpenID+Connect)

OIDC on modernimpi tapa toteuttaa kirjautuminen. Erillistä Shibbolethin kaltaista palvelua ei tarvita vaan sovellus keskustelee suoraan yliopiston OIDC-providerin kanssa. Sovellustason toteutus riippuu omista teknologiavalinnoista. Todennäköisesti on tarvetta jonkin OpenID-kirjaston käytölle ja jollekin sessionhallintaratkaisulle.

Sovellukselle pitää määritellä paluuosoite sp-rekisterin kautta. Muut tarvittavat tiedot kuten palvelun salaisuus löytyvät myös sieltä sekä OpenShiftissa olevasta esimerkkitoteutuksesta.

Esimerkkitoteutus ks. [openid-mongo-example](https://github.com/UniversityOfHelsinkiCS/openid-mongo-example/blob/main/src/server/util/oidc.ts).
