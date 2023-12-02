# MongoDB OpenShiftiin

## 1. Luo PersistentVolumeClaim

Jotta tietokannan sisältö tallennetaan pysyvästi pitää luoda PVC. Mene OpenShiftissä `Administrator` -> `Storage` -> `PersistentVolumeClaims` -> `Create PersistentVolumeClaim`.

Valitse oikea `storageClass` jonka olet saanut projektin tilauksen yhteydessä.

Laita `PersistentVolumeClaim name` kohtaan claimin nimi, Tässä esimerkissä käytetään `mongo-claim`.

Valitse tietokannan koko sen mukaan mitä olet tilannut. (Esimerkiksi 10GiB)

## 2. Luo MongoDB podi

Mene lisäämään mongo kohdasta `Developer` -> `+Add` -> `Container images`.

Laita imagen registryyn: `docker.io/mongo:<version>`, laita versioon se versio, jota haluat käyttää, esimerkiksi `docker.io/mongo:7.0.0`.

Vaihda kuvakkeeksi mongodb, jos haluat.

Laita nimeksi mongo, tätä nimeä käytetään tietokantaan yhdistäessä, älä kuitenkaan luo routea. (Varmista että `Create a route` ei ole valittu.)

## 3. Luo secretit

Mene kohtaan `Developer` -> `Secrets` -> `Create` -> `Key/value secret`.

Luo secretit:

- `MONGO_INITDB_DATABASE`: <tietokannan_nimi>
- `MONGO_INITDB_ROOT_PASSWORD`: <tähän joku vahva salasana, älä kuitenkaan käytä erikoismerkkejä>
- `MONGO_INITDB_ROOT_USERNAME`: root

Paina `Save`, sen jälkeen `Add Secret to workload` ja valitse mongo.

Käynnistä mongo podi uudestaan, jotta secretit tulevat voimaan.

Podi voidaan käynnistää uudestaan painamalla alanuolta ja sen jälkeen ylänuolta podin `Details` näkymästä.

Tässä vaiheessa voit laittaa sovelluksen tietokantaosoitteeksi:

```bash
mongodb://root:<salasana>@<podin_nimi>/<tietokannan_nimi>?authSource=admin
```

Käynnistä sovelluksesi uudestaan, tietokantayhteyden pitäisi toimia nyt. Sovelluksessa ei kuitenkaan vielä tässä vaiheessa ole määritelty pysyväistallennusta, eli tietokannan tiedot häviävät, jos käynnistät tietokannan uudelleen.

## 4. Ota käyttöön pysyväistallennus

Paina hiiren oikealla näppäimellä mongo podista topology näkymässä ja valitse `Edit Deployment`.

YAML-näkymässä korvaa `volumes` ja `volumeMounts` seuraavilla:

```bash
      volumes:
        - name: mongo-data
          persistentVolumeClaim:
            claimName: mongo-claim
```
```bash
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
```

Nyt tietojen pitäisi säilyä, vaikka mongo podi sammutetaan tai käynnistetään uudelleen.
