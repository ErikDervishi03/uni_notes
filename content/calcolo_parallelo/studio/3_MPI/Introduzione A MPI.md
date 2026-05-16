**MPI (Message Passing Interface)** non è un nuovo linguaggio di programmazione, ma una libreria standardizzata basata sul paradigma a **scambio di messaggi**. Viene tipicamente utilizzata con C, C++ e Fortran per sviluppare applicazioni parallele ad alte prestazioni.

- **Paradigma SPMD**: I programmi MPI seguono il modello **Single Program Multiple Data**. Lo stesso codice sorgente viene compilato e l'eseguibile risultante viene lanciato in copie multiple. Ogni copia (processo) lavora su una porzione diversa di dati.
- **Memoria Distribuita**: I processi hanno spazi di memoria completamente separati.
    - _Pro_: Non esistono le race condition sui dati in memoria (ogni processo vede solo le sue variabili).
    - _Contro_: Possono verificarsi errori di comunicazione (come deadlock) e la condivisione delle informazioni richiede l'invio esplicito di messaggi.

---

## Schema Base di un Programma MPI

Tutti i processi eseguono lo stesso codice dall'inizio alla fine. Il flusso viene differenziato controllando l'identificativo del processo (chiamato **rank**).

```c
#include <mpi.h>

int main(int argc, char *argv[]) {
    // 1. Inizializzazione obbligatoria
    MPI_Init(&argc, &argv); 
    
    // Codice eseguito da tutti i processi
    foo(); 
    
    // Differenziazione del lavoro in base al rank
    if ( my_rank == 0 ) { 
        do_something();      // eseguito SOLO dal processo 0 (spesso il "master")
    } else { 
        do_something_else(); // eseguito da tutti gli altri processi ("worker")
    } 
    
    // 2. Chiusura obbligatoria
    MPI_Finalize(); 
    return 0;
}
```

---
## Categorie di Funzioni nella Libreria MPI

La libreria MPI offre diverse tipologie di funzioni:

