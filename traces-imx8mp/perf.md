# Guida all'uso di `perf` per l'analisi cache (i.MX8)
## 1. Comandi di Esplorazione

- `perf list`: Elenca tutti i simboli degli eventi tracciabili dal kernel.
    
- `perf list cache`: Mostra gli eventi predefiniti legati alle cache (es. L1).
    
- `perf list pmu`: Elenca gli eventi "grezzi" (raw) specifici dell'hardware (es. `l2d_cache_refill`).
    

## 2. Opzioni del comando `perf stat`

Il comando `perf stat` viene utilizzato per raccogliere statistiche aggregate durante l'esecuzione di un programma.

| **Opzione**    | **Nome**           | **Descrizione**                                                            |
| -------------- | ------------------ | -------------------------------------------------------------------------- |
| **`-e`**       | **Event**          | Specifica la lista di eventi hardware da monitorare (separati da virgola). |
| **`-a`**       | **All CPUs**       | Raccoglie i dati da tutti i core della CPU contemporaneamente.             |
| **`-A`**       | **No aggregation** | Mostra i risultati separati per ogni singolo core (fondamentale per L1).   |
| **`-C <n>`**   | **Core**           | Monitora esclusivamente il core specifico indicato (es. `-C 1`).           |
| **`-p <PID>`** | **PID**            | Si aggancia a un processo già in esecuzione tramite il suo identificativo. |


## 3. Eventi hardware 

Poiché gli alias standard (come `LLC`) potrebbero non essere supportati, si utilizzano i nomi del **PMU Kernel**:
- **`L1-dcache-loads`**: Totale accessi alla Cache L1 Dati.
- **`L1-dcache-load-misses`**: Fallimenti in L1 (il dato deve essere cercato in L2).
- **`l2d_cache`**: Totale accessi alla Cache L2 (condivisa/LLC).
- **`l2d_cache_refill`**: Fallimenti in L2 (il dato deve essere cercato in RAM). **Questo è il contatore critico per l'interferenza.**

Per una visione completa degli eventi: 

```bash
perf list cache
```

```sh
perf list pmu
```


## 4. Esempi di comandi 

### Analisi su tutti i core con visualizzazione separata

Utile per vedere il "rumore di fondo" del sistema su ogni CPU:
```
perf stat -a -A -e L1-dcache-loads,L1-dcache-load-misses,l2d_cache_refill sleep 5
```

### Analisi di un processo target 

Per misurare le performance "pulite" di `cyclictest` (Cache Calda):
Due opzioni in questo caso : 
1. usare le opzioni corrette di `cyclictest` : affinity `-a 1`
2. oppure usare `taskset -c` se si tratta di un comando generico
```
perf stat -e L1-dcache-load-misses,l2d_cache_refill taskset -c 1 <comando>[opzioni]
```

### Analisi sotto Interferenza
Mentre `meminterf` (HeSocMark) satura la L2 sul Core 2, misuriamo l'impatto sul Core 1:
```
# Terminale 1: Interferenza
taskset -c 2 meminterf -s 4M

# Terminale 2: Misurazione
perf stat -e L1-dcache-load-misses,l2d_cache_refill cyclictest -t 1 -a 1 -p 99 -i 1000 -q
```

## 5. Interpretazione Risultati

- **L1 Misses alti:** Problema interno all'algoritmo del programma (codice inefficiente).
- **L2 Refill alti (sotto interferenza):** Dimostra che un "vicino rumoroso" (Core 2) sta espellendo i dati del nostro task (Core 1) dalla cache condivisa, forzando accessi lenti alla RAM e causando latenza Real-Time.
