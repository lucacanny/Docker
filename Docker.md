> # DOCKER

**Come si creano i container Docker?**

Riepilogando le caratteristiche di un container Docker, potrebbe non essere chiara l’ultima affermazione: il container è il risultato di un processo ripetibile e perfettamente tracciabile.

Questa affermazione è particolarmente importante perché se vogliamo evitare di perdere qualsiasi tipo di configurazione della nostra applicazione e vogliamo condurre delle analisi efficienti in caso di errore, il container deve poter essere versionato e ricreato facilmente ogni volta che vogliamo.

Questa affermazione è però vera solo se rispettiamo alcune regole:

* i container si basano su immagini di base che devono essere immutabili.
* le immagini Docker immutabili si definiscono utilizzando i Dockerfile.

**Cos’è un Dockerfile**
Il Dockerfile è il file su cui si basa il processo di creazione di un’immagine Docker. Descrive tutti i layer che compongono l’immagine. Per capire il concetto di layer, facciamo un esempio pratico. Consideriamo il Dockerfile dell’immagine debian:latest:

```
FROM scratch
ADD rootfs.tar.xz /
CMD ["bash"]
```

L’immagine ha 2 layer. Il primo layer viene creato sulla base di un’immagine di default chiamata “scratch” aggiungendovi (ADD) il file rootfs.tar.xz. 
Il secondo layer, che usa come immagine di base il risultato del primo layer, corrisponde ad un metadato che viene aggiunto e che stabilisce che quando l’immagine verrà usata, il container dovrà eseguire il comando (CMD) bash. 
Il risultato che si ottiene elaborando il secondo e ultimo layer, corrisponde all’immagine finale.

