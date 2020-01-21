---
layout: post
title:  "Application Containerization and Microservice Orchestration"
description: "A step by step interactive tutorial of application containerization and microservice orchestration using Docker, starting from a simple script to a multi-service application stack"
img: "https://training.play-with-docker.com/images/linkextractor-microservice-diagram.png"
date:   2018-09-22
author: "@ibnesayeed"
tags: [beginner, linux, developer, microservice, orchestration, linkextractor, api, python, php, ruby]
categories: beginner
terms: 1
---

In questo tutorial impareremo la containerizzazione di applicazioni di base usando Docker, eseguendo vari componenti di un'applicazione come microservizi.
Utilizzeremo [Docker Compose] (https://docs.docker.com/compose/) per l'orchestrazione durante lo sviluppo.
Questo tutorial è rivolto ai principianti che hanno familiarità di base con Docker.
Se non conosci Docker, ti consigliamo di consultare prima il tutorial [Docker per principianti] (/ beginner-linux).

Inizieremo da uno script Python di base che estrae i collegamenti da una determinata pagina Web e lo evolve gradualmente in uno stack di applicazioni multi-servizio.
Il codice demo è disponibile nel repository [Link Extractor] (https://github.com/ibnesayeed/linkextractor).
Il codice è organizzato in passaggi che introducono in modo incrementale modifiche e nuovi concetti.
Dopo il completamento, lo stack dell'applicazione conterrà i seguenti microservizi:

* Un'applicazione Web scritta in PHP, pubblicata utilizzando un server Apache, accetta un URL come input e riepiloga i collegamenti estratti da esso
* L'applicazione Web comunica con un server API scritto in Python (e Ruby) che si occupa dell'estrazione del collegamento e restituisce una risposta JSON
* Una cache Redis utilizzata dal server API per evitare ripetute operazioni di recupero e estrazione dei collegamenti per le pagine già scartate

Il server API caricherà la pagina del collegamento di input dal Web solo se non si trova nella cache.
Lo stack alla fine apparirà come nella figura seguente:

! [A Microservice Architecture dell'applicazione Link Extractor] (/ images / linkextractor-microservice-diagram.png)

Questo tutorial è stato inizialmente sviluppato per un colloquio nel [Dipartimento di Informatica] (https://odu.edu/compsci) della [Old Dominion University] (https://www.odu.edu/), Norfolk, Virginia. A [registrazione video] (https://www.youtube.com/watch?v=Y_X0F2FgYm8), [diapositive di presentazione] (https://www.slideshare.net/ibnesayeed/introducing-docker-application-containerization-service- orchestrazione) e una breve descrizione del discorso in [un post sul blog] (https://ws-dl.blogspot.com/2017/12/2017-12-03-introducing-docker.html).


> ** Passaggi: **
> * Sommario
> {: toc}


## Preparazione ambiente

Cominciamo prima clonando il repository di codice demo, cambiando la directory di lavoro e passando al branch `demo`.

```.term1
git clone https://github.com/ibnesayeed/linkextractor.git
cd linkextractor
git checkout demo
```


## Passo 0: Semplice script di estrazione dei collegamenti

Passa al branch `step0` ed elenca i file al suo interno.

```.term1
git checkout step0
tree
```

```
.
├── README.md
└── linkextractor.py

0 directories, 2 files
```

Il file `linkextractor.py` è quello interessante qui, quindi diamo un'occhiata al suo contenuto:

```.term1
cat linkextractor.py
```

```py
#!/usr/bin/env python

import sys
import requests
from bs4 import BeautifulSoup

res = requests.get(sys.argv[-1])
soup = BeautifulSoup(res.text, "html.parser")
for link in soup.find_all("a"):
    print(link.get("href"))
```

Questo è un semplice script Python che importa tre pacchetti: `sys` dalla libreria standard e due popolari pacchetti di terze parti` requests` e `bs4`.
L'argomento della riga di comando fornito dall'utente (che dovrebbe essere un URL per una pagina HTML) viene usato per recuperare la pagina usando il pacchetto `requests`, quindi analizzato usando `BeautifulSoup`.
L'oggetto analizzato viene quindi ripetuto per trovare tutti gli elementi di ancoraggio (ovvero tag <a> `) e stampare il valore del loro attributo` href` che contiene il collegamento ipertestuale.

Tuttavia, questo script apparentemente semplice potrebbe non essere il più semplice da eseguire su una macchina che non soddisfa i suoi requisiti.
Il file `README.md` suggerisce come eseguirlo, quindi proviamo:

```.term1
./linkextractor.py http://example.com/
```

```
bash: ./linkextractor.py: Permission denied
```

Quando abbiamo provato a eseguirlo come uno script, abbiamo ricevuto l'errore `Permission denied`.
Controlliamo le autorizzazioni correnti su questo file:

```.term1
ls -l linkextractor.py
```

```
-rw-r--r--    1 root     root           220 Sep 23 16:26 linkextractor.py
```

Questa autorizzazione corrente `-rw-r - r -` indica che lo script non è impostato per essere eseguibile.
Possiamo cambiarlo eseguendo `chmod a + x linkextractor.py` oppure eseguirlo come un programma Python invece di uno script auto-eseguito come illustrato di seguito:

```.term1
python linkextractor.py
```

```py
Traceback (most recent call last):
  File "linkextractor.py", line 5, in <module>
    from bs4 import BeautifulSoup
ImportError: No module named bs4
```

Qui abbiamo ricevuto il primo messaggio `ImportError` perché ci manca il pacchetto di terze parti necessario allo script.
Possiamo installare quel pacchetto Python (e potenzialmente altri pacchetti mancanti) usando una delle molte tecniche per farlo funzionare, ma è troppo lavoro per uno script così semplice, che potrebbe non essere ovvio per coloro che non hanno familiarità con l'ecosistema di Python .

A seconda della macchina e del sistema operativo su cui stai tentando di eseguire questo script, su quale software è già installato e su quanto accesso hai, potresti incontrare alcune di queste potenziali difficoltà:

* Lo script è eseguibile?
* Python è installato sulla macchina?
* È possibile installare software sulla macchina?
* `Pip` è installato?
* Le librerie Python `requests` e` beautifulsoup4` sono installate?

È qui che gli strumenti di containerizzazione delle applicazioni come Docker sono utili.
Nel prossimo passaggio cercheremo di containerizzare questo script e di semplificarne l'esecuzione.

## Passo 1: Script estrattore di collegamenti containerizzato

Col branch `step1` ed elenca i file al suo interno.

```.term1
git checkout step1
tree
```

```
.
├── Dockerfile
├── README.md
└── linkextractor.py

0 directories, 3 files
```

Abbiamo aggiunto un nuovo file (ad esempio, `Dockerfile`) in questo passaggio.
Diamo un'occhiata al suo contenuto:

```.term1
cat Dockerfile
```

```dockerfile
FROM       python:3
LABEL      maintainer="Sawood Alam <@ibnesayeed>"

RUN        pip install beautifulsoup4
RUN        pip install requests

WORKDIR    /app
COPY       linkextractor.py /app/
RUN        chmod a+x linkextractor.py

ENTRYPOINT ["./linkextractor.py"]
```

Usando questo `Dockerfile` possiamo preparare un'immagine Docker per questo script.
Partiamo dall'immagine Docker `python` ufficiale che contiene l'ambiente runtime di Python e gli strumenti necessari per installare i pacchetti e le dipendenze di Python.
Aggiungiamo quindi alcuni metadati come etichette (questo passaggio non è essenziale, ma è comunque una buona pratica).
Le successive due istruzioni eseguono il comando `pip install` per installare i due pacchetti di terze parti necessari per il corretto funzionamento dello script.
Quindi creiamo una directory di lavoro `/ app`, copiamo il file` linkextractor.py` in esso e cambiamo le sue autorizzazioni per renderlo uno script eseguibile.
Infine, impostiamo lo script come punto di entrata per l'immagine.

Finora, abbiamo appena descritto come vogliamo che sia la nostra immagine Docker, ma non ne abbiamo davvero creata una.
Quindi facciamo solo questo:

```.term1
docker image build -t linkextractor:step1 .
```

Questo comando dovrebbe produrre un output come illustrato di seguito:

```
Sending build context to Docker daemon  171.5kB
Step 1/8 : FROM       python:3

... [OUTPUT REDACTED] ...

Successfully built 226196ada9ab
Successfully tagged linkextractor:step1
```

Abbiamo creato un'immagine Docker denominata `linkextractor: step1` basata sul` Dockerfile` illustrato sopra.
Se la compilazione ha avuto esito positivo, dovremmo essere in grado di vederlo nell'elenco delle immagini:

```.term1
docker image ls
```

```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
linkextractor       step1               e067c677be37        2 seconds ago       931MB
python              3                   a9d071760c82        2 weeks ago         923MB
```

Questa immagine dovrebbe contenere tutti gli ingredienti necessari per eseguire lo script ovunque su una macchina che supporti Docker.
Ora, eseguiamo un container unico con questa immagine per estrarre i collegamenti da alcune pagine Web live:

```.term1
docker container run -it --rm linkextractor:step1 http://example.com/
```

Questo genera un singolo link presente nella semplice pagina web [esempio.com] (http://esempio.com/):

```
http://www.iana.org/domains/example
```

Proviamolo su una pagina web con più link al suo interno:

```.term1
docker container run -it --rm linkextractor:step1 https://training.play-with-docker.com/
```

```
/
/about/
#ops
#dev
/ops-stage1
/ops-stage2
/ops-stage3
/dev-stage1
/dev-stage2
/dev-stage3
/alacart
https://twitter.com/intent/tweet?text=Play with Docker Classroom&url=https://training.play-with-docker.com/&via=docker&related=docker
https://facebook.com/sharer.php?u=https://training.play-with-docker.com/
https://plus.google.com/share?url=https://training.play-with-docker.com/
http://www.linkedin.com/shareArticle?mini=true&url=https://training.play-with-docker.com/&title=Play%20with%20Docker%20Classroom&source=https://training.play-with-docker.com
https://2018.dockercon.com/
https://2018.dockercon.com/
https://success.docker.com/training/
https://community.docker.com/registrations/groups/4316
https://docker.com
https://www.docker.com
https://www.facebook.com/docker.run
https://twitter.com/docker
https://www.github.com/play-with-docker/play-with-docker.github.io
```

Sembra buono, ma possiamo migliorare l'output.
Ad esempio, alcuni collegamenti sono relativi, possiamo convertirli in URL completi e fornire anche il testo di ancoraggio a cui sono collegati.
Nel prossimo passaggio apporteremo queste modifiche e alcuni altri miglioramenti allo script.


## Passo 2: Modulo di estrazione collegamento con URI completo e testo di ancoraggio

Passa al branch `step2` ed elenca i file al suo interno.

```.term1
git checkout step2
tree
```

```
.
├── Dockerfile
├── README.md
└── linkextractor.py

0 directories, 3 files
```

In questo passo lo script `linkextractor.py` viene aggiornato con le seguenti modifiche funzionali:

* I percorsi sono stati resi tutti URL completi
* Segnalazione di collegamenti e testi di ancoraggio
* Utilizzabile come modulo in altri script

Diamo un'occhiata allo script aggiornato:

```.term1
cat linkextractor.py
```

```py
#!/usr/bin/env python

import sys
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin

def extract_links(url):
    res = requests.get(url)
    soup = BeautifulSoup(res.text, "html.parser")
    base = url
    # TODO: Update base if a <base> element is present with the href attribute
    links = []
    for link in soup.find_all("a"):
        links.append({
            "text": " ".join(link.text.split()) or "[IMG]",
            "href": urljoin(base, link.get("href"))
        })
    return links

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("\nUsage:\n\t{} <URL>\n".format(sys.argv[0]))
        sys.exit(1)
    for link in extract_links(sys.argv[-1]):
        print("[{}]({})".format(link["text"], link["href"]))
```

La logica di estrazione dei collegamenti viene astratta in una funzione "extract_links" che accetta un URL come parametro e restituisce un elenco di oggetti contenenti testi di ancoraggio e collegamenti ipertestuali normalizzati.
Questa funzionalità può ora essere importata in altri script come modulo (che utilizzeremo nel passaggio successivo).

Ora costruiamo una nuova immagine e vediamo queste modifiche in vigore:

```.term1
docker image build -t linkextractor:step2 .
```

Abbiamo usato un nuovo tag `linkextractor: step2` per questa immagine in modo da non sovrascrivere l'immagine da` step0` per illustrare che possono coesistere e che i contenitori possono essere eseguiti usando una di queste immagini.

```.term1
docker image ls
```

```
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
linkextractor       step2               be2939eada96        3 seconds ago        931MB
linkextractor       step1               673d045a822f        About a minute ago   931MB
python              3                   a9d071760c82        2 weeks ago          923MB
```

L'esecuzione di un container unico usando l'immagine `linkextractor: step2` dovrebbe ora produrre un output migliorato:

```.term1
docker container run -it --rm linkextractor:step2 https://training.play-with-docker.com/
```

```
[Play with Docker classroom](https://training.play-with-docker.com/)
[About](https://training.play-with-docker.com/about/)
[IT Pros and System Administrators](https://training.play-with-docker.com/#ops)
[Developers](https://training.play-with-docker.com/#dev)
[Stage 1: The Basics](https://training.play-with-docker.com/ops-stage1)
[Stage 2: Digging Deeper](https://training.play-with-docker.com/ops-stage2)
[Stage 3: Moving to Production](https://training.play-with-docker.com/ops-stage3)
[Stage 1: The Basics](https://training.play-with-docker.com/dev-stage1)
[Stage 2: Digging Deeper](https://training.play-with-docker.com/dev-stage2)
[Stage 3: Moving to Staging](https://training.play-with-docker.com/dev-stage3)
[Full list of individual labs](https://training.play-with-docker.com/alacart)
[[IMG]](https://twitter.com/intent/tweet?text=Play with Docker Classroom&url=https://training.play-with-docker.com/&via=docker&related=docker)
[[IMG]](https://facebook.com/sharer.php?u=https://training.play-with-docker.com/)
[[IMG]](https://plus.google.com/share?url=https://training.play-with-docker.com/)
[[IMG]](http://www.linkedin.com/shareArticle?mini=true&url=https://training.play-with-docker.com/&title=Play%20with%20Docker%20Classroom&source=https://training.play-with-docker.com)
[[IMG]](https://2018.dockercon.com/)
[DockerCon 2018 in San Francisco](https://2018.dockercon.com/)
[training.docker.com](https://success.docker.com/training/)
[Register here](https://community.docker.com/registrations/groups/4316)
[Docker, Inc.](https://docker.com)
[[IMG]](https://www.docker.com)
[[IMG]](https://www.facebook.com/docker.run)
[[IMG]](https://twitter.com/docker)
[[IMG]](https://www.github.com/play-with-docker/play-with-docker.github.io)
```

L'esecuzione di un container usando l'immagine precedente `linkextractor: step1` dovrebbe comunque produrre il vecchio output:

```.term1
docker container run -it --rm linkextractor:step1 https://training.play-with-docker.com/
```

Finora, abbiamo imparato a containerizzare uno script con le sue dipendenze necessarie per renderlo più portatile.
Abbiamo anche imparato come apportare modifiche all'applicazione e creare diverse varianti di immagini Docker che possono coesistere.
Nel passaggio successivo creeremo un servizio Web che utilizzerà questo script e renderà il servizio eseguito all'interno di un container Docker.


## Passo 3: Collegamento del servizio API di Extractor

Passa al branch `step3` ed elenca i file al suo interno.

```.term1
git checkout step3
tree
```

```
.
├── Dockerfile
├── README.md
├── linkextractor.py
├── main.py
└── requirements.txt

0 directories, 5 files
```

In questo passaggio sono state apportate le seguenti modifiche:

* Aggiunto uno script server `main.py` che utilizza il modulo di estrazione dei collegamenti scritto nell'ultimo passaggio
* Il `Dockerfile` viene aggiornato per fare invece riferimento al file` main.py`
* Il server è accessibile come API WEB su `http: // <nomehost> [: <prt>] / api / <url>`
* Le dipendenze vengono spostate nel file `requirements.txt`
* Utilizzo della mappatura delle porte per rendere accessibile il servizio all'esterno del contenitore (il server `Flask` usato qui ascolta sulla porta `5000` di default)

Diamo prima un'occhiata al `Dockerfile` per le modifiche:

```.term1
cat Dockerfile
```

```dockerfile
FROM       python:3
LABEL      maintainer="Sawood Alam <@ibnesayeed>"

WORKDIR    /app
COPY       requirements.txt /app/
RUN        pip install -r requirements.txt

COPY       *.py /app/
RUN        chmod a+x *.py

CMD        ["./main.py"]
```

Da quando abbiamo iniziato a usare `requirements.txt` per le dipendenze, non è più necessario eseguire il comando` pip install` per i singoli pacchetti.
La direttiva `ENTRYPOINT` è sostituita da` CMD` e si riferisce allo script `main.py` che ha il codice del server perché non vogliamo usare questa immagine per comandi una tantum ora.

Il modulo `linkextractor.py` rimane invariato in questo passaggio, quindi diamo un'occhiata al file` main.py` appena aggiunto:

```.term1
cat main.py
```

```py
#!/usr/bin/env python

from flask import Flask
from flask import request
from flask import jsonify
from linkextractor import extract_links

app = Flask(__name__)

@app.route("/")
def index():
    return "Usage: http://<hostname>[:<prt>]/api/<url>"

@app.route("/api/<path:url>")
def api(url):
    qs = request.query_string.decode("utf-8")
    if qs != "":
        url += "?" + qs
    links = extract_links(url)
    return jsonify(links)

app.run(host="0.0.0.0")
```

Qui, stiamo importando la funzione `extract_links` dal modulo` linkextractor` e stiamo convertendo la lista di oggetti restituiti in una risposta JSON.

È tempo di creare una nuova immagine con queste modifiche in atto:

```.term1
docker image build -t linkextractor:step3 .
```

Quindi eseguire il container in modalità staccata (flag `-d`) in modo che il terminale sia disponibile per altri comandi mentre il container è ancora in esecuzione.
Si noti che stiamo mappando la porta `5000` del container con il` 5000` dell'host (usando l'argomento `-p 5000: 5000`) per renderlo accessibile dall'host.
Stiamo anche assegnando un nome (`--name = linkextractor`) al container per facilitare la visualizzazione dei registri e l'uccisione o la rimozione del container.

```.term1
docker container run -d -p 5000:5000 --name=linkextractor linkextractor:step3
```

Se le cose vanno bene, dovremmo essere in grado di vedere il container elencato nella condizione `Up`:

```.term1
docker container ls
```

```
CONTAINER ID        IMAGE                 COMMAND             CREATED             STATUSPORTS                    NAMES
d69c0150a754        linkextractor:step3   "./main.py"         9 seconds ago       Up 8 seconds0.0.0.0:5000->5000/tcp   linkextractor
```

Ora possiamo fare una richiesta HTTP nel formato `/ api / <url>` per comunicare con questo server e recuperare la risposta contenente i collegamenti estratti:

```.term1
curl -i http://localhost:5000/api/http://example.com/
```

```json
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 78
Server: Werkzeug/0.14.1 Python/3.7.0
Date: Sun, 23 Sep 2018 20:52:56 GMT

[{"href":"http://www.iana.org/domains/example","text":"More information..."}]
```

Ora, abbiamo il servizio API in esecuzione che accetta le richieste nel formato `/ api / <url>` e risponde con un JSON contenente collegamenti ipertestuali e testi di ancoraggio di tutti i collegamenti presenti nella pagina Web a dare `<url>`.

Dal momento che il container è in esecuzione in modalità distaccata, non potremo vedere cosa sta succedendo all'interno, ma possiamo vedere i log usando il nome `linkextractor` che abbiamo assegnato al nostro container:

```.term1
docker container logs linkextractor
```

```
* Serving Flask app "main" (lazy loading)
* Environment: production
  WARNING: Do not use the development server in a production environment.
  Use a production WSGI server instead.
* Debug mode: off
* Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
172.17.0.1 - - [23/Sep/2018 20:52:56] "GET /api/http://example.com/ HTTP/1.1" 200 -
```


Possiamo vedere i messaggi registrati dal momento in cui il server parte a lavorare, e una voce del registro richieste (HTTP) quando abbiamo eseguito il comando `curl.
Ora possiamo fermare e rimuovere questo container:

```.term1
docker container rm -f linkextractor
```

In questo passaggio abbiamo eseguito con successo un servizio API in ascolto sulla porta `5000`.
Questo è fantastico, ma le risposte API e JSON sono per le macchine, quindi nel prossimo passaggio eseguiremo un servizio web con un'interfaccia web a misura d'uomo in aggiunta a questo servizio API.


## Passo 4: Link Extractor API e Web Front End Services

Passa al branch `step4` ed elenca i file al suo interno.

```.term1
git checkout step4
tree
```

```
.
├── README.md
├── api
│   ├── Dockerfile
│   ├── linkextractor.py
│   ├── main.py
│   └── requirements.txt
├── docker-compose.yml
└── www
    └── index.php

2 directories, 7 files
```

In questo passaggio sono state apportate le seguenti modifiche dall'ultimo passaggio:

* Il servizio API JSON di estrazione link (scritto in Python) viene spostato in una cartella `./api` separata che ha lo stesso codice esatto del passaggio precedente
* Un'applicazione Web front-end è scritta in PHP nella cartella `. /www` che comunica con l'API JSON
* L'applicazione PHP è montata all'interno dell'immagine Docker `php: 7-apache` ufficiale per una più facile modifica durante lo sviluppo
* L'applicazione Web è resa accessibile su `http: // <nomehost> [: <prt>] /? Url = <url-encoded-url>`
* Una variabile d'ambiente `API_ENDPOINT` viene utilizzata all'interno dell'applicazione PHP per configurarla per comunicare con il server API JSON
* Un file `docker-compose.yml` è scritto per creare vari componenti e incollarli insieme

In questo passaggio stiamo pianificando di eseguire due contenitori separati, uno per l'API e l'altro per l'interfaccia Web.
Quest'ultimo ha bisogno di un modo per comunicare con il server API.
Per consentire ai due container di comunicare tra loro, possiamo mappare le loro porte sul computer host e utilizzarlo per il routing delle richieste oppure possiamo posizionare i container in una singola rete privata e accedervi direttamente.
Docker ha un eccellente supporto di rete e fornisce comandi utili per gestirle.
Inoltre, in una rete Docker i container si identificano usando i loro nomi come nomi host per evitare di cercare i loro indirizzi IP nella rete privata.
Tuttavia, non eseguiremo alcuna operazione manualmente, ma utilizzeremo Docker Compose per automatizzare molte di queste attività.

Diamo un'occhiata al file `docker-compose.yml` che abbiamo:

```.term1
cat docker-compose.yml
```

```yml
version: '3'

services:
  api:
    image: linkextractor-api:step4-python
    build: ./api
    ports:
      - "5000:5000"
  web:
    image: php:7-apache
    ports:
      - "80:80"
    environment:
      - API_ENDPOINT=http://api:5000/api/
    volumes:
      - ./www:/var/www/html
```

Questo è un semplice file YAML che descrive i due servizi `api` e` web`.
Il servizio `api` userà l'immagine` linkextractor-api: step4-python` che non è ancora stata costruita, ma sarà costruita su richiesta usando il `Dockerfile` dalla directory `./api`.
Questo servizio sarà esposto sulla porta `5000` dell'host.

Il secondo servizio chiamato `web` utilizzerà l'immagine` php: 7-apache` ufficiale direttamente da DockerHub, ecco perché non abbiamo un file Docker per esso.
Il servizio sarà esposto sulla porta HTTP predefinita (cioè, `80`).
Forniremo una variabile d'ambiente chiamata `API_ENDPOINT` con il valore `http://api:5000/api/` per dire allo script PHP dove connettersi per l'accesso all'API.
Si noti che qui non si utilizza un indirizzo IP, invece viene utilizzato `api:5000` perché avremo una voce dinamica del nome host nella rete privata per il servizio API corrispondente al nome del servizio.
Infine, assoceremo mount la cartella `./www` per rendere disponibile il file` index.php` all'interno del container del servizio `web` in `/var/www/html`, che è la radice web predefinita per Apache server web.

Ora diamo un'occhiata al file `www / index.php` rivolto all'utente:

```.term1
cat www/index.php
```

Questo è un file lungo che contiene principalmente tutto il markup e gli stili della pagina.
Tuttavia, l'importante blocco di codice si trova all'inizio del file, come illustrato di seguito:

```php
$api_endpoint = $_ENV["API_ENDPOINT"] ?: "http://localhost:5000/api/";
$url = "";
if(isset($_GET["url"]) && $_GET["url"] != "") {
  $url = $_GET["url"];
  $json = @file_get_contents($api_endpoint . $url);
  if($json == false) {
    $err = "Something is wrong with the URL: " . $url;
  } else {
    $links = json_decode($json, true);
    $domains = [];
    foreach($links as $link) {
      array_push($domains, parse_url($link["href"], PHP_URL_HOST));
    }
    $domainct = @array_count_values($domains);
    arsort($domainct);
  }
}
```

La variabile `$ api_endpoint` è inizializzata con il valore della variabile d'ambiente fornita dal file` docker-compose.yml` come `$ _ENV [" API_ENDPOINT "]` (altrimenti ricade su un valore predefinito di `http: // localhost: 5000 / api / `).
La richiesta viene fatta usando la funzione `file_get_contents` che usa la variabile` $ api_endpoint` e l'URL fornito dall'utente da `$ _GET [" url "]`.
Alcune analisi e trasformazioni vengono eseguite sulla risposta ricevuta che vengono successivamente utilizzate nel markup per popolare la pagina.

Portiamo questi servizi in modalità staccata usando l'utility `docker-compose`:

```.term1
docker-compose up -d --build
```

```
Creating network "linkextractor_default" with the default driver
Pulling web (php:7-apache)...
7-apache: Pulling from library/php

... [OUTPUT REDACTED] ...

Status: Downloaded newer image for php:7-apache
Building api
Step 1/8 : FROM       python:3

... [OUTPUT REDACTED] ...

Successfully built 1f419be1c2bf
Successfully tagged linkextractor-api:step4-python
Creating linkextractor_web_1 ... done
Creating linkextractor_api_1 ... done
```

Questo output mostra che Docker Compose ha creato automaticamente una rete chiamata `linkextractor_default`, ha estratto l'immagine` php: 7-apache` da DockerHub, ha creato l'immagine `api: python` usando il nostro` Dockerfile` locale e infine ha fatto girare due container `linkextractor_web_1` e `linkextractor_api_1` che corrispondono ai due servizi che abbiamo definito nel file YAML sopra.

Il controllo dell'elenco dei contenitori in esecuzione conferma che i due servizi sono effettivamente in esecuzione:

```.term1
docker container ls
```

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS   PORTS                    NAMES
268b021b5a2c        php:7-apache                     "docker-php-entrypoi…"   3 minutes ago       Up 3 minutes        0.0.0.0:80->80/tcp       linkextractor_web_1
5bc266b4e43d        linkextractor-api:step4-python   "./main.py"              3 minutes ago       Up 3 minutes        0.0.0.0:5000->5000/tcp   linkextractor_api_1
```

Ora dovremmo essere in grado di parlare con il servizio API come prima:

```.term1
curl -i http://localhost:5000/api/http://example.com/
```

Per accedere all'interfaccia Web [fare clic per aprire l'Estrattore link] (/) {: data-term = ". Term1"} {: data-port = "80"}.
Quindi compila il modulo con `https: // training.play-with-docker.com /` (o qualsiasi URL della pagina HTML di tua scelta) e invia per estrarre i collegamenti da esso.

Abbiamo appena creato un'applicazione con architettura a microservizi, isolando le singole attività in servizi separati rispetto alle applicazioni monolitiche in cui tutto è riunito in un'unica unità.
Le applicazioni di microservizi sono relativamente più semplici da ridimensionare, gestire e spostare.
Consentono inoltre di sostituire facilmente i componenti con un servizio equivalente.
Ne parleremo più avanti.

Ora, modifichiamo il file `www / index.php` per sostituire tutte le occorrenze di` Link Extractor` con `Super Link Extractor`:

```.term1
sed -i 's/Link Extractor/Super Link Extractor/g' www/index.php
```

Il ricaricamento dell'interfaccia Web dell'applicazione (o [facendo clic qui] (/) {: data-term = ". Term1"} {: data-port = "80"}) dovrebbe ora riflettere questa modifica nel titolo, nell'intestazione e piè di pagina.
Questo accade perché la cartella `./www` è montata all'interno del container, quindi qualsiasi modifica apportata all'esterno si rifletterà all'interno del container o viceversa.
Questo approccio è molto utile nello sviluppo, ma nell'ambiente di produzione preferiremmo che le nostre immagini Docker fossero autonome.
Ripristiniamo ora queste modifiche per pulire il tracciamento di Git:

```.term1
git reset --hard
```

Prima di passare al passaggio successivo, è necessario chiudere questi servizi, ma Docker Compose può aiutarci a gestirlo molto facilmente:

```.term1
docker-compose down
```

```
Stopping linkextractor_api_1 ... done
Stopping linkextractor_web_1 ... done
Removing linkextractor_api_1 ... done
Removing linkextractor_web_1 ... done
Removing network linkextractor_default
```

Nel passaggio successivo aggiungeremo un ulteriore servizio al nostro stack e creeremo un'immagine personalizzata autonoma per il nostro servizio di interfaccia web.


## Passo 5: Servizio Redis per la memorizzazione nella cache

Passa al branch `step5` ed elenca i file al suo interno.

```.term1
git checkout step5
tree
```

```
.
├── README.md
├── api
│   ├── Dockerfile
│   ├── linkextractor.py
│   ├── main.py
│   └── requirements.txt
├── docker-compose.yml
└── www
    ├── Dockerfile
    └── index.php

2 directories, 8 files
```

Alcune modifiche evidenti rispetto al passaggio precedente sono le seguenti:

* Un altro `Dockerfile` viene aggiunto nella cartella `./www` per l'applicazione web PHP per creare un'immagine autonoma ed evitare il montaggio di file live
* Un container Redis viene aggiunto per la memorizzazione nella cache utilizzando l'immagine Docker Redis ufficiale
* Il servizio API parla con il servizio Redis per evitare di scaricare e analizzare pagine che erano già state cancellate in precedenza
* Una variabile d'ambiente `REDIS_URL` viene aggiunta al servizio API per consentirgli di connettersi alla cache Redis

Ispezioniamo prima il `Dockerfile` appena aggiunto nella cartella `./www`:

```.term1
cat www/Dockerfile
```

```dockerfile
FROM       php:7-apache
LABEL      maintainer="Sawood Alam <@ibnesayeed>"

ENV        API_ENDPOINT="http://localhost:5000/api/"

COPY       . /var/www/html/
```

Questo è un `Dockerfile` piuttosto semplice che usa l'immagine` php:7-apache` ufficiale come base e copia tutti i file dalla cartella `./www` nella cartella `/var/www/html/` della immagine.
Questo è esattamente ciò che stava accadendo nel passaggio precedente, ma che è stato montato in un bind usando un volume, mentre qui stiamo rendendo il codice parte dell'immagine autonoma.
Abbiamo anche aggiunto la variabile d'ambiente `API_ENDPOINT` con un valore predefinito, che suggerisce implicitamente che si tratta di un'informazione importante che deve essere presente affinché il servizio funzioni correttamente (e dovrebbe essere personalizzato in fase di esecuzione con un valore appropriato ).

Successivamente, esamineremo il file `api / main.py` del server API in cui stiamo utilizzando la cache Redis:

```.term1
cat api/main.py
```

Il file ha molte righe, ma i bit importanti sono come illustrato di seguito:

```py
redis_conn = redis.from_url(os.getenv("REDIS_URL", "redis://localhost:6379"))
# ...
    jsonlinks = redis.get(url)
    if not jsonlinks:
        links = extract_links(url)
        jsonlinks = json.dumps(links, indent=2)
        redis.set(url, jsonlinks)
```

Questa volta il servizio API deve sapere come connettersi all'istanza di Redis poiché la utilizzerà per la memorizzazione nella cache.
Queste informazioni possono essere rese disponibili in fase di esecuzione usando la variabile d'ambiente `REDIS_URL`.
Una voce `ENV` corrispondente viene aggiunta anche nel` Dockerfile` del servizio API con un valore predefinito.

Un'istanza client `redis` viene creata usando il nome host` redis` (uguale al nome del servizio come vedremo più avanti) e la porta Redis predefinita `6379`.
In primo luogo stiamo provando a vedere se è presente una cache nell'archivio Redis per un determinato URL, in caso contrario utilizziamo la funzione `extract_links` come prima e popoliamo la cache per tentativi futuri.

Ora, diamo un'occhiata al file aggiornato `docker-compose.yml`:

```.term1
cat docker-compose.yml
```

```yml
version: '3'

services:
  api:
    image: linkextractor-api:step5-python
    build: ./api
    ports:
      - "5000:5000"
    environment:
      - REDIS_URL=redis://redis:6379
  web:
    image: linkextractor-web:step5-php
    build: ./www
    ports:
      - "80:80"
    environment:
      - API_ENDPOINT=http://api:5000/api/
  redis:
    image: redis
```

La configurazione del servizio `api` rimane sostanzialmente la stessa di prima, ad eccezione del tag immagine aggiornato e della variabile d'ambiente aggiunta `REDIS_URL` che punta al servizio Redis.
Per il servizio `web`, stiamo usando l'immagine personalizzata `linkextractor-web:step5-php` che sarà costruita usando il file Dockerfile appena aggiunto nella cartella `./www`.
Non stiamo più montando la cartella `./www` usando la configurazione di` volumi`.
Finally, a new service named `redis` is added that will use the official image from DockerHub and needs no specific configurations for now.
This service is accessible to the Python API using its service name, the same way the API service is accessible to the PHP front-end service.

Let's boot these services up:

```.term1
docker-compose up -d --build
```

```
... [OUTPUT REDACTED] ...

Creating linkextractor_web_1   ... done
Creating linkextractor_api_1   ... done
Creating linkextractor_redis_1 ... done
```

Now, that all three services are up, access the web interface by [clicking the Link Extractor](/){:data-term=".term1"}{:data-port="80"}.
There should be no visual difference from the previous step.
However, if you extract links from a page with a lot of links, the first time it should take longer, but the successive attempts to the same page should return the response fairly quickly.
To check whether or not the Redis service is being utilized, we can use `docker-compose exec` followed by the `redis` service name and the Redis CLI's [monitor](https://redis.io/commands/monitor) command:

```.term1
docker-compose exec redis redis-cli monitor
```

Now, try to extract links from some web pages using the web interface and see the difference in Redis log entries for pages that are scraped the first time and those that are repeated.
Before continuing further with the tutorial, stop the interactive `monitor` stream as a result of the above `redis-cli` command by pressing `Ctrl + C` keys while the interactive terminal is in focus.

Now that we are not mounting the `/www` folder inside the container, local changes should not reflect in the running service:

```.term1
sed -i 's/Link Extractor/Super Link Extractor/g' www/index.php
```

Verify that the changes made locally do not reflect in the running service by reloading the web interface and then revert changes:

```.term1
git reset --hard
```

Now, shut these services down and get ready for the next step:

```.term1
docker-compose down
```

```
Stopping linkextractor_web_1   ... done
Stopping linkextractor_redis_1 ... done
Stopping linkextractor_api_1   ... done
Removing linkextractor_web_1   ... done
Removing linkextractor_redis_1 ... done
Removing linkextractor_api_1   ... done
Removing network linkextractor_default
```

We have successfully orchestrated three microservices to compose our Link Extractor application.
We now have an application stack that represents the architecture illustrated in the figure shown in the introduction of this tutorial.
In the next step we will explore how easy it is to swap components from an application with the microservice architecture.


## Step 6: Swap Python API Service with Ruby

Checkout the `step6` branch and list files in it.

```.term1
git checkout step6
tree
```

```
.
├── README.md
├── api
│   ├── Dockerfile
│   ├── Gemfile
│   └── linkextractor.rb
├── docker-compose.yml
├── logs
└── www
    ├── Dockerfile
    └── index.php

3 directories, 7 files
```

Some significant changes from the previous step include:

* The API service written in Python is replaced with a similar Ruby implementation
* The `API_ENDPOINT` environment variable is updated to point to the new Ruby API service
* The link extraction cache event (HIT/MISS) is logged and is persisted using volumes

Notice that the `./api` folder does not contain any Python scripts, instead, it now has a Ruby file and a `Gemfile` to manage dependencies.

Let's have a quick walk through the changed files:

```.term1
cat api/linkextractor.rb
```

```rb
#!/usr/bin/env ruby
# encoding: utf-8

require "sinatra"
require "open-uri"
require "uri"
require "nokogiri"
require "json"
require "redis"

set :protection, :except=>:path_traversal

redis = Redis.new(url: ENV["REDIS_URL"] || "redis://localhost:6379")

Dir.mkdir("logs") unless Dir.exist?("logs")
cache_log = File.new("logs/extraction.log", "a")

get "/" do
  "Usage: http://<hostname>[:<prt>]/api/<url>"
end

get "/api/*" do
  url = [params['splat'].first, request.query_string].reject(&:empty?).join("?")
  cache_status = "HIT"
  jsonlinks = redis.get(url)
  if jsonlinks.nil?
    cache_status = "MISS"
    jsonlinks = JSON.pretty_generate(extract_links(url))
    redis.set(url, jsonlinks)
  end

  cache_log.puts "#{Time.now.to_i}\t#{cache_status}\t#{url}"

  status 200
  headers "content-type" => "application/json"
  body jsonlinks
end

def extract_links(url)
  links = []
  doc = Nokogiri::HTML(open(url))
  doc.css("a").each do |link|
    text = link.text.strip.split.join(" ")
    begin
      links.push({
        text: text.empty? ? "[IMG]" : text,
        href: URI.join(url, link["href"])
      })
    rescue
    end
  end
  links
end
```

This Ruby file is almost equivalent to what we had in Python before, except, in addition to that it also logs the link extraction requests and corresponding cache events.
In a microservice architecture application swapping components with an equivalent one is easy as long as the expectations of consumers of the component are maintained.

```.term1
cat api/Dockerfile
```

```dockerfile
FROM       ruby:2.6
LABEL      maintainer="Sawood Alam <@ibnesayeed>"

ENV        LANG C.UTF-8
ENV        REDIS_URL="redis://localhost:6379"

WORKDIR    /app
COPY       Gemfile /app/
RUN        bundle install

COPY       linkextractor.rb /app/
RUN        chmod a+x linkextractor.rb

CMD        ["./linkextractor.rb", "-o", "0.0.0.0"]
```

Above `Dockerfile` is written for the Ruby script and it is pretty much self-explanatory.

```.term1
cat docker-compose.yml
```

```yml
version: '3'

services:
  api:
    image: linkextractor-api:step6-ruby
    build: ./api
    ports:
      - "4567:4567"
    environment:
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./logs:/app/logs
  web:
    image: linkextractor-web:step6-php
    build: ./www
    ports:
      - "80:80"
    environment:
      - API_ENDPOINT=http://api:4567/api/
  redis:
    image: redis
```

The `docker-compose.yml` file has a few minor changes in it.
The `api` service image is now named `linkextractor-api:step6-ruby`, the port mapping is changed from `5000` to `4567` (which is the default port for Sinatra server), and the `API_ENDPOINT` environment variable in the `web` service is updated accordingly so that the PHP code can talk to it.

With these in place, let's boot our service stack:

```.term1
docker-compose up -d --build
```

```
... [OUTPUT REDACTED] ...

Successfully built b713eef49f55
Successfully tagged linkextractor-api:step6-ruby
Creating linkextractor_web_1   ... done
Creating linkextractor_api_1   ... done
Creating linkextractor_redis_1 ... done
```

We should now be able to access the API (using the updated port number):

```.term1
curl -i http://localhost:4567/api/http://example.com/
```

```json
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 96
X-Content-Type-Options: nosniff
Server: WEBrick/1.4.2 (Ruby/2.5.1/2018-03-29)
Date: Mon, 24 Sep 2018 01:41:35 GMT
Connection: Keep-Alive

[
  {
    "text": "More information...",
    "href": "http://www.iana.org/domains/example"
  }
]
```

Now, open the web interface by [clicking the Link Extractor](/){:data-term=".term1"}{:data-port="80"} and extract links of a few URLs.
Also, try to repeat these attempts for some URLs.
If everything is alright, the web application should behave as before without noticing any changes in the API service (which is completely replaced).

We can shut the stack down now:

```.term1
docker-compose down
```

Since we have persisted logs, they should still be available after the services are gone:

```.term1
cat logs/extraction.log
```

```
1537753295      MISS    http://example.com/
1537753600      HIT     http://example.com/
1537753635      MISS    https://training.play-with-docker.com/
```

This illustrates that the caching is functional as the second attempt to the `http://example.com/` resulted in a cache `HIT`.

In this step we explored the possibility of swapping components of an application with microservice architecture with their equivalents without impacting rest of the parts of the stack.
We have also explored data persistence using bind mount volumes that persists even after the containers writing to the volume are gone.

So far, we have used `docker-compose` utility to orchestrate the application stack, which is good for development environment, but for production environment we use `docker stack deploy` command to run the application in a [Docker Swarm Cluster](/swarm-stack-intro).
It is left for you as an assignment to deploy this application in a Docker Swarm Cluster.


## Conclusions

We started this tutorial with a simple Python script that scrapes links from a give web page URL.
We demonstrated various difficulties in running the script.
We then illustrated how easy to run and portable the script becomes onces it is containerized.
In the later steps we gradually evolved the script into a multi-service application stack.
In the process we explored various concepts of microservice architecture and how Docker tools can be helpful in orchestrating a multi-service stack.
Finally, we demonstrated the ease of microservice component swapping and data persistence.

The next step would be to learn how to deploy such service stacks in a [Docker Swarm Cluster](/swarm-stack-intro).

As an aside, here are some introductory Docker slides.

<iframe src="//www.slideshare.net/slideshow/embed_code/key/qAjE65X2E8tkxe" width="700" height="450" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/ibnesayeed/introducing-docker-application-containerization-service-orchestration" title="Introducing Docker - Application Containerization &amp; Service Orchestration" target="_blank">Introducing Docker - Application Containerization &amp; Service Orchestration</a> </strong> by <strong><a href="//www.slideshare.net/ibnesayeed" target="_blank">Sawood Alam</a></strong> </div>

---
