## OpenShift-konttialustan käyttö staging-ympäristönä

Tämä ohje olettaa, että hallitset jossain määrin Dockeria, esim. vähintään luvun _Docker basics_ verran kurssilta [DevOps with Docker](https://courses.mooc.fi/org/uh-cs/courses/devops-with-docker). Docker-kurssi jatkuu vielä 16.6. asti, seuraava versio kurssista alkaa toisen periodin alussa. Jokaisen käpistelijän kannattaa ehdottomasti suorittaa kurssi!

Jos käytät fuksiläppäriä ja koneellasi ei ole jo Dockeria asennettuna, seuraa [tätä](https://version.helsinki.fi/cubbli/cubbli-help/-/wikis/Docker) ohjetta!

### Esimerkkisovellus

Seuraavassa asennetaan yliopiston tietotekniikkakeskuksen OpenShift-klusterille Reactilla ja NodeJS:llä toteutettu SPA-sovellus, jonka koodi löytyy [GitHubista](https://github.com/mluukkai/openshift-demo).

Sovellus on hyvin yksinkertainen laskuri. Laskurin arvo on talletettu Postgres-tietokantaan, johon backend on yhteydessä [Sequelize](https://sequelize.org/)-kirjaston avulla. Frontend sisältää napit laskurin kasvattamiseen sekä nollaamiseen. Käytössä ovat siis kurssilta [Full stack open](https://fullstackopen.com/) tutut teknologiat.

Projektiin on määritelty GitHub Action -workflow, joka luo projektista Docker-imagen ja pushaa sen Dockerhubiin. Sama image sisältää sekä backendin, että frontendin.

Sovellus on GitHubissa siinä tilanteessa mihin tämä tutoriaali päättyy. Alkutilanne on branchissa [start](https://github.com/mluukkai/openshift-demo/tree/start). Koodissa ei muutoksia ole, mutta tutoriaalin aikana tehdyt konfiguraatiot puuttuvat vielä haarasta start.

### OpenShift

Käytössämme on Tietotekniikkakeskuksen [OpenShift](https://devops.pages.helsinki.fi/guides/platforms/tike-container-platform.html)-klusteri. OpenShift on [Kubernetes](https://github.com/mluukkai/openshift-demo/blob/main/.github/workflows/main.yaml)-klusteri tietyin lisämaustein.

Kubernetes on melko monimutkainen olio, kurssi [DevOps with Kubernetes](https://devopswithkubernetes.com/) käsittelee aihetta laajasti. Seuraavassa käydään läpi minimioppimäärä yksinkertaisen sovelluksen tarpeisiin.

Ytimessä olevan Kuberneteksen lisäksi OpenShift sisältää mm. graafisen käyttöliittymän, jonka kautta konfiguraatioita on mahdollista tehdä, mutta se **ei ole suositeltua** sillä näin päädytään usein hallitsemattoman epämääräisiin konfiguraatioihin. On suositeltavaa pitäytyä määrittelyissä mahdollisimman "puhtaassa" Kuberneteksessa, ja näin tulemme seuraavassakin tekemään.

**Eli älä määrittele mitään OpenShiftin käyttöliittymän kautta. Jos teet näin, teknistä tukea ei kurssin puolesta ole luvassa.**

Käytämme klusteria yksinomaan komentoriviltä, komennon [oc](https://docs.redhat.com/en/documentation/openshift_container_platform/4.11/html/cli_tools/openshift-cli-oc) avulla. `oc` toimii samoin kun Kubernetesin [kubectl](https://kubernetes.io/docs/reference/kubectl/), mutta se sisältää muutamia OpenShift-spesifejä komentoja.

Asenna nyt koneellesi `oc` [tämän ohjeen](https://devops.pages.helsinki.fi/guides/platforms/tike-container-platform.html#openshift-client) mukaan. Kannattaa myös ehdottomasti konfiguroida [tabulaattoritäydennys](https://docs.redhat.com/en/documentation/openshift_container_platform/4.9/html/cli_tools/openshift-cli-oc#cli-enabling-tab-completion).

Oletetaan nyt, että `oc` asennettu. Jotta yhteys klusteriin toimisi, on oltava Eduroamissa tai HY:n vpn:ssä. 

Kirjaudu klusterille suorittamalla komento `oc login -u <username> https://api.ocp-test-0.k8s.it.helsinki.fi:6443`.

Jostain syystä kirjautuminen ei kaikilla toimi. Jos käy näin, kirjaudu OpenShift-webkonsoliin <https://console-openshift-console.apps.ocp-test-0.k8s.it.helsinki.fi> ja valitse _Copy login command_:

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/openshift/images/k1.png?raw=true" width="600">

Kirjaudu uudelleen, ja saat toimivan kirjautumiskomennon:

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/openshift/images/k5.png?raw=true" width="600">

Kirjautumisen jälkeen voidaan vaikkapa suorittaa komento `oc status`, joka kertoo että olemme onnistuneesti kirjautuneet, omassa tapauksessani projektiin _toska-playground_:

```bash
$ oc status
In project toska-playground on server https://api.ocp-test-0.k8s.it.helsinki.fi:6443
```

Esimerkissä on käytössä projekti `toska-playground`. Ohtuprojekteissa käytetään projektia `ohtuprojekti-staging`. Projektista on olemassa sekä tuotanto- että testipuoli. Kysy ohjaajaltasi kumpaa ryhmäsi käyttää. 

Testipuolen osoite on https://api.ocp-test-0.k8s.it.helsinki.fi:6443 ja tuotantopuolen https://api.ocp-prod-0.k8s.it.helsinki.fi:6443, eli kirjautuessa käytä oikeaa osoitetta! Tuotantopuolen web-konsolin osoite on <https://console-openshift-console.apps.ocp-prod-0.k8s.it.helsinki.fi>

### Pod ja deployment

OpenShiftissa sovelluksen paketoinnin perusyksikkö on [podi](https://kubernetes.io/docs/concepts/workloads/pods/) (engl. pod). Podit ovat Kubernetes-ympäristön perusyksiköitä, ja ne sisältävät yleensä yhden Docker-imagen. Joissain tilanteissa podissa voi olla useita imageja, mutta tämä on yleensä mielekästä vain, jos imaget ovat tiiviisti yhteydessä toisiinsa, esimerkiksi jakamalla verkkoyhteyden tai tallennustilan.

Podeja ei yleensä laiteta klusteriin suoraan. Näiden sijaan käytetään [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)-objekteja.

Määritellään deployment sovellustamme varten hakemistoon `manifests` sijoitettavaan tiedostoon `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapp-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoapp
  template:
    metadata:
      labels:
        app: demoapp
    spec:
      containers:
        - name: demoapp
          image: mluukkai/demoapp:1
          ports:
            - containerPort: 3000
          env:
           - name: DB_URL
             value: postgresql://ohtuprojektitesti:passwordhere@hostnamehere:5432/ohtuprojekti?targetServerType=primary&ssl=true
```

Avaimen `spec` arvona määritellään deploymentin hallitseman podin (tai podien jos `replicas` on suurempi kuin 1) kontit. Tapauksessamme on yksi kontti, jonka käyttämä image on `mluukkai/demoapp:1`. Kontille on annettu sen käyttämä tietokantaosoite määrittelemällä ympäristömuuttuja `DB_URL`. Lue [täältä](https://github.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/tree/master/openshift#tietokannan-hankkiminen) miten saat Postgres-tietokannan jos projektisi sellaista tarvitsee!

Deploymentin määrittelevä yaml-tiedosto näyttää monimutkaiselta. Osa määrittelyn sisällöstä on sovelluksesta riippumatta suunilleen sama, tärkein osuus on juurikin avaimen `spec` arvona. Metadatassa oleva `name: demoapp-dep` määrittelee deploymentin nimen.

Deployment luodaan klusterille seuraavalla komennolla ($ on komentokehote):

```bash
$ oc apply -f manifests/deployment.yaml
```

Deployment näyttää onnistuneen:

```bash
$ oc get deployments
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
demoapp-dep   1/1     1            1           16s
```

Deploymentin on siis tarkoitus käynnistää yksi podi. Katsotaan miltä podien tilanne näyttää:

```bash
$ oc get pod
NAME                           READY   STATUS    RESTARTS   AGE
demoapp-dep-5dbb664966-p6kfh   1/1     Running   0          25s
```

Podi on käynnissä.

Testataan nyt sovellusta. Sovellus ei näy vielä klusterin ulkopuolelle, mutta pääsemme siihen käsiksi klusterin sisältä podin IP-osoitteen avulla. Saamme IP-osoitteen selville komennolla `oc describe pod <pod>`:

```bash
$ oc describe po demoapp-dep-7499f5c5bd-8lm9p
Name:             demoapp-dep-7499f5c5bd-8lm9p
Namespace:        toska-playground
Priority:         0
Service Account:  default
Node:             worker-1.ocp-test-0.k8s.it.helsinki.fi/128.214.137.138
Start Time:       Tue, 06 May 2025 17:00:38 +0300

...

Status:           Running
IP:               10.12.2.177
```

Käynnistetään klusterin sisälle [curl](https://curl.se/)-komennon sisältävä Docker [image](https://hub.docker.com/r/curlimages/curl):

```bash
$ oc run curlimage --image=curlimages/curl --restart=Never --command -- sleep infinity
$ oc exec -it curlimage sh
```

Olemme nyt podin sisällä, ja voimme kokeilla ottaa yhteyttä sovellukseen tekemällä GET-pyynnön testirajapintaan _api/ping_:

```bash
$ curl 10.12.2.177:3000/api/ping
{"message":"pong"} 
```

Jostain syystä GET-pyyntö laskurin arvon palauttavaan rajapintaan _api/counter_: ei toimi:

```bash
$ curl 10.13.2.114:3000/api/counter
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Error</title>
</head>
<body>
<pre>Internal Server Error</pre>
</body>
</html>
```

Tutkitaan sovelluksen lokia komennolla `oc logs <pod>`:

```bash
$ oc logs demoapp-dep-68c95df467-w4glg

> demoapp@0.0.0 prod
> NODE_ENV=production node server/index.js

Server running on http://localhost:3000
Unable to connect to the database: ConnectionError [SequelizeConnectionError]: no pg_hba.conf entry for host
"128.214.137.139", user "ohtuprojektitesti", database "ohtuprojekti", SSL encryption
```

Tietokantayhteyden suhteen näyttää olevan jotain häikkää. Ja kun virheilmoitusta lukee tarkemmin, huomaa, että sovelluksen käyttämä tietokantaurl on väärä, tietokannan nimi on _ohtuprojektitesti_, ei _ohtuprojekti_ kuten konfiguraatio vahingossa määritteli.

Korjataan konfiguraatio ja päivitetään klusteri komennolla `apply -f manifests/deployment.yaml`. Katsotaan lokia uudelleen:

```bash
$ oc logs demoapp-dep-6746c5d5dc-jjp88
> demoapp@0.0.0 prod
> NODE_ENV=production node server/index.js

Server running on http://localhost:3000
Executing (default): SELECT 1+1 AS result
Connected to the database
Executing (default): SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' AND table_name = 'counters'
Executing (default): SELECT i.relname AS name, ix.indisprimary AS primary, ix.indisunique AS unique, ix.indkey AS indkey, array_agg(a.attnum) as column_indexes, array_agg(a.attname) AS column_names, pg_get_indexdef(ix.indexrelid) AS definition FROM pg_class t, pg_class i, pg_index ix, pg_attribute a WHERE t.oid = ix.indrelid AND i.oid = ix.indexrelid AND a.attrelid = t.oid AND t.relkind = 'r' and t.relname = 'counters' GROUP BY i.relname, ix.indexrelid, ix.indisprimary, ix.indisunique, ix.indkey ORDER BY i.relname;
```

Hyvältä näyttää! Yritetään uudelleen

```bash
$ curl 10.13.2.114:3000/api/counter
curl: (7) Failed to connect to 10.13.2.114 port 3000 after 18688 ms: Could not connect to server
```

Yhteys ei enää yllättäen toimi. Syynä on se, että podin IP-osoite ei ole pysyvä. Tarkastetaan komennolla `oc describe pod <po>` uusi osoite, ja curlataan tähän osoitteeseen:

```bash
$ curl 10.15.3.15:3000/api/counter
{"id":1,"value":0}
```

Sovellus näyttää toimivan!

### Deklaratiivinen vs imperatiivinen määrittely

Veimme sovelluksen tuotantoon määrittelemällä yaml-tiedoston ja antamalla komennon 
`oc apply -f manifests/deployment.yaml`

Curl-podi taas käynnistettiin seuraavasti

```bash
$ oc run curlimage --image=curlimages/curl --restart=Never --command -- sleep infinity
```

Mistä on kyse? Ensimmäinen tapa, missä käytettiin yamlia on ns. [deklaratiivinen](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/#declarative-object-configuration) konfigurointitapa, joka on jossain määrin ehkä haastavampi, mutta ehdottomasti suositeltava tapa. Etuna on mm. se, että konfiguraatiot on mahdollista tallettaa versionhallintaan.

Jälkimmäinen komento `oc run  ...` taas edustaa [imperatiivistä](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/#imperative-object-configuration) tyyliä Kubernetes-objektien luomiseen. Imperatiivisen tyylin ongelma on se, että klusterin tila ei pysy samalla tavalla hallittavasti näkyvillä kuin vaikkapa versionhallintaan tallennetuissa yaml-manifesteissä. Imperatiivista tyyliä kannattaa käyttää lähinnä yksinkertaisiin tilanteisiin, mm. esimerkkimme tapaan debugatessa.

Käytettäköön siis deklaratiivista tyyliä, joka on myös tämän hetkisen teollisen parhaan käytänteen [GirOpsin](https://www.redhat.com/en/topics/devops/what-is-gitops) taustalla. Tätäkin teemaa käsitellään kurssilla [DevOps with Kubernetes](https://devopswithkubernetes.com/).

Laajennetaan ohtuprojektimanifestia:

**Älä määrittele mitään OpenShiftin käyttöliittymän kautta tai imperatiivisin käskyin. Jos teet näin, teknistä tukea ei kurssin puolesta ole luvassa.**

### Service

Podin IP-osoite siis näyttää vaihtuvan. Tämä johtuu siitä, että podi itseasiassa luodaan aina uudelleen kun siihen tehdään minkäänlaisia muutoksia, podit eivät siis ole pysyviä. Tarvitaankin jotain pysyvämpää, jotta ohjelma tavoitetaan varmemmin klusterin sisältä. Tätä varten on Kubernetesissa tarjolla [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Tehdään tiedostoon `service.yaml` seuraava määrittely:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demoapp-svc
spec:
  selector:
    app: demoapp
  ports:
    - protocol: TCP
      port: 80         
      targetPort: 3000  
  type: ClusterIP
```

Spec-osassa määritellään, että service "kohdistuu" sovellukseen _demoapp_, joka on portissa 3000, service tarjoaa ulospäin portin 80.

Seuraava havainnollistaa delpoymentin ja servicen yhteyttä:

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/openshift/images/k6.png?raw=true" width="600">

Luodaan service, ja varmistetaan heti, että kaikki meni hyvin:

```bash
$ oc apply -f manifests/service.yaml
service/demoapp-svc created
$ oc get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
demoapp-svc   ClusterIP   172.30.117.37   <none>        80/TCP    3s
```

Klusteri on luonut servicelle IP-osoitteen jonka avulla pääsisimme käsiksi sovellukseen. Parempi tapa on käyttää servicen nimeä  _demoapp-svc_, tämä toimii klusterin sisällä DNS-nimenä palvelulle:

```bash
$ curl http://demoapp-svc/api/ping
{"message":"pong"}
$ curl http://demoapp-svc/api/counter
{"id":1,"value":0}
$ curl http://172.30.117.37/api/counter
{"message":"pong"}
```

Eli vaikka alla oleva podi vaihtuu, osoite pysyy servicen ansiosta ennallaan. Voisimme myös käynnistää podista useampia replikoita muuttamalla konfiguraatiota:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapp-dep
spec:
  replicas: 3 
  ...
```

Kun komento `oc apply -f manifests/deployment.yaml` on suoritettu uudelleen, näemme että klusterille on käynnistynyt uusia kopioita samasta podista:

```bash
$ oc get pod
NAME                          READY   STATUS    RESTARTS   AGE
curlimage                     1/1     Running   0          16h
demoapp-dep-5bb7578b6-2xljw   1/1     Running   0          13h
demoapp-dep-5bb7578b6-4bk6c   1/1     Running   0          47s
demoapp-dep-5bb7578b6-w2b9j   1/1     Running   0          47s
```

Kun podiin otetaan yhteyttä servicen osoitteen kautta, yhdistää service pyynnön yhdelle podeista [round robin](https://en.wikipedia.org/wiki/Round-robin_scheduling) -periaateella.

Voimme tässä vaiheessa poistaa debuggausta varten käynnistämämme podin `curlimage`:

```bash
$ oc delete curlimage
```

### Näkyvyys klusterin ulkopuolelle

Sovelluksemme on muuten oikein hyvä, mutta emme pääse käyttämään sitä klusterin ulkopuolelta. Kubernetes tarjoaa muutamia ratkaisuja liikenteen ohjaamiseksi klusteriin.

Debuggaustarkoituksiin kätevä ratkaisu on komento [port-forward](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/), joka ohjaa jonkun paikallisen koneen portin klusterin sisälle.

Voimme tehdä portinohjauksen palveluun seuraavasti

```bash
$ oc port-forward svc/demoapp-svc 8080:80
Forwarding from 127.0.0.1:8080 -> 3000
```

Nyt pääsemme sovellukseen käsiksi selaimella portista 8080:

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/openshift/images/k2.png?raw=true" width="600">

Portinohjaus voidaan tehdä myös suoraan yksittäiseen podiin:

```bash
$ oc port-forward demoapp-dep-5bb7578b6-2xljw 8080:3000
```

Portinohjaus sopii hyvin debuggaukseen, esim. sen tarkastamiseen että sovellus toimii kokonaisuudessaan.

Tarvitsemme kuitenkin todelliseen käyttöön jotain muuta. Kubernetes tarjoaa tähän kaksi ratkaisua: Ingressin ja uudemman Gateway API:n joita molemia käsitellään kurssilla [DevOps with Kubernetes](https://devopswithkubernetes.com/). Tiken OpenShiftissä joudumme kuitenkin käyttämään OpenShit-spesifiä ratkaisua [Routea](https://docs.redhat.com/en/documentation/openshift_container_platform/4.11/html/networking/configuring-routes#route-configuration).

Tehdään seuraava määrittely tiedostoon `manifests/route.yaml`

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: demoapp-route
  namespace: toska-playground
  labels:
    app: demoapp
    type: external
spec:
  host: demoapp-toska-playground.ext.ocp-test-0.k8s.it.helsinki.fi 
  port:
    targetPort: 3000 
  to:
    kind: Service
    name: demoapp-svc
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None
```

Namespace on tässä tapauksessa _toska-playground_, se vastaa OpenShift-projektin nimeä, ohtuprojekteilla se on _ohtuprojekti-staging_. Host-nimen pitää olla Tiken klusteritasolla uniikki, sopiva nimi on esim. sovelluksen nimi ja sen perässä namespacen nimi. `spec/to` määrittelee reitityksen kohteena olevan palvelun. Kohdeportiksi pitää määritellä servicen takana olevan podin portti, ei siis servicen portti (joka oli tapauksessamme 80), servicen sisäistä porttia käytetään tapauksessamme klusterin sisäisessä kommunikoinnissa.

Sovellus toimii nyt koko maailmalle osoitteessa https://demoapp-toska-playground.ext.ocp-test-0.k8s.it.helsinki.fi/

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/openshift/images/k3.png?raw=true" width="600">

Host-nimi riippuu myös käytetystä klusteriympäristöstä, jos käytetään tuotantoklusteria, vaihtuu sana `test` sanaksi `prod`.

### Image stream

Sovelluksessamme on nyt eräs hieman ikävä puoli. Jos haluamme käynnistää uuden version, tulee luoda Docker-image jolla on uusi tagi, esim mluukkai/demoapp:2 ja deploymentia on muutettava siten, että se viittaa muuttuneeseen tagiin.

OpenShift tarjoaa [image steream](https://docs.redhat.com/en/documentation/openshift_container_platform/4.8/html/images/managing-image-streams) -nimisen objektin, jonka ansiosta deploymentin on mahdollista viitata koko ajan samaan tagiin, ja taustalla olevan imagen päivittyminen Dockerhubiin otetaan tästä huolimatta huomioon.

Muutetaan Dockerhubiin pushattavan imagen tagiksi _mluukkai/demoapp:staging_:

```yaml
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/demoapp:staging
```

Luodaan nyt seuraava määrittely tiedostoon _imagestream.yaml_:

```yaml
kind: ImageStream
apiVersion: image.openshift.io/v1

metadata:
  name: demoapp
  labels:
    app: demoapp
spec:
  lookupPolicy:
    local: false
  tags:
    - name: staging
      from:
        kind: DockerImage
        name: mluukkai/demoapp:staging
      importPolicy:
        scheduled: true
      referencePolicy:
        type: Local
```

Tämä määrittelee imagestreamin nimeltään _demoapp_, ja sille tägin _staging_, mihin viitataan esim. deploymenteissa nimellä `demoapp:staging`.

Luodaan imagestream ja tarkistetaan vielä miltä se näyttää

```bash
$ oc apply -f manifests/imagestream.yaml
imagestream.image.openshift.io/demoapp created
$ oc get imagestream
NAME      IMAGE REPOSITORY                                                       TAGS      UPDATED
demoapp   registry.apps.ocp-test-0.k8s.it.helsinki.fi/toska-playground/demoapp   staging   4 seconds ag
```

Voimme nyt ottaa image streamin viittaaman imagen käyttöön muokkaamalla deploymentia seuraavasti

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    alpha.image.policy.openshift.io/resolve-names: "*"
    image.openshift.io/triggers: >-
      [{"from":{"kind":"ImageStreamTag","name":"demoapp:staging","namespace":"toska-playground"},"fieldPath":"spec.template.spec.containers[?(@.name==\"demoapp\")].image","pause":"false"}]
  name: demoapp-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoapp
  template:
    metadata:
      labels:
        app: demoapp
    spec:
      containers:
        - name: demoapp
          image: demoapp:staging
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: DB_URL
              value: postgresql://ohtuprojektitesti:passwordhere@hostnamehere:5432/ohtuprojektitesti?targetServerType=primary&ssl=true     
```

Uutta tässä on avaimeen `meta/annotations` lisätyt määreet, jotka saavat deploymentin seuraamaan image streamissa tapahtuvia muutoksia. Toinen muutos on kontainerin `image`, joka arvo on nyt `demoapp:staging`, eli viite image streamiin.

Image stream päivittyy 15 min välein, eli jos pushaamme sovelluksesta uuden version Dockerhubiin, kestää korkeintaan 15 minuuttia, ennen kuin klusterilla oleva imagestream päivittyy, ja sovelluksen uusi versio käynnistyy.

Jos on tarve nopeampaan päivitykseen, voidaan suorittaa komento `oc import-image demoapp:staging` joka päivittää imagestreamin välittömästi, sekä käynnistää podin uudelleen jos image streamin osoittama image on muuttunut.

### Konfiguraatiot

Sovellus saa tietokannan osoitteen ympäristömuuttujan `DB_URL` avulla. Ympäristömuuttujan arvo määritellään deploymentissa:

```yaml
    spec:
      containers:
        - name: demoapp
          image: demoapp:staging
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: DB_URL
              value: postgresql://ohtuprojektitesti:passwordhere@hostnamehere:5432/ohtuprojekti?targetServerType=primary&ssl=true
```           

Tämä tapa on ok mutta ei optimaali, emme esim. voi laittaa deployment.yaml:ia GitHubiin koska URL sisältää salasanan. Kubernetes tarjoaa pari mekanismia, joiden avulla konfiguraatiot voidaan eriyttää deploymenteista, käytetään nyt näistä [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/):ia.

Tehdään tätä varten tiedosto `configmap.yaml`        

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demoapp-config
data:
  DB_URL: postgresql://ohtuprojektitesti:passwordhere@hostnamehere:5432/ohtuprojekti?targetServerType=primary&ssl=true
```

Lisätään tiedoto välittömästi `.gitignore`:n, jotta tietokannan salasana ei livahda internettiin.

Deploymentissa oleva ympäristömuuttujan määrittely muuttuu seuraavasti

```yaml
    spec:
      containers:
        - name: demoapp
          image: demoapp:staging
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: DB_URL
              valueFrom:
              configMapKeyRef:
                name: demoapp-config
                key: DB_URL 
```

Kubernetes tarjoaa myös resurssin [Secret](https://kubernetes.io/docs/concepts/configuration/secret/), joka periaatteessa sopisi paremmin salasanan sisältävän tietokantaurlin tarpeisiin. Nimestään huolimatta secretit eivät oikeastaan tuo juurikaan turvaa, joten sivuutamme asian nyt. Myös secretejä käsitellään kurssilla [DevOps with Kubernetes](https://devopswithkubernetes.com/).

### Kustomize

Sovelluksen Kubernetes-konfiguraatiot, eli manifestit on nyt talletettu hakemistoon `manifests`, ja konfiguraatioista muut paitsi salaista tietoa sisältävä `configmap.yaml` on tallennettu versionhallintaan.

Koko sovelluksen konfiguraatiot voi päivittää klusterille komennolla `oc apply -f manifests`, eli antamalla parametriksi manifestit sisältämän hakemiston. Monimutkaisemmassa sovelluksessa manifestitiedostot voivat olla hajaantuneet useampaan hakemistoon ja klusterin synkronointi voi olla hankalampaa.

[Kustomize](https://kustomize.io/)-työkalu (joka on nykyään sisäänrakennettu Kubernetesin komentoriville) tuo helpotusta tähän (ja tarjoaa paljon muutakin). Määritellään tiedosto `kustomize.yaml`, joka listaa yksittäiset manifestit:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - manifests/configmap.yaml
  - manifests/route.yaml
  - manifests/service.yaml
  - manifests/imagestream.yaml
  - manifests/deployment.yaml
```

Manifestit saadaan synkronoitua klusterille komennolla `oc apply -k .`

Esimerkkimme tapauksessa Kustomize ei tuo juurikaan etuja, suuremmassa projektissa tilanne on toinen. Kurssilla [DevOps with Kubernetes](https://devopswithkubernetes.com/) asiaa käsitellään enemmän.

### Resurssirajat

Klusteri priorisoi sovelluksia, joille on asetettu resurssirajat. Jos resursseista on pulaa, niin ilman rajojen asettamista sovellus ei välttämättä edes käynnisty.

Deploymenteille onkin syytä asettaa [resurssirajat](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    alpha.image.policy.openshift.io/resolve-names: "*"
    image.openshift.io/triggers: >-
      [{"from":{"kind":"ImageStreamTag","name":"demoapp:staging","namespace":"toska-playground"},"fieldPath":"spec.template.spec.containers[?(@.name==\"demoapp\")].image","pause":"false"}]
  name: demoapp-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoapp
  template:
    metadata:
      labels:
        app: demoapp
    spec:
      containers:
        - name: demoapp
          image: demoapp:staging
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: DB_URL
              valueFrom:
                configMapKeyRef:
                  name: demoapp-config
                  key: DB_URL
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
            requests:
              memory: "256Mi"
              cpu: "250m"
```

Limits määrittää enimmäismäärän laskentaresursseja, joita kontti voi käyttää. Requests määrittää resurssimäärän, joka taataan olevan saatavilla kontille.

Voit säätää memory ja cpu arvoja sovelluksesi vaatimusten mukaan. Muisti määritetään MiB-yksiköissä ja CPU milliCPU-yksiköissä.

Älä pyydä turhaan liikaa resursseja!

### Kooste tärkeimmistä komennoista

| Command                  | Description                                      |
|--------------------------|--------------------------------------------------|
| `oc get po`             | Listaa podit                                     |
| `oc get svc`            | Listaa servicet                                  |
| `oc describe po <pod>`  | Katso podin tarkemmat tiedot                     |
|                          | Komento toimii myös muille resursseille, esim. svc, deployments |
| `oc exec -it <pod> bash` | suorita podilla komento bash eli komentotulkki |
| `oc apply -f mainifest.yaml ` | luo/päivitä manifestin määrittelemät objektit |
| `oc delete -f mainifest.yaml ` | tuhoa manifestin määrittelemät objektit |
| `oc import-image image:tagi` | päivitä imagesream heti |
| `oc logs <pod>` |  näytä sovelluksen lokit |
| `oc logs -f <pod>` |  seuraa sovelluksen lokeja |
| `oc port-forward <pod>` | ohjaa lokaalin koneen portin liikenne podiin |
| `oc port-forward svc/<service>` | ohjaa lokaalin koneen portin liikenne palveluun |

### Tietokannan hankkiminen

Esimerkkisovellus käyttää Tiken ylläpitämää yhteiskäyttöistä tietokantaa. Tietokannan hankkiminen on helppoa, yksi email riittää, ks https://devops.pages.helsinki.fi/guides/platforms/shared-dbs.html

Pyyntöön voi käyttää seuraavaa email-pohjaa

```
Saisimmeko uuden tietokannan yhteiskäyttöiseen Postgresin testikantaan esim. possu-test-1-21.it?

kantatunnus: omansovelluksennimi
sovelluksen osoite: openshift
kannan ylläpitäjä: oili.opiskelija@helsinki.fi
Klusteri, jossa sovellus pyörii: prod tai testi (ocp-prod/ocp-test klusterin urlissa)
```

Osoitteena siis _openshift_. Jos kyse on tuotantosovelluksesta, tulee pyytää _possu-test-1-21.it_ sijaan tuotantokantaan.

Kannattaa huomata, että tietokanta edellyttää SSL:n käyttöä, eli tietokantaurl on muotoa `postgresql://ohtuprojektitesti:passwordhere@hostnamehere:5432/ohtuprojekti?targetServerType=primary&ssl=true` asia kyllä mainitaan yo dokumentaatiossa mutta esim. allekirjoittanut ei lukenut dokumenttia aluksi tarpeeksi tarkasti...

Jos haluat ottaa tietokantaan yhdeyden suoraan, on projektiin `ohtuprojekti-staging` on konfiguroitu podi `db-tools`, joka sisältää tietokantayhteyksien kannalta tarvittavat komentorivityökalut, kuten `psql` ja `mongosh`. Tietokantayhteys avataan tämän podin avulla seuraavasti

```bash
$ oc exec -it $(oc get pods -l deployment=db-tools -o jsonpath='{.items[0].metadata.name}') -- psql postgres://kayttaja:salasana@possu-test.it.helsinki.fi:5432/tietokanta
```

Komentorivilt yhdistäessäsi riittää tietokantaurlin lyhempi muoto.

### Mongo ja tiedostojen tallentaminen

Jos päätät käyttää MongoDB:tä, ei yliopistolla ole valitettavasti tarjolla hostattua palvelua. On käytettävä ulkoista palvelua tai konfiguroitava Mongo itse OpenShiftin. Tämäkään ei ole vaikeaa. Pienen mutkan matkaan aiheuttaa se, että Mongoa varten on varattava pysyvään talletukseen sopivaa levytilaa.

Levytilan varaaminen tapahtuu luomalla [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims). Tehdään tiedosto `volumeclaim.yaml` ja sinne sisältö:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1

metadata:
  name: demoapp-claim
  namespace: toska-playground
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: data-2
  volumeMode: Filesystem
```

Määritelty levypyyntö varaa yhden gigan levyä. Tilauksessa on määritelty `storageClassName: data-2`, tämä ohjaa pyynnön Tiken meille varaamalle levylle eli PersistentVolumelle.

Kokeillaan ensin liittää levy normaaliin podiin. 

Tehdään tiedostoon `ubuntu-deployment.yaml` deployment joka käynnistää Ubuntun:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-ubuntu-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-ubuntu
  template:
    metadata:
      labels:
        app: my-ubuntu
    spec:
      containers:
        - name: ubuntu
          image: ubuntu
          imagePullPolicy: Always
          command:
            - sleep
            - "3600"  
```

Ennen kun liitämme pysyväislevyn deploymentin luovaan podiin, kokeillaan mitä tapahtuu jos kirjoitamme tiedostoja podin sisälle ilman pysyväislevyn käyttöä.

Luodaan deployment, mennään podiin ja tehdään podin hakemistoon /tmp kaksi tiedostoa:

```bash
$ oc apply -f ubuntu-deployment.yaml
$ oc exec -it my-ubuntu-deployment-d7b6d85d4-drsbr bash
$ cd tmp
$ touch tiedosto1
$ touch tiedosto2
$ ls 
tiedosto1
tiedosto2
```

Tehdään nyt muutos deploymentiin, esim. muutetaan komennon sleep sekuntimäärää. Kun suoritetaan `apply`, luodaan uusi podi edellisen tilalle. 

Kun mennään nyt uuden podin hakemistoon `/tmp` huomataan että se on tyhjä:

```bash
$ oc apply -f ubuntu-deployment.yaml
$ oc exec -it my-ubuntu-deployment-6b947f9bbc-ztls bash
$ cd tmp
$ ls -l
total 0
```

Podin levylle kirjoittaminen siis on turhaa, jos on tarkoitus tallettaa asioita pysyvästi.

Otetaan nyt käyttöön klusterilta pyytämämme levyvoluumi, ja mäpätään se podin hakemistoon `/tmp`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-ubuntu-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-ubuntu
  template:
    metadata:
      labels:
        app: my-ubuntu
    spec:
      containers:
        - name: ubuntu
          image: ubuntu
          imagePullPolicy: Always
          command:
            - sleep
            - "3601"
          volumeMounts:
            - mountPath: /tmp
              name: demoapp-volume
      volumes:
        - name: demoapp-volume
          persistentVolumeClaim:
            claimName: demoapp-claim
```

Deploymentissa on kaksi lisäystä, ensinnäkin `template/spec`:in alla on osa `volumes`, joka ketoo mitä volumeja deployment käyttää. Käyttöön on otettu levypyyntöä `demoapp-claim` vastaava voluumi ja on annettu sille nimi `demoapp-volume`. Osan `container`s alle on määritelty, että käyttöön otettu voluumi mäpätään podiin polulle `/tmp`.

Kokeillaan sitten samaa uusiksi, eli lisätään volumille mäpättyyn hakemistoon `/tmp` tiedostoja, pakotetaan podin uudelleenluonti, ja tarkastetaan että tiedostot säilyvät:

```bash
$ oc exec -it my-ubuntu-deployment-755d7b8889-t2ctk bash
$ cd tmp
$ ls
lost+found
$ touch tiedosto1
$ touch tiedosto2
$ ls
lost+found  tiedosto1  tiedosto2
$ exit
exit
$ of apply -f ubuntu-deployment.yaml
deployment.apps/my-ubuntu-deployment configured
$ oc exec -it my-ubuntu-deployment-76bfb959bb-jpdng bash
$ cd tmp
$ ls
lost+found  tiedosto1  tiedosto2
```

Kuten olettaa saattaa, tiedostot säilyvät. Volumella olevat tiedot säilyvät vaikka deployment tuhottaisiin ja luotaisiin uudelleen:

```bash
$ oc delete -f ubuntu-deployment.yaml
$ oc apply -f ubuntu-deployment.yaml
$ oc  exec -it my-ubuntu-deployment-76bfb959bb-qxckz ls /tmp
lost+found  tiedosto1  tiedosto2
```

Uskomme nyt että persistent volumet ovat sitä mitä tarvitsemme, tuhotaan kokeiluun käytetty Ubuntu ja luodaan Mongoa varten deployment ja service. Luodaan tällä kertaa molemmat samaan tiedostoon `mongo-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapp-mongo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoapp-mongo
  template:
    metadata:
      labels:
        app: demoapp-ubuntu
    spec:
      containers:
        - name: my-mongo
          image: mongo
          imagePullPolicy: Always
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: demo_user
            - name: MONGO_INITDB_ROOT_PASSWORD
              value: demo_password
          volumeMounts:
            - mountPath: /data/db
              name: demoapp-volume
      volumes:
        - name: demoapp-volume
          persistentVolumeClaim:
            claimName: demoapp-claim

---

apiVersion: v1
kind: Service
metadata:
  name: demoapp-mongo-svc
spec:
  selector:
    app: demoapp-mongo
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
  type: ClusterIP
```

Levyvoluumi on mäpätty Mongo-podin polulle `/data/db`, kyseessä on [dokumentaation](https://hub.docker.com/_/mongo) mukaan hakemisto minne Mongo tallettaa tietokannan.

Suoritetaan tuttu komento `oc apply -f mongo-deployment.yaml`. Komentoja `oc get po` ja `oc logs` tarkastelemalla olemme vakuuttuneita, että Mongo on käynnissä.

Klusterin sisältä kantaan päästään nyt käsiksi osoitteella `mongodb://demo_user:demo_password@demoapp-mongo-svc:27017`.

Laskurisovelluksesta on tehty Mongoa käyttävä versio repositorion branchiin [mongo](https://github.com/mluukkai/openshift-demo/tree/mongo). Mongoa käyttävän versiosta luodaan Docker-image on `demoapp-mongo`.

Laajennetaan image streamia ottamaan huomioon myös tämä tägi:

```yaml
kind: ImageStream
apiVersion: image.openshift.io/v1

metadata:
  name: demoapp
  labels:
    app: demoapp
spec:
  lookupPolicy:
    local: false
  tags:
    - name: staging
      from:
        kind: DockerImage
        name: mluukkai/demoapp:staging
      importPolicy:
        scheduled: true
      referencePolicy:
        type: Local
    - name: mongo
      from:
        kind: DockerImage
        name: mluukkai/demoapp:mongo
      importPolicy:
        scheduled: true
      referencePolicy:
        type: Local
```

Muutetaan deploymentia seuraavasti:

```yaml
    spec:
      containers:
        - name: demoapp
          image: demoapp:mongo
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: MONGO_DB_URL
              value: mongodb://demo_user:demo_password@demoapp-mongo-svc:27017
```


Olemme tällä kertaa laiskoja ja määrittelemme tietokantaurlin suoraan GitHubiin menevässä tiedostossa `deployment.yaml`. **Älä yritä tätä kotona äläkä missään muuallakaan!**

Kokeillaan! Sovellus toimii:

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/openshift/images/k8.png?raw=true" width="600">

### Ongelmatilanteita

Kubernetes/OpenShift on loistava työkalu, mutta moni asia voi mennä pieleen. Kielimalleista on erittäin suuri apu yaml-manifestien kanssa, esim ChatGPT osaa bongata virheet manifesteista ja luoda niitä ainakin pääpiirteissään jos promptaus on kohtuullisella tasolla.

Debugatessa ei auta oikein muu kun systemaattisuus. Komentoja `oc get`, `oc describe` ja `oc logs` ei voi koskaan käyttää liikaa...

#### Lokaalisti toimiva kontti ei toimi...

Käyttöoikeusongelmien kirjo voi olla moninainen riippuen kontissa ajettavasta teknologiasta. Tähän liittyy yleensä jonkinlainen tiedostojen käyttöön liittyvä virheilmoitus: 'not writable', 'permission denied'. 

Kun kontti käynnistetään OpenShiftissa, arvotaan kontin käyttäjäksi satunnainen UID ja käyttäjälle tulee asettaa sopivat oikeudet kontin sisälle. Esimerkkimme tilanteessa ei mitään toimenpiteitä tarvittu. Aina ei ole näin. Joissain tilanteissa selviää seuraavasti:

```dockerfile
WORKDIR /app

COPY . .
```

voit asettaa käyttöoikeudet Dockerfilessa yksinkertaisella (joskaan ei kovin mallikelpoisella) tavalla,

```dockerfile
RUN chmod -R 777 *
```

jonka jälkeen käyttäjän oikeudet ulottuvat kansion /app alikansioihin ja tiedostoihin.

Esimerkkitapauksena mainittakoon Pythonin virtuaaliympäristö, jota saatetaan yrittää suorittaa jossain muussa hakemistossa kuin /app, jolloin venv-moduuli ei välttämättä käynnisty laisinkaan. Sopivaa ympäristömuuttujaa käyttämällä virtuaaliympäristö voidaan asettaa suoritettavaksi /app kansion sisällä. Ympäristömuuttujia voi esitellä kontille Dockerfilen käskyllä ENV.

Seuraava esimerkki toi ratkaisun ainakin erääseen Poetry-projektiin:

```dockerfile
ENV POETRY_VIRTUALENVS_IN_PROJECT=true
```

Jossain tilanteessa ongelman voi aiheuttaa kolmannen osapuolen image. Esim. avain-arvo-tietokanta [Redisin](https://redis.io/) virallinen [Docker image](https://hub.docker.com/_/redis) olettaa suorittavansa koodia rootin oikeuksilla. OpenShift kuitenkin vaihtaa tilalle satunnaisen ei-rootkäyttäjän ja oletusarvoisesti konfiguroitu kontti törmää ongelmiin kirjoittaessaan dataa tiedostoihin. Ongelma ratkeaa ottamalla käyttöön [Red Hatin](https://catalog.redhat.com/software/containers/rhel9/redis-7/64881353e0e10aaf1cbac8b7?gs&q=redis) versio Redisistä, esim. `registry.redhat.io/rhel9/redis-7`, joka ei edellytä root-käyttäjää. Red Hatilta löytyy OpenShiftille sopiva versio monesta yleisestä palvelusta.

### Objektien nimennästä

On oleellisen tärkeää, että jokainen ryhmä nimeää Kubernetes-objektit järkevästi ja yhdenmukaisesti, siten että on mahdollista helposti päätellä minkä ryhmän tuotoksista on kyse. Jos projektin nimi olisi esim. labtool, objektien nimet noudattaisivat seuraavaa kaavaa

```
labtool-backend-dep
labtool-backend-svc
labtool-route
labtool-mongo-dep
labtool-mongo-svc
```

Nimi siis alkaa ryhmän nimeä edustavalla kuvaavalla merkkijonolla.

Klusterille ei saa jättää mitään roskaa eikä ylimääräisiä Kubernetes-objekteja. Eli jos jokin rersurssi todetaan turhaksi, se poistetaan.

Jos klusterilta löytyy holtittomasti nimettyjä objekteja, ne saatetaan poistaa ilman varoitusta.

### HY-kirjautuminen

Katso erillinen [ohje](kirjautuminen.md)
