## Paradigmi di Calcolo: HPC vs HTC

Il nostro corso si concentra sull'High Performance Computing(HPC). È fondamentale distinguere questo approccio da quello tipico del Cloud Computing.

| Caratteristica           | High Performance Computing(HPC)                                                  | High Throughput Computing (HTC) / Cloud                                           |
| :----------------------- | :------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------- |
| **Obiettivo Principale** | Minimizzare il tempo di esecuzione (Makespan) di una singola applicazione.       | Massimizzare il numero di task completati nell'unità di tempo (Throughput).       |
| **Natura del Task**      | Unica applicazione massiccia e complessa che elabora enormi moli di dati.        | Tante piccole applicazioni o istanze dello stesso problema indipendenti tra loro. |
| **Suddivisione**         | Il problema viene decomposto in sottoproblemi interdipendenti.                   | Tanti task isolati (problemi "imbarazzantemente paralleli").                      |
| **Comunicazione**        | Alta. I sottoproblemi necessitano di cooperare e scambiarsi dati frequentemente. | Bassa o nulla. Le componenti non hanno bisogno di comunicare.                     |
| **Infrastruttura**       | Supercomputer con reti di interconnessione fittissime a **bassissima latenza**.  | Data center distribuiti costituiti da server o PC standard commerciali.           |

## La Legge di Moore e la Barriera Termica

La Legge di Moore postula che il numero di transistor in un circuito integrato tenda a raddoppiare circa ogni 18-24 mesi. 

* **Il problema (Power/Thermal Wall):** L'aumento della densità dei transistor porta a un aumento esponenziale del calore generato (dissipazione termica). Non è più fisicamente possibile aumentare le frequenze di clock come in passato senza fondere il chip.
* **La soluzione:** Per mantenere le prestazioni in crescita, l'industria è passata all'Architettura Multicore. I processori moderni integrano molteplici core logici a frequenze più basse all'interno di un singolo chip.

## Fasi di Parallelizzazione di un Problema

Per risolvere un problema computazionale in ottica parallela, si abbandona la tradizionale esecuzione di istruzioni lineari e si seguono quattro fasi logiche:

1. **Decomposizione (Partitioning):** Suddividere il programma e i dati in sottoproblemi più piccoli e gestibili.
2. **Distribuzione (Mapping):** Assegnare e distribuire questi sottoproblemi alle varie unità di calcolo (core o nodi) disponibili.
3. **Risoluzione / Comunicazione:** Le unità di calcolo risolvono i sottoproblemi. Poiché spesso non sono del tutto indipendenti, è necessario orchestrare la comunicazione e la sincronizzazione tra di esse.
4. **Aggregazione:** Combinare i risultati parziali elaborati dalle singole unità per comporre il risultato finale e globale.

## Livelli di Parallelismo e Istruzioni SIMD

Il parallelismo può essere sfruttato a diversi livelli hardware. Uno dei più vicini al processore è l'approccio SIMD(Single Instruction, Multiple Data).

![[2_introduction_page_43.png|fix]]

| Modello                 | Funzionamento                                                      | Utilizzo Tipico                                          |
| :---------------------- | :----------------------------------------------------------------- | :------------------------------------------------------- |
| **SISD (Tradizionale)** | Singola istruzione applicata a un singolo dato.                    | Esecuzione sequenziale classica.                         |
| **SIMD**                | Singola istruzione applicata simultaneamente a un vettore di dati. | Parallelismo a livello di dati (Data-Level Parallelism). |

* **Funzionamento SIMD:** La CPU dispone di speciali registri vettoriali (più capienti di quelli standard) che possono contenere array di valori (es. 4 interi adiacenti caricati dalla memoria). Una singola istruzione è in grado di eseguire la stessa operazione (es. una somma) su tutti gli elementi del registro contemporaneamente.
* **Implementazione:** In C/C++, si utilizzano le **funzioni intrinsic**, le quali vengono mappate direttamente dal compilatore alle corrispondenti istruzioni assembly SIMD del processore. ^9a64f9

## Architettura Multicore e Gerarchia di Memoria

Un Core è, a tutti gli effetti, una CPU indipendente. Spesso i processori supportano tecnologie come l'**Hyper-threading**, che permette a un singolo core fisico di esporsi al sistema operativo come due core logici, gestendo più thread concorrentemente.

![[2_introduction_page_47.png|fix]]

Il parallelismo moderno è gerarchico: si distribuisce il carico tra i vari core e, all'interno di ciascun core, si sfrutta il parallelismo SIMD.

Per alimentare i core con i dati necessari senza rallentarli, i processori implementano una **Gerarchia di Cache**:

| Livello di Cache | Posizionamento             | Velocità                      | Capacità       | Condivisione                                    |
| :--------------- | :------------------------- | :---------------------------- | :------------- | :---------------------------------------------- |
| **Cache L1**     | Interna al singolo core.   | Altissima.                    | Molto piccola. | Privata (separata tra L1-Istruzioni e L1-Dati). |
| **Cache L2**     | Adiacente al singolo core. | Alta.                         | Media.         | Tipicamente privata per ogni singolo core.      |
| **Cache L3**     | Esterna ai core, sul chip. | Media (più veloce della RAM). | Grande.        | **Condivisa** tra tutti i core del processore.  |


![[2_introduction_page_53.png|fix]]

*Nota: nei processori di fascia alta (es. HPC), l'immagine dell'architettura può presentare decine o centinaia di core interconnessi attorno a banchi di Cache L3 condivisa e controller di memoria complessi.*
