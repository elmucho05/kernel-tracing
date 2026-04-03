Per la riproduzione del problema:
- al posto del task real-time forniti, provare ad eseguire cyclictest, con opzioni come --tracemark o -b. Se il problema si riproduce, allora cyclictest scatenerà una azione di tracing. Dal tracing probabilmente capiremo di più sulla causa.

Problema principale: scoprire la componente hardware, la componente kernel (interrupt handling, locking, ...) e la componente scheduling della latenza.

Altro modo per visualizzare quello che succede: fare tracing dello scheduling dei processi. Problema: eccesso di informazioni.

Usare offwaketime per analizzare le componente di tempo di risveglio in generale: tutte le componenti sommate.

Usare il tracer wakeup_rt invece per scoprire il tempo che passa dal wake up a quando il task viene effettivamente schedulato.

Usare rtla per indagare ulteriormente da dove arriva la latenza, dall'hardware.

Strumentare il codice per misurare i tempi da dentro il task. Per la strumentazione: scoprire come fa cyclictest a scatenare una traccia, e fare lo stesso nella propria strumentazione.

Ricordarsi che per trarre vantaggio dalle ottimizzazione di preempt-rt bisogna alzare la priorità dei task ai quali si vuole garantire la latenza. Questo permette di sfruttare priority inheritance, e forse anche gli sleeping locks.

Se rimangono casi di alta latenza
- controllare anche page faults. In quel caso considerare funzionalità come mlockall(), eventualmente tramite un wrapper script che invoca mlockall() e poi fa partire il processo da proteggere con una execvp().
- se ancora non si risolve, considerare l'effetto di System Management Interrupts (SMIs). Per misurare questi effetti, si può pensare di usare il tracer `hwlat`, per esempio attraverso il tool `hwlatdetect`.
- controllare se ci sono driver che lasciano gli interrupt disabilitati per tanto tempo. A questo scopo si può utilizzare il tracer `irqsoff`

-----------------
