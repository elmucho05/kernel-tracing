### TAG delle fasi
Durante il processo di identificazione delle fasi del ciclo di vita di `cyclictest` è nata la necessità di effettuare ricerche mirate e veloci all'interno delle tracce.

Perciò per rendere il processo più immediato è stato pensato un sistema di TAG che identificano a grandi linee una delle 4 fasi da cercare.
- **Fase di addormentamento:** Tag `SLP:` (Sleep)
- **Fase di interrupt:** Tag `IRQ:` (Interrupt Request)
- **Fase di risveglio:** Tag `WKP:` (Wakeup)
- **Fase di scheduling:** Tag `SCH:` (Schedule)

### Modifica delle funzioni

Una volta definito il sistema di etichettatura (TAG), il passo successivo è stato mappare le fasi logiche del ciclo di vita di cyclictest sulle funzioni fisiche presenti nel codice sorgente del kernel.

Partendo dalle tracce generate, sono state identificate alcune funzioni candidate. Dopodiché è stato utilizzato lo strumento **Elixir Bootlin** per navigare i sorgenti del kernel e studiare tali funzioni.

Per ciascuna delle quattro fasi, è stata identificata almeno una funzione candidata  :
1. **Fase di Addormentamento (SLP):** `do_nanosleep`. Scelta perché rappresenta il punto di arrivo della system call `nanosleep` prima della cessione della CPU. Non è stata scelta la funzione `__arm64_sys_clock_nanosleep()` perché queste modifiche si stanno eseguendo su un kernel linux standard per processore x86_64, ma guardando le tracce, la funzione invocata dopo di essa è la `do_nanosleep`.
	1. `hrtimer_nanosleep` : altra funzione candidata che appare nella traccia addirittura prima delle funzione `do_nanosleep`
2. **Fase di Interrupt (IRQ):** `hrtimer_interrupt` e `hrtimer_wakeup` Scelte perché appaiono nella traccia in momenti critici dell'interrupt handler quando scatta il timer.
3. **Fase di Risveglio (WKP):** `try_to_wake_up`. La funzione dello scheduler che gestisce la logica di attivazione di un processo dormiente. Questa funzione chiama poi `ttwu_do_activate`. Anch'essa è stata scelta come candidata.
4. **Fase di Scheduling (SCH):** `finish_task_switch`. La prima funzione eseguita dal task non appena riprende il controllo della CPU dopo il cambio di contesto.

Per rendere il tracing è stata  sfruttata la struttura dati del kernel:**`struct task_struct`**.

Ogni processo in Linux è rappresentato in memoria da questa struttura. All'interno del codice kernel, la macro **`current`** funge da puntatore globale al processo che in quell'istante sta occupando la CPU.

Nello specifico, abbiamo utilizzato il campo **`current->comm`**:
- un array di caratteri, che contiene il nome del comando o dell'eseguibile (es. "cyclictest"). Ha una lunghezza fissa definita dalla costante `TASK_COMM_LEN` (solitamente 16 byte).
    
- **Utilizzo:** Attraverso la funzione `strncmp(current->comm, "cyclictest", 10)`, siamo in grado di verificare se il processo che sta eseguendo la funzione è esattamente l'istanza che vogliamo monitorare.



- Le funzioni relative alla gestione dei timer risiedono in `kernel/time/time.c`
- Le funzioni relative allo scheduling risiedono in `kernel/sched/core.c`

### Funzioni modificate in `kernel/sched/core.c`
#### `finish_task_switch`

```c
static struct rq *finish_task_switch(struct task_struct *prev)
        __releases(__rq_lockp(this_rq()))
{
        struct rq *rq = this_rq();
        struct mm_struct *mm = rq->prev_mm;
        unsigned int prev_state;
        if (strncmp(current->comm, target_process_name, 16) == 0) {
                trace_printk("SCHD: %s %s %d\n", __func__,current->comm, current->pid);
        }
        /*
         * The previous task will have left us with a preempt_count of 2
         * because it left us after:
         *
         *      schedule()
         *        preempt_disable();                    // 1
         *        __schedule()
         *          raw_spin_lock_irq(&rq->lock)        // 2
         *
         * Also, see FORK_PREEMPT_COUNT.
         */
        if (WARN_ONCE(preempt_count() != 2*PREEMPT_DISABLE_OFFSET,
                      "corrupted preempt_count: %s/%d/0x%x\n",
                      current->comm, current->pid, preempt_count()))
                preempt_count_set(FORK_PREEMPT_COUNT);

        rq->prev_mm = NULL;
...
...
...
}
```   

#### `try_to_wakeup`

```c
int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
        guard(preempt)();
        int cpu, success = 0;
        if (strncmp(p->comm, target_process_name, 16) == 0) {
                trace_printk("WKP: %s %s %d\n",__func__, p->comm, p->pid);
        }

        wake_flags |= WF_TTWU;


```
#### `ttwu_do_activate`

