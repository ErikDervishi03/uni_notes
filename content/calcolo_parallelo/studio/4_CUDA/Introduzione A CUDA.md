**CUDA (Compute Unified Device Architecture)** è una tecnologia proprietaria di NVIDIA basata sul paradigma di programmazione *[[3_design-of-parallel-programs#Paradigmi di Parallelismo Task vs Data|Data-Parallel]]*. 

## Terminologia di Base
* **Host**: La CPU e la sua memoria RAM di sistema.
* **Device**: La GPU e la sua memoria video (VRAM).
Queste sono due entità fisicamente distinte che comunicano attraverso il bus **PCI-Express**.

## Flusso di Esecuzione di un Programma CUDA
Ogni programma CUDA è composto da codice seriale che gira sulla CPU e codice parallelo che gira sulla GPU. Il flusso tipico è:
1.  **Copia dei dati (Host $\to$ Device)**: L'Host trasferisce i dati di input nella memoria del Device.
2.  **Lancio del Kernel**: L'Host invoca l'esecuzione del programma parallelo sulla GPU.
3.  **Computazione Asincrona**: La CPU, dopo aver lanciato la GPU, non è bloccata. Può continuare a eseguire altre operazioni mentre la GPU lavora.
4.  **Copia dei risultati (Device $\to$ Host)**: Al termine, l'Host recupera i risultati trasferendoli dalla memoria della GPU alla propria.

---

## Modello di Esecuzione: Grid, Block, Thread e Warp
Dal punto di vista del programmatore, la GPU è un'entità a cui inviare funzioni (chiamate **Kernel**). Un Kernel viene eseguito contemporaneamente da migliaia di **Thread** indipendenti.

La gerarchia dei thread è rigorosa:
* **Thread**: L'unità base di esecuzione.
* **Block (Blocco)**: Un gruppo di thread (organizzati in 1, 2 o 3 dimensioni). Un blocco può contenere al **massimo 1024 thread**. I thread dello stesso blocco possono cooperare e condividere memoria veloce.
* **Grid (Griglia)**: Un array di blocchi (organizzati in 1, 2 o 3 dimensioni) che eseguono lo stesso Kernel.
### Il concetto di Warp (Modello Hardware)
Mentre blocchi e griglie sono astrazioni software, a livello hardware i thread vengono raggruppati in **Warp**, formati esattamente da **32 thread**.
* La GPU agisce in modalità **[[1_Introduzione al Calcolo Parallelo#Livelli di Parallelismo e Istruzioni SIMD|SIMD]] (Single Instruction, Multiple Data)** a livello di Warp: l'unità SIMD è il Warp, quindi tutti i 32 thread eseguono *sempre la stessa istruzione* contemporaneamente.
* *Nota sull'efficienza:* Se i thread all'interno di un warp prendono percorsi diversi a causa di salti condizionali (`if/else`), la GPU deve eseguire i rami in modo seriale. È altamente inefficiente, quindi bisogna limitare le divergenze dei Warp.

![[10_CUDA_page_8.png|fix]]

### Lo Scheduler Hardware 
Dove vanno a finire i thread? L'hardware è ottimizzato per fare parallelismo a grana finissima. Se creiamo 100.000 thread divisi in blocchi, non importa quanti SM (Streaming Multiprocessor) fisici abbia la scheda: lo scheduler della GPU assegnerà in modo trasparente i blocchi agli SM disponibili, con politiche simili a quelle di un OS. Noi creiamo i thread, l'hardware ci pensa.

![[10_CUDA_page_11.png|fix]]

---

## Sintassi Base: Il primo programma CUDA

La keyword `__global__` permette di definire una funzione Kernel (chiamata dall'Host, eseguita sul Device). Deve sempre restituire `void`.

```cpp
/* cuda-hello1.cu */ 
#include <stdio.h> 

__global__ void mykernel(void) { 
    // Codice eseguito sulla GPU
} 

int main(void) { 
    mykernel<<<1,1>>>(); // Sintassi di lancio: <<<N_Blocchi, N_Thread_per_Blocco>>>
    printf("Hello World!\\n"); 
    return 0; 
}
```
**Compilazione:**
```bash
$ nvcc cuda-hello1.cu 
$ ./a.out 
Hello World!
```

---

## Gestione della Memoria
I puntatori usati nel Kernel puntano all'area di memoria del Device, non dell'Host. Per trasferire dati ci servono:
* `cudaMalloc()`, `cudaFree()`, `cudaMemcpy()` (simili a malloc, free, memcpy del C).

**Tipi di copia:** `cudaMemcpyHostToDevice`, `cudaMemcpyDeviceToHost`.

Per la somma dei 2 valori faremo cosi':

```cpp
#include <stdio.h>  
#include <stdlib.h>  
#include "hpc.h"  
  
__global__ void add( int *a, int *b, int *c )  
{  
   *c = *a + *b;  
}  
  
int main( void )    
{  
   int a, b, c;                  /* host copies of a, b, c */    
   int *d_a, *d_b, *d_c;         /* device copies of a, b, c */  
   const size_t size = sizeof(int);  
   /* Allocate space for device copies of a, b, c */  
   cudaMalloc((void **)&d_a, size);  
   cudaMalloc((void **)&d_b, size);  
   cudaMalloc((void **)&d_c, size);  
   /* Setup input values */  
   a = 2; b = 7;  
   /* Copy inputs to device */  
   cudaMemcpy(d_a, &a, size, cudaMemcpyHostToDevice);  
   cudaMemcpy(d_b, &b, size, cudaMemcpyHostToDevice);  
   /* Launch add() kernel on GPU */  
   add<<<1,1>>>(d_a, d_b, d_c); cudaCheckError();  
   /* Copy result back to host */  
   cudaMemcpy(&c, d_c, size, cudaMemcpyDeviceToHost);  
   /* Cleanup */  
   cudaFree(d_a); cudaFree(d_b); cudaFree(d_c);  
   return EXIT_SUCCESS;  
}
```
### Sincronismo e Asincronismo
Anche se i Kernel sono asincroni, i trasferimenti via `cudaMemcpy` avvengono tramite DMA e **bloccano la CPU** finché la copia non è terminata. 
Per la copia asincrona:
* `cudaMemcpyAsync()`: Non blocca la CPU.
* `cudaDeviceSynchronize()`: Blocca la CPU finché tutte le chiamate CUDA precedenti non sono terminate.

---

## Esempio pratico: Somma di Vettori

### Step 1: Somma di Vettori (1 Blocco, N Thread)
Se creassimo un solo blocco con al massimo 1024 thread, useremmo l'indice del thread all'interno del blocco (`threadIdx.x`):

```cpp
__global__ void add(int *a, int *b, int *c) { 
    c[threadIdx.x] = a[threadIdx.x] + b[threadIdx.x]; 
}
// Chiamata: add<<<1, N>>>(d_a, d_b, d_c); // Funziona solo se N <= 1024
```

![[10_CUDA_page_35.png|fix]]

### Step 2: Somma Scalabile (N Blocchi, 1 Thread per blocco)
Invece di usare i thread, potremmo usare solo i blocchi (`blockIdx.x`):

```cpp
__global__ void add(int *a, int *b, int *c) { 
    c[blockIdx.x] = a[blockIdx.x] + b[blockIdx.x]; 
}
// Chiamata: add<<<N, 1>>>(d_a, d_b, d_c);
```

![[10_CUDA_page_30.png|fix]]

### Step 3: Somma Ottimale (Combinare Blocchi e Thread)
La situazione reale richiede di combinare l'indice del blocco e del thread per mappare vettori di grandissime dimensioni in un unico array lineare. La formula è:
`int index = threadIdx.x + blockIdx.x * blockDim.x;`

```cpp
__global__ void add(int *a, int *b, int *c, int n) { 
    int index = threadIdx.x + blockIdx.x * blockDim.x; 
    
    // Controllo dei limiti: potremmo aver lanciato più thread dell'array
    if (index < n) { 
        c[index] = a[index] + b[index]; 
    } 
}

// Chiamata nel main:
// #define BLKDIM 1024
// add<<<(N + BLKDIM - 1)/BLKDIM, BLKDIM>>>(d_a, d_b, d_c, N);
```

![[10_CUDA_page_30.png|fix]]

---

## Cooperazione tra Thread e Memoria Condivisa (Shared Memory)

### Il Problema dello Stencil 1D
Vogliamo calcolare uno Stencil 1D: ogni elemento del nuovo array è la somma di se stesso, dei `RADIUS` elementi prima e dei `RADIUS` elementi dopo.

Se ogni thread legge direttamente dalla Global Memory, ci saranno tantissime letture ridondanti nella stessa area di memoria, rendendo l'algoritmo inefficiente.

### Il Modello di Memoria CUDA e la Soluzione
Per risolvere, i thread dello stesso blocco possono condividere dati tramite la **Memoria Condivisa (Shared Memory)**, dichiarata usando `__shared__`. È molto più veloce di quella globale, ma è visibile solo ai thread di quel blocco.

![[10_CUDA_page_51.png|fix]]

### Uso della Memoria Condivisa nello Stencil
L'idea è far copiare ad ogni thread del blocco il proprio elemento dalla memoria globale a quella condivisa. 

![[10_CUDA_page_55.png|fix]]

La singola istruzione `temp[lindex] = in[gindex];`, essendo eseguita in parallelo da tutti i thread, consente di copiare l'intero blocco di dati quasi istantaneamente. I thread ai bordi copieranno anche le "ghost cells" (l'alone del raggio).

### Attenzione alla Race Condition e `__syncthreads()`
Siccome un blocco ha più di 32 thread (più di un warp), un warp potrebbe finire di copiare i suoi dati e iniziare ad applicare lo stencil su elementi della shared memory non ancora inizializzati dagli altri warp (Race Condition). 

Per risolvere si usa `__syncthreads()`, che crea una barriera di sincronizzazione tra tutti i thread del blocco.

```cpp
#define BLKDIM 1024
#define RADIUS 3

__global__ void stencil_1d(int *in, int *out) { 
    // Array in Shared Memory (dimensione blocco + aloni sx/dx)
    __shared__ int temp[BLKDIM + 2 * RADIUS]; 
    
    // Indice globale (con offset del radius per i bordi esterni)
    int gindex = threadIdx.x + blockIdx.x * blockDim.x + RADIUS; 
    
    // Indice locale nella shared memory
    int lindex = threadIdx.x + RADIUS; 
    
    int result = 0, offset; 
    
    // 1. Lettura elementi principali
    temp[lindex] = in[gindex]; 
    
    // 2. Lettura dell'alone (Ghost Cells)
    if (threadIdx.x < RADIUS) { 
        temp[lindex - RADIUS] = in[gindex - RADIUS]; 
        temp[lindex + blockDim.x] = in[gindex + blockDim.x]; 
    } 
    
    // Sincronizzazione fondamentale contro le Race Condition
    __syncthreads(); 
    
    // 3. Applicazione dello Stencil 
    for (offset = -RADIUS ; offset <= RADIUS ; offset++) { 
        result += temp[lindex + offset]; 
    } 
    
    // 4. Scrittura risultato
    out[gindex] = result; 
}
```

## Gestione del Device e Misurazione dei Tempi

Per misurare le prestazioni di un kernel, possiamo utilizzare la libreria standard `hpc.h` e la funzione `hpc_gettime()`.
**Importante:** Poiché i kernel sono lanciati in modo asincrono, se non inseriamo una barriera prima di prendere il tempo finale, misureremmo solo il tempo impiegato dalla CPU per "lanciare" il comando, non quello reale di elaborazione della GPU.

```c
#include "hpc.h" 
// ... 
double tstart, tend; 
tstart = hpc_gettime(); 

mykernel<<<X, Y>>>(); // Lancio asincrono
cudaDeviceSynchronize(); // BLOCCA la CPU finché la GPU non finisce

tend = hpc_gettime(); 
printf("Elapsed time %f\\n", tend - tstart); 
```
Spesso è utile misurare anche il tempo speso in `cudaMemcpy` (che rappresenta il collo di bottiglia del bus PCI-Express) separatamente dal tempo di elaborazione puro del kernel.

---

## Qualificatori delle Funzioni (`__device__`, `__host__`)

Oltre a `__global__` (funzione chiamata dall'Host ed eseguita sul Device), CUDA introduce altri qualificatori:

### Il qualificatore `__device__`
Indica una funzione che può essere chiamata **solo** da un kernel (`__global__`) o da un'altra funzione `__device__`. Gira esclusivamente sulla GPU.
```c
__device__ float cuda_fmaxf(float a, float b) { 
    return (a > b ? a : b); 
} 

__global__ void my_kernel(float *v, int n) { 
    int i = threadIdx.x + blockIdx.x * blockDim.x; 
    if (i < n) { 
        v[i] = cuda_fmaxf(1.0, v[i]); // Chiamata device-to-device
    } 
}
```

*Uso sulle variabili:* Se `__device__` viene applicato a una variabile globale fuori dalle funzioni, questa viene allocata direttamente nella memoria globale della GPU senza dover chiamare `cudaMalloc`. Tuttavia, la sua dimensione deve essere costante e nota a tempo di compilazione. Per scambiare dati con queste variabili dall'Host si usa:
* `cudaMemcpyToSymbol(dev_var, host_ptr, size)`
* `cudaMemcpyFromSymbol(host_ptr, dev_var, size)`

### Il qualificatore `__host__`
Indica che la funzione gira sulla CPU. È il qualificatore implicito di default. 
Diventa utile quando usato in combinazione con `__device__`:
```c
__host__ __device__ float my_fmaxf(float a, float b) { 
    return (a > b ? a : b); 
}
```
In questo caso, il compilatore `nvcc` genererà automaticamente due versioni della funzione: una eseguibile dalla CPU e una eseguibile dalla GPU.

### Recap Qualificatori
| Qualificatore | Eseguita su | Chiamabile da |
| :--- | :--- | :--- |
| `__device__` | Device (GPU) | Device |
| `__host__` | Host (CPU) | Host |
| `__global__` | Device (GPU) | Host |

---

## Gestione degli Errori in CUDA
Poiché i kernel non restituiscono valori (sono void e asincroni), eventuali errori non interrompono l'esecuzione nativamente e rischiano di passare inosservati.

Per controllare l'ultimo errore generato dalla GPU:
```c
cudaError_t err = cudaGetLastError();
if (err != cudaSuccess) {
    printf("%s\\n", cudaGetErrorString(err));
}
```

La libreria `hpc.h` fornisce due macro comodissime per semplificare:
* `cudaSafeCall( expr );`: Da avvolgere attorno alle chiamate API (es. `cudaMemcpy`, `cudaMalloc`).
* `cudaCheckError();`: Da mettere subito dopo l'invocazione di un kernel.

```c
cudaSafeCall( cudaMemcpy(d_a, h_a, size, cudaMemcpyHostToDevice) ); 
my_kernel<<< 1, 1 >>>(d_a); 
cudaCheckError(); 
cudaSafeCall( cudaMemcpy(h_a, d_a, size, cudaMemcpyDeviceToHost) ); 
```

---

## Griglie Bi- e Tri-dimensionali (dim3)
I thread e i blocchi non devono per forza essere lineari (1D). Per problemi geometrici o matriciali, è comodissimo organizzarli in 2D o 3D usando la struttura dati `dim3`.

```c
dim3 blk1(3);       // Blocco 1D: 3 x 1 x 1
dim3 blk2(3, 4);    // Blocco 2D: 3 x 4 x 1
dim3 blk3(3, 4, 7); // Blocco 3D: 3 x 4 x 7

// Per lanciare un kernel con una griglia 2D di blocchi 3D:
dim3 grid(16, 4);   // 16x4 blocchi
dim3 block(8, 8, 8); // 8x8x8 thread per blocco
mykernel<<<grid, block>>>();
```

---

## Esempio: Moltiplicazione di Matrici (MatMul) e Tiling

Vogliamo moltiplicare due matrici quadrate $N \times N$. In CPU richiederemmo 3 cicli annidati. In GPU mappiamo i primi due cicli (righe e colonne) agli indici 2D del blocco e del thread, eliminandoli.

```c
__global__ void matmul(const float *p, const float *q, float *r, int n) { 
    // y = riga, x = colonna
    const int i = blockIdx.y * blockDim.y + threadIdx.y; 
    const int j = blockIdx.x * blockDim.x + threadIdx.x; 
    
    if ( i < n && j < n ) { 
        float val = 0.0; 
        for (int k = 0; k < n; k++) { 
            // Matrice memorizzata linearmente: indice = riga * dim + colonna
            val += p[i*n + k] * q[k*n + j]; 
        } 
        r[i*n + j] = val; 
    } 
}
```

### Ottimizzazione MatMul con Tiling (Shared Memory)
L'approccio precedente fa tanta pressione sulla memoria globale (Global Memory) perché ogni thread deve scorrere intere righe e colonne.
Con la tecnica del **Tiling**, la matrice viene caricata a tiles nella Shared Memory. I thread del blocco collaborano per caricare una sottomatrice, si sincronizzano, e poi eseguono i calcoli sulle veloci memorie locali.

```c
__global__ void matmulb(const float *p, const float *q, float *r, int n) { 
    // Mattonelle in Shared Memory
    __shared__ float local_p[BLKDIM][BLKDIM]; 
    __shared__ float local_q[BLKDIM][BLKDIM]; 
    
    const int ty = threadIdx.y, tx = threadIdx.x; 
    const int i = blockIdx.y * BLKDIM + ty; 
    const int j = blockIdx.x * BLKDIM + tx; 
    
    float v = 0.0; 
    
    // Scorriamo le matrici a passi di dimensione del TILE (BLKDIM)
    for (int m = 0; m < n; m += BLKDIM) { 
        // 1. Ogni thread carica un singolo elemento nelle mattonelle condivise
        local_p[ty][tx] = p[i*n + (m + tx)]; 
        local_q[ty][tx] = q[(m + ty)*n + j]; 
        __syncthreads(); // Attende che la mattonella sia completamente caricata
        
        // 2. Calcolo del prodotto parziale usando i dati in Shared Memory
        for (int k = 0; k < BLKDIM; k++) { 
            v += local_p[ty][k] * local_q[k][tx]; 
        } 
        __syncthreads(); // Attende che tutti abbiano finito di usare questa mattonella
    } 
    r[i*n + j] = v; 
}
```
*Identificazione del Warp in griglie multi-dimensionali:* In CUDA, all'interno di un blocco i thread formano i warp raggruppandosi prima lungo la dimensione $X$, poi la $Y$ e infine la $Z$.
L'ID lineare del thread (e di conseguenza a quale Warp appartiene) si calcola così:
`tid = threadIdx.x + threadIdx.y * blockDim.x;`
`warp_id = tid / 32;`

---

## Riduzione Parallela sulla GPU
Come facciamo una `sum-reduction` di un vettore in CUDA? Non esiste un `#pragma omp reduction`. Vediamo l'evoluzione delle strategie:

1.  **Approccio Naive (Blocchi + Host):** Ogni blocco calcola la somma parziale della sua porzione di vettore usando la Shared Memory, e salva il singolo risultato in un array globale `d_sums`. Infine l'Host (CPU) legge `d_sums` e somma l'ultimo migliaio di valori serialmente.
2.  **Riduzione ad Albero in Shared Memory:** Ottimizza il calcolo interno al blocco. Al posto di usare un thread per sommare in un ciclo for seriale, si fanno sommare i thread a coppie dimezzando lo stride ad ogni step.
3.  **Ottimizzazione del Lavoro per Thread (Third Solution):** Il costo per lanciare un blocco è alto. Invece di far sommare un elemento per thread, lanciamo meno blocchi e facciamo scorrere i thread a salti di dimensione `blockDim.x`. In questo modo un thread somma più elementi in modo interleaved (garantendo l'accesso coalescente, vedi prossimo paragrafo).

---

## Ottimizzazione degli Accessi in Memoria (Coalescing)
Questa è forse la regola fondamentale per ottenere vere prestazioni in CUDA. 

L'accesso alla **Memoria Globale** avviene sempre con trasferimenti raggruppati (chunk) da 32, 64 o 128 byte alla volta, corrispondenti alle "cache-lines".
La GPU cerca sempre di **fondere (coalesce)** gli accessi alla memoria dei 32 thread appartenenti allo stesso Warp.

| Tipo di Accesso del Warp | Efficienza del Bus |
| :--- | :--- |
| **Allineato e Consecutivo:** I 32 thread chiedono elementi adiacenti in memoria. | **100%**. La GPU legge l'intera cache-line da 128 byte in una singola transazione utile. |
| **Permutato:** I 32 thread chiedono elementi all'interno degli stessi 128 byte, ma mischiati. | **100%**. Hardware moderno mappa correttamente la cache-line. |
| **Disallineato:** L'accesso consecutivo cade a cavallo di due cache-line. | **50%**. La GPU deve importare due blocchi da 128 byte, sprecandone una metà. |
| **Broadcast (stesso elemento):** Tutti i 32 thread leggono la stessa singola variabile. | **3.125%**. Importa 128 byte per usare 4 byte. |
| **Sparpagliato (Scattered):** I 32 thread leggono in 32 punti lontanissimi tra loro. | **Catastrofe**. La GPU deve fare 32 transazioni separate importando un totale di 4096 byte, sprecandone quasi tutti. |

### L'esempio della Rotazione Immagine
Se leggiamo un'immagine per riga (coalesced) e la scriviamo trasposta per colonna, la scrittura sarà *scattered* (i thread scriveranno in indirizzi lontani $N$ byte tra loro), causando un crollo totale del memory throughput.
**La Soluzione:** Si usa la Shared Memory! I thread caricano un Tile dell'immagine (lettura coalesced), ruotano la matrice piccola *dentro la velocissima Shared Memory*, e poi la scrivono in output riga per riga (scrittura coalesced).

---

## AoS vs SoA (Array of Structures vs Structure of Arrays)
Un classico errore concettuale nell'organizzazione dei dati in C rispetto a CUDA.

**AoS (Array of Structures):**
```c
typedef struct { float x; float y; float z; } point3d;
// ... point3d points[N]; ...
float x = points[i].x; // Accesso del thread
```
*Problema:* In memoria i dati sono `x0 y0 z0 x1 y1 z1`. Se il Warp 1 ha bisogno di tutte le `x`, leggerà a salti di 12 byte (Scattered), forzando l'importazione inutile di `y` e `z` sprecando larghezza di banda.

**SoA (Structure of Arrays) - La via CUDA:**
```c
typedef struct { float *x; float *y; float *z; } points3d;
// ...
float x = points->x[i]; // Accesso del thread
```
*Soluzione:* In memoria ci sarà `x0 x1 x2 ... y0 y1 y2 ...`. Quando il Warp chiede le prime 32 `x`, sono tutte perfettamente contigue in RAM. L'accesso è **100% Coalesced**.

---

## Misura delle Prestazioni in CUDA
Non possiamo usare i classici concetti di "Speedup vs $p$ processori" come in OpenMP o MPI, perché in CUDA non controlliamo il numero fisico di core usati, ma solo la griglia logica che l'hardware distribuisce.

Si usano metriche alternative:
1.  **Throughput (Capacità di smaltimento):** $$Throughput = \frac{\text{Numero totale di Operazioni}}{\text{Wall-Clock Time}}$$
    Misura quante operazioni (spesso in GFLOPS) o quanti elementi al secondo il dispositivo elabora.
2.  **Speedup Competitivo Relativo:**
    $$CUDA\_Speedup = \frac{\text{Wall-Clock Time dell'implementazione OpenMP Ottimizzata (con max core)}}{\text{Wall-Clock Time di CUDA}}$$
    *Nota bene:* Misurare le prestazioni di una GPU da migliaia di core contro un programma C seriale a thread singolo è intellettualmente disonesto. Il paragone equo si fa sempre mettendo al 100% dell'efficienza la CPU (usando OpenMP) contro il 100% dell'efficienza della GPU.