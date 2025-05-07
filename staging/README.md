## HOWTO staging Openshiftissä

Tämä ohje olettaa, että käyttäjä hallitsee jossain määrin Dockeria, suunilleen ensimmäisen kahden osan verran kurssilta [DevOps with Docker](https://courses.mooc.fi/org/uh-cs/courses/devops-with-docker).

### Esimerkkisovellus

Seuraavassa asennetaan yliopiston OpenShift-klusteriin Reactilla ja NodeJS:llä toteutettu SPA-sovellus, jonka koodi löytyy [GitHubista](https://github.com/mluukkai/openshift-demo).

Sovellus on hyvin yksinkertainen laskuri. Laskurin arvo on talletettu Postgres-tietokantaan, johon backend on yhteydessä [Sequelize](https://sequelize.org/)-kirjaston avulla. Frontend sisältää napit laskurin kasvattamiseen sekä nollaamiseen.

Projektiin on määritelty GitHub Action -workflow, joka luo projektista Docker-imagen ja pushaa sen Dockerhubiin. Sama image sisältää sekä backendin, että frontendin.

### Openshift

Käytössämme on Tietotekniikkakeskuksen OpenShift-klusteri. OpenShift on Kubernetes-klusteri tietyin lisämaustein. On suositeltavaa pitäytyä määrittelyissä mahdollisimman "puhtaassa" Kuberneteksessa, ja näin tulemme seuraavassakin tekemään. OpenShift sisältää mm. graafisen käyttöliittymän jonka kautta konfiguraatioita on mahdollista tehdä, mutta se ei ole suositeltua sillä näin päädytään usein hallitsemattoman epämääräisiin konfiguraatioihin.

Käytämmekin klusteria komentoriviltä ´oc´-komennon avulla. ´oc´ toimii samoin kun Kubernetesin ´kubectl´, mutta se sisältää muutamia OpenShift-spesifejä komentoja. Komennon ´kubectl´ käyttö voi olla järkevämpää niissä tapauksissa kun se riittää (eli melkein aina) sillä ko

Oletetaan, että oc asennettu ja ollaan Eduroamissa tai HY:n vpn:ssä. Protip konfaa [tabcomplete](https://docs.redhat.com/en/documentation/openshift_container_platform/4.9/html/cli_tools/openshift-cli-oc#cli-enabling-tab-completion).

Kirjaudu

```markdown
![Openshift Login](images/k1.ong)
```

### Pod ja deployment

### Podit Openshiftissä

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

Avaimen `spec` arvona määritellään deploymentin hallitseman podin (tai podien jos `replicas` on suurempi kuin 1) kontit. Eli tapauksessamme on yksi kontti, jonka image on `mluukkai/demoapp:1`. Kontille on annettu sen käyttämä tietokantaosoite määrittelemällä ympäristömuuttuja `DB_URL`.

Deployment luodaan klusterille seuraavalla komennolla ($ on komentokehote):

´´´
$ oc apply -f manifests/deployment.yaml
´´´

Deployment näyttää onnistuneen:

´´´
$ oc get deployments
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
demoapp-dep   1/1     1            1           16s
´´´

Deploymentin on siis tarkoitus käynnistää yksi podi. Katsotaan miltä podien tilanne näyttää:

´´´
oc get pod
NAME                           READY   STATUS    RESTARTS   AGE
demoapp-dep-5dbb664966-p6kfh   1/1     Running   0          25s
´´´

Podi on käynnissä.

Testataan nyt sovellusta. Sovellus ei näy vielä klusterin ulkopuolelle, mutta pääsemme siihen käsiksi klusterin sisältä podin IP-osoitteen avulla. Saamme IP-osoitteen selville komennolla ´oc describe pod´:

´´´
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
´´´

Käynnistetään klusterin sisälle [curl](https://curl.se/)-komennon sisätävä Docker [image](https://hub.docker.com/r/curlimages/curl):

´´´
$ oc run curlimage --image=curlimages/curl --restart=Never --command -- sleep infinity
$ oc exec -it curlimage sh
´´´

Olemme nyt podin sisällä, ja voimme kokeilla ottaa yhteyttä sovelluksen tekemällä GET-pyynnön testirajapintaan _api/ping_:

´´´
$ curl 10.12.2.177:3000/api/ping
{"message":"pong"}/home/curl_user 
´´´

Jostain syystä GET-pyyntö laskurin arvon palauttavaan rajapintaan _api/counter_: ei toimi:

´´´
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
´´´

Tutkitaan sovelluksen lokia komennolla ´oc logs´:

´´´
$ oc logs demoapp-dep-68c95df467-w4glg

> demoapp@0.0.0 prod
> NODE_ENV=production node server/index.js

Server running on http://localhost:3000
Unable to connect to the database: ConnectionError [SequelizeConnectionError]: no pg_hba.conf entry for host "128.214.137.139", user "ohtuprojektitesti", database "ohtuprojekti", SSL encryption
´´´

Tietokantayhteyden suhteen näyttää olevan jotain häikkää. Ja kun virheilmoitusta lukee tarkemmin, huomaa, että sovelluksen käyttämä tietokantaurl on väärä, tietokannan nimi on _ohtuprojektitesti_, ei _ohtuprojekti_ kuten konfiguraatio vahingossa määritteli.

Korjataan konfiguraatio ja päivitetään korjattu tilanne komennolla ´apply -f manifests/deployment.yaml´. Katsotaan lokia uudelleen:


$ demoapp git:(main) ✗ k logs demoapp-dep-6746c5d5dc-jjp88
´´´
> demoapp@0.0.0 prod
> NODE_ENV=production node server/index.js

Server running on http://localhost:3000
Executing (default): SELECT 1+1 AS result
Connected to the database
Executing (default): SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' AND table_name = 'counters'
Executing (default): SELECT i.relname AS name, ix.indisprimary AS primary, ix.indisunique AS unique, ix.indkey AS indkey, array_agg(a.attnum) as column_indexes, array_agg(a.attname) AS column_names, pg_get_indexdef(ix.indexrelid) AS definition FROM pg_class t, pg_class i, pg_index ix, pg_attribute a WHERE t.oid = ix.indrelid AND i.oid = ix.indexrelid AND a.attrelid = t.oid AND t.relkind = 'r' and t.relname = 'counters' GROUP BY i.relname, ix.indexrelid, ix.indisprimary, ix.indisunique, ix.indkey ORDER BY i.relname;
´´´

Yritetään jälleen

´´´
$ curl 10.13.2.114:3000/api/counter
curl: (7) Failed to connect to 10.13.2.114 port 3000 after 18688 ms: Could not connect to server
´´´

Ei toimi. Mikä vikana? IP-osoite ei pysy, tarkastetaan uusi:

´´´
curl 10.15.3.15:3000/api/counter
{"id":1,"value":0}
´´´

Toimii!


### Service

Podin IP-osoite siis näyttää vaihtuvan. Tämä johtuu siitä, että podi luodaan aina uudelleen kun siihen tehdään minkäänlaisia muutoksia. Tarvitaan siis jotain pysyvämpää, jotta ohjelma tavoitetaan varmemmin klusterin sisältä. Tätä varten on Kubernetesissa tarjolla [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Tehdään tiedostoon ´service.yaml´ seuraava määrittely:

´´´
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
´´´

Spec-osassa määritellään, että service "kohdistuu" sovellukseen _demoapp_, joka on portissa 3000, service tajoaa ulospäin portin 80.

Luodaan service, ja varmistetaan heti, että kaikki meni hyvin:

´´´
$ oc apply -f manifests/service.yaml
service/demoapp-svc created
$ oc get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
demoapp-svc   ClusterIP   172.30.117.37   <none>        80/TCP    3s
´´´

Klusteri on luonut servicelle IP-osoitteen jonka avulla pääsisimme käsiksi sovellukseen. Parempi tapa on käyttää servicen nimeä  _demoapp-svc_, tämä toimii klusterin sisällä DNS-nimenä palvelulle:

´´´
$ curl http://demoapp-svc/api/ping
{"message":"pong"}
$ curl http://demoapp-svc/api/counter
{"id":1,"value":0}
$ curl http://172.30.117.37/api/counter
{"message":"pong"}
´´´

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

### Imagestream

### Konfiguraatiot

### Tietokannan hankkiminen

### Kun joku menee vikaan

avaa possu

kubectl run -it --rm postgres-client --image=postgres:latest sh