```c
static void
ttwu_do_activate(struct rq *rq, struct task_struct *p, int wake_flags,
                 struct rq_flags *rf)
{
        int en_flags = ENQUEUE_WAKEUP | ENQUEUE_NOCLOCK;

        lockdep_assert_rq_held(rq);

        if (p->sched_contributes_to_load)
                rq->nr_uninterruptible--;

        if (wake_flags & WF_RQ_SELECTED)
                en_flags |= ENQUEUE_RQ_SELECTED;
        if (wake_flags & WF_MIGRATED)
                en_flags |= ENQUEUE_MIGRATED;
        else
        if (p->in_iowait) {
                delayacct_blkio_end(p);
                atomic_dec(&task_rq(p)->nr_iowait);
        }
        if (strncmp(p->comm, target_process_name, 16) == 0) {
                trace_printk("WKP: %s %s %d\n",__func__, p->comm, p->pid);
        }
        activate_task(rq, p, en_flags);
        wakeup_preempt(rq, p, wake_flags);

        ttwu_do_wakeup(p);

        if (p->sched_class->task_woken) {
                /*
                 * Our task @p is fully woken up and running; so it's safe to
                 * drop the rq->lock, hereafter rq is only used for statistics.
                 */
                rq_unpin_lock(rq, rf);
                p->sched_class->task_woken(rq, p);
                rq_repin_lock(rq, rf);
        }
}
```


### Funzioni modificate in `kernel/time/hrtimer.c`


#### `hrtimer_nanosleep`

```c
long hrtimer_nanosleep(ktime_t rqtp, const enum hrtimer_mode mode, const clockid_t clockid)
{               
        struct restart_block *restart;
        struct hrtimer_sleeper t;
        int ret;
        if(strncmp(current->comm,target_process_name,16)==0){
                trace_printk("SLP: %s %s %d\n",__func__,current->comm,current->pid);
        }
        hrtimer_setup_sleeper_on_stack(&t, clockid, mode);
        hrtimer_set_expires_range_ns(&t.timer, rqtp, current->timer_slack_ns);
        ret = do_nanosleep(&t, mode);
        if (ret != -ERESTART_RESTARTBLOCK)
                goto out;

        /* Absolute timers do not update the rmtp value and restart: */
        if (mode == HRTIMER_MODE_ABS) {
                ret = -ERESTARTNOHAND;
                goto out;
        }

        restart = &current->restart_block;
        restart->nanosleep.clockid = t.timer.base->clockid;
        restart->nanosleep.expires = hrtimer_get_expires(&t.timer);
        set_restart_fn(restart, hrtimer_nanosleep_restart);
out:
        destroy_hrtimer_on_stack(&t.timer);
        return ret;
}
```

#### `hrtimer_interrupt`

```c
void hrtimer_interrupt(struct clock_event_device *dev)
{
        struct hrtimer_cpu_base *cpu_base = this_cpu_ptr(&hrtimer_bases);
        ktime_t expires_next, now, entry_time, delta;
        unsigned long flags;
        int retries = 0;

        if (strncmp(current->comm, target_process_name, 16) == 0) {
                trace_printk("IRQ: %s %s %d\n",__func__, current->comm,current->pid);
        }

        BUG_ON(!cpu_base->hres_active);
        cpu_base->nr_events++;
        dev->next_event = KTIME_MAX;
        dev->next_event_forced = 0;
...
...
...
}
```

#### `hrtimer_wakeup`

```c
static enum hrtimer_restart hrtimer_wakeup(struct hrtimer *timer)
{
        struct hrtimer_sleeper *t = container_of(timer, struct hrtimer_sleeper, timer);
        struct task_struct *task = t->task;

        if (task && strncmp(task->comm, target_process_name, 16) == 0) {
                trace_printk("IRQ: %s %s %d\n", __func__, task->comm, task->pid);
        }
        t->task = NULL;
        if (task)
                wake_up_process(task);

        return HRTIMER_NORESTART;
}
```

#### `do_nanosleep()`

```c
static int __sched do_nanosleep(struct hrtimer_sleeper *t, enum hrtimer_mode mode)
{
        struct restart_block *restart;

        do {
                set_current_state(TASK_INTERRUPTIBLE|TASK_FREEZABLE);
                hrtimer_sleeper_start_expires(t, mode);

                if (likely(t->task)){
                        if (strncmp(current->comm, target_process_name, 16) == 0) {
                                trace_printk("SLP: %s %s %d\n",__func__,current->comm,current->pid);
                        }
                        schedule();
                }

                hrtimer_cancel(&t->timer);
                mode = HRTIMER_MODE_ABS;

        } while (t->task && !signal_pending(current));

        __set_current_state(TASK_RUNNING);

        if (!t->task){
                if (strncmp(current->comm, target_process_name, 16) == 0) {
                        trace_printk("SCHD: %s %s %d\n",__func__,current->comm, current->pid);
                }
                return 0;
        }

        restart = &current->restart_block;
        if (restart->nanosleep.type != TT_NONE) {
                ktime_t rem = hrtimer_expires_remaining(&t->timer);
                struct timespec64 rmt;

                if (rem <= 0){
                        if (strncmp(current->comm, target_process_name, 16) == 0) {
                                trace_printk("SCHD: sched_end target=%s:%d\n", current->comm, current->pid);
                        }
                        return 0;
                }
                rmt = ktime_to_timespec64(rem);

                return nanosleep_copyout(restart, &rmt);
        }
        return -ERESTART_RESTARTBLOCK;
}
```
