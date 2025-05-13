# Yliopisto-kirjautuminen

Jos projekti vaatii yliopiston kirjautumista vaihtoehdot ovat käytännössä SAML-pohjainen Shibboleth kirjautuminen tai modernimpaan OAuth:iin ja OpenID Connect:iin (OIDC) perustuva ratkaisu. Tutustutaan seuraavassa modernimpaan ratkaisuun.

Muutetaan sovellusta siten, että jos käyttäjä ei ole kirjautunut, ei nollausnappia näytetyä:

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/openshift/images/k9.png?raw=true" width="600">

Kun käyttäjä kirjautuu, on nollaaminen mahdollista.

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/openshift/images/k10.png?raw=true" width="600">

Koodi on kokonaisuudessaan GithHubissa haarassa [login](https://github.com/mluukkai/openshift-demo/tree/login?tab=readme-ov-file).

Katsotaan ensin kirjautumista frontendin kannalta.

Sovellus kysyy aina etusivulle tultaessa kirjaantuneen käyttäjän tietoja tekemällä HTTP GET -pyynnön osoitteeseen `/api/user`. 

Pyyntä on toteutettu Reactille tyypillisellä tavalla, eli `useEffect`-hookissa:

```js
  useEffect(() => {
    axios.get('/api/user')
      .then(response => {
        setUser(response.data)
      })
      .catch(error => {
        console.log('not logged in')
      })
  }, [])
```

Jos käyttäjä on kirjautunut, palauttaa `/api/user` käyttäjän, jotka talletetaan tilaan `user` kutsumalla `setUser`.

Jos käyttäjä ei ole kirjautunut, on tilan `user` arvona `null`. Eri napit näytetään tämän perusteella.

Sisään- ja ulkoskirjautumisen napinkäsittelijät ovat seuraavassa:

```js
  const onLogin = () => {
    window.location.href = '/api/login'
  }

  const onLogut = async () => {
    await axios.post('/api/logout')
    setUser(null)
  }
```

Sisäänkirjautuessa vaihdetaan selaimen osoiteriville `/api/login`, tämä saa aikaan sen, että selain tekee HTTP GET -pyynnön tähän osoitteeseen. Kirjautuminen on pakko tehdä näin sensijaan että pyyntö tehtäisiin JavaScript-koodista (lisätietoa esim. [täällä](https://community.auth0.com/t/cors-error-when-initiating-silent-auth-requests/103208)).

Kirjautumisen yhteydessä backend tekee selaimelle uudelleeohjauksen sovelluksen juuriosoitteeseen `/`, tämän ansiosta sovellus tekee heti pyynnön `/api/user` ja saa kirjautuneen käyttäjän tiedot.

Uloskirjautuminen tapahtuu tekemällä HTTP POST -pyyntö backendiin ja asettamalla tilan`user` arvoon `null`.

Tarkastellaan seuraavaksi backendin koodia. Backend käyttää kirjautumisen apuna [passport](https://www.passportjs.org/)-kirjastoa. Jos käytät jotain muuta kuin Node/Expressiä backendin toteutukseen, joudut turvautumaan googleen ja kielimalleihin.

Toteutus edellyttää seuraavien kirjastojen asentamista:

```
passport
express-session
passport-openid
openid-client@5.4.3
ioredis
connect-redis
```

Backend käyttää [Redis](https://redis.io/)-avain-arvo-tietokantaa kirjautuneiden käyttäjien sessioiden tietojen tallettamiseen. Redisistä huolehtiva deployment ja service löytyvät [täältä](https://github.com/mluukkai/openshift-demo/blob/login/manifests/redisdeployment.yaml). Määrittely on suoraviivainen, ainoa huomionarvoinen seikka on käytössä oleva image
`registry.redhat.io/rhel9/redis-7` jota joudumme käyttämään OpenShiftin rajoitusten takia sillä Dockerhubissa oleva image suorittaa koodia root-tilassa ja se aiheuttaa ongelman OpenShiftin oikeuksien suhteen.

Aluksi määritellään Redis, sekä [sessioista](https://www.passportjs.org/concepts/authentication/sessions/) vastaava middleware passportin käyttöön:

```js
const passport = require('passport')
const session = require('express-session')

const SESSION_SECRET = process.env.SESSION_SECRET
const REDIS_HOST = process.env.REDIS_HOST

const redis = new Redis({
  host: REDIS_HOST,
  port: 6379,
})

app.use(session({
  secret: SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  store: new RedisStore({ client: redis }),
}))

app.use(passport.initialize())
app.use(passport.session())
```

Kirjautumista ja käyttäjänhallintaa varten tarvitaan neljä routea:

```js
app.get('/api/login', passport.authenticate('oidc'))

app.get('/api/login/callback', passport.authenticate('oidc', { failureRedirect: '/' }),  (req, res) => {
  res.redirect('/')
})

app.post('/api/logout', (req, res, next) => {
  req.logout((err) => {
    if (err) { return next(err); }
    res.redirect('/');
  });
});

app.get('/api/user', async (req, res) => {
  console.log('User:', req.user)
  if (req.user) {
    res.json(req.user);
  } else {
    res.status(401).json({ message: 'Unauthorized' });
  }
});
```

Ensimmäinen route ohjaa HTTP GET `/api/login` -pyynnön yliopiston kertakirjaantumispalveluun. Routeista toinen on _takaisunkutsuroute_, jota kertakirjaantuminen kutsuu, kirjaantumisen onnistuessa. Kirjaantumisen yhteydessä passport suorittaa kirjaantumistoimenpiteet (joiden koodiin palaamme kohta), sekä tallettaa tiedon kirjaantuneesta käyttäjästä sessioon, eli käytännössä selaimelle asetettavaan cookieen. Sovellus uudelleenohjataan juurisoitteeseen `/`, joka saa siis aikaan sen, että selain tekee uuden pyynnön osoitteeseen `/api/user` joka kirjaantimisen ansiosta voi palauttaa `req.user`:iin passporton asettaman kirjaantuneen käyttäjän. 

Ulkoskirjutumisen route on yksinkertainen, se kutsuu funktiota [req.logout](https://www.passportjs.org/concepts/authentication/logout/), ja ohjaa selaimen juuriosoiteeseen (joka ei tapauksessamme ole aivan välttämätöntä).

Käynnistyessään sovellus vielä alustaa autentikoinnin kutsumalla erillisessä tiedostossa määriteltyä funktiota:

```js
app.listen(PORT, async () => {
  const { setupAuthentication }  = await import('./oicd.mjs');

  await setupAuthentication()
  // ...
}
```

Oleelliset osat tiedostosta `oicd.mjs` (joka käyttää JavaScriptin ES moduuleita) näyttävät seuraavalta:

```js
const verifyLogin = async (_tokenSet, userinfo, done) => {
  console.log('userinfo', userinfo)

  const user = {
    id: userinfo.sub,
    username: userinfo.uid,
    name: userinfo.name,
  }

  // save user to db if that is required in your app

  done(null, user)
}

export const setupAuthentication = async () => {
  const client = await getClient()

  const object = {
    cn: { essential: true },
    name: { essential: true },
    given_name: { essential: true },
    hyGroupCn: { essential: true },
    email: { essential: true },
    family_name: { essential: true },
    uid: { essential: true },
  }

  const params = {
    scope: 'openid profile email',
    claims: {
      id_token: object,
      userinfo: object,
    },
  }
  
  passport.use('oidc', new openidClient.Strategy({ client, params }, verifyLogin))
}
```

Määrittelyn ytimessä on funktio [verifyLogin](https://www.passportjs.org/concepts/authentication/openid/), joka suoritetaan onnistuneen kirjautumisen yhteydessä. Funktion toinen parametri on kirjautumispalvelun palauttama kirjaantuneen käyttäjän tiedot. Jos käyttäjien tiedot on esim. tarve tallettaa tietokantaan, tallennus tulee tehdä tässä kohtaa jos kirjaantunutta käyttäjää ei vielä tietokannasta löydy. Funktion lopussa kutsutaan kolmatta parametria antamalla kutsissa parametrina ne tiedot joita passportin halutaan palauttavan kutsun `req.user` yhteydessä (jota käytettiin reitin GET `/api/user` käsittelijässä).

Tiedoston [oidc.mjs](https://github.com/mluukkai/openshift-demo/blob/login/server/oicd.mjs) metodissa `getClient` konfiguroidaan kirjaantumispalvelimelle yhteydessä oleva openidClient, joka konfiguroidaan sovelluksen kertakirjautumisjärjestelmään määritellyillä arvoilla.

Kertakirjaantuminen konfiguroidaan osoitteessa <https://sp-registry.it.helsinki.fi/>, painamalla nappia _Add a new OICD relying party_.
- Käyttöoikeuksista [käyttöohje](https://wiki.helsinki.fi/xwiki/bin/view/SO/User%20management/SP%20Registry) sanoo seuraavasti: _If you are not verified as a university employee during login, please contact atk-autentikointi@helsinki.fi after login, so that an administrator can add the necessary user rights for you. If necessary, an administrator can also create local usernames for SP-Registry._

Mallia voi ottaa tiedostoista [conf1.png](https://github.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/blob/master/openshift/images/conf1.png) ja [conf2.png](https://github.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/blob/master/openshift/images/conf2.png)

Sovelluksen käyttämät attribuutit, eli kirjaantumispalvelulta pyydettävät käyttäkohtaiset arvot määritellään välilehdeltä _Attributes_:

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/openshift/images/k12.png?raw=true" width="600">

Selitys attribuuttien merkityksestä on [täällä](https://wiki.helsinki.fi/xwiki/bin/view/SO/User%20management/User%20attributes/). Lisää jonkinlaista ohjeistusta SP-registryn käyttöön on [täällä](https://wiki.helsinki.fi/xwiki/bin/view/SO/User%20management/SP%20Registry/).

Client secretin arvo löytyy välilehden _Technical attributes_ alaosasta.

Salaisuuksien arvot on määritelty sovellukselle tiedostossa `configmap.yaml`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demoapp-config
data:
  DB_URL: postgresql://ohtuprojektitesti:passwordhere@hostnamehere:5432/ohtuprojekti?targetServerType=primary&ssl=true
  OIDC_CLIENT_ID: id_valuesehe_see_the_spregistry
  OIDC_CLIENT_SECRET: secret_secretvaluehere_see_the_spregistry
  OIDC_REDIRECT_URI: https://demoapp-toska-playground.apps.ocp-test-0.k8s.it.helsinki.fi/api/login/callback
  OIDC_ISSUER: https://login-test.it.helsinki.fi/.well-known/openid-configuration
  REDIS_HOST: redis-svc
  SESSION_SECRET: randomsecretstringhere
```

Muutetaan tiedostoa `deployment.yaml` siten, että se antaa samantien kaikki config mapin määrittelemät ympäristömuuttujat podille:

```yaml
    spec:
      containers:
        - name: demoapp
          image: demoapp:login
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: demoapp-config
```

## Sovelluskehitys paikallisella koneella ja kertakirjautuminen

Kertakirjautuminen aiheuttaa pienen haasteen sovelluksen kehittämiselle omalla koneella. Eräs ratkaisu tähän on ohittaa OpenID ja kovakoodata kirjautumsen tiedot backendiin. Tämä on tehty nyt tiedostoon [index.js](https://github.com/mluukkai/openshift-demo/blob/login/server/index.js#L67) seuraavasti:

```js
let loggedIn = false

if (process.env.NODE_ENV !== 'production') {
  const setMocUser = (req, res, next) => {
    if (!loggedIn) {
      return next()
    }

    req.user = {
      "id": "2Q6XGZP4DNWAEYVIDZV2KLXKO3Z4QEBM",
      "username": "mluukkai-test",
      "name": "Matti Luukkainen"
    }
    next()
  }
  
  app.use(setMocUser)

  app.get('/api/login', (req, res) => {
    loggedIn = true
    res.redirect('/');
  })

  app.post('/api/logout', (req, res, next) => {
    loggedIn = false
    res.redirect('/');
  });
}

// normaalisti käytettävät routet
```

Jos sovellus ei ole tuotannossa, eli `process.env.NODE_ENV !== 'production'` määritelläänkin käyttäjän feikkaava middleware sekä endpointit `/api/login` ja `/api/logout`. Kun kirjaantuminen tehdään, asetetaan `loggedIn = true`, ja middleware lisää `req.user `:n arvoksi kovakoodatun käyttäjän.

Esimerkissä on nyt kirjoitettu lähes kaikki koodi yhteen tiedostoon. Tämä ei tietenkään ole ohtuprojektissa järkevää, ja eri asiat, mm. tietokantayhteyksien avaaminen ja erityisesti feikkikirjautuminen on hyvä eriyttää omiin moduuleihin.