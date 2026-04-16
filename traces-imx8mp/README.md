
## Comandi eseguiti
Tutte le registrazioni sono state eseguite mentre era in esecuzione il seguente comando di modo da poter identificare un pattern del suo comportamento:
```sh
cyclictest -t 1 -a 1 -p 99 -i 2000000 -q
```
- `-t 1` : numero di thread spawnati
- `-a 1` : affinity, su quale core eseguire il processo
- `-i 2000000` : intervallo di 2mln di microsecondi, quindi 2 secondi
- `-q` quiet

> [!NOTE]
> Il comando `cyclictest` genera nonostante l'opzione `-t 1`, due thread, di cui solo il secondo(avente pid maggiore) è quello che esegue la `nanosleep` di attesa di due secondi. 
> Lo si può verificare mediante `strace -p pid_thread`.

Per la registrazione delle tracce sono stati eseguiti i seguenti comandi :

- Per tracciare l'intero kernel alla massima profondità :
```sh
trace-cmd record -p function_graph 
```
- Per tracciare un singolo processo tramite il pid :
```sh
trace-cmd record -p function_graph -P pid
```

- Per tracciare una parte degli eventi come `timer:`
	- obiettivo: isolare e registrare l'esatta catena che lega i timer ad alta risoluzione (hrtimer) allo scheduling del sistema operativo
	- - **`timer:hrtimer_start`**: registra l'istante esatto in cui un processo richiede l'attivazione di un timer (impostazione della scadenza).
	- **`timer:hrtimer_expire_entry`**: registra il momento in cui il timer hardware scade e il kernel ne gestisce l'interrupt.
	- **`sched:sched_wakeup`**: registra l'istante immediatamente successivo in cui lo scheduler, in risposta alla scadenza del timer, inserisce il processo in attesa nella coda di esecuzione (Ready Queue).
```sh
trace-cmd record -e timer:hrtimer_start -e timer:hrtimer_expire_entry -e sched:sched_wakeup
```

> [!CAUTION]
> NOTA: nelle tracce complete del kernel, a seconda della memoria disponibile sulla board, si possono trovare delle nelle quali ci sono salti di diversi timestamp e che sono segnalati con `EVENTS_DROPPED`. Questo indica che il buffer del kernel si è riempito più velocemente di quanto `trace-cmd` riesca a leggere e salvare i dati sul disco. Perciò è necessario aumentare la dimensione del buffer. Per fortuna `trace-cmd` offre l'opzione `-b` tramite la quale è possibile specificare la dimensione del buffer.
> 	*ATTENZIONE* a non aumentare troppo perché la dimensione massima dipende dalla board. Un suggerimento potrebbe essere `-b 120000`
> 



# Come leggere le tracce
> [!TIP]
> Le tracce sono state registrate in diversi momenti nel tempo. 
> Es: quando cyclictest aveva il pid 2641 per il thread manager che stampa su schermo e pid 2642 per il thread che esegue effettivamente la nanosleep.
> In altri momenti i due thread avevano pid diversi.

I numero di file di questa repo potrebbe cambiare e quindi la documentazione potrebbe non ricoprire tutti i nuovi file. Al momento il contenuto della directory è così distribuito.

- `cyclictest-thread-loop-all-cores` : questo è un'estrazione del momento in cui il thread `cyclictest` si addormenta e del salto di 2 secondi comprendente tutto quello che accade nel kernel prima del momento del risveglio fino al momento stesso.
	- il file da cui è avvenuta tale estrazione è un file non presente nella repository poiché di dimensioni di 1.1GB.
	- NOTA : in questo file possiamo notare del RUMORE derivante dagli altri core, perciò è stata fatta un'ulteriore estrazione dalla quale è derivata il file `cyclictest-thread-filtered-loop-pattern`
- `cyclictest-thread-filtered-loop-patter` è il file ottenuto dopo l'analisi del file precedente dove si sono notate operazioni interessanti relative a `cyclictest` nei `core 001 e 003`. L'estrazione è stata effettuata mediante il comando `sed '/\[001\]d/' | sed '/\[003\]d/ > nomefile`.
	- in questo file filtrato sono stati segnalati i momenti salienti del ciclo di vita del thread di cyclictest che esegue la nanosleep.

