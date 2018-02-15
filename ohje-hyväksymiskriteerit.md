## Hyväksymiskriteerien parhaat käytännöt

### User story formaatti

User Storyn tuttu formaatti kertoo korkealla tasolla mitä toteutettava toiminnallisuus pitää sisällään ja mitä lisäarvoa se tuottaa käyttäjälle.

_“As a [type of user],  
I want to [some goal]   
(so that [some reason])”_

Jos otetaan esimerkiksi jo kliseeksi muodostunut verkkokauppa, yksi User Story voisi olla vaikka 
 
_“As a shopper,  
I want to be able to add products to my shopping cart  
so that I can later order them.”_

### User storyn laajuus

Nyt herää kysymys mitä Story oikeasti pitää sisällään, eli mikä on Storyn “laajuus” (engl scope)
  - Näkeekö shopper jostain Shopping cartin sisällön, tuotteiden määrän tai niiden yhteissumman?
  - Voiko tuotteita lisätä tai poistaa? 
  - Kuinka kauan cart muistaa sinne kerätyt ostokset? 
  - Voiko tuotteen lisätä, vaikka sitä ei olisi varastossa? 
  - Kerätäänkö sisällön muutoksista dataa analytiikkaa varten? 

On tärkeätä että tiimin kaikki jäsenet ja asiakas ovat yhtä mieltä storyjen laajuudesta.
Jos storyjen laajuudesta ei ole yksimielisyyttä, on mahdotonta sanoa kuinka “valmiina” asiat ovat tai projektin valmiuden asteesta on useita näkemyksiä.
Ja se taas ei ole ollenkaan hyvä asia, ketterän yksi perusperiaate on transparency, eli läpinäkyvyys ja sitä taas ei voida saavuttaa jos tiimiläiset eivät jaa yhteistä näkemystä tärkeistä asioista kuten projektin valmiusasteesta.

### Hyväksymiskriteerit

User Storyn laajuus määritellään hyväksymiskriteereillä. Storya voidaan pitää valmiina vain, jos kaikki sille asetetut kriteerit täyttyvät. 
Hyväksymiskriteerit jaetaan kahteen yleisesti ohjelmistojen vaatimusmäärittelyssä käytettävään luokkaan:
- Toiminnalliset vaatimukset 
  - _"Mitä tekee"_
- Ei-toiminnalliset vaatimukset
  - _"Miten tekee"_
  - Suorituskyky
  - Tietoturva
  - Datan tallennus myöhempää käyttöä varten
  - Dokumentaatio
  - Ulkonäkö

Hyvä kriteeri ei ole liian yksityiskohtainen, mutta ei myöskään liian ylimalkainen. 
Taso on sopiva silloin, kun kriteereitä joudutaan muuttamaan vain harvoin sprintin aikana ja storyn valmistuessa väärinymmärryksiä ei ole tapahtunut. 
Tavallisesti kannattaa jättää pois tarkat ulkonäköön liittyvät seikat. Esimerkiksi kriteeri:

_“The purchaser will see a big red box containing a message with text ‘You have insufficient funds’  
when clicking the purchase button if they cannot afford the purchase.”_ 

on parempi muotoilla 

_“The purchaser will be informed that they have insufficient funds  
when they are attempting to make a purchase they cannot afford.”_

Hyväksymiskriteerien tulee ehdottomasti olla sellaisia, että on kaikkien tiimin jäsenten ja mielellään myös asiakkaan helppo testata täyttyykö kriteeri. 
**Optimaalisessa tapauksessa kriteerit ovat automaattisesti suoritettavia testejä.**
Jos kriteerien toteutuminen testataan manuaalisesti, tulee kriteerien olla siten ilmaistuja, että kaikki tiimiläiset (ja asiakas) ovat yksimielisiä kriteerien täyttymisen suhteen.

### Hyväksymiskriteerien formaatti

Kriteerien kirjaamiseen on olemassa useita eri tapoja, mutta tässä on esiteltynä niistä kaksi yleisesti käytettyä tapaa. Ensimmäinen tapa, Given-When-Then/Gherkin, on tuttu Ohjelmistotuotanto-kurssilta. Kriteereitä kirjoitettaisiin tällöin esimerkiksi


