## OpenShift-konttialustan käyttö staging-ympäristönä

Tämä ohje olettaa, että hallitset jossain määrin Dockeria, esim. vähintään luvun _Docker basics_ verran kursilta [DevOps with Docker](https://courses.mooc.fi/org/uh-cs/courses/devops-with-docker).

### Esimerkkisovellus

Seuraavassa asennetaan yliopiston tietotekniikkakeskuksen OpenShift-klusterille Reactilla ja NodeJS:llä toteutettu SPA-sovellus, jonka koodi löytyy [GitHubista](https://github.com/mluukkai/openshift-demo).

Sovellus on hyvin yksinkertainen laskuri. Laskurin arvo on talletettu Postgres-tietokantaan, johon backend on yhteydessä [Sequelize](https://sequelize.org/)-kirjaston avulla. Frontend sisältää napit laskurin kasvattamiseen sekä nollaamiseen. Käytössä ovat siis kurssilta [Full stack -open](https://fullstackopen.com/) tutut teknologiat.

Projektiin on määritelty GitHub Action -workflow, joka luo projektista Docker-imagen ja pushaa sen Dockerhubiin. Sama image sisältää sekä backendin, että frontendin.

Sovellus on GitHub siinä tilanteessa mihin tämä tutoriaali päättyy. Alkutilanne on branchissa [start](https://github.com/mluukkai/openshift-demo/tree/start). Koodissa ei muutoksia ole, mutta konfiguraatiot tutoriaalin aikana tehdyt konfiguraatiot puuttuvat vielä haarasta start.

### OpenShift

Käytössämme on Tietotekniikkakeskuksen [OpenShift](https://devops.pages.helsinki.fi/guides/platforms/tike-container-platform.html)-klusteri. OpenShift on [Kubernetes](https://github.com/mluukkai/openshift-demo/blob/main/.github/workflows/main.yaml)-klusteri tietyin lisämaustein.

Kubernetes on melko monimutkainen olio, kurssi [DevOps with Kubernetes](https://devopswithkubernetes.com/) käsittelee aihetta laajassti. Seuraavassa käydään läpi minimioppimäärä yksinkertaisen sovelluksen tarpeisiin.

Ytimessä olevan Kuberneteksen lisäksi OpenShift sisältää mm. graafisen käyttöliittymän jonka kautta konfiguraatioita on mahdollista tehdä, mutta se **ei ole suositeltua** sillä näin päädytään usein hallitsemattoman epämääräisiin konfiguraatioihin. On suositeltavaa pitäytyä määrittelyissä mahdollisimman "puhtaassa" Kuberneteksessa, ja näin tulemme seuraavassakin tekemään.

**Eli älä määrittele mitään OpenShiftin käyttöliittymän kautta, jos teet näin, teknistä tukea ei kurssin puolesta ole luvassa.**

Käytämme klusteria yksinomaan komentoriviltä, komennon [ok](https://docs.redhat.com/en/documentation/openshift_container_platform/4.11/html/cli_tools/openshift-cli-oc) avulla. `oc` toimii samoin kun Kubernetesin [kubectl](https://kubernetes.io/docs/reference/kubectl/), mutta se sisältää muutamia OpenShift-spesifejä komentoja.

Asenna nyt koneellesi `oc` [tämän ohjeen](https://devops.pages.helsinki.fi/guides/platforms/tike-container-platform.html#openshift-client) mukaan. Kannattaa myös ehdottomasti konfiguroida [tabulaattoritäydennys](https://docs.redhat.com/en/documentation/openshift_container_platform/4.9/html/cli_tools/openshift-cli-oc#cli-enabling-tab-completion).

Oletetaan nyt, että `oc` asennettu. Jotta yhteys klusteriin toimisi, on oltava Eduroamissa tai HY:n vpn:ssä. 

Kirjaudu klusterille suorittamalla komento `oc login -u <username> https://api.ocp-test-0.k8s.it.helsinki.fi:6443`.

Kirjaantumisen jälkeen voidaan vaikkapa suorittaa komento `oc status`, joka kertoo että olemme onnisstuneessti kirjautuneet, omassa tapauksessani projektiin _toska-playground_:

```
$ oc status
In project toska-playground on server https://api.ocp-test-0.k8s.it.helsinki.fi:6443
```

Esimerkissä on käytössä projekti `toska-playground`. Ohtuprojekteissa käytetän projektia `ohtuprojekti-staging`. Projektista on olemassa sekä tuotanto, että testipuoli. Kysy ohjaajaltasi kumpaa ryhmäsi käyttää. 

Testipuolen osoite on https://api.ocp-test-0.k8s.it.helsinki.fi:6443 ja tuotantopuolen https://api.ocp-prod-0.k8s.it.helsinki.fi:6443, eli kirjautuessa käytä oikeaa osoitetta!

### Pod ja deployment

Openshiftissa sovelluksen paketoinnin perusyksikkö on [podi](https://kubernetes.io/docs/concepts/workloads/pods/) (engl. pod). Podit ovat Kubernetes-ympäristön perusyksiköitä, ja ne sisältävät yleensä yhden Docker-imagen. Joissain tilanteissa podissa voi olla useita imageja, mutta tämä on yleensä mielekästä vain, jos imaget ovat tiiviisti yhteydessä toisiinsa, esimerkiksi jakamalla verkkoyhteyden tai tallennustilan.

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

Avaimen `spec` arvona määritellään deploymentin hallitseman podin (tai podien jos `replicas` on suurempi kuin 1) kontit. Tapauksessamme on yksi kontti, jonka käyttämä image on `mluukkai/demoapp:1`. Kontille on annettu sen käyttämä tietokantaosoite määrittelemällä ympäristömuuttuja `DB_URL`.

Deployment luodaan klusterille seuraavalla komennolla ($ on komentokehote):

```
$ oc apply -f manifests/deployment.yaml
```

Deployment näyttää onnistuneen:

```
$ oc get deployments
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
demoapp-dep   1/1     1            1           16s
```

Deploymentin on siis tarkoitus käynnistää yksi podi. Katsotaan miltä podien tilanne näyttää:

```
oc get pod
NAME                           READY   STATUS    RESTARTS   AGE
demoapp-dep-5dbb664966-p6kfh   1/1     Running   0          25s
```

Podi on käynnissä.

Testataan nyt sovellusta. Sovellus ei näy vielä klusterin ulkopuolelle, mutta pääsemme siihen käsiksi klusterin sisältä podin IP-osoitteen avulla. Saamme IP-osoitteen selville komennolla `oc describe pod`:

```
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

Käynnistetään klusterin sisälle [curl](https://curl.se/)-komennon sisätävä Docker [image](https://hub.docker.com/r/curlimages/curl):

```
$ oc run curlimage --image=curlimages/curl --restart=Never --command -- sleep infinity
$ oc exec -it curlimage sh
```

Olemme nyt podin sisällä, ja voimme kokeilla ottaa yhteyttä sovelluksen tekemällä GET-pyynnön testirajapintaan _api/ping_:

```
$ curl 10.12.2.177:3000/api/ping
{"message":"pong"} 
```

Jostain syystä GET-pyyntö laskurin arvon palauttavaan rajapintaan _api/counter_: ei toimi:

```
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

Tutkitaan sovelluksen lokia komennolla `oc logs`:

```
$ oc logs demoapp-dep-68c95df467-w4glg

> demoapp@0.0.0 prod
> NODE_ENV=production node server/index.js

Server running on http://localhost:3000
Unable to connect to the database: ConnectionError [SequelizeConnectionError]: no pg_hba.conf entry for host "128.214.137.139", user "ohtuprojektitesti", database "ohtuprojekti", SSL encryption
```

Tietokantayhteyden suhteen näyttää olevan jotain häikkää. Ja kun virheilmoitusta lukee tarkemmin, huomaa, että sovelluksen käyttämä tietokantaurl on väärä, tietokannan nimi on _ohtuprojektitesti_, ei _ohtuprojekti_ kuten konfiguraatio vahingossa määritteli.

Korjataan konfiguraatio ja päivitetään korjattu tilanne komennolla `apply -f manifests/deployment.yaml`. Katsotaan lokia uudelleen:

```
$ oc logs demoapp-dep-6746c5d5dc-jjp88
> demoapp@0.0.0 prod
> NODE_ENV=production node server/index.js

Server running on http://localhost:3000
Executing (default): SELECT 1+1 AS result
Connected to the database
Executing (default): SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' AND table_name = 'counters'
Executing (default): SELECT i.relname AS name, ix.indisprimary AS primary, ix.indisunique AS unique, ix.indkey AS indkey, array_agg(a.attnum) as column_indexes, array_agg(a.attname) AS column_names, pg_get_indexdef(ix.indexrelid) AS definition FROM pg_class t, pg_class i, pg_index ix, pg_attribute a WHERE t.oid = ix.indrelid AND i.oid = ix.indexrelid AND a.attrelid = t.oid AND t.relkind = 'r' and t.relname = 'counters' GROUP BY i.relname, ix.indexrelid, ix.indisprimary, ix.indisunique, ix.indkey ORDER BY i.relname;
```

Yritetään jälleen

```
$ curl 10.13.2.114:3000/api/counter
curl: (7) Failed to connect to 10.13.2.114 port 3000 after 18688 ms: Could not connect to server
```

Ei toimi. Mikä vikana? IP-osoite ei pysy, tarkastetaan uusi:

```
curl 10.15.3.15:3000/api/counter
{"id":1,"value":0}
```

Toimii!

### yaml vs imperatiiviset komennot

...

### Service

Podin IP-osoite siis näyttää vaihtuvan. Tämä johtuu siitä, että podi luodaan aina uudelleen kun siihen tehdään minkäänlaisia muutoksia. Tarvitaan siis jotain pysyvämpää, jotta ohjelma tavoitetaan varmemmin klusterin sisältä. Tätä varten on Kubernetesissa tarjolla [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Tehdään tiedostoon `service.yaml` seuraava määrittely:

```
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

Spec-osassa määritellään, että service "kohdistuu" sovellukseen _demoapp_, joka on portissa 3000, service tajoaa ulospäin portin 80.

Luodaan service, ja varmistetaan heti, että kaikki meni hyvin:

```
$ oc apply -f manifests/service.yaml
service/demoapp-svc created
$ oc get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
demoapp-svc   ClusterIP   172.30.117.37   <none>        80/TCP    3s
```

Klusteri on luonut servicelle IP-osoitteen jonka avulla pääsisimme käsiksi sovellukseen. Parempi tapa on käyttää servicen nimeä  _demoapp-svc_, tämä toimii klusterin sisällä DNS-nimenä palvelulle:

```
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

```
$ oc get pod
NAME                          READY   STATUS    RESTARTS   AGE
curlimage                     1/1     Running   0          16h
demoapp-dep-5bb7578b6-2xljw   1/1     Running   0          13h
demoapp-dep-5bb7578b6-4bk6c   1/1     Running   0          47s
demoapp-dep-5bb7578b6-w2b9j   1/1     Running   0          47s
postgres-client               1/1     Running   0          16h
```

Kun podiin otetaan yhteyttä servicen osoitteen kautta, yhdistää service pyynnön yhdelle podeista [round robin](https://en.wikipedia.org/wiki/Round-robin_scheduling) -periaateella.

### Näkyvyys klusterin ulkopuolelle

Sovelluksemme on muuten oikein hyvä, mutta emme pääse käyttämään sitä klusterin ulkopuolelta. Kubernetes tarjoaa muutamia ratkaisuja liikenteen klusteriin ohjaamiseksi.

Debuggaustarkoituksiin kätevä ratkaisu on komenot [port-forward](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/), joka ohjaa jonkun paikallisen koneen portin klusterin sisälle.

Voimme tehdä portinohjauksen palveluun seuraavasti

```
$ oc port-forward svc/demoapp-svc 8080:80
Forwarding from 127.0.0.1:8080 -> 3000
```

Nyt pääsemme sovellukseen käsiksi selaimella portista 8080:

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/staging/images/k2.png?raw=true" width="600">


Portinohjaus voidaan tehdä myös suoraan yksittäiseen podiin:

```
$ oc port-forward demoapp-dep-5bb7578b6-2xljw  8080:3000
```

Portinohjaus sopii hyvin debuggaukseen, esim. sen tarkastamiseen että sovellus toimii kokonaisuudessaan.

Tarvitsemme kuitenkin todelliseen käyttöön jotain muuta. Kubernetes tarjoaa tähän kaksi ratkaisua Ingressin ja uudemman Gateway API:n, molemia käsitellään kurssilla [DevOps with Kubernetes](https://devopswithkubernetes.com/). OpenShiftissä joudumme kuitenkin käyttämään OpenShit-spesifiä ratkaisua [Routea](https://docs.redhat.com/en/documentation/openshift_container_platform/4.11/html/networking/configuring-routes#route-configuration).

Tehdään seuraava määrittely tiedostoon `manifests/route.yaml`

```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: demoapp-route
  namespace: toska-playground
  labels:
    app: demoapp
    type: external
spec:
  host: demoapp-toska-playground.apps.ocp-test-0.k8s.it.helsinki.fi 
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

Namespace on tässä tapauksessa _toska-playground_, se vastaa OpenShift-projektin nimeä, ohtuprojekteilla se on _ohtuprojekti-staging_. Host-nimen pitää olla yliopistotaolla uniikki, sopiva nimi on esim. sovelluksen nimi ja sen perässä namespacen nimi. spec/to määrittelee reitityksen kohteena olevan palvelun. Kohdeportiksi pitää määritellä servicen takana olevan podin portti, ei siis servicen portti (joka oli tapauksessamme 80), sericen sisäistä porttia käytetään tapauksessamme klusterin sisäisessä kommunikoinnissa.

Sovellus toimii nyt koko maailmalle osoitteessa https://demoapp-toska-playground.apps.ocp-test-0.k8s.it.helsinki.fi/

<img src="https://raw.githubusercontent.com/HY-TKTL/TKT20007-Ohjelmistotuotantoprojekti/refs/heads/master/staging/images/k3.png?raw=true" width="600">

### Image stream

Sovelluksessamme on nyt eräs hieman ikävä puoli. Jos haluamme käynnistää uuden version, tulee luoda Docker-image jolla on uusi tagi, esim mluukkai/demoapp:2 ja deploymentia on muutettava siten, että se viittaa muuttuneeseen tagiin.

OpenShift tarjoaa [image steream](https://docs.redhat.com/en/documentation/openshift_container_platform/4.8/html/images/managing-image-streams) -nimisen resurssin, jonka ansiosta deploymentin om mahdollista viitata koko ajan samaan tagiin, ja taustalla olevan imagen päivitys otetaan tästä huolimatta huomioon.

Muutetaan Dockerhubiin pushattavan imagen tagiksi _mluukkai/demoapp:staging_:

```
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/demoapp:staging
```

Luodaan nyt määrittely tiedostoon _imagesteram.yaml_:

```
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

```
$ oc apply -f manifests/imagestream.yaml
imagestream.image.openshift.io/demoapp created
$ oc get imagestream
NAME      IMAGE REPOSITORY                                                       TAGS      UPDATED
demoapp   registry.apps.ocp-test-0.k8s.it.helsinki.fi/toska-playground/demoapp   staging   4 seconds ag
```

Voimme nyt ottaa image streamin määrittelemän imagen käyttöön muokkaamalla deploymentia seuraavasti

```
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
              value: postgresql://ohtuprojektitesti:passwordhere@hostnamehere:5432/ohtuprojekti?targetServerType=primary&ssl=true     
```

Uutta tässä on avaimen `meta/annotations` lisätyt määreet, jotka saavat deploymentin seuraamaan image streamissa tapahtuvia muutoksia. Toinen muutos on kontainerin `image`, joka arvo on nyt `demoapp:staging`

Image steram päivttyy 15 min välein, eli jos pushaamme sovelluksesta uuden version Dockerhubiin, kestää korkeintaan 15 minuuttia, ennen kuin klusterilla oleva imagestream päivittyy, ja sovelluksen uusi versio käynnistyy.

Jos tarve nopeampaan päivitykseen, voidaan antaa komento `oc import-image demoapp:staging` joka päivittää imagestreamin välittömästi.

### Konfiguraatiot

Sovellus saa nyt tietokannan osoitteen ympäristömuuttujan DB_URL avulla. Ympäristömuuttujan arvo määritellään deploymentissa:

```
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

Tämä tapa on ok mutta ei optimaali, emme esim. voi laittaa nyt deployment.yaml:ia GitHubiin. Kubernetes tarjoaa pari eri mekanismia minkä avulla konfiguraatiot voidaan eriyttää deploymenteista, käytetään nyt näistä [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/):ia. Tehdään tätä varten tiedosto `configmap.yaml`        

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demoapp-config
data:
  DB_URL: postgresql://ohtuprojektitesti:passwordhere@hostnamehere:5432/ohtuprojekti?targetServerType=primary&ssl=true
```

Lisätään tiedoto välittömästi ´.gitignore´:n!

Deployment muuttuu seuraavasti

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

Kubernetes tarjoaa myös resurssin [Secret](https://kubernetes.io/docs/concepts/configuration/secret/), joka periaatteessa sopisi paremmin salasanan sisältävän tietokantaurlin tarpeisiin. Nimestään huolimatta secretit eivät oikeastan tuo juurikaan turvaa, joten sivuutamme asian nyt.

### Kustomize

Sovelluksen Kubernetes-konfiguraatiot, eli manifestit on nyt talletettu hakemistoon manifests, konfiguraatioista muut paitsi salaista tietoa sisältävä configmap.yaml kannattaa talletaa versionhallintaan.

Koko sovelluksen tapauksessamme viedä klusterille komennolla `oc apply -f manifests`. Monimutkaisemmassa sovelluksessa manifestitiedostot voivat olla hajaantuneet useampaan hakemistoon ja klusterille vieminen voi olla hankalampaa. [Kustomize](https://kustomize.io/)-työkalu (joka on nykyään sisäänrakennettu Kubernetesin komentoriville) tuo helpotusta tähän (ja tarjoaa paljon muutakin).

Määritellään tiedosto `kustomize.yaml` joka listaa käytettävät resurssit:

```
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

### Tietokannan hankkiminen

### Kooste tärkeimmistä komennoista

|----------|----------|
| oc get po        | listaa podit       |
| oc get svc        | listaa servicet       |
| oc describe po podintunniste        | katso podin tarkemmat tiedoot      |
|  | komento toimi myös muille resursseille esim svc, deployments|
|----------|----------|

### Kun joku menee vikaan

avaa possu

`kubectl run -it --rm postgres-client --image=postgres:latest sh`

Komento `oc get imagestream demoapp`

### Mongo ja tiedostojen tallentaminen


### HY login