1. **Comunicazione**:
    - _Point-to-point (Pairwise)_: Comunicazione diretta tra due specifici processi (mittente e destinatario,
    - _Collettiva (Collectives)_: Comunicazione che coinvolge tutti i processi di un gruppo simultaneamente (es. broadcast).
2. **Sincronizzazione**:
    - Principalmente gestita tramite _barriere_ (tutti i processi si fermano finché l'ultimo non è arrivato).
3. **Interrogazione del Sistema**:
    - Domande sull'ambiente di esecuzione: "Quanti processi ci sono in totale?", "Qual è il mio ID?", "Ci sono messaggi in attesa?".

---
## I Comunicatori (Communicators)

In MPI, è possibile raggruppare i processi in sotto-gruppi detti **comunicatori**. Le operazioni collettive avvengono solo all'interno del comunicatore specificato.

Il comunicatore di default, che utilizzeremo sempre, è **`MPI_COMM_WORLD`**. Questo comunicatore rappresenta l'insieme totale di tutti i processi lanciati dall'utente all'avvio del programma.

---
## Le 7 Funzioni Fondamentali di MPI

Con queste funzioni principali è possibile costruire quasi ogni programma MPI:

1. `MPI_Init`: Inizializza l'ambiente MPI.
2. `MPI_Finalize`: Termina l'ambiente MPI e rilascia le risorse.
3. `MPI_Comm_size`: Interroga il sistema per sapere _quanti_ processi totali sono in esecuzione nel comunicatore.
4. `MPI_Comm_rank`: Interroga il sistema per sapere _chi sono io_ (il mio ID/rank all'interno del comunicatore).
5. `MPI_Send`: Invia un messaggio (modalità bloccante).
6. `MPI_Recv`: Riceve un messaggio (modalità bloccante).
7. `MPI_Abort`: Termina forzatamente l'intera computazione di tutti i processi (utile per la gestione degli errori fatali).

---
## Esempio 1: Hello World in MPI

Questo programma stampa un saluto da ogni processo lanciato, mostrando come recuperare le informazioni base.

```c
/* mpi-hello.c */
#include <stdio.h> 
#include <mpi.h>

int main( int argc, char *argv[] ) { 
    int rank, size, len; 
    char hostname[MPI_MAX_PROCESSOR_NAME]; 
    
    MPI_Init( &argc, &argv );

    // Interrogazioni
    MPI_Comm_rank( MPI_COMM_WORLD, &rank );
    MPI_Comm_size( MPI_COMM_WORLD, &size ); 
    MPI_Get_processor_name( hostname, &len );

    printf("Greetings from process %d of %d running on %s\n", rank, size, hostname);

    MPI_Finalize();
    return 0;
}
```

---
## Esempio 2: Point-to-Point Communication

Il programma base per lo scambio di messaggi: il processo 0 invia un numero al processo 1.

**Per compilare ed eseguire:**

```bash
mpicc mpi-point-to-point.c -o mpi-point-to-point  
mpirun -n 2 mpi-point-to-point  
```

**Codice:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>  

int main( int argc, char *argv[]) {
    int my_rank, comm_size, buf;
    MPI_Status status;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &my_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &comm_size);  
    
    // Gestione errori delegata al processo 0 per evitare stampe multiple
    if (0 == my_rank && comm_size < 2) {
        fprintf(stderr, "FATAL: you must run at least 2 processes\n");
        MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
    }  
    
    // Logica Point-to-Point
    if (my_rank == 0) {
        buf = 123456;
        MPI_Send( 
            &buf,           /* send buffer: puntatore al dato da spedire */
            1,              /* count: numero di elementi */
            MPI_INT,        /* datatype: tipo del dato */
            1,              /* destination: rank del destinatario */
            0,              /* tag: etichetta del messaggio */
            MPI_COMM_WORLD  /* communicator */
        );
    }
    else if (my_rank == 1) {
        MPI_Recv( 
            &buf,           /* receive buffer: dove salvare il dato */
            1,              /* count: MAX numero di elementi accettabili */
            MPI_INT,        /* datatype: tipo del dato */
            0,              /* source: rank del mittente */
            0,              /* tag: etichetta attesa */
            MPI_COMM_WORLD, /* communicator */
            &status         /* status: info extra sulla ricezione */
        );
        printf( "Received %d\n", buf );
    }  
    
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

---
## Dettagli sulla Comunicazione in MPI

### I Dati e i Datatypes

Ogni dato inviato/ricevuto in MPI è descritto da una tripla: `(address, count, datatype)`.

- **address**: Il puntatore alla locazione di memoria da cui leggere (nella Send) o dove scrivere (nella Recv).
- **count**: Il numero esatto di elementi da spedire, o il _massimo_ consentito in ricezione (NON è la dimensione in byte).
- **datatype**: Identifica la natura del dato per garantire la compatibilità, anche in esecuzioni su hardware misto (es. tra macchine a 32 e 64 bit).

Tipi base comuni: `MPI_CHAR`, `MPI_INT`, `MPI_FLOAT`, `MPI_DOUBLE`, `MPI_BYTE`.

### Il Tag del Messaggio

I messaggi viaggiano con un'etichetta intera chiamata **tag**. Serve al processo destinatario per distinguere messaggi logici diversi provenienti dallo stesso mittente.

- Spesso useremo `0` di default per inviare.
- In ricezione, si può indicare il tag specifico atteso, oppure usare il jolly `MPI_ANY_TAG` per accettare il primo messaggio utile indipendentemente dal suo tag.

### MPI_Send (Blocking Send)

```c 
int MPI_Send(const void *buf, int count, MPI_Datatype datatype, int dest, int tag, MPI_Comm comm)
```

- Invia il messaggio al processo `dest`. Se `dest` è `MPI_PROC_NULL`, la funzione si chiude subito senza fare nulla.
- **Significato di "Bloccante"**: Quando la `MPI_Send` ritorna (termina l'esecuzione), significa unicamente che **i dati sono stati estratti in modo sicuro da `buf`**. Dopo la return, il programmatore è libero di modificare, sovrascrivere o deallocare l'array `buf` senza corrompere la trasmissione.
- _Attenzione_: NON significa affatto che il messaggio sia già giunto al destinatario (potrebbe essere in buffering di rete).
    

### MPI_Recv (Blocking Receive)

```c
int MPI_Recv(void *buf, int count, MPI_Datatype datatype, int source, int tag, MPI_Comm comm, MPI_Status *status)
```

- **Bloccante**: Sospende l'esecuzione del processo finché non viene ricevuto un messaggio con il corretto `source` e `tag`. Una volta sbloccata, i dati dentro `buf` sono pronti per essere utilizzati.
- Il parametro `source` può essere un rank specifico o il jolly `MPI_ANY_SOURCE`. Se è `MPI_PROC_NULL`, la funzione ritorna subito senza far nulla.
- Il parametro `count` rappresenta la **dimensione massima** del buffer fornito.

    - Se il messaggio in arrivo ha meno elementi, va bene (il resto del buffer non viene toccato).
    - Se il messaggio in arrivo ha _più_ elementi di `count`, si verifica un errore grave a runtime (buffer overflow evitato da MPI interrompendo il programma).
    
- L'argomento `status` salva metadati sul messaggio (es. per scoprire il reale mittente se si è usato `ANY_SOURCE`, o la dimensione esatta). Se non interessa, si usa la macro `MPI_STATUS_IGNORE`.

_Nota:_ Assumiamo sempre che l'hardware sottostante e la libreria MPI siano affidabili. Se i parametri sono corretti, la spedizione e ricezione non falliscono "silenziosamente" come potrebbe accadere con chiamate socket grezze, la comunicazione andrà sempre a buon fine.

#### Gestione degli Errori e MPI_Status

Quando si utilizza `MPI_Recv`, il parametro `count` specifica la dimensione *massima* del buffer di ricezione. 
* Se il messaggio in arrivo ha un numero di elementi **minore o uguale** a `count`, la ricezione ha successo.
* Se il messaggio contiene **più elementi** di `count`, si verifica un errore di tipo `MPI_ERR_TRUNCATE` (buffer overflow), e il programma tipicamente si interrompe.

### La struttura `MPI_Status`
Quando si riceve un messaggio (specialmente usando `MPI_ANY_SOURCE` o `MPI_ANY_TAG`), la funzione popola una struttura `MPI_Status` per fornire metadati sul messaggio appena arrivato. In C, contiene (tra gli altri) questi campi:
* `status.MPI_SOURCE`: Il rank effettivo del processo mittente.
* `status.MPI_TAG`: Il tag effettivo del messaggio ricevuto.
* `status.MPI_ERROR`: Il codice di errore associato alla ricezione.

### `MPI_Get_count`
Poiché `MPI_Recv` accetta fino a `count` elementi, per sapere **esattamente quanti elementi sono stati effettivamente ricevuti**, si usa:
```c
int MPI_Get_count(const MPI_Status *status, MPI_Datatype datatype, int *count);
```
Questa funzione ispeziona lo `status` e salva nella variabile puntata da `count` il numero reale di elementi ricevuti.

---

## Il Problema del Deadlock e le Soluzioni

Osserva questo scenario di comunicazione circolare:
* **Processo 0:** Chiama `MPI_Send` verso il Processo 1, poi `MPI_Recv` dal Processo 1.
* **Processo 1:** Chiama `MPI_Send` verso il Processo 0, poi `MPI_Recv` dal Processo 0.

**Cosa succede?** Spesso MPI utilizza dei buffer di sistema nascosti. Se i messaggi sono piccoli, vengono copiati nei buffer di rete, la `MPI_Send` si sblocca, e tutto funziona. **Tuttavia**, se i messaggi sono molto grandi e riempiono i buffer, le `MPI_Send` di entrambi i processi si bloccano in attesa che l'altro processo chiami la `MPI_Recv` per svuotarli. Poiché entrambi sono bloccati sulla `Send`, nessuno chiamerà mai la `Recv`: questo è un **Deadlock**.

**Soluzioni al Deadlock:**
1.  **Inversione delle chiamate:** Un processo fa prima la Send e poi la Recv; l'altro fa prima la Recv e poi la Send. (Richiede logica asimmetrica).
2.  **Uso di `MPI_Sendrecv`:** Una funzione sicura fornita da MPI che esegue invio e ricezione simultaneamente, garantendo l'assenza di deadlock.
3.  **Comunicazione Non Bloccante:** (Migliore approccio prestazionale) Usare le versioni asincrone delle funzioni di invio e ricezione.

---

## Comunicazione Non Bloccante (Asincrona)
Le operazioni non bloccanti permettono di sovrapporre la comunicazione con il calcolo utile (hiding communication latency). 

### `MPI_Isend` e `MPI_Irecv` 
* `int MPI_Isend(..., MPI_Request *req)`
* `int MPI_Irecv(..., MPI_Request *req)`

A differenza delle versioni bloccanti, queste funzioni **ritornano immediatamente**, prima ancora che il messaggio sia stato trasferito o che il buffer sia sicuro da usare. Forniscono un *handle* di tipo `MPI_Request` che identifica in modo univoco quell'operazione in background.

**Attenzione:** Dopo aver chiamato una funzione asincrona, **NON devi modificare/leggere il buffer** finché non verifichi che l'operazione sia effettivamente conclusa tramite le funzioni di attesa:

### `MPI_Wait` e `MPI_Test`
* `int MPI_Wait(MPI_Request *request, MPI_Status *status)`: **Blocca** l'esecuzione finché l'operazione asincrona identificata da `request` non è completata.
* `int MPI_Test(MPI_Request *request, int *flag, MPI_Status *status)`: **Non blocca**. Controlla istantaneamente se l'operazione è completata (scrive `1` in `flag`) o meno (`0` in `flag`). Permette al processo di fare altro mentre aspetta.

*Esistono anche le varianti per attendere array di richieste: `Waitall`, `Waitany`, `Waitsome`.*

### Esempio: Send Asincrona (Sovrapposizione calcolo-comunicazione)
```c
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>  

void big_computation( void ) {
    printf("Some big computation while message travels...\n");
}  

int main( int argc, char *argv[]) {
    int rank, size, buf;
    MPI_Status status;
    MPI_Request req;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank( MPI_COMM_WORLD, &rank );
    MPI_Comm_size( MPI_COMM_WORLD, &size );  
    
    if ( size < 2 ) {
        fprintf(stderr, "FATAL: you must run at least 2 processes\n");
        MPI_Abort(MPI_COMM_WORLD, EXIT_FAILURE);
    }  
    
    if (rank == 0) {
        buf = 123456;
        // La Isend ritorna subito, la comunicazione avviene in background
        MPI_Isend( &buf, 1, MPI_INT, 1, 0, MPI_COMM_WORLD, &req);
        
        // Faccio lavoro utile senza aspettare!
        big_computation();
        
        // Ora mi assicuro che l'invio sia concluso prima di uscire o modificare 'buf'
        MPI_Wait(&req, &status);
        printf("Master terminates\n");
    } else if (rank == 1) {
        // Il worker può usare una recv bloccante standard per aspettare il dato
        MPI_Recv( &buf, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, &status );
        printf("Received %d\n", buf);
    }  
    
    MPI_Finalize();
    return EXIT_SUCCESS;
}
```

---

## Interruzione dell'esecuzione: `MPI_Abort`
Per interrompere un programma MPI a causa di un errore grave, NON usare `exit()` o `abort()` del C standard, perché non ripuliscono lo stato dei demoni MPI distribuiti.
Usa:
```c
int MPI_Abort(MPI_Comm comm, int errorcode);
```
Questa funzione termina "graziosamente" tutti i processi MPI in esecuzione nel comunicatore specificato e restituisce l'errorcode al sistema operativo.

---

## Esempio 1: Integrazione con Metodo dei Trapezi (MPI)
Vogliamo parallelizzare il calcolo dell'area sottesa da una curva, suddividendo l'intervallo $[a, b]$ in $n$ trapezi. 
*Approccio SPMD:* Ogni processo calcola quanti e quali trapezi gli spettano in base al suo rank, calcola l'area locale e la invia al Master (rank 0) che somma i risultati parziali.

```c
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>  

// Funzione da integrare
double f( double x ) {
    return 4.0/(1.0 + x*x);
}  

// Calcolo dell'area locale
double trap( int my_rank, int comm_sz, double a, double b, int n ) {
    const double h = (b-a)/n;
    // Partizionamento del dominio a blocchi (grana grossa)
    const int local_n_start = (n * my_rank) / comm_sz;
    const int local_n_end = (n * (my_rank + 1)) / comm_sz;
    
    double x = a + local_n_start * h;
    double my_result = 0.0;  
    
    for (int i = local_n_start; i < local_n_end; i++) {
        my_result += h*(f(x) + f(x+h))/2.0;
        x += h;
    }
    return my_result;
}  

int main( int argc, char* argv[] ) {
    double a = 0.0, b = 1.0, partial_result, result = 0.0;
    int n = 1000000;
    int my_rank, comm_sz;  
    
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &my_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &comm_sz);  
    
    // Tutti i nodi leggono da riga di comando autonomamente
    if ( argc == 4 ) {
        a = atof(argv[1]);
        b = atof(argv[2]);
        n = atoi(argv[3]);
    }  
    
    // Tutti i processi calcolano il loro risultato locale in parallelo
    partial_result = trap( my_rank, comm_sz, a, b, n );  
    
    if ( my_rank != 0 ) {
        // I worker inviano il risultato parziale al master
        MPI_Send(&partial_result, 1, MPI_DOUBLE, 0, 0, MPI_COMM_WORLD);
    } else {
        // Il Master raccoglie e accumula
        result = partial_result;
        for ( int p=1; p<comm_sz; p++ ) {
            // Usa MPI_ANY_SOURCE per evitare deadlock nel caso i messaggi 
            // arrivino in disordine e riempiano i buffer di sistema!
            MPI_Recv(&partial_result, 1, MPI_DOUBLE, MPI_ANY_SOURCE, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            result += partial_result;
        }
        printf("Area: %f\n", result);
    }  
    
    MPI_Finalize();  
    return EXIT_SUCCESS;
}
```

---

## Primitive di Comunicazione Collettiva
Le applicazioni MPI utilizzano spesso il pattern **Bulk Synchronous**: i processi si alternano tra fasi di pura computazione locale e fasi di comunicazione collettiva in cui aggiornano lo stato globale.

*Regola d'oro:* Tutte le funzioni collettive **devono essere chiamate da TUTTI i processi** del comunicatore, altrimenti il programma andrà in deadlock irreversibile.

| Funzione | Descrizione |
| :--- | :--- |
| `MPI_Barrier` | È l'unica primitiva di pura sincronizzazione. Quando un processo chiama la Barrier, si ferma. Riparte solo quando *tutti* i processi del comunicatore hanno raggiunto la stessa barriera. |
| `MPI_Bcast` | **Broadcast**: Un processo (root) prende i dati dal suo buffer e li invia identici a tutti gli altri processi nel comunicatore. Tutti usano la stessa chiamata di funzione. |
| `MPI_Scatter` | Il root prende un array e lo **sparpaglia**, distribuendolo a pezzi di uguale dimensione (*chunk*) in ordine tra i processi (es. l'elemento 0 al proc 0, l'elemento 1 al proc 1...). |
| `MPI_Gather` | L'opposto della Scatter. Raccoglie i chunk dai processi e li concatena in ordine nell'array del processo root. |
| `MPI_Allgather` | È come una Gather seguita da un Broadcast. Raccoglie i chunk da tutti, ma distribuisce l'array finale concatenato a *tutti* i processi, non solo al root. |

---

## Esempio 2: Somma di Vettori (Distribuzione Semplice)
Il processo 0 genera due vettori, li distribuisce, i nodi fanno la somma pezzo per pezzo, e i risultati vengono raccolti dal processo 0.

**Limitazione Importante:** Usando `MPI_Scatter` / `MPI_Gather` ordinarie, assumiamo che la dimensione totale del vettore $N$ sia **esattamente un multiplo** del numero di processi $P$. Se c'è del resto, il programma fallirà o richiederà gestione manuale.

```c
#include <stdio.h>
#include <stdlib.h>
#include <math.h> 
#include <mpi.h>  

// Computazione locale
void sum( const double *x, const double *y, double *z, int n ) {
    for (int i=0; i<n; i++) z[i] = x[i] + y[i];
}  

int main( int argc, char *argv[] ) {
    double *x = NULL, *local_x = NULL;
    double *y = NULL, *local_y = NULL;
    double *z = NULL, *local_z = NULL;
    int n = 1000, local_n, my_rank, comm_sz;  
    
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &my_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &comm_sz);  
    if ( argc == 2 ) n = atoi(argv[1]);  
    
    // Controlla che N sia un multiplo esatto di P
    if ( (my_rank == 0) && (n % comm_sz) ) {
        fprintf(stderr, "FATAL: vector length must be multiple of comm size\n");
        MPI_Abort( MPI_COMM_WORLD, EXIT_FAILURE );
    }  
    
    local_n = n / comm_sz; // Dimensione del chunk
    
    if ( my_rank == 0 ) {
        // Solo il master alloca i vettori completi
        x = (double*)malloc( n * sizeof(double) );
        y = (double*)malloc( n * sizeof(double) );
        z = (double*)malloc( n * sizeof(double) );
        for ( int i=0; i<n; i++ ) { x[i] = i; y[i] = n-1-i; }
    }  
    
    // Tutti (compreso il master) allocano lo spazio per la loro parte locale
    local_x = (double*)malloc( local_n * sizeof(double) );
    local_y = (double*)malloc( local_n * sizeof(double) );
    local_z = (double*)malloc( local_n * sizeof(double) );  
    
    // Sparpaglia x
    MPI_Scatter(x, local_n, MPI_DOUBLE, local_x, local_n, MPI_DOUBLE, 0, MPI_COMM_WORLD);  
    // Sparpaglia y
    MPI_Scatter(y, local_n, MPI_DOUBLE, local_y, local_n, MPI_DOUBLE, 0, MPI_COMM_WORLD);  
    
    // Calcolo parallelo distribuito
    sum( local_x, local_y, local_z, local_n );  
    
    // Raccoglie i risultati parziali ricostruendo l'array z nel master
    MPI_Gather(local_z, local_n, MPI_DOUBLE, z, local_n, MPI_DOUBLE, 0, MPI_COMM_WORLD);  
    
    if ( my_rank == 0 ) {
        printf("Test OK\n");
        free(x); free(y); free(z);
    }  
    free(local_x); free(local_y); free(local_z);  
    
    MPI_Finalize();  
    return EXIT_SUCCESS;
}
```

---

## Distribuzione Irregolare: `MPI_Scatterv` e `MPI_Gatherv`
Quando la dimensione dei dati da dividere **non è un multiplo** esatto dei processi (o si vogliono distribuire pezzi di dimensioni differenti), la Scatter ordinaria non basta. Serve la versione V (Vector): `MPI_Scatterv`.

Queste funzioni non prendono più un singolo intero `count`, ma richiedono **due array** aggiuntivi (di dimensione pari al numero di processi):
1.  **`sendcounts[]`**: Specifica *quanti* elementi mandare ad ogni specifico processo.
2.  **`displs[]`** (Displacements): Specifica l'*indice di partenza* nell'array origine da cui iniziare a pescare i dati per quello specifico processo. Questo permette persino di lasciare buchi logici nei dati!

---

## Esempio 3: Somma di Vettori (Distribuzione Avanzata Irregolare)

Questo codice gestisce qualsiasi dimensione di vettore, spalmando il resto in modo equo.

```c
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <mpi.h>  

void sum( const double *x, const double *y, double *z, int n ) {
    for (int i=0; i<n; i++) z[i] = x[i] + y[i];
}  

int main( int argc, char *argv[] ) {
    double *x=NULL, *local_x=NULL, *y=NULL, *local_y=NULL, *z=NULL, *local_z=NULL;
    int *counts, *displs; // Vettori per Scatterv/Gatherv
    int n = 1000, local_n, my_rank, comm_sz;  
    
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &my_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &comm_sz);  
    if ( argc == 2 ) n = atoi(argv[1]);  
    
    // Tutti i processi allocano e calcolano come verranno divisi i dati
    counts = (int*)malloc( comm_sz * sizeof(int) );
    displs = (int*)malloc( comm_sz * sizeof(int) );
    
    for (int i=0; i<comm_sz; i++) {
        // Formule standard per mappare un dominio 1D equamente
        const int start = (n * i) / comm_sz;
        const int end = (n * (i+1)) / comm_sz;
        counts[i] = end - start;
        displs[i] = start;
    }  
    
    // La porzione di questo specifico processo
    local_n = counts[my_rank];  
    
    if ( my_rank == 0 ) {
        x = (double*)malloc( n * sizeof(double) );
        y = (double*)malloc( n * sizeof(double) );
        z = (double*)malloc( n * sizeof(double) );
        for ( int i=0; i<n; i++ ) { x[i] = i; y[i] = n-1-i; }
    }  
    
    // Alloca array locali di dimensione dinamica
    local_x = (double*)malloc( local_n * sizeof(double) );
    local_y = (double*)malloc( local_n * sizeof(double) );
    local_z = (double*)malloc( local_n * sizeof(double) );  
    
    // Scatterv per x
    MPI_Scatterv(x, counts, displs, MPI_DOUBLE, 
                 local_x, local_n, MPI_DOUBLE, 0, MPI_COMM_WORLD);  
    // Scatterv per y
    MPI_Scatterv(y, counts, displs, MPI_DOUBLE, 
                 local_y, local_n, MPI_DOUBLE, 0, MPI_COMM_WORLD);  
    
    sum( local_x, local_y, local_z, local_n );  
    
    // Gatherv per ricostruire l'array z finale 
    MPI_Gatherv(local_z, local_n, MPI_DOUBLE, 
                z, counts, displs, MPI_DOUBLE, 0, MPI_COMM_WORLD);  
    
    if ( my_rank == 0 ) {
        printf("Test OK\n");
        free(x); free(y); free(z);
    }  
    
    free(local_x); free(local_y); free(local_z);  
    free(counts); free(displs);  
    
    MPI_Finalize();  
    return EXIT_SUCCESS;
}
```

## MPI_Reduce
La funzione `MPI_Reduce` esegue un'operazione di riduzione globale (es. somma, massimo) sui dati distribuiti tra i processi e salva il risultato finale **esclusivamente nel buffer del processo root (destinazione)**.

```c
int MPI_Reduce(const void *sendbuf, void *recvbuf, int count, 
               MPI_Datatype datatype, MPI_Op op, int root, MPI_Comm comm)
```
* `sendbuf`: I dati forniti dal singolo processo (input).
* `recvbuf`: Il buffer in cui verrà scritto il risultato. Deve essere allocato e valido solo per il processo `root`.
* `count`: Se `count > 1`, l'operazione di riduzione viene applicata **elemento per elemento** (component-wise). Ad esempio, il primo elemento dell'array risultato sarà la riduzione di tutti i primi elementi degli array di input.

### Operatori Predefiniti (`MPI_Op`)
| Valore | Significato |
| :--- | :--- |
| `MPI_MAX` / `MPI_MIN` | Massimo / Minimo globale |
| `MPI_SUM` / `MPI_PROD` | Somma / Prodotto |
| `MPI_LAND` / `MPI_LOR` / `MPI_LXOR` | Operatori Logici (AND, OR, XOR) |
| `MPI_BAND` / `MPI_BOR` / `MPI_BXOR` | Operatori Bit a Bit |
| `MPI_MAXLOC` / `MPI_MINLOC` | Calcola il Max/Min e restituisce anche l'**indice** in cui si trova. Richiede un datatype composto, ad esempio `struct {double val; int idx}` abbinato al datatype MPI `MPI_DOUBLE_INT`. |

---

## Operazioni Collettive Avanzate

* **`MPI_Allreduce`**: È identica a `MPI_Reduce`, ma non esiste un processo `root`. Il risultato globale viene distribuito (Broadcast) a **tutti** i processi.
	![[9_MPI_page_57.png|fix]]
* **`MPI_Alltoall`**: Ogni processo invia una porzione di dati (Scatter) a tutti gli altri processi. È l'equivalente di una comunicazione tutti-con-tutti (trasposizione).
	![[9_MPI_page_60.png|fix]]
* **`MPI_Scan`**: (Inclusive Scan). Calcola l'operazione in base ai prefissi (rango dei processi). Il processo $i$ riceverà la riduzione dei valori forniti dai processi da $0$ a $i$.
![[9_MPI_page_62.png|fix]]
---

## Integrazione del Metodo dei Trapezi con MPI_Reduce
Riscriviamo il calcolo dell'area tramite trapezi eliminando il loop manuale di `Recv` e usando direttamente l'efficienza della libreria per raccogliere e sommare.

```c
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>  

double f( double x ) { return 4.0/(1.0 + x*x); }  

double trap( int my_rank, int comm_sz, double a, double b, int n ) {
    const double h = (b-a)/n;
    const int local_n_start = (n * my_rank) / comm_sz;
    const int local_n_end = (n * (my_rank + 1)) / comm_sz;
    double x = a + local_n_start * h;
    double my_result = 0.0;  
    
    for (int i = local_n_start; i < local_n_end; i++) {
        my_result += h*(f(x) + f(x+h))/2.0;
        x += h;
    }
    return my_result;
}  

int main( int argc, char* argv[] ) {
    double a = 0.0, b = 1.0, partial_result, result = 0.0;
    int n = 1000000, my_rank, comm_sz;  
    
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &my_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &comm_sz);  
    
    if ( argc == 4 ) {
        a = atof(argv[1]); b = atof(argv[2]); n = atoi(argv[3]);
    }  
    
    // 1. Calcolo Locale
    partial_result = trap( my_rank, comm_sz, a, b, n );  
    
    // 2. Riduzione Globale (Sum)
    MPI_Reduce( 
        &partial_result, /* send buffer (input di ogni processo) */
        &result,         /* receive buffer (dove salvare il tot) */
        1,               /* elementi da ridurre */
        MPI_DOUBLE, 
        MPI_SUM,         /* operatore */
        0,               /* root rank (processo 0) */
        MPI_COMM_WORLD 
    );  
    
    // Solo il processo 0 ha il risultato valido
    if ( my_rank == 0 ) printf("Area: %f\n", result);
    
    MPI_Finalize();  
    return EXIT_SUCCESS;
}
```

---

## Esercizio: Odd-Even Transposition Sort in MPI
L'algoritmo Odd-Even per sistemi a memoria distribuita si basa sulla **suddivisione in blocchi**. Se abbiamo $n$ elementi e $P$ processi, a ogni processo viene assegnato un blocco contiguo di dimensione $n/P$.

**Funzionamento:**
1.  **Ordinamento Locale:** Ogni processo ordina inizialmente il proprio blocco in modo indipendente (ad es. tramite Quicksort locale).
2.  **Fasi Alternate (Pari/Dispari):** Per $P$ fasi totali, i processi adiacenti (determinati in base alla fase corrente) si scambiano interi blocchi.
3.  **Merge-Split:** Dopo lo scambio, ogni processo unisce il proprio blocco con quello ricevuto dal vicino. Siccome entrambi i blocchi erano già localmente ordinati, **non serve un intero sort da capo**, basta un'operazione di *Merge* (costo lineare $O(n/P)$).
4.  **Selezione:** Dopo il Merge, il processo col rank inferiore si tiene la metà più piccola degli elementi fusi, e il processo col rank superiore si tiene la metà più grande.

![[9_MPI_page_69.png|fix]]

---

## Il Problema del Deadlock nello Scambio Dati
La fase più critica è il momento in cui due processi vicini devono scambiarsi i propri blocchi.

**Codice Pericoloso:**
```c
for (phase=0; phase < comm_sz; phase++) { 
    partner = compute_partner(phase, my_rank); 
    if (partner != IDLE) {
        MPI_Send(my_data, partner); // Rischio Deadlock!
        MPI_Recv(partner_data, partner);
        // ... Merge e Split ...
    } 
}
```
Se entrambi i processi chiamano `MPI_Send` e i blocchi di dati superano le dimensioni dei buffer di rete, si bloccheranno per sempre aspettando che l'altro chiami la `MPI_Recv`.

### Soluzioni Possibili
1.  **Spezzare la Simmetria:**
    ```c
    if (my_rank % 2 == 0) { 
        MPI_Send(...); MPI_Recv(...); 
    } else { 
        MPI_Recv(...); MPI_Send(...); 
    }
    ```
    *Funziona, ma rende il codice meno pulito.*

2.  **Uso della Comunicazione Asincrona:**
    ```c
    MPI_Request req; 
    MPI_Isend(...); // Invia in background
    MPI_Recv(...);  // Riceve
    MPI_Wait(&req, MPI_STATUS_IGNORE); // Attende fine invio
    ```
    *Ottima soluzione, ma non la migliore.*

---

## La Soluzione Definitiva: `MPI_Sendrecv`
La libreria MPI offre una funzione ottimizzata appositamente per le operazioni di scambio incrociato (shift o scambi diretti). Esegue **simultaneamente** l'invio e la ricezione senza alcun rischio di blocco.

```c
int MPI_Sendrecv(
    const void *sendbuf, int sendcount, MPI_Datatype sendtype, int dest, int sendtag,
    void *recvbuf, int recvcount, MPI_Datatype recvtype, int source, int recvtag,
    MPI_Comm comm, MPI_Status *status
)
```
* Unifica i parametri di un Invio e di una Ricezione in un'unica chiamata atomica e sicura.
* Il `dest` e il `source` possono essere lo stesso processo o processi diversi (utile per pipeline o comunicazioni circolari).

*Nota: Se la dimensione dell'array non fosse un multiplo esatto di $P$, i blocchi avrebbero dimensioni variabili. In questo caso, prima della fase di scambio dati tramite `MPI_Sendrecv`, i processi dovrebbero comunicarsi a vicenda quanti elementi stanno effettivamente per inviare.*

Analizziamo l'algoritmo dal punto di vista teorico :

![[OddEvenSort_analisi_MPI.png]]![[OddEvenSort_analisi_OptimalP_MPI.png|697]]


