Questo directory contiene i log e le analisi di tracing eseguite sul kernel Linux PREEMPT_RT (`6.6.23-rt28`) in esecuzione su i.MX8. 

L'obiettivo dell'analisi è isolare, riconoscere e misurare i pattern fisiologici di addormentamento (sleep) e risveglio (wakeup) di un task Real-Time (`cyclictest`), distinguendoli dal rumore di fondo del sistema operativo e identificando l'infrastruttura di gestione dei timer e degli interrupt.

## 1. Struttura della directory e guida alla lettura

I file all'interno di questa directory seguono una un pattern di naming per separare i dati grezzi dalle analisi estratte:

* **`traccia_*`**: Questi file contengono il **dump completo e raw** generato da `trace-cmd`. Sono file grandi che rappresentano l'intero periodo di registrazione e contengono tutto il "rumore" del sistema intercettato dal tracer.
* **`*-occurrency`**: Questi file contengono **frammenti isolati (pattern o snippet)** estratti dalle tracce complete. Rappresentano i momenti esatti (le "occorrenze") in cui i fenomeni studiati (context switch, interrupt latencies, timer expirations) si sono manifestati. Sono i file di riferimento per leggere i risultati dell'analisi.

### Elenco dei file
* `traccia_funcgraph` / `funcgraph-occurrency`
* `traccia_timers` / `timers-occurrency`
* `traccia_irqsoff` / `irsqoff-occurrency`
* `traccia_wakeup` / `wakeup-occurrency`

## 2. Comandi di generazione tracce (`trace-cmd`)

Per garantire la coerenza dell'analisi e minimizzare le migrazioni di processo, tutti i test sono stati eseguiti bloccando `cyclictest` su un singolo core (`-a 1`), impostando priorità FIFO massima (`-p 99`) e un ciclo di sleep di 1 millisecondo (`-i 1000`).
Le interfacce grafiche (HMI) e i carichi di memoria artificiali sono stati tenuti spenti 

I **file** **raw** sono stati generati con i seguenti comandi:

**1. Function Graph (Isolato sul processo):**
Traccia l'albero delle chiamate di funzione e i relativi tempi di esecuzione, filtrando solo per il PID di cyclictest e limitando la profondità per leggibilità.
```sh
trace-cmd record -p function_graph --max-graph-depth 4 -F cyclictest -p 99 -m -a 1 -t 1 -i 1000 -D 5 -q
```

**2. Tracciamento degli Eventi Timer (HRTimers):** Traccia esclusivamente i momenti di avvio e scadenza dei timer hardware per verificare l'infrastruttura di risveglio.

```bash
trace-cmd record -e timer:hrtimer_start -e timer:hrtimer_expire_entry -e timer:hrtimer_expire_exit cyclictest -p 99 -m -a 1 -t 1 -i 1000 -D 5 -q
```


**3. Tracer Latenza IRQSOFF:** Misura il tempo massimo in cui gli interrupt hardware sono stati disabilitati a livello di CPU.

```bash
trace-cmd record -p irqsoff cyclictest -p 99 -m -a 1 -t 1 -i 1000 -D 5 -q
```

**4. Tracer Latenza WAKEUP_RT:** Misura il tempo totale trascorso dal momento in cui scatta l'evento di risveglio per il task RT al momento in cui avviene il context switch fisico sulla CPU.

```bash
trace-cmd record -p wakeup_rt cyclictest -p 99 -m -a 1 -t 1 -i 1000 -D 5 -q
```

_Estrazione dei log (comune a tutti):_

```bash
trace-cmd report > nome_traccia
```



## 3. Risultati e Analisi dei pattern occurrencies

Di seguito l'analisi dei fenomeni isolati documentati nei rispettivi file `*-occurrency`.
### A. Function Graph (`funcgraph-occurrency`)

- **Obiettivo:** Riconoscere il pattern architetturale di Sleep e Wakeup.
- **Analisi:** La traccia evidenzia il momento dell'addormentamento: `cyclictest` invoca `__arm64_sys_clock_nanosleep() {` lasciando la parentesi aperta. A causa del filtro sul singolo PID, l'interrupt hardware che innesca il risveglio non è visibile. Tuttavia, si rileva un netto "salto" temporale di ~10ms (corrispondente all'intervallo di cyclictest). Il risveglio è marcato dall'ingresso del thread `ktimers/1`, il quale esegue le funzioni architetturali ARM64 per il context switch (`fpsimd_thread_switch`, `tls_preserve_current_state`), cedendo fisicamente la CPU a `cyclictest` che chiude la funzione nanosleep con un delta registrato di `# 10184 us`.

### B. Timers (`timers-occurrency`)
- **Obiettivo:** Dimostrare che il kernel usa un High-Resolution Timer dedicato e non il tick di sistema (housekeeping) per i risvegli Real-Time.
- **Analisi:** Il file cattura uso degli HRTimers. Filtriando il rumore di fondo (`function=tick_sched_timer`), emerge un pattern ciclico : `cyclictest` imposta un timer con `hrtimer_start` mirato alla callback `hrtimer_wakeup`. Calcolando la differenza tra i parametri `expires` di due iterazioni successive, si ottiene un delta esatto di `1.000.000` nanosecondi (1 millisecondo). Quando la CPU esce dall'idle, gestisce `hrtimer_expire_entry` unicamente per la callback di wakeup, bypassando totalmente la routine generale di housekeeping.
    
### C. IRQSOFF (`irsqoff-occurrency`)

- **Obiettivo:** Valutare la sregolatezza fisiologica degli interrupt.
    
- **Analisi:** È stato isolato un evento fisiologico da ~209 µs. Lo stack trace mostra che un `kworker` (driver Ethernet `igb` in operazione `igb_rd32`) è stato interrotto da un interrupt MSI del PCIe. L'occorrenza è utile per decifrare l'anatomia della colonna di preemption: l'interrupt imposta il flag `N` (Need Resched) a 130 µs, ma essendo nel contesto dell'Hard IRQ (flag `h`), il kernel non può invocare lo scheduler. Deve completare l'ISR e lo sblocco del GIC hardware in sicurezza. A 206 µs il contesto IRQ termina, e solo a quel punto il kernel gestisce il flag `N` chiamando `preempt_schedule_irq`. Questo dimostra che ritardi sotto i 250 µs sono meccanismi di sicurezza dell'hardware e non anomalie.
    

### D. Wakeup_RT (`wakeup-occurrency`)

- **Obiettivo:** Analizzare la latenza di attivazione di un task ad alta priorità in assenza di Memory Contention.
    
- **Analisi:** È stata isolata una latenza eccellente di 198 µs, non relativa a cyclictest ma all'infrastruttura Mailbox IPC (`irq/32-imx_mu_c`). La traccia svela la transizione del flag `N` durante la fase di scheduling:
    
1. L'interrupt alza il flag `N`.
2. A 167 µs inizia la gestione di `__schedule`.
3. A 181 µs lo scheduler decide il task subentrante e _rimuove_ il flag `N`, sebbene il context switch non sia ancora ultimato.
4. A 193 µs avviene lo swap fisico dei registri (`==>`). 