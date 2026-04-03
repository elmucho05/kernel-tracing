# Ftrace
> La documentazione seguente e' un traduzione della documentazione del Kernel Linux. Per approfondire l'argomento consultare la documentazione ufficiale [Ftrace](https://docs.kernel.org/trace/ftrace.html)

**Ftrace** è un tracer interno progettato per aiutare gli sviluppatori e i progettisti di sistemi a comprendere cosa accade all'interno del kernel. Può essere utilizzato per il debugging o per analizzare latenze e problemi di prestazioni che si verificano al di fuori dello **user-space**.

Sebbene ftrace sia tipicamente considerato un tracer di funzioni (**function tracer**), in realtà è un framework composto da diverse utilità di tracing. Esiste il tracing delle latenze per esaminare ciò che accade tra la disattivazione e l'attivazione degli interrupt, così come per la **preemption** e per misurare il tempo che intercorre dal momento in cui un task viene risvegliato a quando viene effettivamente pianificato (**scheduled in**).

Uno degli utilizzi più comuni di ftrace è il tracciamento degli eventi (**event tracing**). In tutto il kernel sono presenti centinaia di punti di evento statici che possono essere abilitati tramite il file system **tracefs** per monitorare cosa stia accadendo in determinate parti del kernel.

Ftrace usa il tracefs file system per tenere i file di controllo ed anche i file per mostrare l'output.

Quando il tracefs e' configurato nel kernel, la directory `/sys/kernl/tracing` sara' creata. 


### Formato di una generica traccia
Quando l'opzione di *latency-formato* e' abilitata oppure quando uno dei tracer di latenza e' impostato, il file `trace` ci permette di comprendere meglio il perche' di una determinata latenza. Ecco una tipica traccia :

```sh
# tracer: irqsoff
#
# irqsoff latency trace v1.1.5 on 3.8.0-test+
# --------------------------------------------------------------------
# latency: 259 us, #4/4, CPU#2 | (M:preempt VP:0, KP:0, SP:0 HP:0 #P:4)
#    -----------------
#    | task: ps-6143 (uid:0 nice:0 policy:0 rt_prio:0)
#    -----------------
#  => started at: __lock_task_sighand
#  => ended at:   _raw_spin_unlock_irqrestore
#                  _------=> CPU#
#                 / _-----=> irqs-off
#                | / _----=> need-resched
#                || / _---=> hardirq/softirq
#                ||| / _--=> preempt-depth
#                |||| /     delay
#  cmd     pid   ||||| time  |   caller
#     \   /      |||||  \    |   /
      ps-6143    2d...    0us!: trace_hardirqs_off <-__lock_task_sighand
      ps-6143    2d..1  259us+: trace_hardirqs_on <-_raw_spin_unlock_irqrestore
      ps-6143    2d..1  263us+: time_hardirqs_on <-_raw_spin_unlock_irqrestore
      ps-6143    2d..1  306us : <stack trace>
 => trace_hardirqs_on_caller
 => trace_hardirqs_on
 => _raw_spin_unlock_irqrestore
 => do_task_stat
 => proc_tgid_stat
 => proc_single_show
 => seq_read
 => vfs_read
 => sys_read
 => system_call_fastpath
```


Questa traccia mostra che il tracer corrente e' `irqsoff` il quale traccia il tempo nel quale gli interrupt sono stati abilitati. 
- Viene segnata la versione del tracer e la versione del kernel (3.8 in questo esempio). 
- Poi viene mostrata la latenza massima in *microsecondi* ($259 \mu s$)
- Il numero delle entry della traccia e il totale di cpu
	- entrambi qui sono `#4/4`
	- `VP,KP,SP e HP` sono sempre a 0 e sono riservarti per later user. 
	- `#P` e' il numero di processori online `#P:4`


Successivamente abbiamo : `task: ps-6143 (uid:0 nice:0 policy:0 rt_prio:0)` quindi il processo in esecuzione dove la latenza e' stata verificata.


```
#  => started at: __lock_task_sighand
#  => ended at:   _raw_spin_unlock_irqrestore
```

- lock_task_sighand is where the interrupts were disabled.
- raw_spin_unlock_irqrestore is where they were enabled again.

Le prossime righe dopo l'header sono per la traccia vera e propria, l'header spiega solo cosa sia cosa:

- cmd: The name of the process in the trace.
- pid: The PID of that process.
- CPU#: The CPU which the process was running on.
- irqs-off: ‘d’ interrupts are disabled. ‘.’ otherwise.
- need-resched
- hardirq/softirq
	- ‘Z’ - NMI occurred inside a hardirq
	- ‘z’ - NMI is running
	- ‘H’ - hard irq occurred inside a softirq.
	- ‘h’ - hard irq is running
	- ‘s’ - soft irq is running
	- ‘.’ - normal context.
- preempt-depth: The level of preempt_disabled
- `time` :
	- quando il formato latancy e' abilitato, il file `trace`  include anche un timestamp relativo all'inzio della traccia. Questo cambia dall'output quando il *latency-format* e' disabilitato.
- `delay`:
	- permette di capire a colpo d'occhio vari tipi di ritardo. Ha bisogno di essere *fixed* per poter essere relativa solo alla stessa CPU.
	- - ‘$’ - greater than 1 second
	- ‘@’ - greater than 100 millisecond
	- `*` - greater than 10 millisecond
	- ‘#’ - greater than 1000 microsecond
	- ‘!’ - greater than 100 microsecond
	- ‘+’ - greater than 10 microsecond
	- ‘ ‘ - less than or equal to 10 microsecond.


### `trace_options`
Il file `trace_options` e' utilizzato per controllare quello viene stampato nell'output della traccia 

```sh
cat trace_options
      print-parent
      nosym-offset
      nosym-addr
      noverbose
      noraw
      nohex
      nobin
      noblock
      nofields
      trace_printk
      annotate
      nouserstacktrace
      nosym-userobj
      noprintk-msg-only
      context-info
      nolatency-format
      record-cmd
      norecord-tgid
      overwrite
      nodisable_on_free
      irq-info
      markers
      noevent-fork
      function-trace
      nofunction-fork
      nodisplay-graph
      nostacktrace
      nobranch
```

Per disabilitare una delle opzioni, echo "no" nell'opzione preferita:
```sh
echo noprint-parent > trace_options
```

Per **abilitare** una delle opzioni :
```sh
echo sys-offset > trace_options
```

# Irqsoff
Quando gli interrupt sono disabilitati, la cpu non puo' piu' reagire ad altri eventi esterni (Tranne gli NMI e SMI). Questo di per se non consentirebbe ad interrupt come un timer oppure quello del mouse, di far sapere al Kernel dell'avvenimento di un nuovo evento. Il che risulta in una **latenza col tempo di reazione**

Il tracer **irqsoff** tiene traccia del tempo per il quale gli interrupt sono disabilitati. Quando un nuovo massimo di latenza e' raggiunto, il tracer salva la traccia che ha portato a quell'evento, di modo che ogni volta che si raggiunge un nuovo massimo, la vecchia traccia venga scartata.

Per cambiare il valore massimo  : `echo 0 > tracing_max_latency`.

Un esempio di traccia con questo tracer e' la seguente:

```sh
echo 0 > options/function-trace
echo irqsoff > current_tracer
echo 1 > tracing_on
echo 0 > tracing_max_latency
ls -ltr
[...]
echo 0 > tracing_on
cat trace
# tracer: irqsoff
#
# irqsoff latency trace v1.1.5 on 3.8.0-test+
# --------------------------------------------------------------------
# latency: 16 us, #4/4, CPU#0 | (M:preempt VP:0, KP:0, SP:0 HP:0 #P:4)
#    -----------------
#    | task: swapper/0-0 (uid:0 nice:0 policy:0 rt_prio:0)
#    -----------------
#  => started at: run_timer_softirq
#  => ended at:   run_timer_softirq
#
#
#                  _------=> CPU#
#                 / _-----=> irqs-off
#                | / _----=> need-resched
#                || / _---=> hardirq/softirq
#                ||| / _--=> preempt-depth
#                |||| /     delay
#  cmd     pid   ||||| time  |   caller
#     \   /      |||||  \    |   /
  <idle>-0       0d.s2    0us+: _raw_spin_lock_irq <-run_timer_softirq
  <idle>-0       0dNs3   17us : _raw_spin_unlock_irq <-run_timer_softirq
  <idle>-0       0dNs3   17us+: trace_hardirqs_on <-run_timer_softirq
  <idle>-0       0dNs3   25us : <stack trace>
 => _raw_spin_unlock_irq
 => run_timer_softirq
 => __do_softirq
 => call_softirq
 => do_softirq
 => irq_exit
 => smp_apic_timer_interrupt
 => apic_timer_interrupt
 => rcu_idle_exit
 => cpu_idle
 => rest_init
 => start_kernel
 => x86_64_start_reservations
 => x86_64_start_kernel
```
#### L'Intestazione (Header)


```plaintext
# tracer: irqsoff
# latency: 16 us, #4/4, CPU#0 | ...
```

- **latency: 16 us**: Qui ftrace ti dice subito il "colpevole". Gli interrupt sono rimasti disabilitati per **16 microsecondi**.
    
- **started at / ended at**: Ti indica la funzione che ha spento le luci e quella che le ha riaccese. In questo caso, tutto è successo dentro `run_timer_softirq`.

```
#                  _------=> CPU#            (Su quale processore siamo)
#                 / _-----=> irqs-off        (d = interrupt disabilitati)
#                | / _----=> need-resched    (N = il kernel deve cambiare processo)
#                || / _---=> hardirq/softirq (s = siamo in un softirq)
#                ||| / _--=> preempt-depth   (Livello di annidamento preemption)
#                |||| /     delay
#  cmd     pid   ||||| time  |   caller
#     \   /      |||||  \    |   /
  <idle>-0       0d.s2    0us+: _raw_spin_lock_irq <-run_timer_softirq
```

- **`<idle>-0`**: Il processo che stava girando era il processo "idle" (ID 0).
- **`0d.s2`**:
    - `0`: CPU 0.
    - `d`: Gli interrupt sono **disabilitati** (fondamentale per irqsoff!).
    - `.`: Il flag `need_resched` non è attivo.
    - `s`: Siamo all'interno di un `softirq`.
    - `2`: Il livello di preemption è 2.
        
- **`0us+`**: Il tempo dall'inizio dell'evento. Il simbolo `+` indica che il tempo sta scorrendo.
    
- **`_raw_spin_lock_irq`**: Questa è la funzione chiamata che ha effettivamente acquisito un lock e **disabilitato gli interrupt**.
    

#### Lo Stack Trace

```
  <idle>-0       0dNs3   17us : _raw_spin_unlock_irq <-run_timer_softirq
```

Qui gli interrupt vengono riabilitati da `_raw_spin_unlock_irq`. Notare che il tempo è **17us**.

> **Nota sulla discrepanza:** Ti chiederai perché la latenza dichiarata è **16us** ma qui leggi **17us** o **25us**. Questo accade perché il clock di sistema continua a girare leggermente tra il momento in cui viene registrata la latenza massima e il momento in cui viene scritto il log della funzione.

#### Il "Perché" (Backtrace)
Infine, ftrace ti mostra la "catena di comando" (Stack Trace):
```
 => _raw_spin_unlock_irq
 => run_timer_softirq
 => __do_softirq
 ...
 => start_kernel
```

Questo ti permette di risalire l'intera gerarchia delle chiamate: partendo dal basso (`start_kernel`), vedi esattamente quale percorso ha portato il sistema a eseguire quel lock che ha bloccato gli interrupt per 16 microsecondi.


# wakeup_rt

In un ambiente Real-Time (RT), il dato più critico non è la media, ma il **caso peggiore (worst case)**. La "latenza di scheduling" è il tempo che intercorre dal momento in cui un task ad alta priorità viene svegliato al momento in cui inizia effettivamente a girare sulla CPU.

Il tracer `wakeup_rt` è progettato specificamente per i task RT. Ignora i task normali perché le loro latenze, spesso imprevedibili e alte, "inquinerebbero" i dati, sovrascrivendo i record di latenza dei task RT che stiamo cercando di ottimizzare.

### 1. Configurazione del Test

Per testarlo, non usiamo un semplice comando, ma forziamo un processo (`sleep`) ad avere una priorità Real-Time usando il comando `chrt` il quale ci permette di manipolare gli attributi *real-time* di un processo.

```sh
# Setup iniziale
echo 0 > options/function-trace
echo wakeup_rt > current_tracer
echo 0 > tracing_max_latency
echo 1 > tracing_on

# Eseguiamo 'sleep 1' con politica SCHED_FIFO (-f) e priorità 5
chrt -f 5 sleep 1

# Disattiviamo per leggere i risultati
echo 0 > tracing_on
```

### 2. Analisi dell'Output 
```plaintext
# cat trace
# tracer: wakeup
#
# tracer: wakeup_rt
#
# wakeup_rt latency trace v1.1.5 on 3.8.0-test+
# --------------------------------------------------------------------
# latency: 5 us, #4/4, CPU#3 | (M:preempt VP:0, KP:0, SP:0 HP:0 #P:4)
#    -----------------
#    | task: sleep-2389 (uid:0 nice:0 policy:1 rt_prio:5)
#    -----------------
#
#                  _------=> CPU#
#                 / _-----=> irqs-off
#                | / _----=> need-resched
#                || / _---=> hardirq/softirq
#                ||| / _--=> preempt-depth
#                |||| /     delay
#  cmd     pid   ||||| time  |   caller
#     \   /      |||||  \    |   /
  <idle>-0       3d.h4    0us :      0:120:R   + [003]  2389: 94:R sleep
  <idle>-0       3d.h4    1us+: ttwu_do_activate.constprop.87 <-try_to_wake_up
  <idle>-0       3d..3    5us : __schedule <-schedule
  <idle>-0       3d..3    5us :      0:120:R ==> [003]  2389: 94:R sleep
```
#### L'Header
```
# tracer: wakeup_rt
# latency: 5 us, #4/4, CPU#3 | ...
# task: sleep-2389 (uid:0 nice:0 policy:1 rt_prio:5)
```

- **latency: 5 us**: Il sistema ha impiegato solo 5 microsecondi per far partire il processo dopo il risveglio. Eccellente.
- **policy: 1**: Indica `SCHED_FIFO` (la politica Real-Time scelta).
- **rt_prio: 5**: Questa è la priorità impostata dall'utente (da 1 a 99).
    
#### Il Diagramma di tracing
Vediamo cosa succede nel momento esatto del risveglio:

```
#  cmd     pid   ||||| time  |   caller
#     \   /      |||||  \    |   /
  <idle>-0       3d.h4    0us :      0:120:R   + [003]  2389: 94:R sleep
```

- **`<idle>-0`**: La CPU 3 era in stato di riposo (idle).
- **`3d.h4`**:
    - `h`: Siamo all'interno di un **hardirq** (un interrupt hardware ha svegliato il task).
    - `4`: Profondità di preemption.
- **`0:120:R + [003] 2389: 94:R sleep`**:
    - Il task `0` (idle) con priorità kernel `120` sta svegliando (`+`) il task `2389` (sleep) sulla CPU `003`.
    - **94**: Questa è la priorità interna del kernel. Si calcola come ($99 - \text{rt\_prio}$), quindi $99 - 5 = 94$.
    - **R**: Il task è ora in stato "Running" (pronto a correre).

#### Il Cambio di Contesto (Context Switch)

```
  <idle>-0       3d..3    5us : __schedule <-schedule
  <idle>-0       3d..3    5us :      0:120:R ==> [003]  2389: 94:R sleep
```

- **5us**: Dopo 5 microsecondi, viene chiamata la funzione `__schedule`.
- **`==>`**: Questo simbolo indica il **cambio di contesto**. Il processo idle lascia il posto al processo `sleep`.
  
Il `0:120:R` significa idle stava eseguendo con una priority di 0 (120-120) e nello stato running R. Il task di `sleep` era stato schedulato con 2389:94:R. Quella e' la priority del kernel e anche lui e' n stato running
### 3. Punti Chiave da Ricordare

1. **Punto di arresto**: Il tracciamento si ferma un istante prima che il task inizi effettivamente a eseguire il proprio codice. Si ferma dentro lo scheduler quando il cambio è ormai deciso.
    
2. **Priorità Invertita**: Nel kernel Linux, più basso è il numero di priorità, più "importante" è il compito. Ecco perché la priorità utente `5` diventa `94` internamente (un task con priorità utente `99` avrebbe priorità kernel `0`).
    
3. **Perché non usare il wakeup normale?** Se usassi il semplice `wakeup`, un qualsiasi processo in background (come un aggiornamento di sistema o un browser) potrebbe generare una latenza di 100us, nascondendo il fatto che il tuo task critico sta invece reagendo in soli 5us. `wakeup_rt` "filtra" il rumore e tiene d'occhio solo ciò che conta per il Real-Time.

##### Tracing con lo scheduler RR tramite `chrt -r 5`

```sh
chrt -r 5 sleep 1
echo 1 > options/function-trace

# tracer: wakeup_rt
#
# wakeup_rt latency trace v1.1.5 on 3.8.0-test+
# --------------------------------------------------------------------
# latency: 29 us, #85/85, CPU#3 | (M:preempt VP:0, KP:0, SP:0 HP:0 #P:4)
#    -----------------
#    | task: sleep-2448 (uid:0 nice:0 policy:1 rt_prio:5)
#    -----------------
#
#                  _------=> CPU#
#                 / _-----=> irqs-off
#                | / _----=> need-resched
#                || / _---=> hardirq/softirq
#                ||| / _--=> preempt-depth
#                |||| /     delay
#  cmd     pid   ||||| time  |   caller
#     \   /      |||||  \    |   /
  <idle>-0       3d.h4    1us+:      0:120:R   + [003]  2448: 94:R sleep
  <idle>-0       3d.h4    2us : ttwu_do_activate.constprop.87 <-try_to_wake_up
  <idle>-0       3d.h3    3us : check_preempt_curr <-ttwu_do_wakeup
  <idle>-0       3d.h3    3us : resched_curr <-check_preempt_curr
  <idle>-0       3dNh3    4us : task_woken_rt <-ttwu_do_wakeup
  <idle>-0       3dNh3    4us : _raw_spin_unlock <-try_to_wake_up
  <idle>-0       3dNh3    4us : sub_preempt_count <-_raw_spin_unlock
  <idle>-0       3dNh2    5us : ttwu_stat <-try_to_wake_up
  <idle>-0       3dNh2    5us : _raw_spin_unlock_irqrestore <-try_to_wake_up
  <idle>-0       3dNh2    6us : sub_preempt_count <-_raw_spin_unlock_irqrestore
  <idle>-0       3dNh1    6us : _raw_spin_lock <-__run_hrtimer
  <idle>-0       3dNh1    6us : add_preempt_count <-_raw_spin_lock
  <idle>-0       3dNh2    7us : _raw_spin_unlock <-hrtimer_interrupt
  <idle>-0       3dNh2    7us : sub_preempt_count <-_raw_spin_unlock
  <idle>-0       3dNh1    7us : tick_program_event <-hrtimer_interrupt
  <idle>-0       3dNh1    7us : clockevents_program_event <-tick_program_event
  <idle>-0       3dNh1    8us : ktime_get <-clockevents_program_event
  <idle>-0       3dNh1    8us : lapic_next_event <-clockevents_program_event
  <idle>-0       3dNh1    8us : irq_exit <-smp_apic_timer_interrupt
  <idle>-0       3dNh1    9us : sub_preempt_count <-irq_exit
  <idle>-0       3dN.2    9us : idle_cpu <-irq_exit
  <idle>-0       3dN.2    9us : rcu_irq_exit <-irq_exit
  <idle>-0       3dN.2   10us : rcu_eqs_enter_common.isra.45 <-rcu_irq_exit
  <idle>-0       3dN.2   10us : sub_preempt_count <-irq_exit
  <idle>-0       3.N.1   11us : rcu_idle_exit <-cpu_idle
  <idle>-0       3dN.1   11us : rcu_eqs_exit_common.isra.43 <-rcu_idle_exit
  <idle>-0       3.N.1   11us : tick_nohz_idle_exit <-cpu_idle
  <idle>-0       3dN.1   12us : menu_hrtimer_cancel <-tick_nohz_idle_exit
  <idle>-0       3dN.1   12us : ktime_get <-tick_nohz_idle_exit
  <idle>-0       3dN.1   12us : tick_do_update_jiffies64 <-tick_nohz_idle_exit
  <idle>-0       3dN.1   13us : cpu_load_update_nohz <-tick_nohz_idle_exit
  <idle>-0       3dN.1   13us : _raw_spin_lock <-cpu_load_update_nohz
  <idle>-0       3dN.1   13us : add_preempt_count <-_raw_spin_lock
  <idle>-0       3dN.2   13us : __cpu_load_update <-cpu_load_update_nohz
  <idle>-0       3dN.2   14us : sched_avg_update <-__cpu_load_update
  <idle>-0       3dN.2   14us : _raw_spin_unlock <-cpu_load_update_nohz
  <idle>-0       3dN.2   14us : sub_preempt_count <-_raw_spin_unlock
  <idle>-0       3dN.1   15us : calc_load_nohz_stop <-tick_nohz_idle_exit
  <idle>-0       3dN.1   15us : touch_softlockup_watchdog <-tick_nohz_idle_exit
  <idle>-0       3dN.1   15us : hrtimer_cancel <-tick_nohz_idle_exit
  <idle>-0       3dN.1   15us : hrtimer_try_to_cancel <-hrtimer_cancel
  <idle>-0       3dN.1   16us : lock_hrtimer_base.isra.18 <-hrtimer_try_to_cancel
  <idle>-0       3dN.1   16us : _raw_spin_lock_irqsave <-lock_hrtimer_base.isra.18
  <idle>-0       3dN.1   16us : add_preempt_count <-_raw_spin_lock_irqsave
  <idle>-0       3dN.2   17us : __remove_hrtimer <-remove_hrtimer.part.16
  <idle>-0       3dN.2   17us : hrtimer_force_reprogram <-__remove_hrtimer
  <idle>-0       3dN.2   17us : tick_program_event <-hrtimer_force_reprogram
  <idle>-0       3dN.2   18us : clockevents_program_event <-tick_program_event
  <idle>-0       3dN.2   18us : ktime_get <-clockevents_program_event
  <idle>-0       3dN.2   18us : lapic_next_event <-clockevents_program_event
  <idle>-0       3dN.2   19us : _raw_spin_unlock_irqrestore <-hrtimer_try_to_cancel
  <idle>-0       3dN.2   19us : sub_preempt_count <-_raw_spin_unlock_irqrestore
  <idle>-0       3dN.1   19us : hrtimer_forward <-tick_nohz_idle_exit
  <idle>-0       3dN.1   20us : ktime_add_safe <-hrtimer_forward
  <idle>-0       3dN.1   20us : ktime_add_safe <-hrtimer_forward
  <idle>-0       3dN.1   20us : hrtimer_start_range_ns <-hrtimer_start_expires.constprop.11
  <idle>-0       3dN.1   20us : __hrtimer_start_range_ns <-hrtimer_start_range_ns
  <idle>-0       3dN.1   21us : lock_hrtimer_base.isra.18 <-__hrtimer_start_range_ns
  <idle>-0       3dN.1   21us : _raw_spin_lock_irqsave <-lock_hrtimer_base.isra.18
  <idle>-0       3dN.1   21us : add_preempt_count <-_raw_spin_lock_irqsave
  <idle>-0       3dN.2   22us : ktime_add_safe <-__hrtimer_start_range_ns
  <idle>-0       3dN.2   22us : enqueue_hrtimer <-__hrtimer_start_range_ns
  <idle>-0       3dN.2   22us : tick_program_event <-__hrtimer_start_range_ns
  <idle>-0       3dN.2   23us : clockevents_program_event <-tick_program_event
  <idle>-0       3dN.2   23us : ktime_get <-clockevents_program_event
  <idle>-0       3dN.2   23us : lapic_next_event <-clockevents_program_event
  <idle>-0       3dN.2   24us : _raw_spin_unlock_irqrestore <-__hrtimer_start_range_ns
  <idle>-0       3dN.2   24us : sub_preempt_count <-_raw_spin_unlock_irqrestore
  <idle>-0       3dN.1   24us : account_idle_ticks <-tick_nohz_idle_exit
  <idle>-0       3dN.1   24us : account_idle_time <-account_idle_ticks
  <idle>-0       3.N.1   25us : sub_preempt_count <-cpu_idle
  <idle>-0       3.N..   25us : schedule <-cpu_idle
  <idle>-0       3.N..   25us : __schedule <-preempt_schedule
  <idle>-0       3.N..   26us : add_preempt_count <-__schedule
  <idle>-0       3.N.1   26us : rcu_note_context_switch <-__schedule
  <idle>-0       3.N.1   26us : rcu_sched_qs <-rcu_note_context_switch
  <idle>-0       3dN.1   27us : rcu_preempt_qs <-rcu_note_context_switch
  <idle>-0       3.N.1   27us : _raw_spin_lock_irq <-__schedule
  <idle>-0       3dN.1   27us : add_preempt_count <-_raw_spin_lock_irq
  <idle>-0       3dN.2   28us : put_prev_task_idle <-__schedule
  <idle>-0       3dN.2   28us : pick_next_task_stop <-pick_next_task
  <idle>-0       3dN.2   28us : pick_next_task_rt <-pick_next_task
  <idle>-0       3dN.2   29us : dequeue_pushable_task <-pick_next_task_rt
  <idle>-0       3d..3   29us : __schedule <-preempt_schedule
  <idle>-0       3d..3   30us :      0:120:R ==> [003]  2448: 94:R sleep
````
### 1. Il momento del risveglio (Latenza 1-3 us)
Tutto inizia quando un interrupt hardware sveglia il task `sleep`.
```
<idle>-0  3d.h4   1us+:  0:120:R   + [003]  2448: 94:R sleep
<idle>-0  3d.h4   2us :  ttwu_do_activate <-try_to_wake_up
```

- **`+`**: Il task `sleep` viene inserito nella coda di esecuzione (runqueue).
- **`h4`**: Siamo ancora dentro un gestore di interrupt hardware (`hardirq`).
    
### 2. La comparsa del flag `N` (Latenza 4 us)
Qui accade qualcosa di fondamentale per lo scheduling:

```
<idle>-0  3dNh3   4us :  task_woken_rt <-ttwu_do_wakeup
```

- **`N` (Need Resched)**: Il kernel ha capito che il task appena svegliato (`sleep`) ha una priorità più alta di quello attuale (`idle`). Imposta quindi il flag `TIF_NEED_RESCHED`. Questo flag dice al kernel: _"Appena finisci quello che stai facendo, chiama lo scheduler immediatamente!"_.
### 3. "Pulizie di casa" del Kernel (Latenza 5-24 us)

Questa è la parte più densa della traccia. Poiché la CPU era in stato di risparmio energetico (`idle`), prima di passare al task RT, il kernel deve uscire correttamente dallo stato di "quiete":

- **Gestione Timer (`hrtimer`)**: Il kernel deve ricalcolare quando scadranno i prossimi timer hardware (`tick_program_event`, `lapic_next_event`).
- **RCU (`rcu_idle_exit`)**: Il kernel aggiorna il sottosistema RCU (Read-Copy Update) comunicando che la CPU non è più inattiva.
- **Contabilità del carico (`cpu_load_update_nohz`)**: Viene aggiornata la statistica del carico della CPU.
    

> **Nota:** Tutte queste funzioni (che vedi tra i 5us e i 24us) sono il motivo per cui la latenza è passata da 5us (dell'esempio precedente) a 29us. È l' "overhead" necessario per svegliare una CPU che stava dormendo.

### 4. Il Cambio di Contesto Finale (Latenza 25-30 us)

Finalmente, il kernel è pronto per il passaggio di testimone:

```
<idle>-0  3.N..  25us :  schedule <-cpu_idle
<idle>-0  3.N..  25us :  __schedule <-preempt_schedule
...
<idle>-0  3dN.2  28us :  pick_next_task_rt <-pick_next_task
<idle>-0  3d..3  30us :  0:120:R ==> [003]  2448: 94:R sleep
```

- **`pick_next_task_rt`**: Lo scheduler sceglie specificamente il task RT (`sleep`) dalla coda.
    
- **`==>`**: Il context switch è completato. Il processo `sleep` è ora in esecuzione.
    
### Sintesi dei Flag e del Flusso

Per capire meglio l'evoluzione, ecco come cambiano i flag nella colonna centrale:

1. **`3d.h4`**: (Inizio) Interrupt attivo, preemption disabilitata.
2. **`3dNh3`**: Viene aggiunto il flag **`N`**. Lo scheduler è "prenotato".
3. **`3dN.1`**: Uscita dall'interrupt hardware, si eseguono le pulizie di sistema.
4. **`3d..3`**: (Fine) Il task RT prende il controllo della CPU.


# # Hardware Latency Detector[](https://docs.kernel.org/trace/ftrace.html#hardware-latency-detector "Permalink to this heading")

The hardware latency detector is executed by enabling the “hwlat” tracer.

NOTE, this tracer will affect the performance of the system as it will periodically make a CPU constantly busy with interrupts disabled.

```
# echo hwlat > current_tracer
# sleep 100
# cat trace
# tracer: hwlat
#
# entries-in-buffer/entries-written: 13/13   #P:8
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
           <...>-1729  [001] d...   678.473449: #1     inner/outer(us):   11/12    ts:1581527483.343962693 count:6
           <...>-1729  [004] d...   689.556542: #2     inner/outer(us):   16/9     ts:1581527494.889008092 count:1
           <...>-1729  [005] d...   714.756290: #3     inner/outer(us):   16/16    ts:1581527519.678961629 count:5
           <...>-1729  [001] d...   718.788247: #4     inner/outer(us):    9/17    ts:1581527523.889012713 count:1
           <...>-1729  [002] d...   719.796341: #5     inner/outer(us):   13/9     ts:1581527524.912872606 count:1
           <...>-1729  [006] d...   844.787091: #6     inner/outer(us):    9/12    ts:1581527649.889048502 count:2
           <...>-1729  [003] d...   849.827033: #7     inner/outer(us):   18/9     ts:1581527654.889013793 count:1
           <...>-1729  [007] d...   853.859002: #8     inner/outer(us):    9/12    ts:1581527658.889065736 count:1
           <...>-1729  [001] d...   855.874978: #9     inner/outer(us):    9/11    ts:1581527660.861991877 count:1
           <...>-1729  [001] d...   863.938932: #10    inner/outer(us):    9/11    ts:1581527668.970010500 count:1 nmi-total:7 nmi-count:1
           <...>-1729  [007] d...   878.050780: #11    inner/outer(us):    9/12    ts:1581527683.385002600 count:1 nmi-total:5 nmi-count:1
           <...>-1729  [007] d...   886.114702: #12    inner/outer(us):    9/12
```

The above output is somewhat the same in the header. All events will have interrupts disabled ‘d’. Under the FUNCTION title there is:

> #1
> 
> This is the count of events recorded that were greater than the tracing_threshold (See below).
> 
> inner/outer(us): 11/11
> 
> > This shows two numbers as “inner latency” and “outer latency”. The test runs in a loop checking a timestamp twice. The latency detected within the two timestamps is the “inner latency” and the latency detected after the previous timestamp and the next timestamp in the loop is the “outer latency”.
> 
> ts:1581527483.343962693
> 
> > The absolute timestamp that the first latency was recorded in the window.
> 
> count:6
> 
> > The number of times a latency was detected during the window.
> 
> nmi-total:7 nmi-count:1
> 
> > On architectures that support it, if an NMI comes in during the test, the time spent in NMI is reported in “nmi-total” (in microseconds).
> > 
> > All architectures that have NMIs will show the “nmi-count” if an NMI comes in during the test.


# Function tracer

Mentre i tracciatori precedenti (`irqsoff`, `wakeup_rt`) si concentrano sulla misura del tempo (latenza), il **function tracer** è il "microscopio" di ftrace. Registra ogni singola funzione del kernel che viene chiamata in tempo reale.

Questo strumento permette di seguire il flusso di esecuzione del kernel. È estremamente potente, ma genera una quantità enorme di dati in pochissimo tempo.

### 1. Requisiti e Attivazione

Per funzionare, deve essere attivo l'interruttore globale `ftrace_enabled`. Se è impostato a 0, il tracer non produrrà alcun output (operazione _nop_).

```bash
# Verifica o attiva il supporto globale
sysctl kernel.ftrace_enabled=1

# Imposta il tracer
echo function > current_tracer
echo 1 > tracing_on

# Eseguiamo un comando brevissimo (usleep 1 microsecondo)
usleep 1

# Spegniamo subito per non riempire il buffer
echo 0 > tracing_on
```

### 2. Analisi dell'Output (Il Flusso delle Funzioni)

```plaintext
# cat trace
# tracer: function
#
# entries-in-buffer/entries-written: 24799/24799   #P:4
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
            bash-1994  [002] ....  3082.063030: mutex_unlock <-rb_simple_write
            bash-1994  [002] ....  3082.063031: __mutex_unlock_slowpath <-mutex_unlock
            bash-1994  [002] ....  3082.063031: __fsnotify_parent <-fsnotify_modify
            bash-1994  [002] ....  3082.063032: fsnotify <-fsnotify_modify
            bash-1994  [002] ....  3082.063032: __srcu_read_lock <-fsnotify
            bash-1994  [002] ....  3082.063032: add_preempt_count <-__srcu_read_lock
            bash-1994  [002] ...1  3082.063032: sub_preempt_count <-__srcu_read_lock
            bash-1994  [002] ....  3082.063033: __srcu_read_unlock <-fsnotify
[...]
```

Guardiamo cosa succede quando scriviamo nel file system (probabilmente proprio mentre stavamo dando il comando per fermare il tracciamento):

```
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
            bash-1994  [002] ....  3082.063030: mutex_unlock <-rb_simple_write
            bash-1994  [002] ....  3082.063031: __mutex_unlock_slowpath <-mutex_unlock
```

- **`bash-1994`**: Il processo che ha scatenato la chiamata è la shell (PID 1994).
- **`[002]`**: Tutto avviene sulla CPU 2.
- **`....`**: I flag (irqs, need-resched, etc.) sono tutti spenti in questo frammento.
- **`mutex_unlock <- rb_simple_write`**: Questa è la parte fondamentale. Si legge così: "È stata chiamata la funzione `mutex_unlock`, ed è stata invocata da `rb_simple_write`".
    

In pratica, ftrace ci mostra non solo cosa sta succedendo, ma anche **chi ha chiamato chi**.

### 3. Il Problema del Buffer (Ring Buffer)

Ftrace usa un **ring buffer** (un buffer circolare). Immaginalo come un nastro trasportatore ad anello: quando il nastro è pieno, i nuovi dati iniziano a sovrascrivere quelli più vecchi.

**Il rischio:** Se esegui un comando e poi scrivi manualmente `echo 0 > tracing_on`, il tempo che impieghi a digitare o il tempo che il sistema impiega a processare il comando `echo` potrebbe generare così tante chiamate di funzioni da "spingere fuori" dal buffer i dati che ti interessavano davvero.



### 4. Soluzione: Disattivazione via Software (C Code)

Per catturare un evento con precisione chirurgica, la soluzione migliore è fermare il tracciamento direttamente dall'interno di un programma C non appena si verifica la condizione che vuoi studiare.

Ecco come si fa in pratica:


```c
int trace_fd;

int main(int argc, char *argv[]) {
    // 1. Apriamo il file che controlla il tracciamento
    // (tracing_file è una funzione ipotetica che restituisce il percorso corretto, 
    // solitamente /sys/kernel/debug/tracing/tracing_on)
    trace_fd = open("/sys/kernel/debug/tracing/tracing_on", O_WRONLY);

    // ... esecuzione del codice ...

    // 2. Se succede qualcosa di interessante (es. un errore o un evento raro)
    if (condition_hit()) {
        // Scriviamo "0" nel file per stoppare ftrace IMMEDIATAMENTE
        write(trace_fd, "0", 1);
        // Ora il buffer è "congelato" al momento esatto dell'evento
    }
    
    close(trace_fd);
}
```
### Perché è meglio?
Usando `write()` nel codice:
- **Latenza zero:** Il tracciamento si ferma nel microsecondo esatto in cui la condizione è colpita.
- **Dati puliti:** Il buffer conterrà esattamente le chiamate di funzione che hanno portato a quel momento, senza i log inutili del comando `echo` digitato dall'utente.

# Function-Graph
Il **Function Graph Tracer** è l'evoluzione del function tracer: non si limita a dirti quali funzioni vengono chiamate, ma disegna un vero e proprio **grafico gerarchico** dell'esecuzione, mostrandoti visivamente l'apertura e la chiusura di ogni funzione (come se fosse codice C).

Per tracciare sia l'entrata che l'uscita, ftrace usa uno stack speciale memorizzato nel `task_struct` di ogni processo. Quando una funzione inizia, il tracer **sovrascrive l'indirizzo di ritorno** della funzione con un "probe" personalizzato. L'indirizzo originale viene salvato nello stack di ftrace e ripristinato alla fine, permettendo così di misurare esattamente quanto tempo è passato tra `{` e `}`.

```plaintext
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |

 0)               |  sys_open() {
 1)               |    do_sys_open() {
 2)               |      getname() {
 3)               |        kmem_cache_alloc() {
 4)   1.382 us    |          __might_sleep();
 5)   2.478 us    |        }
 6)               |        strncpy_from_user() {
 7)               |          might_fault() {
 8)   1.389 us    |            __might_sleep();
 9)   2.553 us    |          }
 10)   3.807 us    |        }
 11)   7.876 us    |      }
 12)               |      alloc_fd() {
 13)   0.668 us    |        _spin_lock();
 14)   0.570 us    |        expand_files();
 15)   0.586 us    |        _spin_unlock();
