# TKT20007-Ohjelmistoprojekti

Kurssin yleinen kuvaus: [courses.helsinki.fi]([https://courses.helsinki.fi/fi/tkt20007/](https://studies.helsinki.fi/kurssit/opintojakso/otm-f07aa52f-df4c-4a9a-8e89-d6222b88e2f2/TKT20007))

Ohtuprojekti osallistujan silmin: syksyn 2024 NOW-tietokanta-tiimin [kertomus](https://www.youtube.com/watch?v=yrxqB_y0lMs&t=165s)

[Projektien repositorioita](https://github.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/tree/master/history)

## Ajankohtaista

- Kurssilla on Discord-kanava _ohtuproj_, liity nyt https://study.cs.helsinki.fi/discord/join/ohtuproj

Demot:

* Kesäkuun lopun demo pe 27.6. klo 9.00-11 Chemicumin A129
* Pitkien projektien loppudemo elokuun viimeisellä viikolla, TBA

Aloitetaan tasalta ilman akateemista varttia

Vertaisarviot:

* puolessa välissä ja lopussa vertaisarvio

[https://study.cs.helsinki.fi/projekti/peerreview](https://study.cs.helsinki.fi/projekti/peerreview) 

## Tekninen tuki

Ryhmille on tarjolla teknistä tukea! Kysy [discordissa](https://study.cs.helsinki.fi/discord/join/ohtuproj), kanavalla *ohtuproj_tekninen_tuki*

## Arvosteluperusteet

Arvostelu perustuu seuraaviin asioihin
- Ryhmän sopimien työskentelytapojen noudattaminen
- Ryhmän työskentelytapojen (=prosessin) kehittäminen
- Ryhmätyöskentely
- Tekninen kontribuutio - koodi tai asiantuntijuus
- Valmistautuminen asiakastapaamisiin ja toiminta asiakaspalaverissa
- Työtuntien määrä ja tasaisuus sekä merkintöjen asiallisuus

Katso tarkemmat kriteerit [arvostelumatriisista](https://docs.google.com/spreadsheets/d/1fMmlOMQDZMRFMCbgGtPuXB1n9w3hXnmotDZO3n_TdUI/edit#gid=428576263)

## Projektin kulku

- Noudatetaan Scrum-henkistä prosessia.
- Kahden viikon välein asiakastapaaminen (Sprint Review), jonka jälkeen Sprint Planning ja Sprint Retrospective.
- Mahdollisuuksien mukaan päivän aluksi Daily Scrum.
- Ensimmäinen viikko ns. nollasprintti, josta lisää alempana.
- Tarkastellaan jatkuvasti prosessin toimivuutta Sprint Retrospectivessä. 
  - Transparency, Inspect & Adapt! 
- Ensimmäisessä asiakastapaamisessa pyritään määrittelemään [Minimum Viable Product](https://en.wikipedia.org/wiki/Minimum_viable_product), koska tarkoituksena on saada ohjelma tuotantoon mahdollisimman nopeasti.

### Nollasprintti

Perustakaa **heti** jonkinlainen yhteinen TODO-lista. Kirjatkaa sinne nollasprintin tehtävät. Siellä tulisi olla ainakin nämä:
- [ ] Slack, tai vastaava keskustelualusta pystyyn.
  - Ohjaajalle kutsu
- [ ] Product backlogin laatiminen
- [ ] Sprint Task Board / Sprint backlog (fyysinen tai sähköinen)
- [ ] Tuntikirjanpito, josta näkee jokaiseen viikkoon käytetyt tunnit opiskelijoittain
- [ ] Luokaa GitHub-organisaatio ja repository
  - [README standardin mukaiseksi](https://guides.github.com/features/wikis/), lisäksi "päärepoon" linkit muihin repositorioihin, backlogiin sekä sovellukseen.
- [ ] Sopikaa käytettävät teknologiat
  - Huom! Asiakkailla voi olla mielipide käytettävistä teknologioista tai koodauskäytännöistä.
- [ ] CI- ja staging-ympäristö mahdollisimman nopeasti pystyyn
- [ ] Branching-käytännöistä sopiminen
  - Hyvä käytäntö on pitää master-haarassa vain tuotantokelpoista (deployable) koodia. Näin voidaan aina siirtää koodi staging-palvelimelle (ja myöhemmin tuotantoon.)
  - Ohtuprojektien [OpenShift-konttialustaa](https://github.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/tree/master/openshift) voi käyttää staging-ympäristönä, erityisesti jos asiakas HY:llä
- [ ] Koodauskäytännöt
  - Sopikaa Definition of Done
  - Pitää olla sellainen, että DoDin kriteerit täyttävä story voitaisiin viedä sellaisenaan tuotantoon!
  - Huom: DoD:ia voi päivittää tiukemmaksi projektin edetessä
- [ ] Valituilla teknologioilla toteutettu "hello world" / [Walking skeleton](http://wiki.c2.com/?WalkingSkeleton) -sovellus staging-ympäristöön. 
  - _"A Walking Skeleton is a tiny implementation of the system that performs a small end-to-end function"_

## Kurssin vaatimuksia

### Backlogit

- **DEEP** Product Backlog pitää olla [DEEP](https://www.romanpichler.com/blog/make-the-product-backlog-deep/).
- **User storyt** Vaatimukset User Story -muodossa. INVEST on tärkeä!
- **Hyväksymiskriteerit** Storylla pitää olla hyväksymiskriteerit (Acceptance critieria), jos se on Product Backlogissa korkealla (=tulee kohta tehtäväksi). Kriteerit kannattaa käydä läpi koko tiimin ja asiakkaan kanssa, vaikka kaikkia ei tarvitse kirjoittaa asiakkaan läsnäollessa. Lue hyvistä käytännöistä alla.
  - lue lisää hyväksymiskriteereistä omasta [ohjeestaan](https://github.com/ohtu-ohjaajat/opiskelijalle/blob/master/ohje-hyv%C3%A4ksymiskriteerit.md)
- **Storyjen seuranta** Seurataan missä sprintissä valmistuneen storyn tekeminen on aloitettu. Tarkoituksena on, että kaikki storyt olisivat valmiita samassa sprintissä kuin ne on aloitettu, mutta metriikoiden optimointi ei saa missään nimessä olla itsetarkoitus — storyn pitää olla aidosti laadukkaasti tehty ja muilta osin valmis ennen kuin se lasketaan tehdyksi.

### Scrum-tapaamiset

- Sprint planning 
- Sprint review
- Retrospektiivi
- Daily

### Koodi

- **Open source**
- **GitHub** Ohjaajan päästävä näkemään lähdekoodi ja commitit

### Deployment environments

  - **Staging-ympäristö** mahdollisimman tuotannon kaltainen.
     - ks [ohtuprojektin OpenShift](https://github.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/tree/master/openshift)
  - **Tuotantoon!** Tuotos täytyy saada tuotantoon jo projektin aikana, mielellään mahdollisimman nopeasti.
  - **CD** Tuotantoon päivityksen pitää olla [tehokasta ja/tai jatkuvaa](https://puppet.com/blog/continuous-delivery-vs-continuous-deployment-what-s-diff)
  - **Demot stagingista tai tuotannosta** Asiakkaalle demotaan ainoastaan vain ja ainoastaan staging- tai tuotantopalvelimelta, ei omalta koneelta.
- **Testaus**
  - Automaatiotestauksen pitää olla riittävän kattava ja hyvin toteutettu
  - Kaikkien yksikkötestien ajaminen on hyvä kestää korkeintaan muutaman minuutin. Korkeamman tason testien, kuten UI-testien ajo saa kestää pidempään, eikä niitä tarvitse ajaa yhtä usein.
  - Erittäin tärkeä konsepti on ns. testipyramidi http://martinfowler.com/bliki/TestPyramid.html. Parin minuutin lukeminen voi säästää kymmeniä tunteja aikaa.
  - Testikoodin laatu tulee mielellän olla yhtä hyvää kuin muunkin koodin.

### Työtunnit

- **Tuntikirjapito** Kurssin aikana seurataan tuntimääriä. Työtunneiksi lasketaan myös tapaamiset, esim. tiimin kanssa yhdessä lounastamiset ja itseopiskelu.
  - Kirjaukset tehdään [timelogs](https://study.cs.helsinki.fi/projekti/timelogs) -palvelussa kuluvan sprintin aikana - mieluiten päivittäin.
  - Lähtökohtaisesti *työtunteja ei voi merkitä edelliselle sprintille*, vaan ne jää silloin merkitsemättä.
  - Huomaathan, että sprintti vaihtuu tasan kello 0:00.
- **Tasaisuus** Ehdottomana vaatimuksena työmäärien tasaisuus viikkotasolla, eli tuntimäärän kuuluu olla viikosta toiseen suunnilleen sama. Sairastumiset yms. neuvoteltava poikkeus. Ilmoita reilusti etukäteen jos tiedät, että osallistumisesi viikon töihin estyy jollain tapaa.
- **Kurssi = 200h** Vaatimuksena noin 200 tuntia työtä koko kurssin aikana, mikä on noin 15-16 tuntia viikossa.

### Sekalaisia vaatimuksia

- **Commit esiin** Varmista että committisi näkyvät GitHubissa oikein. Ks. [ohje](https://github.com/mluukkai/ohjelmistotuotanto2018/wiki/miniprojektin-arvosteluperusteet#commitit-kadoksissa)
- **Kunnioita** kanssaopiskelijoitasi, kyseessä ei ole yksilökurssi. 
  - Käyttäydy samojen standardien mukaan kuin olisit töissä.
  - Sovittuihin yhteisiin tapaamisiin pitää tulla. Myöhästymisistä, esteistä yms. pitää ilmoittaa ajoissa.
- Siiloutumista tulee välttää. Koodikatselmoinnit!
- **Arvot ja käytännöt**
  - Ryhmän sovittava yhteiset käytännöt. Ks. [esimerkki](https://github.com/ohtu-ohjaajat/opiskelijalle/blob/master/tiimin_kaytanteet.md)
- **Asiakastapaamiset**
  - **Tilavaraukset** Ensimmäisen asiakastapaamisen jälkeen ryhmä on vastuussa asiakastapaamisten järjestämisestä.
  - **Agenda** Asiakastapaamisiin kannattaa luoda melko yksityiskohtainen agenda ja se on hyvä lähettää asiakkaalle ennen varsinaista tapaamista.
  - **Roolit** Kiertävä puheenjohtajan roolit. Lisäksi ainakin kirjuri ja demovastaava. 
  * **Muistiinpanot** Asiakastapaamisista tehtävä muistiinpanot. Keskusteltuja asioita ei saa jättää muistin varaan vaan ne tulee kirjata backlogille.
  - [Vinkkejä asiakaspalaveriin](ohjeita-asiakaspalaveriin.md)
- **Retrospektiivit dokumentoitava**
  - Ohjaajan oltava paikalla jokaisessa retrospektiivissä. Doodlaus tiimin vastuulla.
- **"Kamerapakko"**
  - Etäyhteydellä tapahtuvissa asiakastapaamisissa, retrospektiiveissä ja dailyissä on pidettävä kamera päällä.

## Projektin tavoite

- Ryhmätyötaitojen harjoittelu.
  - Tehtävien ja työn jakaminen
  - Kommunikaatio
  - Organisointi
- Ohjelmistotuotantomenetelmien käytännön harjoittelu.
  - Yhteisen codebasen kanssa työskentely (git)
  - Deployment
- Ketterät toimintatavat
  - Transparency
  - Inspect & Adapt
- [Miksi Scrum?](learning-goals-of-software-engineering-lab.md) Kirjoitus ohtuprojektin tavoitteista ja miten scrum auttaa niiden saavuttamisessa.

## Vinkkejä
- Töitä kannattaa tehdä mahdollisimman paljon muiden kanssa fyysisesti samassa tilassa.
- Sprintin aikana on hyvä olla tekeillä ainoastaan 2-3 storya kerrallaan. Tällöin storyt valmistuvat varmemmin. Muutama valmis > melkein kaikki kesken.
- Ohjelmiston arkkitehtuuriin kannattaa käyttää paljon huomiota ja jättää sprinttiin aikaa tehdä parannuksia sisäiseen laatuun. Puhdas User Story -hikipaja, jossa keskitytään ainoastaan mahdollisimman nopeaan ominaisuuksien tuottamiseen ei pitkällä tähtäimellä tuota hyviä tuloksia.
- Voi olla hyvä idea hahmotella hyväksymiskriteerit ennen varsinaista asiakastapaamista. Kannattaa kuitenkin jättää hieman pureskeltavaa kriteereihin ennen tapaamista.
- On hyvä käytäntö pitää agenda näkyvillä asiakastapaamisen aikana, jotta keskustelu saadaan pysymään paremmin aiheessa. Käydään kokouskäytäntöjä tarpeen mukaan läpi yhdessä.
- [Vinkkejä asiakaspalaveriin](ohjeita-asiakaspalaveriin.md)
- [ohtuprojektin OpenShift-konttialusta](https://github.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/tree/master/openshift)
- Eräs hyväksi havaittu sovellus retrojen pitoon on <https://retrotool.io/>

## Roolit

[Rooleista](roolit.md)

## Käytännön ohjeita

- **Tilat:** 
  - DK107 eli "Luola" ensisijaisesti ohtuprojektiryhmien käytössä
  - Myös muita saleja voi käyttää: esim B221 (yläpaja), BK107 (alapaja), C221, DK108 (haxo)
- **Sali asiakaspalaverille:** Palaverin voi pitää tiimitilassa tai "neukkarissa". Pyydä tilavaraus Luukkaiselta (email tai discord) tai kysy ohjaajalta. Myös kirjaston ryhmätyötiloja voi käyttää. Varausohjeet: http://www.helsinki.fi/kirjasto/fi/asioi/tyoskentele-kirjastossa/ryhmatyotilat/

## Ohtuprojekti staging

Staging nykyään Tiken [OpenShift](https://github.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/blob/master/openshift/README.md)-klusterissa

## Parhaat käytänteet

Aina ei tarvitse keksiä pyörää uudelleen - katso vinkkejä [täältä](best-practices.md) tai [edellisten ryhmien repoista](history/).
