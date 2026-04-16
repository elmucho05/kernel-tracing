
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

### `cyclictest-thread-loop-all-cores`
Questo è un'estrazione del momento in cui il thread `cyclictest` si addormenta e del salto di 2 secondi comprendente tutto quello che accade nel kernel prima del momento del risveglio fino al momento stesso.
	- il file da cui è avvenuta tale estrazione è un file non presente nella repository poiché di dimensioni di 1.1GB.
	- NOTA : in questo file possiamo notare del RUMORE derivante dagli altri core, perciò è stata fatta un'ulteriore estrazione dalla quale è derivata il file y

- `cyclictest-thread-filtered-loop-patter` è il file ottenuto dopo l'analisi del file precedente dove si sono notate operazioni interessanti relative a `cyclictest` nei `core 001 e 003`. L'estrazione è stata effettuata mediante il comando `sed '/\[001\]d/' | sed '/\[003\]d/ > nomefile`.
	- in questo file filtrato sono stati segnalati i momenti salienti del ciclo di vita del thread di cyclictest che esegue la nanosleep.

- `trace-funcgraph-pid2641 e trace-funcgraph-pid2641` sono due file nei quali si distingue il comportamento di entrambi i thread generati da `cyclictest`. Sono stati anch'essi generati con i metodi precedentemente indicati. Il loro scopo in questo caso è semplicemente identificare un pattern generale del comportamento nel kernel di `cyclictest`. Analisi più accurate nel file `cyclictest-thread-filtered-loop-pattern`

- `hrtimer-full-trace e hrtimer-pattern` sono due file generati mediante il tracciamento degli eventi `timer` come indicato precedentemente. Nel primo file troviamo la traccia completa e nel secondo il pattern di risveglio cyclictest all'interno di tale traccia. 

- `trace-irqsoff` : questa traccia mostra il task che ha subito il massimo ritardo dovuto agli interrupt disabilitati. Scorrendo il log e lo stack trace finale, è evidente che il blocco non è avvenuto in spazio utente. Il responsabile è un thread interno del kernel (kworker/0:1-8) che stava gestendo un'interrupt hw. Il tracer irqsoff monitora lo stato hardware globale della CPU, ignorando la logica dei singoli processi o PID.  Il fatto che `cyclictest` non compaia significa che il codice di cyclictest non causa mai la disabilitazione prolungata degli interrupt. 


- `trace-wakeup_rt` : Questra traccia mostra il record della peggior latenza di scheduling catturata durante l'esecuzione (in questo caso, un ritardo di 230 µs). Analizzando il log, notiamo che l'evento registrato non è correlato al thread cyclictest, ma riguarda il risveglio di un processo kernel dedicato alla gestione dell'interfaccia di comunicazione: irq/32-imx_mu_c (poid 122).
  
	- `cyclictest`, risvegliandosi "solo" ogni 2 secondi, subisce latenze inferiori a questa soglia. Di conseguenza, i suoi risvegli subiscono mai un ritardo superiore al massimo già registrato nel sistema, e il tracer scarta i suoi log, lasciando visibile solo il record peggiore.
	
	- la prima stack trace è una foto che viene scatta (dalla funzione probe_wakeup) alla wakeup, ed è lo stato del waker
	
	- la seconda stack trace è una foto che viene scattata (dalla funzione probe_wakeup_sched_switch) all'atto del context switch, che quasi certamente accade nella funzione `__schedule`. Fornisce lo stato del processo che ha eseguito la `__schedule`. Potrebbe tanto essere il waker stesso, perché è arrivato alla `__schedule` direttamente nella stessa call chain della wakeup, quanto un diverso processo, che è stato interrotto da un interrupt. L'handler di tale interrupt è il codice che arriva ad eseguire la `__schedule`
	
	- NOTA: anche nel caso di waker che coincide con l'esecutore di `__schedule`, la seconda stack trace può non essere un sovrainsieme della prima, perché tra la wakeup e la `__schedule`, il waker può aver concluso l'esecuzione di funzioni che erano ancora da concludere all'atto della wakeup
	- La parte di traccia tra le due stack trace mostra tutto quello che fa la CPU su cui andrà in esecuzione il task svegliato. In particolare mostra ciò che tale CPU fa tra l'istante in cui il task viene svegliato e l'istante in cui il task finalmente va in esecuzione. Lo mostra col formato del function tracer.
	- Per nostra sfortuna, è molto complessa, perché accade una cosa particolare. Nel mezzo della attesa di schedulazione del task, arriva un secondo interrupt, ed è quel secondo interrupt che porta alla schedule. Per quel motivo, le due stack trace all'inizio ed alla fine non hanno niente a che fare l'una con l'altra
		- La riga in cui si vede l'arrivo del nuovo interrupt è questa: `kworker/-130       0dN..1   67us : preempt_schedule_irq <-el1_interrupt`
