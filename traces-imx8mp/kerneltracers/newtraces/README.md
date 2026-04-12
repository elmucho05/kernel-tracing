# Come leggere le tracce
> [!TIP]
> Le tracce sono state registrate quando cyclictest aveva il pid 2641 per il thread manager che stampa su schermo e pid 2642 per il thread che esegue effettivamente la nanosleep.
> 
> Le tracce chiamate hrtimerentry* invece sono state registrare in un secondo momento, perciò il pid dei due thread appare diverso.


## Comandi eseguiti
Tutte le registrazioni sono state eseguite mentre era in esecuzione il seguente comando di modo da poter identificare un pattern del suo comportamento:
```sh
cyclictest -t 1 -a 1 -p 99 -i 2000000 -q
```


Per la registrazione delle tracce sono stati eseguiti i seguenti comandi e nelle seguenti condizioni:

- `traccia-all-pid*`
	- la sigla all significa che si sta tracciando tutto i kernel al massimo della profondità
	- questi file mostrano il risultato del seguente comando applicato ad entrambi i pid identificando il pattern di esecuzione di `cyclictest`
```sh
trace-cmd record -p function_graph -P pid_processo
```

- `occorrenza`
	- questo file contiene invece una occorrenza di schedulazione, esecuzione, addormentamento e ripresa di esecuzione presa dalla traccia dell'intero kernel

- `newtrace-all-kernel`: contiene la traccia di tutto il kernel da cui è stata presa l'occorrenza. Il file è stato generato dal seguente comando
```sh
trace-cmd record -p function_graph
```

- `hrtimerentry*`
	- file generati dall'esecuzione del seguente comando il quale è utilizzato per isolare e registrare l'esatta catena che lega i timer ad alta risoluzione (hrtimer) allo scheduling del sistema operativo
	- - **`timer:hrtimer_start`**: registra l'istante esatto in cui un processo richiede l'attivazione di un timer (impostazione della scadenza).
	- **`timer:hrtimer_expire_entry`**: registra il momento in cui il timer hardware scade e il kernel ne gestisce l'interrupt.
	- **`sched:sched_wakeup`**: registra l'istante immediatamente successivo in cui lo scheduler, in risposta alla scadenza del timer, inserisce il processo in attesa nella coda di esecuzione (Ready Queue).
```sh
trace-cmd record -e timer:hrtimer_start -e timer:hrtimer_expire_entry -e sched:sched_wakeup
```


