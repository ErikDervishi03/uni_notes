**Indice degli Argomenti:** 
* [[#Modello di Esecuzione (Fork-Join)]] 
* [[#Direttive (Pragma)]] 
* [[#La Direttiva omp parallel]] 
* [[#Funzioni Utili di OpenMP]] 
* [[#Parallelismo Annidato]] 
* [[#Misurare i Tempi di Esecuzione (Wall-Clock Time)]] 
* [[#Scope (Visibilità) delle Variabili]] 
* [[#Comportamento degli Array in OpenMP: Stack vs Heap]] 
* [[#Esempio: Il Metodo dei Trapezi]] 
* [[#Direttiva omp for]] 
* [[#Loop-Carried Dependencies]]
* [[#Schedule (Gestione del partizionamento)]] 
* [[#Direttiva collapse]] 
* [[#Mappatura dei Thread ai Core delle CPU (Thread Affinity)]] 
* [[#P-Core vs E-Core (Esempio: Transposed Matrix-Matrix Multiply)]] 
* [[#Esempio: Odd-Even Transposition Sort]] 
* [[#Direttive di Sincronizzazione]] 
* [[#Gestione Esplicita dei Task (Task Parallelism)]] 

---

OpenMP è il modello standard, basato su direttive al compilatore, per la programmazione parallela in architetture a memoria condivisa. È portabile tra architetture diverse ed è supportato dai moderni compilatori C, C++ e Fortran.

**Vantaggi:**
*   Permette la **parallelizzazione incrementale**: si parte da un programma seriale e, aggiungendo direttive specifiche, si identificano e parallelizzano progressivamente le regioni di codice che possono essere eseguite concorrentemente.
*   Fornisce primitive per la gestione della sincronizzazione (es. barrier) per proteggersi dalle race condition, sebbene sia responsabilità del programmatore usarle correttamente.

**Limitazioni:**
*   **Non parallelizza automaticamente**, non garantisce di per sé uno speedup (in quanto vi è l'overhead per la creazione dei thread, ad esempio) e non previene magicamente le race condition.

## Modello di Esecuzione (Fork-Join)
Un programma OpenMP inizia l'esecuzione come un singolo processo (il **master thread**).
1.  **Fork:** Quando il master thread incontra una regione parallela (`#pragma omp parallel`), crea (effettua un *fork*) un team di thread (worker). Tutti i thread eseguono la regione di codice in modo asincrono, ma duplicato per ognuno.
2.  **Join:** Al termine della regione parallela vi è una **sincronizzazione implicita (barrier)**: i thread si aspettano tra loro. Completata la regione, i thread collassano (*join*) e solo il master thread continua l'esecuzione.
## Direttive (Pragma)
OpenMP si basa su direttive del precompilatore (`#pragma omp`). Se un compilatore non supporta OpenMP, ignora i pragma e il programma funziona in modo seriale.
*   Le direttive si applicano ai **blocchi strutturati**: blocchi di codice con un singolo punto di ingresso in cima e un singolo punto di uscita alla fine.
*   *Restrizione:* Non è possibile utilizzare `goto` o `return` per uscire anticipatamente dall'interno di una regione parallela.

---

## La Direttiva `omp parallel`
```c
#pragma omp parallel [clause ...]
````

Crea un team di thread, dove ogni thread esegue il blocco di codice seguente. Ogni thread ha un proprio ID univoco all'interno del team (il master ha ID 0). Come detto, al termine del blocco c'è una barriera implicita.

**Esempio Hello World:**

```c
/* omp-demo0.c */ 
#include <stdio.h> 

int main( void ) { 
    #pragma omp parallel 
    { 
        printf("Hello, world!\n"); 
    } 
    return 0; 
}
```

**Compilazione ed esecuzione:** Per compilare (usando gcc) è necessario il flag `-fopenmp`.

```bash
gcc -fopenmp omp-demo0.c -o omp-demo0 
$ ./omp-demo0 
Hello, world! 
Hello, world! 
```

Il numero di thread di default dipende dall'implementazione. Può essere modificato a tempo di esecuzione tramite variabile d'ambiente:

```bash
$ OMP_NUM_THREADS=4 ./omp-demo0 
Hello, world! 
Hello, world! 
Hello, world! 
Hello, world!
```

---

## Funzioni Utili di OpenMP

Includendo la libreria `<omp.h>`, si ha accesso a varie funzioni runtime:

- `omp_get_thread_num()`: Restituisce l'ID del thread chiamante all'interno del team corrente.
    
- `omp_get_num_threads()`: Restituisce il numero totale di thread _attivi_ nella regione parallela corrente (se chiamato in regione seriale restituisce 1).
    
- `omp_get_max_threads()`: Restituisce il numero massimo di thread che possono essere creati in una nuova regione parallela.

```c
/* omp-demo1.c */ 
#include <stdio.h> 
#include <omp.h> 

void say_hello( void ) { 
    int my_rank = omp_get_thread_num(); 
    int thread_count = omp_get_num_threads(); 
    printf("Hello from thread %d of %d\n", my_rank, thread_count); 
} 

int main( void ) { 
    #pragma omp parallel 
    say_hello(); 
    return 0; 
}
```

**Impostare il numero di thread da codice:** Invece di usare `OMP_NUM_THREADS`, si può controllare il numero di thread in due modi:

1. **Clausola `num_threads`:** `#pragma omp parallel num_threads(thr)` (dove `thr` può essere un'espressione valutata a runtime, es. un input utente).
    
2. **Funzione API:** `omp_set_num_threads(num);` imposta il numero di thread per le successive regioni parallele.
    

---

## Parallelismo Annidato

È possibile inserire una regione parallela dentro un'altra, ma **di norma è sconsigliato** in quanto raramente utile e spesso dannoso.

- Non è abilitato di default. Per usarlo bisogna impostare la variabile `OMP_NESTED=true`.
- **Problema:** Crea facilmente un numero di thread di gran lunga superiore al numero di core disponibili (_oversubscription_), aumentando enormemente l'overhead di context-switch del Sistema Operativo.

![[6_OpenMP_page_14.png|fix]]

---

## Misurare i Tempi di Esecuzione (Wall-Clock Time)

Per misurare le prestazioni di un programma parallelo, si calcola il tempo trascorso (wall-clock time):

```c
double tstart, tstop; 
tstart = omp_get_wtime(); 

#pragma omp parallel
{ 
    /* block of code to measure */ 
} 
// La barriera implicita assicura che tutti abbiano finito

tstop = omp_get_wtime(); 
printf("Elapsed time: %f\n", tstop - tstart);
```

---

## Scope (Visibilità) delle Variabili

Lo _scope_ di una variabile in OpenMP definisce come i thread vi accedono.

- **Default:** Le variabili dichiarate _prima_ del blocco parallelo sono **condivise** (shared) tra tutti i thread. Le variabili dichiarate _all'interno_ della regione parallela sono **private** per ogni thread.
    
Le clausole applicabili al pragma permettono di alterare il comportamento di default:

| **Clausola**      | **Comportamento**                                                                                                                                                                                                                                                                     |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `shared(x)`       | Tutti i thread accedono e modificano la stessa locazione di memoria. (Default per var. esterne).                                                                                                                                                                                      |
| `private(x)`      | Ogni thread possiede una copia locale (privata) di `x` non inizializzata. Le modifiche fatte nel blocco parallelo vengono perse alla fine della regione parallela.                                                                                                                    |
| `firstprivate(x)` | Come `private`, ma la copia locale viene inizializzata con il valore che `x` possedeva prima di entrare nel blocco parallelo.                                                                                                                                                         |
| `default(none)`   | Annulla il comportamento di default per le variabili esterne, obbligando il programmatore a specificare esplicitamente lo scope (shared, private, etc.) di _tutte_ le variabili usate nel blocco parallelo. **È altamente consigliato per prevenire bug silenziosi o race condition** |

**Esempio di Comportamento dello Scope:**

```c
/* omp-scope.c */ 
#include <stdio.h> 

int main( void ) { 
    int a=1, b=1, c=1, d=1; 
    
    #pragma omp parallel num_threads(10) \ 
            private(a) shared(b) firstprivate(c) 
    { 
        printf("Hello World!\n"); 
        a++; 
        b++; 
        c++; 
        d++; 
    } 
    printf("a=%d\n", a); 
    printf("b=%d\n", b); 
    printf("c=%d\n", c); 
    printf("d=%d\n", d); 
    return 0; 
}
```

**Output e Spiegazione:**

- `a = 1`: Poiché era `private(a)`, gli aggiornamenti locali vanno persi all'uscita dal blocco. Il valore originario è mantenuto.

- `b = ?`: Essendo `shared(b)`, 10 thread hanno incrementato in modo concorrente e non sincronizzato la stessa variabile. C'è una **race condition**, il risultato finale è indefinito.

- `c = 1`: Era `firstprivate(c)`, quindi ogni thread partiva da 1. Ma alla fine del blocco le copie locali vengono distrutte e l'originale non è modificato.

- `d = ?`: Non essendo specificato, di default è `shared(d)`. Come per `b`, si ha una race condition e il risultato è indefinito. L'uso di `default(none)` avrebbe segnalato questo errore a compile-time.


## Comportamento degli Array in OpenMP: Stack vs Heap

Quando si utilizzano le clausole di visibilità (come `firstprivate`), in OpenMP esiste una differenza fondamentale tra il modo in cui vengono gestiti gli array definiti staticamente (allocati sullo stack) e i puntatori ad array allocati dinamicamente (sull'heap).

Osserviamo questo frammento di codice:
```c
int a[3] = {10, 9, 8};
int *b = (int*)malloc(5 * sizeof(*b));

#pragma omp parallel num_threads(3) firstprivate(a, b)
{
    // Corpo della regione parallela
}
````

Ecco cosa succede in memoria per i 3 thread creati:

- **Array sullo stack (`a`)**: OpenMP riconosce la dimensione dell'array statico e crea una **copia privata dell'intera struttura dati** per ogni singolo thread. Ciascun thread (Thread 0, Thread 1, Thread 2) avrà un proprio array indipendente `a`, correttamente inizializzato con i valori originali `{10, 9, 8}`.

- **Array sull'heap tramite puntatore (`b`)**: OpenMP crea una **copia privata del puntatore stesso**, non dei dati a cui punta. Ogni thread riceverà quindi una copia del medesimo indirizzo di memoria (ad esempio `0x0fba`).

**Il problema delle Race Condition:** Come si evince dal caso di `b`, anche se la variabile puntatore è passata come `firstprivate`, tutti i thread conservano lo stesso indirizzo di memoria. Di conseguenza, **i dati sottostanti sull'heap rimangono di fatto condivisi (shared)** tra tutti i thread. Se più thread tentano di modificare gli elementi dell'array `b` contemporaneamente, si verificheranno delle race condition a meno che non si utilizzino costrutti di sincronizzazione.

---

Vediamo ora un esempio pratico di ciò che abbiamo visto fino ad ora.
## Esempio: Il Metodo dei Trapezi

L'obiettivo è calcolare un integrale definito suddividendolo in un numero elevato di intervalli ($n \gg P$, dove $P$ è il numero di thread) e calcolando l'area dei trapezi. L'area di ciascun trapezio può essere calcolata indipendentemente dagli altri.

![[6_OpenMP_page_28.png|fix]]

### Versione 0 (Manuale a Grana Grossa)

In questa versione, implementiamo la suddivisione del lavoro (similmente al pattern [[4_parallel-programming-patterns#Partition|Partition]]) e gestiamo un array condiviso per sommare alla fine i risultati.

```c
/* Version 0: no pragma omp for, manual reduction */
double trap0( double a, double b, long n ) {
    const int thread_count = omp_get_max_threads(); 
    double partial_result[thread_count]; // condiviso tra tutti i thread

    #pragma omp parallel default(none) shared(a, b, n, partial_result)
    {
        const int my_rank = omp_get_thread_num();
        const int thread_count = omp_get_num_threads();
        const double h = (b-a)/n;
        const long local_n_start = ((long)n * my_rank) / thread_count;
        const long local_n_end = ((long)n * (my_rank+1)) / thread_count;
        double x = a + local_n_start * h;
        double local_result = 0.0;
        
        for ( long i = local_n_start; i<local_n_end; i++ ) {
            local_result += h*(f(x) + f(x+h))/2.0;
            x += h;
        }
        partial_result[my_rank] = local_result;
    }
    /* Sincronizzazione implicita */
    
    // Accumulo finale eseguito serialmente
    double result = 0.0;
    for (int i=0; i<thread_count; i++) {
        result += partial_result[i];
    }
    return result;
}
```

**Analisi:** Stiamo facendo una suddivisione statica a blocchi a grana grossa. In questo problema, siccome ogni trapezio ha lo stesso costo computazionale, non c'è grosso sbilanciamento del carico tra i thread. Tuttavia, l'uso dell'array condiviso e del ciclo seriale finale è inefficiente.

### Protezione dalle Race Condition: `atomic` e `critical`

Per sbarazzarci dell'array e accumulare direttamente in una singola variabile `result`, dobbiamo evitare le _race condition_. OpenMP fornisce due direttive per l'accesso in mutua esclusione:

- `#pragma omp critical`: Definisce una sezione critica in cui può entrare **solo un thread alla volta**.
    
- `#pragma omp atomic`: Molto più leggero di `critical`. Permette operazioni base (es. aggiornamenti in formato lettura-modifica-scrittura come `x += expr`) istruendo l'hardware a eseguirle atomicamente, senza i pesanti blocchi logici del sistema operativo. Di solito `atomic` è molto più efficiente.
    

**Attenzione:**

```c
int a = 1; 
#pragma omp parallel 
{ 
    if (a > 0) { 
        #pragma omp critical 
        {
            if ( f() ) { a = a - 1; } 
        }
    } 
}
```

Il primo `if (a > 0)` non è protetto dalla sezione critica, quindi un thread potrebbe leggere un valore mentre un altro lo sta modificando, rendendo il controllo inutile.

### Versione 1 (Uso di atomic)

```c
/* Version 1: no pragma omp for, reduction using atomic construct. */
double trap1( double a, double b, long n ) {
    double result = 0.0;
    #pragma omp parallel default(none) shared(result, a, b, n)
    {
        // ... calcoli di my_rank, start/end, local_result, ecc ...
        double partial_result = 0.0;
        for ( long i = local_n_start; i<local_n_end; i++ ) {
            partial_result += h*(f(x) + f(x+h))/2.0;
            x += h;
        }
        
        #pragma omp atomic
        result += partial_result;
    }
    return result;
}
```

### Versione 2 (La Clausola di Reduction)

L'accumulo di valori parziali è così comune che OpenMP ha una clausola dedicata, `reduction(op:var)`, che automatizza sia le variabili locali sia la loro fusione atomica finale. È molto più efficiente del fare la riduzione manuale con i pragma `atomic`.

Come funziona dietro le quinte:

1. Viene creata una copia privata di `var` per ogni thread.

2. La copia privata viene inizializzata con l'elemento neutro dell'operatore (es. 0 per `+`, 1 per `*`).

3. Alla fine del blocco parallelo, il risultato finale si ottiene applicando l'operatore scelto `op` al valore originale della variabile (prima della regione parallela) e a tutti i valori parziali locali.

![[6_OpenMP_page_38.png|fix]]

![[6_OpenMP_page_39.png|fix]]


```c
/* Version 2: no pragma omp for, "proper" reduction. */
double trap2( double a, double b, long n )
{
    double result = 0.0;
#pragma omp parallel default(none) shared(a, b, n) reduction(+:result)
    {
        const int my_rank = omp_get_thread_num();
        const int thread_count = omp_get_num_threads();
        const double h = (b-a)/n;
        const long local_n_start = ((long)n * my_rank) / thread_count;
        const long local_n_end = ((long)n * (my_rank+1)) / thread_count;
        double x = a + local_n_start * h;

        for ( long i = local_n_start; i<local_n_end; i++ ) {
            result += h*(f(x) + f(x+h))/2.0;
            x += h;
        }
    }
    /* Implicit barrier here */
    return result;
}
```

---

## Direttiva `omp for`

```c
#pragma omp for
```

Si applica immediatamente prima di un ciclo for (all'interno di una regione parallela o combinata come `#pragma omp parallel for`). Distribuisce automaticamente le iterazioni del ciclo for tra i thread del team attivo in quel momento, rimuovendo la necessità di calcolare manualmente indici di inizio e fine.

![[6_OpenMP_page_40.png|fix]]

### Versione 3 (La soluzione finale e più pulita)

```c
/* Version 3: pragma omp for with reduction. */
double trap3( double a, double b, long n ) {
    double result = 0.0;
    const double h = (b-a)/n;
    
    // Apriamo il blocco parallelo e distribuiamo il for in una sola riga
    #pragma omp parallel for default(none) shared(a, b, n, h) reduction(+:result)
    for ( long i = 0; i<n; i++ ) {
        result += h*(f(a+i*h) + f(a+(i+1)*h))/2;
    }
    
    return result;
}
```

**Attenzione:** La direttiva `omp for` impone severi vincoli sulla struttura del ciclo (es. le condizioni di uscita e l'incremento non possono cambiare a runtime, e l'indice del ciclo non deve essere modificato all'interno del corpo).

![[6_OpenMP_page_42.png|fix]]

---

## Loop-Carried Dependencies

Prima di usare `omp for`, bisogna accertarsi che il ciclo non presenti **loop-carried dependencies** (quando il risultato di un'iterazione dipende dall'iterazione precedente).

```c
double factor = 1.0; 
double sum = 0.0; 
for (int k=0; k<n; k++) { 
    sum += factor/(2*k + 1);
    factor = -factor; // DIPENDENZA: dipende dall'iterazione precedente
}
```

Per rimuoverla, si può trasformare `factor` da stato persistente in variabile indipendente e renderla privata per ogni thread:

```c
double factor; 
double sum = 0.0; 
#pragma omp parallel for private(factor) reduction(+:sum) 
for (int k=0; k<n; k++) { 
    if ( k % 2 == 0 ) factor = 1.0; 
    else              factor = -1.0; 
    
    sum += factor/(2*k + 1); 
}
```

_Tip pratico:_ Dichiarando `factor` internamente al ciclo (`double factor = (k%2==0) ? 1.0 : -1.0;`), diventerà automaticamente locale a quel thread.

Non tutte le dipendenze possono essere eliminate. Un test euristico per capire se un ciclo è parallelizzabile è provare ad eseguirlo al contrario: se il risultato non cambia, probabilmente non vi sono dipendenze d'ordine rigoroso.

**Attenzione agli scope impliciti nei cicli annidati:**

```c
int i, j, n, m, temp;
#pragma omp parallel for private(temp) 
for (i=0; i<n; i++) { 
    for (j=0; j<m; j++) { // BUG! j è shared
        temp = b[i]*c[j]; 
        a[i][j] = temp * temp + d[i]; 
    } 
}
```

Mentre la variabile esterna `i` è automaticamente gestita correttamente e resa locale da `omp for`, la variabile interna `j` rimane globale e tutti i thread si pestano i piedi per incrementarla/leggerla. La soluzione è spostare la dichiarazione dentro il ciclo (`for (int j=0; ...)`) oppure dichiarare `private(j)`.

---

## Schedule (Gestione del partizionamento)

Con `schedule(type, chunksize)` si può forzare la modalità in cui OpenMP distribuisce le iterazioni ai thread. Se non specificato, il default è implementation-dependent (GCC tende a usare `static`).

| **Tipo**    | **Descrizione**                                                                                                                                                                                                     |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **static**  | Le iterazioni sono assegnate in blocchi ai thread in modo ciclico pre-determinato. Se `chunksize` non è specificato, le iterazioni vengono divise in blocchi giganti grandi $n_{iterazioni} / n_{threads}$.         |
| **dynamic** | Implementa il pattern [[4_parallel-programming-patterns#Master-Worker\|Master-Worker]]. I thread prelevano dinamicamente chunk di lavoro. Utile per problemi irregolari. Il `chunksize` di default è 1.             |
| **guided**  | Simile a `dynamic`, ma la dimensione del chunk decresce progressivamente man mano che l'esecuzione avanza. Aiuta a ridurre lo sbilanciamento del carico finale e minimizza l'overhead rispetto al dynamic standard. |
| **auto**    | Il compilatore o il run-time decidono la schedulazione ottimale.                                                                                                                                                    |
| **runtime** | La schedulazione non viene cablata nel codice ma ritardata all'esecuzione. Si legge dalla variabile d'ambiente `OMP_SCHEDULE` (es. `OMP_SCHEDULE="static,16" ./my-prog`).                                           |

![[6_OpenMP_page_48.png|fix]]

**Guida alla scelta della clausola schedule:**

|**Clausola**|**Quando usarla**|**Nota sull'Overhead**|
|---|---|---|
|**static**|Il carico di lavoro per iterazione è prevedibile e uniforme.|L'overhead runtime è minimo poiché lo scheduling è quasi del tutto stabilito a tempo di compilazione.|
|**dynamic**|Il carico di lavoro è altamente imprevedibile o variabile (es. set di Mandelbrot).|Richiede molta gestione a runtime, aumentando l'overhead generale. Scegliere bene il chunksize per non penalizzare l'esecuzione.|

Come per i design pattern generali, la scelta della granularità ottima è spesso sperimentale e determina se la soluzione raggiungerà le migliori prestazioni possibili tenendo a bada l'overhead. La teoria per determinare la dimensione ottimale del blocco è in comune con il pattern [[4_parallel-programming-patterns#Partition|Partition]].

---

## Direttiva `collapse`

La clausola `collapse(n)` si aggiunge a un `omp for` per fondere assieme `n` cicli for perfettamente annidati, creando un unico enorme spazio di iterazioni che OpenMP andrà poi a partizionare.

- **Requisito:** "Perfettamente annidati" significa che tra i for coinvolti e al loro interno non devono esserci istruzioni prima o dopo i cicli più interni.

```c
int x, y; 
#pragma omp parallel for collapse(2) 
for ( y = 0; y < ysize; y++ ) { 
    for ( x = 0; x < xsize; x++ ) { 
        drawpixel( x, y ); 
    } 
}
```

L'uso di `collapse` impone automaticamente la privatizzazione di tutte le variabili di controllo dei cicli coinvolti (es. sia `x` che `y` diventano private).

![[6_OpenMP_page_56.png|fix]]

# Appunti: Thread Affinity, Ottimizzazioni e Task in OpenMP

## Mappatura dei Thread ai Core delle CPU (Thread Affinity)
Ottimizzare il posizionamento dei thread sui core fisici è fondamentale per massimizzare le prestazioni.
*   **`OMP_PROC_BIND=true`**: Impedisce al Sistema Operativo di migrare un thread da un core all'altro durante l'esecuzione. 
    *   *Vantaggi:* Migliora l'uso della cache. Quando un thread viene spostato su un nuovo core, la cache di quest'ultimo è "fredda" e deve essere riempita, introducendo latenza.
    *   *Svantaggi:* Priva il SO della flessibilità di riallocare il thread su un core meno carico, limitando il bilanciamento dinamico.
*   **`OMP_PLACES`**: Variabile d'ambiente che permette di specificare esattamente su quali core della CPU devono essere "fissati" i thread.
    *   `OMP_PLACES="0,1,2,3"`: Mappa il thread 0 sul core 0, il thread 1 sul core 1, ecc. Se si lanciano 5 thread, il quinto tornerà in modo ciclico sul core 0.
    *   `OMP_PLACES="0:4"`: Sintassi compatta che viene espansa in `{0}, {1}, {2}, {3}`.
    *   `OMP_PLACES="0:8:2"`: Significa "parti dal core 0, prendi 8 core in totale, saltando di 2 in 2". Viene espanso in `{0}, {2}, {4}, {6}, {8}, {10}, {12}, {14}`.
*   **`OMP_DISPLAY_ENV=true`**: Utile per il debug, stampa a schermo l'ambiente OpenMP e mostra come `OMP_PLACES` viene effettivamente espanso dal sistema.

Questa mappatura esplicita ha molto senso nelle moderne architetture ibride per decidere se usare solo i P-Core (Performance core), se evitare l'Hyper-Threading, o se usare gli E-Core (Efficiency core) solo come ultima risorsa.

![[6_OpenMP_page_62.png|fix]]

---

## P-Core vs E-Core (Esempio: Transposed Matrix-Matrix Multiply)
Consideriamo un processore Intel i9-12900F con 8 P-Core (con Hyper-Threading, quindi 16 core logici) e 8 E-Core (senza HT). Vogliamo moltiplicare due matrici $2048 \times 2048$.

Ecco le politiche di mappatura dei thread testate:
![[6_OpenMP_page_65.png]]

Osserviamo il Wall-Clock Time risultante:
![[6_OpenMP_page_66.png|fix]]

**Analisi del grafico:**
*   Fino a 8 thread (dove vengono impiegati solo i P-Core fisici in tutti gli scenari), i tempi delle varie politiche di scheduling sono identici.
*   Superati gli 8 thread, le prestazioni divergono. Il caso peggiore è lo scheduling statico che, esauriti i P-Core fisici, assegna i task ai thread virtuali (Hyper-Threading) prima di usare gli E-Core fisici.
*   *Sbilanciamento Architetturale:* Lo sbilanciamento del carico qui non è dovuto a un codice scritto male, ma all'hardware (gli E-Core sono molto più lenti dei P-Core).
*   *Soluzione:* In queste architetture eterogenee, lo **scheduling `dynamic`** risulta il migliore, poiché i P-Core smaltiranno i loro chunk molto più velocemente degli E-Core e richiederanno dinamicamente nuovo lavoro, ribilanciando l'esecuzione.

---

## Esempio: Odd-Even Transposition Sort
Variante parallelizzabile del Bubble Sort (costo $O(n^2)$). Esegue sequenzialmente delle fasi per scambiare elementi adiacenti fuori ordine. Nel caso peggiore necessita di $n$ fasi:
*   **Fasi Pari:** Si confrontano/scambiano gli indici pari con i successivi dispari.
*   **Fasi Dispari:** Si confrontano/scambiano gli indici dispari con i successivi pari.

**Prima implementazione OpenMP:**
```c
for (int phase = 0; phase < n; phase++) { 
    if ( phase % 2 == 0 ) { 
        #pragma omp parallel for default(none) shared(v,n) 
        for (int i=0; i<n-1; i+=2) { 
            if (v[i] > v[i+1]) swap( &v[i], &v[i+1] ); 
        } 
    } else { 
        #pragma omp parallel for default(none) shared(v,n) 
        for (int i=1; i<n-1; i+=2) { 
            if (v[i] > v[i+1]) swap( &v[i], &v[i+1] ); 
        } 
    } 
}
```
*Problemi e Considerazioni:*
*   Non si può parallelizzare il ciclo esterno (`phase`) altrimenti fasi pari e dispari si sovrapporrebbero causando data race sugli scambi.
*   Gli scambi dentro la singola fase sono sicuri e parallelizzabili. Non serve uno scheduler dinamico, il carico è perfettamente bilanciato.
*   **Overhead:** In questo codice, ad ogni iterazione del ciclo `phase`, il costrutto `#pragma omp parallel for` **crea e distrugge un team di thread**. Per $n$ grande, questo overhead distrugge le prestazioni.

**Soluzione Ottimizzata (Team di thread riutilizzato):**
```c
#pragma omp parallel default(none) shared(v,n) 
{
    for (int phase = 0; phase < n; phase++) { 
        if ( phase % 2 == 0 ) { 
            #pragma omp for 
            for (int i=0; i<n-1; i+=2) { 
                if (v[i] > v[i+1]) swap( &v[i], &v[i+1] ); 
            } 
        } else { 
            #pragma omp for 
            for (int i=1; i<n-1; i+=2 ) { 
                if (v[i] > v[i+1]) swap( &v[i], &v[i+1] ); 
            } 
        } 
    }
}
```
Si apre la regione parallela all'esterno del ciclo `phase`. I thread vengono creati una sola volta. Il `#pragma omp for` assegna il lavoro e, terminato il ciclo interno, invoca una **barriera implicita** che mantiene la sincronizzazione perfetta prima della fase successiva.

![[6_OpenMP_page_74.png|fix]]

---

## Direttive di Sincronizzazione

| Direttiva             | Comportamento                                                                                                                              | Barriera Implicita alla Fine? |
| :-------------------- | :----------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------- |
| `#pragma omp barrier` | **Barriera esplicita.** Tutti i thread devono raggiungere questo punto prima di poter proseguire l'esecuzione del codice successivo.       | -                             |
| `#pragma omp master`  | Il blocco viene eseguito **solo** dal master thread (ID 0). Gli altri thread saltano il blocco e proseguono.                               | **NO**.                       |
| `#pragma omp single`  | Il blocco viene eseguito **da un solo thread** (il primo che lo raggiunge). Gli altri thread saltano il blocco ma **attendono alla fine**. | **SÌ**.                       |

![[6_OpenMP_page_77.png|fix]]

---

## Gestione Esplicita dei Task (Task Parallelism)
Fino ad ora abbiamo visto il *Data Parallelism* (`omp for`), che richiede di conoscere il numero di iterazioni a priori. Per cicli irregolari (es. esplorazione di liste concatenate o alberi) OpenMP introduce il **Task Parallelism**.

Un task è un'unità indipendente di lavoro composta da codice e dati. Viene quindi usato il paradigma [[4_parallel-programming-patterns#Master-Worker|Master-Worker]] e i thread del team li eseguono dinamicamente quando sono liberi.
*   Il momento in cui un task viene **creato** è separato  al momento in cui viene **eseguito**.
*   I task possono essere annidati (un task può creare altri task).
*   L'ordine di esecuzione dei task non è garantito (un task creato dopo può essere eseguito prima).

**Creazione sicura dei Task:**
```c
#pragma omp parallel 
{ 
    #pragma omp single // FONDAMENTALE!
    { 
        #pragma omp task 
        fred(); 
        #pragma omp task 
        barney(); 
    } 
}
```
È cruciale usare `#pragma omp single` (o `master`). Se non lo si facesse, *ogni* thread della regione parallela creerebbe copie multiple degli stessi task, facendo esplodere il numero di operazioni.

### Scope delle Variabili nei Task
Poiché il task viene eseguito in differita, lo scope delle variabili cambia rispetto alle normali regioni parallele per proteggere i dati:

| Scope alla creazione | Scope dentro il task | Comportamento                                                                                                               |
| :------------------- | :------------------- | :-------------------------------------------------------------------------------------------------------------------------- |
| `shared`             | `shared`             | Il task legge/scrive l'indirizzo originale condiviso.                                                                       |
| `private`            | `firstprivate`       | Il task si salva una **copia privata inizializzata** col valore che la variabile aveva al momento della creazione del task. |
| `firstprivate`       | `firstprivate`       | Funziona come sopra.                                                                                                        |

Vediamo un esempio per capire come si comportano le variabili all'interno di un task:

```c
int a = 1; 

void foo( void ) { 
    int b = 2, c = 3; 
    
    #pragma omp parallel private(b) 
    { 
        int d = 4; // a, c shared; b private, d private 
        
        #pragma omp task 
        { 
            int e = 5; 
            // Scope di a: shared, value 1 
            // Scope di b: firstprivate, uninitialized 
            // Scope di c: shared, value 3 
            // Scope di d: private => firstprivate, value 4 
            // Scope di e: private, value 5 
        } 
    } 
}
```

**Attenzione ai valori indefiniti:** Come si nota in questo esempio, `b` ha un valore iniziale indefinito mentre `d` no. Questo accade perché `b` è stata dichiarata `private` all'ingresso della regione parallela ma non è mai stata inizializzata dal thread prima della creazione del task. Di conseguenza, all'interno del task avremo i seguenti valori:

`a = 1`, `b = ?`, `c = 3`, `d = 4`, `e = 5`.

---

### Esempio: Attraversamento Lista 

**Sbagliato:**
```c
ListNode *p; // Condiviso di default fuori
#pragma omp parallel 
{ 
    #pragma omp single 
    { 
        p = head; 
        while (p) { 
            #pragma omp task 
            process(p);   // ERRORE: p è shared!
            
            p = p->next; 
        } 
    } 
}
```
Essendo `p` condiviso, quando il task viene finalmente eseguito nel futuro, il ciclo `while` principale sarà già andato avanti modificando `p`. Molti task si ritroverebbero a processare nodi avanzati saltando i precedenti.

**Corretto:**
```c
#pragma omp parallel 
{ 
    #pragma omp single 
    { 
        ListNode *p = head; // p è locale (private) al thread che esegue il single
        while (p) { 
            #pragma omp task 
            process(p); // Ora p diventa automaticamente firstprivate nel task!
            
            p = p->next; 
        } 
    } 
}
```
Dichiarando `p` localmente prima del `while`, `p` diventa `private` per il thread creatore. Secondo le regole di OpenMP, le variabili private alla creazione diventano `firstprivate` per i task. Ogni task "fotografa" e conserva il valore esatto di `p` in quel ciclo.

---

### `#pragma omp taskwait` e Ricorsione (Fibonacci)
`#pragma omp taskwait` sospende l'esecuzione del thread corrente finché tutti i task figli da esso generati non terminano. È indispensabile negli algoritmi ricorsivi (divide et impera).

```c
int fib( int n ) { 
    int n1, n2; 
    if (n < 2) return 1; 
    
    #pragma omp task shared(n1) 
    n1 = fib(n-1); 
    
    #pragma omp task shared(n2) 
    n2 = fib(n-2); 
    
    #pragma omp taskwait // Aspetta che n1 ed n2 siano stati calcolati
    
    return n1 + n2; 
} 

int main() { 
    int n = 10, res; 
    #pragma omp parallel 
    {
        #pragma omp master // Solo il master avvia l'albero ricorsivo
        res = fib(n); 
    }
    printf("fib(%d)=%d\n", n, res); 
    return 0; 
}
```
Le variabili `n1` e `n2` sono locali alla funzione e quindi intrinsecamente private. Se non mettessimo `shared(n1)`, il task aggiornerebbe una sua copia `firstprivate` locale e il thread padre non vedrebbe mai il risultato. Specificando `shared`, il task figlio scrive il risultato direttamente nella variabile `n1` del chiamante. Il `taskwait` assicura poi che la somma `n1 + n2` avvenga solo dopo che i risultati siano pronti.