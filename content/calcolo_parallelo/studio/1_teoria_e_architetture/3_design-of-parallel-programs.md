
Scrivere programmi paralleli introduce sfide specifiche (es. sincronizzazione, colli di bottiglia, gestione della memoria) che non esistono nella programmazione sequenziale.
Spesso, un buon algoritmo parallelo richiede un approccio logico completamente diverso rispetto alla sua controparte sequenziale.
## Caso di Studio: Sum-Reduction
**Definizione:** Dato un array di n numeri, vogliamo calcolarne la somma totale. 

### 1. [[2_parallel-architectures#^1b2c6a|Soluzione Sequenziale]]
Un semplice ciclo che itera su tutti gli elementi. La complessità temporale è asintotica lineare Theta(n). Si usa Theta per indicare il limite esatto, non solo il limite superiore come O-grande.

```c
float seq_sum(const float* v, int n) {
    float sum = 0.0; 
    for (int i = 0; i < n; i++) { 
        sum += v[i]; 
    } 
    return sum; 
}
```

---

### 2. Soluzioni a [[2_parallel-architectures#^|Memoria Condivisa]]
Assumiamo di avere P unità di calcolo (processi/thread). Le variabili precedute da "my_" sono private (locali al singolo processo), mentre le altre sono globali e condivise.

Di seguito l'evoluzione della soluzione. **Clicca sui link nella tabella per saltare alla spiegazione e al codice corrispondente:**

