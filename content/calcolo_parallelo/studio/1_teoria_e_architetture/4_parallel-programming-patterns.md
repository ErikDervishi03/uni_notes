
Un design pattern è una descrizione generale di come risolvere una famiglia di problemi ingegneristici ricorrenti, indicando come adattare una soluzione nota a un problema specifico.

**Indice dei Pattern:**
* [[#Embarrassingly Parallel]]
* [[#Scatter/Gather]]
* [[#Partition]]
* [[#Master-Worker]]
* [[#Stencil]]
* [[#Reduce]]
* [[#Scan]]
* [[#List Ranking (Pointer Jumping)]]

---

## Embarrassingly Parallel
Il problema è decomponibile in sottoproblemi completamente indipendenti tra loro. 
* **Caratteristiche:** Non serve interazione o comunicazione tra i processi per la risoluzione dei task.
* **Esempi tipici:** Somma vettoriale, calcolo dell'insieme di Mandelbrot, rendering 3D, brute force per il cracking di password.

## Scatter/Gather
Solitamente viene usata in architetture a memoria distribuita. Consiste nella gestione di una struttura dati (es. array o matrice) in due fasi:
1. **Scatter:** Suddivisione dei dati in blocchi che vengono assegnati ed elaborati da processi diversi.
2. **Gather:** Riassemblaggio dei risultati parziali dai vari processi per formare il risultato globale.

![[5_parallel-programming-patterns_page_14.png|fix]]
## Partition
Il dominio dei dati di input viene suddiviso in regioni disgiunte (partizioni), ciascuna assegnata a un processore. È particolarmente utile quando c'è località di riferimento, riducendo la necessità di comunicazione tra i processi.

### Tipi di Partizionamento
| Tipo           | Descrizione                                                                     | Esempi                                        |
| :------------- | :------------------------------------------------------------------------------ | :-------------------------------------------- |
| **Regolare**   | Il dominio è diviso in partizioni che hanno circa la stessa dimensione e forma. | Prodotto matrice-vettore.                     |
| **Irregolare** | Le partizioni non hanno necessariamente la stessa grandezza o forma.            | Trasferimento di calore su solidi irregolari. |

### Partizionamento 1-D Regolare
* **A blocchi:** Dati divisi in blocchi contigui.
* **Ciclico:** Piccoli blocchi assegnati periodicamente ai processi.

Le formule per gestire il partizionamento a blocchi di dimensione uniforme ($BLKLEN$) sono:

$$
\begin{aligned}
\text{global\_index} &= \text{local\_index} + (\text{block\_id} \times \text{BLKLEN}) \\
\text{block\_id} &=  \text{global\_index} / \text{BLKLEN} \\
\text{local\_index} &= \text{global\_index} \pmod{\text{BLKLEN}}
\end{aligned}
$$

![[5_parallel-programming-patterns_page_21.png|fix]]

---

## Bilanciamento del Carico e Granularità
Nella scelta della dimensione della partizione (granularità) si ha un'alternanza tra tempo di computazione e tempo di comunicazione/sincronizzazione. La soluzione ottima deve minimizzare il *wall-clock time* globale.

| Granularità | Descrizione                            | Pro / Contro                                                                                                                                                                    |
| :---------- | :------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Fine**    | Grande numero di piccole partizioni.   | Migliore bilanciamento del lavoro (specialmente se combinato con il [[#Master-Worker]]); tuttavia, l'overhead di scheduling/comunicazione potrebbe dominare sulla computazione. |
| **Grossa**  | Poche partizioni di grandi dimensioni. | Migliora il rapporto computazione/comunicazione, ma può causare sbilanciamenti del carico se i task hanno durate diverse.                                                       |
![[5_parallel-programming-patterns_page_27.png|fix]]

    come vediamo dal grafico spesso se il partizionamento e' troppo piccolo ci potrebbe essere un maggiore overhead legato alla schedulazione dei processi, mentre se e' troppo grande ci potrebbe essere un sbilanciamento del carico

Per migliorare il bilanciamento del lavoro in problemi irregolari, si può optare per un partizionamento a grana fine oppure per il paradigma [[#Master-Worker]].

---

## Master-Worker
Conosciuto anche come *process farm* o *work pool*, risolve il problema dello sbilanciamento del carico tramite un'assegnazione dinamica dei task.
* **Funzionamento:** I task vengono inseriti in una struttura dati (bag of tasks). Quando un "worker" si libera, preleva il primo task disponibile e lo esegue.
* **Vantaggi/Svantaggi:** Si adatta perfettamente a problemi dove i tempi di esecuzione dei task sono ignoti a priori o molto variabili (es. insieme di Mandelbrot), ma la gestione della struttura dati condivisa introduce un overhead maggiore rispetto all'assegnamento statico.

### Esempio partizionamento [insieme di mandelbrot](https://en.wikipedia.org/wiki/Mandelbrot_set)

* **Grana grossa:** Causa un forte sbilanciamento. I processi sulle aree complesse dell'insieme lavorano moltissimo ($p_1, p_2$), gli altri terminano subito e restano inattivi ($p_0, p_3$)
* **Grana fine (ciclico/statico in [[introduzione a OpenMP|OpenMP]]):** Migliora la distribuzione del lavoro spalmando i task complessi, ma non è la soluzione ottimale in questo caso.
* **Master-Worker (Assegnamento Dinamico):** È l'approccio migliore per bilanciare il carico in questo problema: i task vengono prelevati dinamicamente man mano che i processi si liberano, adattandosi ai tempi di calcolo altamente variabili.

![[5_parallel-programming-patterns_page_35.png|fix]]

* **Nota sull'Efficienza (Overhead):** Sebbene il master-worker bilanci meglio il carico, l'assegnamento statico ha un overhead minore. Il master-worker, infatti, richiede un coordinamento (sincronizzazione) per prelevare i task da una struttura dati condivisa.
## Stencil
Le computazioni stencil si applicano iterativamente su griglie (array o matrici). Il nuovo stato di ogni cella (o pixel) dipende dal suo stato precedente e dal valore delle celle nel suo intorno (es. [smoothing gaussiano](https://en.wikipedia.org/wiki/Gaussian_blur), [Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life)).

Questo pattern presenta alcune difficoltà: 

1. **Sincronismo:** Le computazioni stencil richiedono aggiornamenti sincroni. Poiché l'hardware spesso non è sincrono, si simulano gli step usando due domini: un dominio *corrente* (da cui si legge) e un dominio *successivo* (in cui si scrive l'aggiornamento). I domini vengono scambiati alla fine di ogni iterazione.

	 ![[domini_stensil.png|center]]

2. **Bordi e Ghost Cells:** I pixel sul bordo del dominio non possiedono tutti i vicini necessari. Si utilizzano le **ghost cells**, ovvero un'estensione del dominio inizializzata con valori di default oppure in modo ciclico (condizioni al contorno periodiche) per semplificare e rendere efficiente il calcolo.
	Modo di applicare le ghost cells :
	![[5_parallel-programming-patterns_page_45.png|fix]]
3. **Memoria Distribuita:** Se si partiziona il dominio su più processori (es. partizionamento a blocchi a grana grossa), ogni processo deve scambiare i valori delle proprie celle di bordo con i processi adiacenti ad ogni iterazione per aggiornare le proprie ghost cells locali.
	![[5_parallel-programming-patterns_page_57.png|fix]]
Le computazioni di tipo stensil sono [[#Embarrassingly Parallel|embarrassingly parallel]] :
```
    current domain while (!terminated) 
    { 
	    Init ghost cells 
	    Compute next domain in parallel 
	    Exchange current and next domains 
    }
```
## Reduce
La riduzione applica un operatore binario associativo (es. somma, moltiplicazione, minimo, massimo) agli elementi di un array per restituire un singolo valore scalare.
* Utilizzando un approccio parallelo ad albero, una riduzione su $n$ elementi può essere completata in $O(\log_2 n)$ passi paralleli anziché $O(n)$.

vediamo l'esempio della risoluzione della somma : 

![[5_parallel-programming-patterns_page_67.png|fix]]
## Scan
Molto simile alla riduzione, ma calcola tutti i prefissi dell'array, restituendo un array della stessa dimensione dell'input.
Esistono due tipologie:

| Tipo          | Definizione                                                                                             | Risultato (Esempio)                                                                                                       |
| :------------ | :------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------------------ |
| **Inclusiva** | Il risultato all'indice $i$ include l'elemento all'indice $i$.                                          | $y_0 = x_0$ <br> $y_1 = x_0 \text{ op } x_1$ <br> $y_{n-1} = x_0 \text{ op } x_1 \text{ op } \dots \text{ op } x_{n-1}$   |
| **Esclusiva** | Il risultato all'indice $i$ esclude l'elemento all'indice $i$, inserendo un elemento neutro all'inizio. | $y_0 = 0$ (o elemento neutro) <br> $y_1 = x_0$ <br> $y_{n-1} = x_0 \text{ op } x_1 \text{ op } \dots \text{ op } x_{n-2}$ |

Vediamo un esempio della scan:

![[5_parallel-programming-patterns_page_71.png|fix]]

Ora che abbiamo capito la logica del pattern vediamo **L'implementazione Seriale:**

```c
// Scan Inclusiva
void inclusive_scan(int *x, int *s, int n) { 
    s[0] = x[0]; 
    
    for (int i=1; i<n; i++) 
    { 
	    s[i] = s[i-1] + x[i]; // loop-carried dependence
    } 
} 

// Scan Esclusiva
void exclusive_scan(int *x, int *s, int n) { 
    s[0] = 0; 
    
    for (int i=1; i<n; i++) 
    { 
	    s[i] = s[i-1] + x[i-1]; // loop-carried dependence
	} 
}
```

L'implementazione seriale è semplice ed efficiente ($O(n)$). Tuttavia, il ciclo non può essere banalmente parallelizzato a causa della **loop-carried dependence**: ogni iterazione necessita del risultato elaborato dal passo precedente.

L'algoritmo si può parallelizzare completamente (suddividendolo in fasi di _up-sweep_ e _down-sweep_), in modo che i calcoli nei cicli interni diventino indipendenti ed eseguibili in parallelo: 

![[5_parallel-programming-patterns_page_73.png|fix]]

![[5_parallel-programming-patterns_page_74.png|fix]]

---

## Esempio pratico di Scan: Line of Sight

**Problema:** Date $n$ vette di altezze $h[0..n-1]$, determinare quali vette sono visibili dalla vetta iniziale 0. 

![[5_parallel-programming-patterns_page_85.png|fix]]

**Soluzione Seriale:**

```
bool v[0..n-1]
double a[0..n-1], amax[0..n-1]

a[0] ← -∞
for i ← 1 to n-1 do 
    a[i] ← arctan( (h[i] - h[0]) / i )     // Embarrassingly parallel

amax[0] ← -∞ 
for i ← 1 to n-1 do 
    amax[i] ← max{ a[i-1], amax[i-1] }     

for i ← 0 to n-1 do 
    v[i] ← ( a[i] ≥ amax[i] )              // Embarrassingly parallel
return v
```

**Soluzione Parallela:** Il secondo ciclo è una classica operazione di scan seriale. Sostituendolo con il pattern _Scan_, l'intero algoritmo diventa parallelo:

```
a[0] ← -∞ 
for i ← 1 to n-1 do in parallel 
    a[i] ← arctan( (h[i] - h[0]) / i ) 

amax ← exclusive-scan( max, a ) 

for i ← 0 to n-1 do in parallel 
    v[i] ← ( a[i] ≥ amax[i] ) 
return v
```

---

## List Ranking (Pointer Jumping)

**Input:** Una lista linkata. 
**Output:** Etichettare ogni nodo con la sua posizione (rango) nella lista.

**Soluzione Seriale:**

```c
typedef struct list_node { int rank; struct list_node *next; } list_node; 

void rank( list_node *item ) { 
    int r = 0; 
    while (item != NULL) { 
        item->rank = r++; 
        item = item->next; 
    } 
} 
```

L'approccio seriale è strettamente sequenziale a causa della dipendenza sul puntatore `item`.

**Soluzione Parallela (Pointer Jumping):** Si assegna un processore a ciascun nodo della lista.

- **Init:** Ogni processore imposta localmente `item->rank = 1`.
    
- **Main Loop Sincrono:** Tutti i processori eseguono i seguenti passi in parallelo ad ogni iterazione:
    
    1. **Lettura e Somma (Rank):** Se `item->next != NULL`, il processore legge `item->rank` e `item->next->rank`, poi aggiorna `item->next->rank += item->rank`.
        
    2. **Salto del puntatore (Jump):** Se `item->next != NULL`, il processore legge `item->next->next` e aggiorna il puntatore `item->next = item->next->next`.
        

Questo schema dimezza la lunghezza della lista ad ogni passo, permettendo di etichettare $n$ nodi in $O(\log_2 n)$ passi paralleli. 

![[5_parallel-programming-patterns_page_96.png|fix]]