![img](https://cdn-images-1.medium.com/max/800/1*XreENZEYEID-2XyjL0mJ4g.png)

una possibile visualizzazione dell’immagine debian:latest
Quando questa immagine viene usata per avviare un container, a questi due layer immutabili, viene aggiunto un ulteriore layer, chiamato container layer, con permessi di lettura e scrittura. Se un file di un layer immutabile dovesse essere modificato a runtime, il file verrebbe copiato nel container layer e quindi modificato liberamente.

Vi state chiedendo a cosa serve questa complessità tecnica? Serve a velocizzare l’avvio di un container; dal momento che il filesystem del container è basato su layer (immagini) immutabili, se queste sono già presenti in locale, sono di fatto già disponibili ed utilizzabili dal nuovo container.
Allo stesso modo, se più container usano gli stessi layer (anche se ne hanno solo alcuni in comune) avremo un risparmio di spazio disco perché i dati non saranno duplicati per ciascun container.

**In pratica**

Vediamo ora con un esempio pratico come si esegue il processo di creazione di un’immagine Docker.

In una nuova folder dobbiamo creare il file “Dockerfile”. Il contenuto di questo file come potete immaginare può essere più o meno complesso e dipende sostanzialmente dalla nostra applicazione.
Il set di comandi che possiamo utilizzare è documentato sul sito di docker.
Un esempio di base basato su golang è il seguente:

```
FROM golang
COPY . /go/src/app/
RUN go install app
CMD /go/bin/app
```

Il Dockerfile è usato dal client docker quando si usa il comando build, come in questo esempio:

```
$ docker build -t myapp .
Sending build context to Docker daemon  2.048kB
Step 1/4 : FROM golang
latest: Pulling from library/golang
723254a2c089: Already exists
abe15a44e12f: Pull complete
409a28e3cc3d: Pull complete
Digest: sha256:3983f9b1f0b1509e21...5677cfce3a863df01b29e46550ae84b
Status: Downloaded newer image for golang:latest
---> 3858fd70eed2
Step 2/4 : COPY . /go/src/app/
---> 44bd4f83a342
Step 3/4 : RUN go install app
---> Running in 74931e7c0be4
can't load package: package app: no Go files in /go/src/app
The command '/bin/sh -c go install app' returned a non-zero code: 1
```

Dal risultato ottenuto, possiamo vedere come ogni comando del Dockerfile venga identificato con uno specifico step. L’immagine di base (FROM) è stata scaricata, i file presenti nella folder sono stati copiati (COPY) in un layer, il tentativo di compilare l’applicazione (RUN) ha generato un errore. La build si è interrotta.

Creiamo il file di esempio main.go che leggete di seguito e riproviamo la build

```
package main
import "net/http"
func main(){
  http.ListenAndServe(":80",http.FileServer(http.Dir("/static")))
}
```

questa volta otteniamo l’immagine desiderata:

```
$ docker build -t myapp .
Sending build context to Docker daemon  3.072kB
Step 1/4 : FROM golang
---> 3858fd70eed2
Step 2/4 : COPY . /go/src/app/
---> 7fa2193d24c5
Step 3/4 : RUN go install app
---> Running in 20f48561aab5
Removing intermediate container 20f48561aab5
---> 0d313f3f69fe
Step 4/4 : CMD /go/bin/app
---> Running in 24b9cfb67e3f
Removing intermediate container 24b9cfb67e3f
---> b7e3eee6f9ba
Successfully built b7e3eee6f9ba
Successfully tagged myapp:latest
```

Possiamo notare nell’output dell’esecuzione, che tutti gli step sono stati eseguiti; attenzione al comportamento di alcuni comandi, se RUN viene eseguito durante la fase di build, tanto da generare un errore come nel nostro primo tentativo, l’ultimo step CMD identifica il comando che verrà eseguito a runtime quando l’immagine sarà usata da un container.

**Esercitazioni**

1. Modifica la configurazione per utilizzare la versione 1.8 di Golang.
Se hai esperienza con Golang, prova ad includere delle possibili dipendenze nell’immagine. 
2. Diversamente, prova a creare un’immagine per il linguaggio di programmazione che conosci.

**It works on my computer**

L’immagine Docker che abbiamo creato durante la lezione include tutto ciò che ci serve per avviare la nostra applicazione. E’ il momento di farlo, creando il relativo container Docker con questo comando:

```
$ docker run -d -p 80:80 -v $PWD:/static myapp
b0bb03f49b794978fc83c8180908ab438e2b01d3113a9c4ab063327731ae94
```

Se il comando run è facilmente comprensibile, i suoi parametri richiedono un approfondimento leggendo la documentazione sul sito Docker.
L’applicazione Golang corrisponde ad un semplice web server attivo sulla porta 80 che mostra i file presenti nella directory /static. Avviando il container applicativo con quei parametri:

* rendiamo l’applicazione accessibile sulla porta 80 del nostro computer.
* creiamo un mapping tra la folder del nostro computer in cui abbiamo eseguito il comando e la folder /static del container.

Aprite con il browser l’url http://localhost per verificare il funzionamento.

**Best Practices**

Questa prima lezione non può affrontare tutte le tematiche legate all’utilizzo di Docker, ma vuole indirizzare l’uso di questo strumento verso la pratica dell’immutabilità. Oltre a quando già presentato, le attenzioni che dobbiamo avere quando creiamo un’immagine Docker sono:

* ridurre il più possibile la dimensione dei layer
* pulire eventuali cache (evitando però di allungare i tempi di build dell’immagine)
* creare immagini di base
* usare i comandi ONBUILD per riutilizzare le immagini di base
* Separare in modo chiaro le librerie necessarie per la build da quelle necessarie a runtime (multi-stage build)

**Conclusioni**

In questa lezione abbiamo introdotto gli aspetti principali dei container applicativi. L’esempio proposto, seppur molto semplice, mostra come la pratica DevOps dell’immutabilità consente di isolare l’applicazione che dobbiamo eseguire attraverso un processo preciso e ripetibile.

Ogni modifica all’applicazione può essere applicata facilmente ripetendo il processo di build e creando un nuovo container.
Ogni modifica al processo di build (ad esempio potrei voler aggiungere un altro step per scaricare le librerie esterne richieste dal codice golang) richiede semplicemente la modica del Dockerfile, che può essere versionato.