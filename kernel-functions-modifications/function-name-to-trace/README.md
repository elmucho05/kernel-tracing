# Cambio nome della funzione da tracciare

Per superare il limite dell'hardcoding del nome del processo da tracciare nel codice C del kernel e dover ricompilare ogni volta l'intero kernel è stato deciso di implementare un'interfaccia dinamica direttamente nel sottosistema di tracing nativo di Linux (`tracefs`).

Questo permette di modificare il nome della funzione da tracciare direttamente da cli. Espone una variabile dallo spazio di indirizzamento del kernel come se fosse un normale file di testo in `/sys/kernel/debug/tracing/target_process`.

Per poterlo fare è stato modificato il file sorgente del tracing `kernel/trace/trace.c` nei seguenti punti :

### Aggiunta della variabile globale
```c
char target_process_name[16] = "cyclictest";
EXPORT_SYMBOL(target_process_name);
```

- la macro `EXPORT_SYMBOL` dice al linker del kernel di rendere questa variabile visibile e accessibile anche dagli altri file sorgente (che sarà poi accessibile tramite la direttiva `extern`).

### Aggiunta della funzione di callback per la lettura del file
```c
/*function invoked when reading the file*/
static ssize_t target_process_read(struct file *filp, char __user *ubuf, size_t cnt, loff_t *ppos)
{
    char buf[64];
    int r;

    r = snprintf(buf, sizeof(buf), "%s\n", target_process_name);

    // helper function of the kernel to send data to user space
    return simple_read_from_buffer(ubuf, cnt, ppos, buf, r);
}

```

- questa funzione viene chiamata dal kernel ogni volta che l'utente, tenta di leggere il file ad esempio lanciando il comando `cat /sys/kernel/debug/tracing/target_process`.
- genera una stringa formattata contenente il valore attuale di `target_process_name`. Dopodiché, utilizza la funzione `simple_read_from_buffer` per trasferire i dati dallo spazio di indirizzamento del kernel verso il buffer dello userspace(`ubuf`). Questo è necessario perché lo spazio utente non ha i privilegi necessari per poter leggere i dati dallo spazio di indirizzamento del kernel.


### Aggiunta della callback di scrittura
```c
/*function invoked when writing to the file*/
static ssize_t target_process_write(struct file *filp, const char __user *ubuf, size_t cnt, loff_t *ppos)
{
    char buf[16];

    if (cnt >= sizeof(buf))
        cnt = sizeof(buf) - 1;

    // copy data from user space to kernel space
    if (copy_from_user(buf, ubuf, cnt))
        return -EFAULT;

    buf[cnt] = '\0';
    
    // when you press enter you add \n so wee need to remove it
    if (cnt > 0 && buf[cnt-1] == '\n')
        buf[cnt-1] = '\0';

    strncpy(target_process_name, buf, sizeof(target_process_name));

    return cnt;
}

```

- questa funzione viene chiamata quando l'utente scrive su file, per esempio `echo nome_funzione > /sys/.../target_process`
- esegue l'operazione inversa della funzione read. 

- Parametri delle funzioni dal virtual file system(VFS) del kernel:
	- `cnt` il numero di byte letti in user space con eventuale controllo dell'ultimo carattere per rimuovere il `\n`. Prima però ho fatto un controllo sulla dimensione, dove tronco a 15 byte.
	- Inserisco il terminatore di stringa e nel caso rimuovo \n. 
- La funzione deve sempre ritornare il numero di byte che l'utente le aveva inizialmente chiesto di scrivere.