**Scenario**: The shopper sees the total amount of products in their shopping cart<br/>
&nbsp;&nbsp;Given a shopper who has 2 products in their shopping cart<br/>
&nbsp;&nbsp;When the shopper adds an item to the cart<br/>
&nbsp;&nbsp;Then the shopper sees that they have 3 products in their shopping cart<br/>

**Scenario**: The shopper sees the total price of the products shopping cart<br/>
&nbsp;&nbsp;Given a shopper who has products totaling 8,99 euros in their shopping cart<br/>
&nbsp;&nbsp;When the shopper adds an item costing 20,00 euros to their cart<br/>
&nbsp;&nbsp;Then the shopper sees that the total price of the products in the shopping cart is 28.99 euros<br/>


**Scenario**: The cart is remembered for 3 days<br/>
&nbsp;&nbsp;Given a shopper who last updated their cart 3 days ago<br/>
&nbsp;&nbsp;When the shopper visits the site<br/>
&nbsp;&nbsp;Then the shopper can see their cart unchanged from their last visit<br/>

**Scenario**: The cart is forgotten after 4 days<br/>
&nbsp;&nbsp;Given a shopper who last updated their cart 4 days ago<br/>
&nbsp;&nbsp;When the shopper visits the site<br/>
&nbsp;&nbsp;Then the cart will be empty<br/>

**Scenario**: The cart is remembered for another 3 days after updating it<br/>
&nbsp;&nbsp;Given a shopper who last updated their cart 3 days ago<br/>
&nbsp;&nbsp;And the shopper updates their cart today<br/>
&nbsp;&nbsp;When the shopper visits the site tomorrow<br/>
&nbsp;&nbsp;Then the shopper can see their cart unchanged from their last visit<br/>

Jos hyväksymiskriteerien kirjoittaminen puuduttaa niin voi googlata esim. [software requirements specification example](https://www.google.fi/search?query=software+requirements+specification+example), 
ihmetellä vaikka [tätä](http://www.cse.chalmers.se/~feldt/courses/reqeng/examples/srs_example_2010_group2.pdf) 60 sivuista speksausdokumenttia,
ja todeta, että ei niiden hyväksymiskriteerien kirjoittaminen olekaan niin paha rasti. :D

Toinen tapa ilmaista kriteerit on lyhyesti otsikkotasolla, jolloin hyväksymiskriteeriksi jää ainoastaan äskeisten esimerkkien Scenario-kohta. Kummankin tavan hyvät ja huonot puolet löytyvät pääpiirteissään alla olevassa taulukossa:


| Given-When-Then/Gherkin | Pelkät otsikot |
| ----------------------- | -------------- |
| + Kriteereistä saadaan suoraan automatisoidut testit käyttämällä jotain Gherkiniä tukevaa ohjelmistokehystä. | + Nopea kirjoittaa |
| + Hyvin eksplisiittinen, vähentää väärinymmärryksiä. | + Ohjelmiston toimintaa ei tarvitse suunnitella etukäteen yhtä tarkasti | 
| - Vaatii enemmän etukäteissuunnittelua | - Koska toimintaa ei tarvitse suunnitella samalla tarkkuustasolla, voi lopputulos olla ratkaisevasti erilainen kuin mitä asiakas toivoi |
| - Ei sovellu hyvin ei-toiminnallisten vaatimusten määrittelyyn. | - Voivat jäädä epämääräisiksi |


Joskus hyväksymiskriteerien erittäin suuri määrä saattaa kieliä siitä, että story on liian laaja. Tällöin kannattaa ehdottomasti harkita storyn jakamista. 
Esimerkkitapauksessamme kannattaisi story ehkä jakaa kahteen osaan. Ensimäinen kattaisi kenties ostoskorin perustoiminnallisuuden, eli suunnilleen skenaariot

&nbsp;&nbsp;**Scenario**: The shopper sees the total amount of products in their shopping cart<br/>
&nbsp;&nbsp;**Scenario**: The shopper sees the total price of the products shopping cart<br/>

Jälkimmäiseen jäisivät ne osat, jotka ottaisivat tarkemmin kantaa siihen, kuinka kauan korin sisältö säilyy

&nbsp;&nbsp;**Scenario**: The cart is remembered for 3 days<br/>
&nbsp;&nbsp;**Scenario**: The cart is forgotten after 4 days<br/>
&nbsp;&nbsp;**Scenario**: The cart is remembered for another 3 days after updating it<br/>
