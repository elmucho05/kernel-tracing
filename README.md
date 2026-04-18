# Tracing

Questa repository contiene appunti e tracce per l'analisi dei task real-time e delle relative latenze nel kernel Linux con la patch PREEMPT_RT.


### Documentazione

* [Ftrace](ftrace.md) : guida all'uso di Ftrace, Function Graph e latenze.
* [Interrupts](interrupts.md) : gestione degli interrupt generale e nel kernel linux
* [trace-cmd](trace-cmd.md) : documentazione utile per il front-end di `ftrace`
### Tracce reali

* [Traces IMX8MP](./traces-imx8mp): directory contenente i log e le tracce catturate sulla i.MX8M Plus.

### Modifiche al kernel
La directory [kernel-functions-modifications](./kernel-functions-modifications) contiene una la lista di tutte le modifiche apportate ad alcune funzioni del kernel linux per poter avere una stampa in fase di tracing che ci permetta di identificare meglio le 4 fasi del ciclo di vista di cyclictest. L'obiettivo è quello di arrivare a poter "greppare" tramite dei tag semplici e significativi le varie parti di traccia che ci interessano. 

Le modifiche apportate a queste funzioni sono piuttosto "basilari" in quanto ci si è limitati solo a introdurre in vari punti del codice delle `trace_printk` che stampano messaggi personalizzati in base alle funzioni che il kernel sta eseguendo MA soltanto se quella funzione sta svolgendo operazioni che portano `cyclictest` a svegliarsi/dormire.






