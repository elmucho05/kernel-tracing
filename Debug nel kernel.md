Debugging, internal ecc

Debugging attraverso il tracing

Se ci metti `trace_printk`
- tu tiesci a stampare il messaggio che tu vuoi dare

Il prof si trovava con un ssd che adnava a meta' della velocita', con il suo io scheduler andava lento.
- accendeva il blcok tracer e dentro al suo codice aveva messo dei trace_printk
- il tracing del prof appariva anche il mezz o a tutto l'altro tracing

Il kernel si controlla da solo, dentro al codice del kernel c'e' il codice di debugging, tu lo devi solo trovare.

- il kernel alloca la memoria in pezzi, a **slub** in cui si prealloca un'intera zona di memoria spezzettata, e poi tu modulo del kernel divide lo slub in altri pezzettoni.

Il kernel deve andare veloce, si fida molto del programmatore.

Se invece accendo lo slub debugging, il kernel se ne accorge, quindi devo accedenre lo **slub debugging**

**Memory leak detection** : alloco della memoria ma non la uso, mi mangio la memoria. Mi basta accendere il controller del kernel

**KASan:** runtime memory debugging

**Debug OOPS**, *lockups and hangs* : si accorge se ho un task hanged, una syscall che non esce mai

**Lock debugging:** ha un suo sistema per capire se c'e' una condizione di deadlock.

**Debug kernel data structure** : debug linked list manipulation

**Lanciare il bug quando la memoria e' corrotta**

***Perche non aggiungere sempre questi controlli?***
- overhead, se per ogni accesso in memoria tu devi fare questi controlli, lentezza

**ATTENZIONE** : stanno nel config questi controlli, se tu lo rimetti su un'altro kernel, attenzione, avrai anche la' tali strumenti di debugging.


Il tracing si puo' spegnere, la l'overhead rimane.
- definire una macro per accendere e spegenre il ttracing del kernel


Tool come BPF, ha tanti programming che fanno il tracciamento di quello che succede nel kernel, per esempio off wake up time
- ti da' statistiche su quanto tempo i task passano addormentati, da spento a sveglio
- mi fa una statistica di quanto tmepo ha passato a dormire ogni task, li' vedo un po' e vedo quale task ci ha messo di piu' e cpasico quale sia il problema

Puoi vedere anche una qualche grafica dei task su per quanto si e' addormentato e su queli syscall.
- puoi fare la stessa cosa con uno script chiamato Perf?? e che fa un flame graph il cui il significato di ogni barra e' la percentuale di uso della cpu, quindi **possiamo fare le statistiche per vedere il tempo della CPU, perf me lo dice subito**

Questo e' un problema piuttosto semplice pero' casi in cui usa abbastanza la cpu ma va lento lo stesso, e' abbastanza difficile


Altra alternativa di debug e' la **STRUMENTAZIONE DEL CODICE** : letteralmente mettere dei debug


Il kernel e' **polimorfico** : hanno implementato il polimorfismo in C.
- esempio : mapping->a_ops->readpages() : 
	- readpages pero' non esiste in `aops`, perche' contiene puntatori a funzioni, si chiama il puntatore a funzione `readpages`
	- il prof cerca con una regex dove cerca l'assegnazione a tale puntatore

**Altra questione**
hai un problema in una funzuine del kernel, ma COME CI SEI arrivato la'?
- **dump_stack** : nel dmesg di fa vedere come ci sei arrivato a quella funzione
- attenzione, se la metti un a funzione che si invoca troppo spesso, ho troppe stamppe
	- **WARN_ONCE** : macro per stampare solo una volta
- senno', stampare semplicemente ogni tot, se non mi basta una volta e basta, mi faccio io una frequenza utile
	- i tick del kernel, a ogni tick viene incrementata una var globale che si chiama `jiffies`
 

## Layout del codice kernel
- arch : codice per ogni struttura
- block: codice per dispositivi a blocchi. I dischi sono a blocchi, il codice per gestire i dischi e' qua. altri dispositivi di i/o sono a caratteri
- certs : certificati
- crypto : 
- drivers
- documentation
- firmware: per esempio per nvidia, nvidia ti dava degli eseguibili gia' binari, il vendor non lo vuole aperto. quando forniva una qualche cosa, non voleva far uscire il codice
- fs : per fs, cosa si intende? 
	- il nome e' ambiguo, con fs si intende il contenuto ma anche la specifica tecnologia con cui stai strutturando il tuo disco
	- va a gestire le richeste di io
- include
- init : dove c'era main.c per esempio
- ipc : probabilmente inter process comunication
- kernel : funzionalita' di base del kerne
- lib : librerie
- mm : memory managemnet
- net : rete
- samples : esempi
- script : kbuild
- security
- sound
- tools: probabilmente c'e' anche quella cosa dei test
- usr 
- virt


L'implementazione di un semaforo e' la WAITQUEUE, che a sua volta e' implementata con una lista.

Albero bilanciato di ricerca, red black tree, 

Altra cosa importante e' il tempo, impementata con il timer

## Tracing e profiling
Ho scritto del codice che va lento. Con perf, scoprop dove si passa la maggior parte del tempo, ma devo indagare.

- mettere stampe ovunque, e' troppo lungo come processo lavorativo
- sol : accendo il **tracing** piu' ristretto possibile alla zona su cui ho capito il problema

Mi ritrovo in una della sol piu' difficili, ogni volta che lo eseguo, va ad una velocita' diversa, cambia comportamento.
- ma se sono furbo, lo porto a mio vantaggio, accendo il tracing e ho le tracce per ogni esecuzione.
- confronte le tracce tra il caso lento e veloce, per una questione logica, esiste una differenza, confronto le tracce. QUALCOSA DI DIVERSO STA SUCCEDENDO PER FORZA, lo posso anche dare in pasto all'AI


**PROFILING E INDUZIONE**
esistono dei casi in cui non arrivi per deduzione alla risposta
- allora ci arrivi per induzione, non deduzione

trucco: non ho trovato la causa, ma ho dei spospetti

Inizio con il *se fosse vero che, allora se cambio questo parametro allora deve succedere questo*


### Profiling
Per valutare la latenza : **cyclictest**
Task che si risveglia periodicamente e ti dice quanto ci ha rimesso a svegliarsi
- sotto linux non e' mai cosi'
- carino anche mettere disturbi tramite `stress-ng`

Quando si fa profiling, siamo interessati a sapere informazioni sul nostro sistema:
- lscpu
- lspc
- lsscasi
- lsblk
- df
- fdisk
- mount
- free 

Mentre si fa un test, e' buona prassi manetere la cpu a velocita' costante e senza idle.
- quando gestiamo gli eventi di IO, la cpu si addormenta in attesa che il buffer di IO ritorni
- poi prima si deve svegliare e poi leggere, hai tempo lento
- invece nel caso in cui arriva subito, ho un tempo piu' veloce, quindi due tempi diversi
	- sol : controllare TU la velocita' 


PERF:
- periodicamente va a controllare lo stato dei processi scelti
- attenzione, se la frequenza di esecuzione di perf e' troppo alta, passi troppo tempo ad esseguire perf

BPF:
- vive nel kernel, tende ad avere un overhead molto minore

Instrumentare il codice : stampe nel codice
oppure misuri la ufnzione chiamata quanto dura per capire magari se la funzione che ho chiamato

Il kernel si autocollauda sotto una base di test chiamati Kself test.
- il self test e' un modo per riprodurre il problema

