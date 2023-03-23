Immettere la configurazione warehouse ops.soft, eseguire `./ssl.sh` e verrà creata una cartella `conf` nella **directory superiore** .

## preambolo

SMTP può acquistare direttamente servizi da fornitori di servizi cloud, come:

* [SMTP di Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Invio e-mail Ali cloud](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Puoi anche costruire il tuo server di posta: invio illimitato, basso costo complessivo.

Di seguito, dimostriamo passo dopo passo come costruire il nostro server di posta.

## Selezione del server

Il server SMTP self-hosted richiede un IP pubblico con le porte 25, 456 e 587 aperte.

I cloud pubblici comunemente usati hanno bloccato queste porte per impostazione predefinita e potrebbe essere possibile aprirle emettendo un ordine di lavoro, ma dopotutto è molto problematico.

Consiglio di acquistare da un host che ha queste porte aperte e supporta l'impostazione di nomi di dominio inversi.

Qui, consiglio [Contabo](https://contabo.com) .

Contabo è un provider di hosting con sede a Monaco, in Germania, fondato nel 2003 con prezzi molto competitivi.

Se scegli l'euro come valuta di acquisto, il prezzo sarà più economico (un server con 8 GB di memoria e 4 CPU costa circa 529 yuan all'anno e la quota di installazione iniziale è gratuita per un anno).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Quando effettui un ordine, osserva `prefer AMD` e il server con CPU AMD avrà prestazioni migliori.

Di seguito, prenderò come esempio il VPS di Contabo per dimostrare come costruire il proprio server di posta.

## Configurazione del sistema Ubuntu

Il sistema operativo qui è Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Se il server su ssh visualizza `Welcome to TinyCore 13!` (come mostrato nella figura sottostante), significa che il sistema non è stato ancora installato. Disconnettere ssh e attendere alcuni minuti per accedere nuovamente.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Quando viene visualizzato `Welcome to Ubuntu 22.04.1 LTS` , l'inizializzazione è completa ed è possibile continuare con i passaggi seguenti.

### [Facoltativo] Inizializzare l'ambiente di sviluppo

Questo passaggio è facoltativo.

Per comodità, inserisco l'installazione e la configurazione di sistema del software Ubuntu in [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Esegui il seguente comando per installare con un clic.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Utenti cinesi, utilizzare invece il seguente comando e la lingua, il fuso orario, ecc. verranno impostati automaticamente.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo abilita IPV6

Abilita IPV6 in modo che anche SMTP possa inviare e-mail con indirizzi IPV6.

modifica `/etc/sysctl.conf`

Modifica o aggiungi le seguenti righe

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Segui [il tutorial di contabo: Aggiunta di connettività IPv6 al tuo server](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Modifica `/etc/netplan/01-netcfg.yaml` , aggiungi alcune righe come mostrato nella figura seguente (il file di configurazione predefinito di Contabo VPS ha già queste righe, decommentale).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Quindi `netplan apply` per rendere effettiva la configurazione modificata.

Dopo che la configurazione è andata a buon fine, puoi utilizzare `curl 6.ipw.cn` per visualizzare l'indirizzo IPv6 della tua rete esterna.

## Clona il repository di configurazione ops

```
git clone https://github.com/wactax/ops.soft.git
```

## Genera un certificato SSL gratuito per il tuo nome di dominio

L'invio di posta richiede un certificato SSL per la crittografia e la firma.

Utilizziamo [acme.sh](https://github.com/acmesh-official/acme.sh) per generare certificati.

acme.sh è uno strumento di firma di certificati automatizzato open source,

Immettere la configurazione warehouse ops.soft, eseguire `./ssl.sh` e verrà creata una cartella `conf` nella **directory superiore** .

Trova il tuo provider DNS da [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , modifica `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Quindi esegui `./ssl.sh 123.com` per generare i certificati `123.com` e `*.123.com` per il tuo nome di dominio.

La prima esecuzione installerà automaticamente [acme.sh](https://github.com/acmesh-official/acme.sh) e aggiungerà un'attività pianificata per il rinnovo automatico. Puoi vedere `crontab -l` , c'è una linea come la seguente.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Il percorso per il certificato generato è simile a `/mnt/www/.acme.sh/123.com_ecc。`

Il rinnovo del certificato chiamerà lo script `conf/reload/123.com.sh` , modifica questo script, puoi aggiungere comandi come `nginx -s reload` per aggiornare la cache del certificato delle applicazioni correlate.

## Crea un server SMTP con chasquid

[chasquid](https://github.com/albertito/chasquid) è un server SMTP open source scritto in linguaggio Go.

Come sostituto dei vecchi programmi per server di posta come Postfix e Sendmail, chasquid è più semplice e facile da usare, ed è anche più facile per lo sviluppo secondario.

Esegui `./chasquid/init.sh 123.com` verrà installato automaticamente con un clic (sostituisci 123.com con il tuo nome di dominio di invio).

## Configura la firma e-mail DKIM

DKIM viene utilizzato per inviare firme e-mail per impedire che le lettere vengano trattate come spam.

Dopo che il comando è stato eseguito correttamente, ti verrà chiesto di impostare il record DKIM (come mostrato di seguito).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Basta aggiungere un record TXT al tuo DNS (come mostrato di seguito).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Visualizza lo stato del servizio e i registri

 `systemctl status chasquid` Visualizza lo stato del servizio.

Lo stato di funzionamento normale è quello mostrato nella figura sottostante

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` o `journalctl -xeu chasquid` può visualizzare il log degli errori.

## Configurazione inversa del nome di dominio

Il nome di dominio inverso consente di risolvere l'indirizzo IP nel nome di dominio corrispondente.

L'impostazione di un nome di dominio inverso può impedire che le e-mail vengano identificate come spam.

Quando la posta viene ricevuta, il server ricevente eseguirà l'analisi del nome di dominio inverso sull'indirizzo IP del server di invio per confermare se il server di invio ha un nome di dominio inverso valido.

Se il server di invio non ha un nome di dominio inverso o se il nome di dominio inverso non corrisponde all'indirizzo IP del server di invio, il server di ricezione potrebbe riconoscere l'e-mail come spam o rifiutarla.

Visita [https://my.contabo.com/rdns](https://my.contabo.com/rdns) e configura come mostrato di seguito

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Dopo aver impostato il nome di dominio inverso, ricordarsi di configurare la risoluzione in avanti del nome di dominio ipv4 e ipv6 sul server.

## Modifica il nome host di chasquid.conf

Modifica `conf/chasquid/chasquid.conf` al valore del nome di dominio inverso.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Quindi eseguire `systemctl restart chasquid` per riavviare il servizio.

## Eseguire il backup di conf nel repository git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Ad esempio, eseguo il backup della cartella conf nel mio processo github come segue

Crea prima un magazzino privato

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Entra nella directory conf e invia al magazzino

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Aggiungi mittente

correre

```
chasquid-util user-add i@wac.tax
```

Può aggiungere il mittente

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Verificare che la password sia impostata correttamente

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Dopo aver aggiunto l'utente, `chasquid/domains/wac.tax/users` verrà aggiornato, ricordati di inviarlo al magazzino.

## DNS aggiunge il record SPF

SPF ( Sender Policy Framework ) è una tecnologia di verifica della posta elettronica utilizzata per prevenire le frodi via e-mail.

Verifica l'identità di un mittente di posta controllando che l'indirizzo IP del mittente corrisponda ai record DNS del nome di dominio che afferma di essere, impedendo ai truffatori di inviare e-mail fasulle.

L'aggiunta di record SPF può impedire il più possibile che le e-mail vengano identificate come spam.

Se il tuo server dei nomi di dominio non supporta il tipo SPF, aggiungi semplicemente il record di tipo TXT.

Ad esempio, l'SPF di `wac.tax` è il seguente

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF per `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Nota che ho `include:_spf.google.com` qui, questo perché in seguito configurerò `i@wac.tax` come indirizzo di invio nella casella di posta di Google.

## Configurazione DNS DMARC

DMARC è l'abbreviazione di (Domain-based Message Authentication, Reporting & Conformance).

Viene utilizzato per catturare i rimbalzi SPF (forse causati da errori di configurazione o qualcun altro finge di essere te per inviare spam).

Aggiungi il record TXT `_dmarc` ,

Il contenuto è il seguente

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Il significato di ciascun parametro è il seguente

### p (Politica)

Indica come gestire le e-mail che non superano la verifica SPF (Sender Policy Framework) o DKIM (DomainKeys Identified Mail). Il parametro p può essere impostato su uno dei tre valori:

* none: non viene intrapresa alcuna azione, solo il risultato della verifica viene restituito al mittente tramite il meccanismo di segnalazione e-mail.
* Quarantena: mette la posta che non ha superato la verifica nella cartella spam, ma non rifiuterà direttamente la posta.
* rifiuta: rifiuta direttamente le e-mail che non superano la verifica.

### fo (Opzioni di errore)

Specifica la quantità di informazioni restituite dal meccanismo di reporting. Può essere impostato su uno dei seguenti valori:

* 0: riporta i risultati della convalida per tutti i messaggi
* 1: segnala solo i messaggi che non superano la verifica
* d: segnala solo gli errori di verifica del nome di dominio
* s: segnala solo gli errori di verifica SPF
* l: segnala solo gli errori di verifica DKIM

### rua & ruf

* rua (Reporting URI for Aggregate report): indirizzo e-mail per la ricezione di report aggregati
* ruf (Reporting URI for Forensic reports): indirizzo email per ricevere report dettagliati

## Aggiungi i record MX per inoltrare le email a Google Mail

Poiché non sono riuscito a trovare una casella di posta aziendale gratuita che supporti gli indirizzi universali (Catch-All, può ricevere qualsiasi e-mail inviata a questo nome di dominio, senza restrizioni sui prefissi), ho utilizzato chasquid per inoltrare tutte le e-mail alla mia casella di posta Gmail.

**Se disponi di una tua casella di posta aziendale a pagamento, non modificare l'MX e salta questo passaggio.**

Modifica `conf/chasquid/domains/wac.tax/aliases` , imposta la casella di posta di inoltro

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` indica tutte le email, `i` è il prefisso dell'indirizzo email dell'utente mittente creato sopra. Per inoltrare la posta, ogni utente deve aggiungere una riga.

Quindi aggiungi il record MX (indico qui direttamente l'indirizzo del nome di dominio inverso, come mostrato nella prima riga nella figura seguente).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Al termine della configurazione, puoi utilizzare altri indirizzi email per inviare email a `i@wac.tax` e `any123@wac.tax` per vedere se puoi ricevere email in Gmail.

In caso contrario, controlla il registro chasquid ( `grep chasquid /var/log/syslog` ).

## Invia un'email a i@wac.tax con Google Mail

Dopo che Google Mail ha ricevuto la posta, naturalmente speravo di rispondere con `i@wac.tax` invece di i.wac.tax@gmail.com.

Visita [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) e fai clic su "Aggiungi un altro indirizzo email".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Quindi, inserisci il codice di verifica ricevuto dall'e-mail a cui è stato inoltrato.

Infine, può essere impostato come indirizzo mittente predefinito (insieme all'opzione per rispondere con lo stesso indirizzo).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

In questo modo, abbiamo completato la creazione del server di posta SMTP e allo stesso tempo utilizziamo Google Mail per inviare e ricevere email.

## Invia una mail di prova per verificare se la configurazione è andata a buon fine

Inserisci `ops/chasquid`

Esegui `direnv allow` di installare le dipendenze (direnv è stato installato nel precedente processo di inizializzazione a una chiave ed è stato aggiunto un hook alla shell)

poi corri

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Il significato dei parametri è il seguente

* utente: nome utente SMTP
* passaggio: password SMTP
* a: destinatario

Puoi inviare un'e-mail di prova.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Si consiglia di utilizzare Gmail per ricevere e-mail di prova per verificare se le configurazioni sono andate a buon fine.

### Crittografia standard TLS

Come mostrato nella figura sottostante, è presente questo piccolo lucchetto, il che significa che il certificato SSL è stato abilitato con successo.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Quindi fai clic su "Mostra email originale"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Come mostrato nella figura seguente, la pagina della posta originale di Gmail mostra DKIM, il che significa che la configurazione DKIM è andata a buon fine.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Controlla Ricevuto nell'intestazione dell'e-mail originale e puoi vedere che l'indirizzo del mittente è IPV6, il che significa che anche IPV6 è configurato correttamente.