```
## Analisi dell'Output Standard

Ecco come leggere una traccia tipica:
```
0)               |  sys_open() {
 1)               |    do_sys_open() {
 2)   1.382 us    |      __might_sleep();
 3)   2.478 us    |    }
```

- **`0)`**: Indica il numero della CPU.
- **`1.382 us`**: La durata dell'esecuzione. Se vedi una funzione su una sola riga (punto e virgola `;`), è una "funzione foglia" (non ne chiama altre). Se vedi le graffe `{ }`, la durata totale è riportata sulla graffa di chiusura.

**I Flag di Overhead**
```
+ means that the function exceeded 10 usecs.
! means that the function exceeded 100 usecs.
# means that the function exceeded 1000 usecs.
* means that the function exceeded 10 msecs.
@ means that the function exceeded 100 msecs.
$ means that the function exceeded 1 sec.
```
## Opzioni di Visualizzazione (Personalizzazione)

Puoi attivare o disattivare diverse colonne agendo su `trace_options`. 
### 1. Processo e PID (`funcgraph-proc`)

Utile per capire quale programma sta chiamando le funzioni del kernel. `echo funcgraph-proc > trace_options`

> Esempio: `0) sh-4802 | 4.040 us | call_rcu();`

### 2. Valore di Ritorno (`funcgraph-retval`)

**Fondamentale per il debug.** Ti permette di vedere cosa ha restituito la funzione (es. codici di errore come -22). `echo funcgraph-retval > trace_options`

> Esempio: `} /* cpu_cgroup_can_attach = -22 */` _Nota: Se il valore è un errore, viene mostrato in decimale (-22), altrimenti in esadecimale._

```
1)               |    cgroup_migrate() {
2)   0.651 us    |      cgroup_migrate_add_task(); /* = 0xffff93fcfd346c00 */
3)               |      cgroup_migrate_execute() {
4)               |        cpu_cgroup_can_attach() {
5)               |          cgroup_taskset_first() {
6)   0.732 us    |            cgroup_taskset_next(); /* = 0xffff93fc8fb20000 */
7)   1.232 us    |          } /* cgroup_taskset_first = 0xffff93fc8fb20000 */
8)   0.380 us    |          sched_rt_can_attach(); /* = 0x0 */
9)   2.335 us    |        } /* cpu_cgroup_can_attach = -22 */
10)   4.369 us    |      } /* cgroup_migrate_execute = -22 */
11)   7.143 us    |    } /* cgroup_migrate = -22 */
```

The above example shows that the function cpu_cgroup_can_attach returned the error code -22 firstly, then we can read the code of this function to get the root cause.

### 3. Commenti Personalizzati (`trace_printk`)

Puoi inserire i tuoi commenti nel grafico chiamando `trace_printk()` nel codice C del kernel. Appariranno come commenti in stile C: `/* I'm a comment! */`.

## Limitazioni Attuali (Attenzione!)

Il tracciamento del valore di ritorno (`retval`) ha alcuni limiti tecnici da conoscere:
1. **Void:** Anche se una funzione non restituisce nulla (`void`), ftrace stamperà comunque un valore casuale presente nel registro (puoi ignorarlo).
2. **64-bit su sistemi 32-bit:** Se un valore di ritorno occupa due registri (es. `eax` e `edx` su x86), ftrace attualmente legge solo il primo.
3. **Narrowing (Cast):** Su architetture come ARM64, se una funzione restituisce un tipo piccolo (es. `u8`), i bit alti del registro potrebbero contenere "spazzatura" tecnica, facendo apparire il valore di ritorno più grande di quello che è in realtà.