| Tentativo | Approccio | Problema / Risultato |
| :--- | :--- | :--- |
| [[#Tentativo 1 Nessuna Sincronizzazione\|1. Nessuna Sincronizzazione]] | Suddivisione dei blocchi e somma diretta sulla variabile globale "sum". | **Race Condition.** Più processi si sovrascrivono a vicenda. |
| [[#Tentativo 2 Mutex su ogni elemento\|2. Mutex su ogni elemento]] | Uso di "mutex_lock" prima di aggiornare "sum" nel ciclo "for". | **Inefficiente.** Risolve la race condition, ma genera overhead bloccando n volte. I resti della divisione sono ignorati. |
| [[#Tentativo 3 Mutex a grana grossa\|3. Mutex a grana grossa]] | Calcolo locale di "my_sum" e aggiornamento globale protetto da mutex alla fine. | **Buono ma serializzato.** Mutex acquisito solo P volte, gestisce correttamente i resti. |
| [[#Tentativo 4 Array condiviso senza Barriera\|4. Array condiviso senza Barriera]] | Ogni processo salva la sua somma locale nell'array globale "psum". | **Errato (No sync).** Il Master somma prima che gli altri abbiano finito di scrivere i risultati. |
| [[#Tentativo 5 Array condiviso con Barriera\|5. Array condiviso con Barriera]] | Come il precedente, ma si inserisce una direttiva "barrier()". | **Corretto.** Il Master attende la conclusione di tutti i Worker prima di aggregare. |

#### Tentativo 1 Nessuna Sincronizzazione
**Problema:** Si verifica una **Race Condition**. Più processi tentano di leggere e scrivere simultaneamente sulla stessa variabile "sum", sovrascrivendosi a vicenda e producendo un risultato errato.

```c
sum = 0.0;

do in parallel {
    my_block_len = n / P; 
    my_start = my_id * my_block_len;
    my_end = my_start + my_block_len; 
    
    for (my_i = my_start; my_i < my_end; my_i++) { 
        sum += v[my_i]; /* ERRORE: Race condition */
    }
}
```

#### Tentativo 2 Mutex su ogni elemento
**Problema:** L'acquisizione e il rilascio del lock avvengono per ogni singolo elemento dell'array, creando un enorme collo di bottiglia prestazionale. Inoltre, usando `my_block_len = n/P`, se n non è divisibile perfettamente per P, il resto degli elementi (esattamente n % P) viene ignorato.

```c
mutex m; 
sum = 0.0; 

do in parallel { 
    my_block_len = n / P; 
    my_start = my_id * my_block_len; 
    my_end = my_start + my_block_len; 
    
    for (my_i = my_start; my_i < my_end; my_i++) { 
        mutex_lock(&m); 
        sum += v[my_i]; 
        mutex_unlock(&m); 
    } 
}
```

#### Tentativo 3 Mutex a grana grossa
**Spiegazione:** Ogni processo calcola prima un risultato parziale in una variabile locale. Solo alla fine si acquisisce il Mutex. La divisione calcolata moltiplicando n per l'ID prima di dividere copre tutti gli elementi. Corretta ed efficiente (lock richiamato solo P volte), ma il Mutex impone ancora una parziale serializzazione alla fine.

```c
mutex m; 
sum = 0.0; 

do in parallel { 
    my_start = (n * my_id) / P; 
    my_end = (n * (my_id + 1)) / P; 
    my_sum = 0.0; 
    
    for (my_i = my_start; my_i < my_end; my_i++) { 
        my_sum += v[my_i]; 
    } 
    
    mutex_lock(&m); 
    sum += my_sum; 
    mutex_unlock(&m); 
}
```

#### Tentativo 4 Array condiviso senza Barriera
**Problema:** Manca la sincronizzazione temporale. Il processo Master potrebbe arrivare alla fine del suo calcolo e iniziare la somma totale prima che un altro processo abbia finito di scrivere il proprio risultato parziale su "psum".

```c
psum[0..P-1] = 0.0; 
sum = 0.0; 

do in parallel { 
    my_start = (n * my_id) / P; 
    my_end = (n * (my_id + 1)) / P; 
    
    for (my_i = my_start; my_i < my_end; my_i++) { 
        psum[my_id] += v[my_i]; 
    } 
    
    if (0 == my_id) { /* Solo il master esegue questo */
        for (my_i = 0; my_i < P; my_i++) {
            sum += psum[my_i]; /* ERRORE: Manca attesa */
        }
    } 
}
```

Problema bastardo, questo grafico fa capire meglio forse : 

![[4_design-of-parallel-programs_page_12.png|fix]]
#### Tentativo 5 Array condiviso con Barriera
**Spiegazione:** Riprende l'idea dell'array condiviso per evitare i Mutex, ma introduce una direttiva `barrier()`. Questa istruzione obbliga tutti i processi ad aspettarsi. Il Master procederà alla somma finale solo quando tutti hanno depositato il risultato parziale. È la versione corretta e ottimale per la memoria condivisa.

```c
psum[0..P-1] = 0.0; 
sum = 0.0; 

do in parallel { 
    my_start = (n * my_id) / P; 
    my_end = (n * (my_id + 1)) / P; 
    psum[0..P-1] = 0.0; /* Reset */
    
    for (my_i = my_start; my_i < my_end; my_i++) { 
        psum[my_id] += v[my_i]; 
    } 
    
    barrier(); /* Attesa sincronizzata di tutti i processi */
    
    if (my_id == 0) { 
        for (my_i = 0; my_i < P; my_i++) {
            sum += psum[my_i]; 
        }
    } 
}
```

---

### 3. Soluzioni a [[2_parallel-architectures#^1b2c6a|Memoria Distribuita]]
In questo paradigma non esiste memoria globale, quindi non ci possono essere Race Conditions (tutte le variabili sono private). Si usa lo scambio di messaggi (Message Passing).

**Spiegazione Base (Master-Worker):** Il Master (proc 0) distribuisce un pezzo dell'array a ciascun processo Worker. Ogni processo elabora la somma e la invia al Master, il quale somma tutti i messaggi ricevuti.

```c
my_sum = 0.0; 
my_start = …, my_end = …; /* Calcolo indici di inizio e fine */
my_v[] = receive v[my_start..my_end-1] from proc 0; 

for (int i = 0; i < (my_end - my_start); i++) { 
    my_sum += my_v[i]; 
} 

if (my_id == 0) { 
    /* Il Master riceve e somma i contributi */
    for (int i = 1; i < P; i++) { 
        tmp = receive from proc i; 
        my_sum += tmp; 
    } 
    printf("The sum is %f\n", my_sum); 
} else { 
    /* I Worker inviano al Master */
    send my_sum to proc 0; 
}
```

#### Il Collo di Bottiglia e la Riduzione ad Albero:

Il problema della soluzione descritta sopra è che il Processore 0 deve gestire un traffico e calcoli pari a **Theta(P)**, diventando il collo di bottiglia.

![[4_design-of-parallel-programs_page_15.png|fix]]


La soluzione ottimale in memoria distribuita è la **Riduzione Parallela ad Albero (Tree Reduction)**: 

![[4_design-of-parallel-programs_page_16.png|fix]]

I processi cooperano inviandosi i dati a coppie e aggregandoli progressivamente (come in un torneo a eliminazione). Così, le somme da fare per il *processo 0* crollano da Theta(P) a **Theta(log2 P)**.

---

## Paradigmi di Parallelismo: Task vs Data

| Paradigma            | Descrizione                                                                                                                     | Utilizzo Tipico                                                                                                                                           |
| :------------------- | :------------------------------------------------------------------------------------------------------------------------------ | :-------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Task Parallelism** | Consiste nel distribuire task/funzioni differenti sui vari processori. Ogni processo fa una cosa diversa.                       | Algoritmi Divide et Impera, elaborazione di eventi eterogenei, pipeline di dati.                                                                          |
| **Data Parallelism** | Tutti i processori eseguono lo stesso codice, ma operano su porzioni di dati differenti (SPMD - Single Program, Multiple Data). | È l'approccio principale di questo corso. Utilizzato in [[introduzione a OpenMP\|openMP]], elaborazioni su matrici e GPU ([[Introduzione A CUDA\|CUDA]]). |