# Yliopiston kirjautuminen

Jos projekti vaatii yliopiston kirjautumista vaihtoehdot ovat käytännössä SAML-pohjainen Shibboleth kirjautuminen tai modernimpaan OAuth:iin ja OpenID Connect:iin (OIDC) perustuva ratkaisu.
## Shibboleth

- [Yliopiston Shibboleth ohjeet](https://wiki.helsinki.fi/xwiki/bin/view/IAMasioita/Identiteetin-%20ja%20p%C3%A4%C3%A4synhallinnan%20dokumentaatio/Keskitetyn%20k%C3%A4ytt%C3%A4j%C3%A4tunnistuksen%20vaihtoehdot/1.%20Shibboleth%20%28SAML2%20%20OIDC%29/Ohjeet%20Shibbolointiin)
- [Konttialustan Shibboleth ohjeet](https://wiki.helsinki.fi/xwiki/bin/view/SO/Sovelluskehitt%C3%A4j%C3%A4n%20ohjeet/Alustat/Tiken%20konttialusta/3%20-%20Ohjeet/Shibboleth-kirjautuminen%20sovelluksellesi)

Shibboleth-kirjautumiseen on mahdollista käyttää valmiiksi OpenShift:iin konfiguroitua instanssia. Riittää, että sovelluksen lisää tähän [Apache-konfiguraatiotiedostoon](https://console-openshift-console.apps.ocp-test-0.k8s.it.helsinki.fi/k8s/ns/ohtuprojekti-staging/configmaps/httpd-config) ja uudelleenkäynnistää Shibbolethin. Tämän jälkeen sovellukseen voi tunnistautua osoitteessa [shibboleth.ext.ocp-test-0.k8s.it.helsinki.fi/osoite](https://shibboleth.ext.ocp-test-0.k8s.it.helsinki.fi/sovellus/). Sovellus sää käyttäjän atribuutit pyyntöjen headereissa.

Esimerkkitoteutus ks. [shibboleth-postgres-example](https://github.com/UniversityOfHelsinkiCS/shibboleth-postgres-example/blob/main/src/server/middleware/user.ts).

## OpenID Connect

ks [kirjautuminen.md](irjautuminen.md)
