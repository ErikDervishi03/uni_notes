## Perché studiare diverse architetture?

Non esiste un paradigma di programmazione parallela universale. Tecnologie diverse richiedono approcci diversi: ciò che è ottimale per le GPU non lo è per i supercomputer a Memoria Distribuita o per i processori Multi-core.

## Architettura di Von Neumann

Quasi tutti i processori seriali tradizionali si basano su questo modello:
* **Memoria unificata:** Contiene sia i dati che i programmi (le istruzioni sono rappresentate come byte, esattamente come i dati).
* **Componenti principali:** La CPU (che include la ALU e la Control Unit) è collegata alla memoria tramite un **Bus**.
* **Registri:** Memorie ultra-veloci interne alla CPU. Si dividono in *General Purpose* (per i dati temporanei) e *Speciali* (PC: Program Counter, IR: Instruction Register, PSW: Program Status Word per flag ed errori come l'overflow).

**Il Collo di Bottiglia (Von Neumann Bottleneck):**

Il limite principale di questa architettura è il **bus** che trasferisce informazioni tra memoria e processori, essendo molto più lento della CPU. Per mitigare questo problema si utilizzano tre strategie:
1. Sfruttare al meglio la gerarchia di Caching.
2. Utilizzare l'Hardware Multithreading per nascondere le latenze di memoria.
3. Eseguire più istruzioni contemporaneamente (es. istruzioni vettoriali SIMD).

---

## Caching e Località

Le prestazioni di processori e memorie RAM sono molto diverse, e il divario peggiora nei sistemi multicore. Per questo si interpongono memorie più piccole e veloci (le Cache) tra la CPU e la RAM centrale (DRAM).
Esistono diversi livelli di cache (L1, L2, L3): maggiore è la capienza, minore è la velocità. 

Le cache migliorano le prestazioni mantenendo i dati usati di recente, sfruttando due principi probabilistici:
* **Località Spaziale:** Se un dato viene acceduto, è probabile che i dati adiacenti in memoria vengano richiesti a breve (es. scorrere una matrice per riga in C sfrutta questa località, per colonna no).
* **Località Temporale:** Se un dato viene acceduto, è probabile che venga richiesto nuovamente a breve.

Un valore molto importante da vedere è quindi quello dei cache miss del nostro programma. Questo possiamo farlo in questo modo:

![[3_parallel-architectures_page_21.png|fix]]

---

## Hardware Multithreading

È un meccanismo hardware che permette a una singola unità di esecuzione fisica di operare come se fosse composta da più core logici. 

| Tipologia | Caratteristiche | Utilizzo |
| :--- | :--- | :--- |
| **Grana Grossa (Coarse-grained)** | Gestito dal Sistema Operativo. Il context switch ha un costo elevato. | Efficace quando un processo è bloccato in I/O per lunghi periodi. Non è efficace in stalli brevi. |
| **Grana Fine (Fine-grained)** | Supporto hardware specifico. Context switch a costo quasi zero. | Ideale per nascondere le brevi latenze degli accessi in memoria centrale. |

**Simultaneous Multithreading (SMT) / Hyper-threading** ^[Simultaneous Multithreading e Hyper-Threading possono essere considerati sinonimi, ma c'è una sfumatura: **SMT** è il termine generico per la tecnologia hardware a grana fine; **Hyper-Threading (HTT)** è il marchio commerciale registrato da Intel per la sua specifica implementazione.]:

Il Sistema Operativo vede un singolo processore fisico come due core logici. Condividono risorse fisiche ma hanno registri separati. 
Questo permette di mantenere occupata la **Pipeline** (fasi: *IF, ID, EX, MEM, WB*). Le code rappresentano i flussi di istruzioni che vengono mandate: se una queue è ferma l'altra può andare avanti. L'incremento prestazionale tipico è del 20%.

![[3_parallel-architectures_page_25.png|fix]]

---

## Tassonomia di Flynn

Classificazione delle architetture dei calcolatori basata sul numero di flussi di istruzioni e dati.

![[3_parallel-architectures_page_29.png|fix]]

### SISD

Single Instruction, Single Data. Una singola unità di controllo pilota un'unica ALU operando su un singolo flusso di dati.

![[3_parallel-architectures_page_30.png|fix]]

### MISD

Multiple Instruction, Single Data. Più unità indipendenti eseguono istruzioni diverse sullo stesso identico flusso di dati.

![[3_parallel-architectures_page_31.png|fix]]

### [[1_Introduzione al Calcolo Parallelo#Livelli di Parallelismo e Istruzioni SIMD|SIMD]]

Single Instruction, Multiple Data. Singola unità di controllo (un flusso di istruzioni) ma più ALU che operano su dati diversi, ma l'operazione è la stessa. Usano funzioni *[[1_Introduzione al Calcolo Parallelo#^9a64f9|intrinsic]]*.

![[3_parallel-architectures_page_32.png|fix]]

### MIMD

Multiple Instruction, Multiple Data. Più unità indipendenti, ognuna con la sua ALU e il suo flusso di dati. Esecuzione davvero contemporanea.

![[3_parallel-architectures_page_33.png|fix]]

---

## Architetture MIMD e Ibride

I sistemi moderni appartengono principalmente alla categoria MIMD che si divide in due architetture fondamentali:
### Memoria Condivisa vs Distribuita

| Architettura                                 | Vantaggi                                                                                                     | Svantaggi                                                                                                             |
| :------------------------------------------- | :----------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------- |
| **Memoria Condivisa** (Shared Memory)        | - Facile da programmare.<br>- Ottima per accessi irregolari ai dati (es. grafi).                             | - Il programmatore deve gestire attivamente le *Race Conditions*.<br>- Larghezza di banda limitata.                   |
| **Memoria Distribuita** (Distributed Memory) | - Altamente scalabile aggiungendo nodi.<br>- Ideale per alta località e alto rapporto calcolo/comunicazione. | - Molto complessa da programmare.<br>- Rischio di *Deadlocks*.<br>- Latenza di rete introdotta dall'interconnessione. |

^1b2c6a

Guarda i grafici delle slide, la rappresentazione e' molto intuitiva : 

![[3_parallel-architectures_page_34.png|fix]]

**Architetture Ibride:**

Nei supercomputer, i nodi sono spesso a loro volta architetture a memoria condivisa che in più usano delle GPU, e i nodi usano message passing per comunicare tra di loro.

![[3_parallel-architectures_page_35.png|fix]]


