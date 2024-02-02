Usein kysyttyjä kysymyksiä

## Kontti toimii lokaalisti mutta ei Openshiftissä

### Luku tai kirjoitusoikeuksiin liittyvät virheet ###

Käyttöoikeusongelmien kirjo voi olla moninainen riippuen kontissa ajettavasta teknologiasta. Tähän liittyy yleensä jonkinlainen tiedostojen käyttöön liittyvä virheilmoitus: 'not writable', 'permission denied'. Kun kontti käynnistetään OpenShiftissa, arvotaan kontin käyttäjäksi satunnainen UID ja käyttäjälle tulee asettaa oikeudet suorittaa/ kirjoittaa kontin sisällä. Jos käsket Dockeria luomaan työhakemiston ja kopioimaan sinne projektikansiosi sisällön (muista .dockerignore),
``` bash
WORKDIR /app

COPY . .
```
voit asettaa käyttöoikeudet Dockerfilessa yksinkertaisella (joskaan ei kovin mallikelpoisella) tavalla,

```bash
RUN chmod -R 777 *
```
jonka jälkeen käyttäjän oikeudet ulottuvat kansion /app alikansioihin ja tiedostoihin. Esimerkkitapauksena mainittakoon Pythonin virtuaaliympäristö, jota saatetaan yrittää suorittaa jossain muussa hakemistossa kuin /app, jolloin venv-moduuli ei välttämättä käynnisty laisinkaan. Sopivaa ympäristömuuttujaa käyttämällä virtuaaliympäristö voidaan asettaa suoritettavaksi /app kansion sisällä. Ympäristömuuttujia voi esitellä kontille Dockerfilen käskyllä ENV. Seuraava esimerkki toi ratkaisun ainakin erääseen Poetry-projektiin:
```bash
ENV POETRY_VIRTUALENVS_IN_PROJECT=true
``` 

## Näin saan konsoliyhteyden klusterilla olevaan tietokantaan

Antin ryhmä kirjoittaa ohjeen
