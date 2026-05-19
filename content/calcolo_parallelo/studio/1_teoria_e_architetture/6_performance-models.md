**Indice degli Argomenti:**
* [[#Il Modello Work/Span]]
* [[#Metriche Derivate: Speedup ed Efficienza]]
* [[#Esempi Dettagliati di Analisi Work/Span]]
* [[#Analisi Empirica (Misurazione del Wall-Clock Time)]]
* [[#Strong Scaling vs Weak Scaling ed Esempi di Codice]]
* [[#Limiti Teorici e Legge di Amdahl]]

La **scalabilità** quantifica l'efficienza della parallelizzazione: avendo a disposizione un determinato budget di tempo, misura quanto lavoro aggiuntivo è possibile completare aumentando le unità di esecuzione rispetto a usarne una sola.

---

## Il Modello Work/Span
Questo modello serve per analizzare le prestazioni teoriche, assumendo un sistema a memoria condivisa con $p$ unità di esecuzione indipendenti e asincrone.

**Variabili di base:**
* $n$: dimensione dell'input.
* $p$: numero di unità di esecuzione (processori/core).
* $T_p(n)$: *Wall-Clock Time* (tempo reale trascorso) del programma eseguito con $p$ processori.

| Metrica | Definizione e Formula | Spiegazione Dettagliata |
| :--- | :--- | :--- |
| **Work (Lavoro)** | $T_1(n)$ | Numero totale di operazioni eseguite da tutte le unità. Corrisponde al tempo di esecuzione del programma se eseguito da un solo processore in modo sequenziale. |
| **Span (Cammino Critico)** | $T_\infty(n)$ | Il tempo che il programma impiegherebbe se avesse a disposizione **infiniti processori**. Rappresenta la catena più lunga di dipendenze sequenziali (il *cammino critico*). Il tempo di esecuzione è limitato e dominato da questo percorso incompressibile. |
| **Costo** | $p \times T_p(n)$ | Tempo totale macchina allocato. Se affitti $p$ nodi su un cluster cloud per $T_p$ secondi, pagherai per $p \times T_p(n)$, indipendentemente dal fatto che alcuni processori siano rimasti inattivi in attesa degli altri. |
| **Overhead** | $p \times T_p(n) - T_1(n)$ | Tempo macchina "sprecato". È la differenza tra il costo totale e il lavoro effettivamente utile. È causato da sincronizzazioni, attese, creazione dei thread o sbilanciamenti del carico. |

![[8_performance-models_page_10.png|fix]]
![[8_performance-models_page_11.png|fix]]
![[8_performance-models_page_12.png|fix]]
![[8_performance-models_page_13.png|fix]]


esempio di cammino critico:

![[8_performance-models_page_8.png|fix]]

### Leggi di work/span
1.  **Legge del Lavoro:** $p \times T_p(n) \ge T_1(n)$. Significa che l'overhead è sempre una grandezza positiva. Lavorare in parallelo introduce sempre un costo aggiuntivo di gestione.
2.  **Legge dello Span:** $T_p(n) \ge T_\infty(n)$. Anche se hai un miliardo di processori, non potrai mai risolvere il problema in un tempo inferiore al suo cammino critico strettamente sequenziale.

---

## Metriche Derivate: Speedup ed Efficienza

| Metrica | Formula | Descrizione e Limiti |
| :--- | :--- | :--- |
| **Speedup** | $S(p, n) = \frac{T_1(n)}{T_p(n)}$ | Indica quante volte è più veloce il programma parallelo rispetto a quello seriale. Dalla Legge del Lavoro ricaviamo che $S(p, n) \le p$. Se l'overhead fosse zero, lo speedup sarebbe esattamente $p$ (Speedup ideale o lineare). |
| **Efficienza** | $E(p, n) = \frac{S(p, n)}{p} = \frac{T_1(n)}{p \times T_p(n)}$ | Misura quanto bene stiamo sfruttando le risorse di calcolo fornite. Poiché $S(p, n) \le p$, si ha che $E(p, n) \le 1$. Più è vicina a 1, meno risorse stiamo sprecando in overhead. |

**Lo Speedup Superlineare ($S(p) > p$)**
Sebbene matematicamente l'efficienza non dovrebbe superare 1, in rari casi pratici si osserva uno speedup maggiore di $p$. Questo fenomeno si chiama *speedup superlineare* e accade per:
1.  **Effetti di Cache:** Dividendo i dati su $p$ processori, le partizioni locali diventano così piccole da entrare interamente nelle velocissime memorie Cache (L1/L2), evitando i lenti accessi alla RAM principale.
2.  **Eterogeneità Hardware:** I processori aggiunti potrebbero avere architetture specifiche più veloci per quel task.
3.  **Algoritmi di Ricerca (es. Brute-force):** In alberi di decisione, esplorare in parallelo potrebbe far trovare a un thread la soluzione (e interrompere tutti gli altri) molto prima di quanto farebbe un attraversamento sequenziale deterministico.

---
## Esempi Dettagliati di Analisi Work/Span

### Esempio 1: Riduzione con Task (Approccio ad Albero Divide et Impera)
Vogliamo sommare gli elementi di un array dividendo ricorsivamente l'array a metà e assegnando ogni metà a un task.

```c
float SumReduce( const float v[], int i, int j ) { 
    if (i > j) return 0; 
    else if (i == j) return v[i]; 
    else { 
        const int m = (i + j) / 2; 
        int s1, s2; 
        #pragma omp task shared(s1) 
        s1 = SumReduce(v, i, m); 
        #pragma omp task shared(s2) 
        s2 = SumReduce(v, m+1, j); 
        #pragma omp taskwait 
        return s1 + s2; 
    } 
}

/* Invocazione dal main */
float s; 
float *v = ... 
#pragma omp parallel 
#pragma omp single 
s = SumReduce(v, 0, n-1); 
````

**Analisi Intuitiva:**
![[Screenshot_20260507_122133.png]]
Facciamo un'analisi assumendo che $p = n$

- **Work** $T_1(n)$: 1+ 2 + 4 + ... + n = $O(n)$.
- **Span $T_\infty(n)$:** altezza dell'albero = $O(\log_2 n)$.
- **Speedup Massimo:** $\frac{T_1(n)}{T_\infty(n)} = \frac{O(n)}{O(\log_2 n)}$
- **Efficienza con $n$ processori:** $\frac{\text{Speedup}}{p} = O\left(\frac{1}{\log_2 n}\right)$

**Analisi Accurata:**
Facciamo un'analisi accurata assumendo che $p = \infty$

- **Work $T_1(n)$:** Il programma seriale deve eseguire $(n-1)$ somme per $n$ elementi. Il lavoro totale si esprime con la ricorrenza $W(n) = 2 W(n/2) + O(1)$, che si risolve in $O(n)$.
- **Span $T_\infty(n)$:** Avendo infiniti processori, ogni livello dell'albero di ricorsione viene eseguito in parallelo. Il cammino critico è l'altezza dell'albero. La ricorrenza è $S(n) = S(n/2) + O(1)$, che si risolve in $O(\log_2 n)$.
- **Speedup Massimo:** $\frac{O(n)}{O(\log_2 n)}$
- **Efficienza con $n$ processori:** $\frac{\text{Speedup}}{p} = O\left(\frac{1}{\log_2 n}\right)$

### Esempio 2: Riduzione Lineare (For Loop in OpenMP)

Vediamo un approccio iterativo classico, più facile da scrivere ma con dinamiche diverse.

```c
float SumReduce( const float v[], int n ) { 
    float result = 0.0; 
    #pragma omp parallel for reduction(+:result) 
    for (int i=0; i<n; i++) { 
        result += v[i]; 
    } 
    return result; 
} 
```

**Analisi Realistica (Assumendo $p \ll n$, scheduling statico e riduzione in $O(1)$):**

OpenMP dividerà l'array in blocchi di dimensione $n/p$. Ogni thread somma il suo blocco localmente (lavoro $\approx n/p$), poi OpenMP combina i $p$ risultati parziali alla fine del ciclo for.

![[Screenshot_20260518_210133.png]]

- **Work $T_1(n)$:** $p(n/p) + O(1) = n$. la somma delle zone blu.
- **Span $T_p(n)$:**  $\frac{n}{p} + O(1) = \frac{n}{p}$. l'altezza del primo blocco.
- **Speedup:** $\frac{T_1(n)}{T_p(n)} = p$
- **Efficienza:** $\frac{S_p(n)}{p} = 1$.

**Analisi Realistica (Assumendo $p \ll n$, scheduling statico e riduzione in $O(\log_2 p)$):**

![[Screenshot_20260518_211609.png]]

- **Work $T_1(n)$:** $n + p = n$. la somma delle zone blu.
- **Span $T_p(n)$:**  $\frac{n}{p} + \log p = \frac{n + p\log p}{p} = \frac{n}{p}$. se $p\log p = O(n)$ l'altezza del primo blocco.
- **Speedup:** $\frac{T_1(n)}{T_p(n)} = p$
- **Efficienza:** $\frac{S_p(n)}{p} = 1$.

---
## Analisi Empirica (Misurazione del Wall-Clock Time)

La teoria non sempre rispecchia i colli di bottiglia reali del sistema operativo. Per calcolare Speedup ed Efficienza empiricamente, fissiamo una dimensione $n$ sufficientemente grande e misuriamo $T_p(n)$ variando i processori.

**Strumenti per misurare il tempo:**

- **OpenMP:** `omp_get_wtime()`
- **MPI:** `MPI_Wtime()`
- **Soluzione C standard:** `clock_gettime()`
- **Soluzione Agnostica:** Usare un header personalizzato (es. `hpc.h`) per standardizzare le chiamate tra MPI, CUDA e OpenMP:

```c
    #if _XOPEN_SOURCE < 600 
    #define _XOPEN_SOURCE 600 
    #endif 
    #include "hpc.h"
    
    double start = hpc_gettime(); 
    /* codice da misurare */ 
    double finish = hpc_gettime(); 
```

Non usare la funzione ```clock()``` che misura la somma dei tempi impiegati da tutti i thread dall'inizio alla fine

---

## Strong Scaling vs Weak Scaling ed Esempi di Codice

Esistono due prospettive per valutare la scalabilità di un sistema:

### 1. Strong Scaling (Ridurre il tempo)

Si mantiene **costante la dimensione totale del problema ($n$)** e si aumenta progressivamente il numero di processori ($p$). L'obiettivo è misurare quanto velocemente risolviamo lo stesso identico problema.

$$E(p) = \frac{S(p)}{p} = \frac{T_1(n)}{p \times T_p(n)}$$
### 2. Weak Scaling (Aumentare la complessità)

Si aumenta $p$, ma per bilanciare si **aumenta anche la dimensione del problema ($n_p$)**, in modo che la _quantità di lavoro assegnata a ciascun singolo processore rimanga costante_. L'obiettivo è misurare se il sistema è in grado di risolvere problemi più grandi senza degradare in prestazioni a causa dell'overhead di comunicazione.

$$W(p) = \frac{T_1(n_1)}{T_p(n_p)}$$

Per calcolare il Weak Scaling, definiamo $f(n_p, p)$ come la **quantità di lavoro svolto da ciascun processore**. Vogliamo mantenere questa quantità costante ($f(n_p, p) = cost$). Vediamo come deve crescere la dimensione dell'input $n_p$ in tre casi pratici.

#### Caso 1: Array Lineare 1D

```c
#pragma omp parallel for 
for (int i=0; i<n; i++) 
    c[i] = a[i] + b[i];
```

Il lavoro per processore è la dimensione diviso i processori.

- $f(n_p, p) = \frac{n_p}{p}$
- Affinché $\frac{n_p}{p} = cost$, ricaviamo che $n_p = p \times cost$.
- **Conclusione:** L'input deve crescere in modo **direttamente proporzionale** al numero di processori.    
#### Caso 2: Somma tra Matrici 2D

```c
#pragma omp parallel for collapse(2)
for (int i=0; i<n; i++) { 
    for (int j=0; j<n; j++) { 
        C[i][j] = A[i][j] + B[i][j];
    }
}
```

Nota: `collapse(2)` è eccellente qui perché non ci sono dipendenze e distribuisce perfettamente i loop annidati. Il numero totale di operazioni è $n_p^2$.

- $f(n_p, p) = \frac{n_p^2}{p}$
- Affinché $\frac{n_p^2}{p} = cost$, ricaviamo che $n_p^2 = p \times cost$, ovvero $n_p = \sqrt{p} \times cost'$.
- **Conclusione:** La dimensione del lato della matrice deve crescere in modo **proporzionale alla radice quadrata** del numero di unità di esecuzione.

#### Caso 3: Moltiplicazione tra Matrici 3D (Attenzione alle Race Condition)

```c
// Collassiamo solo i primi due cicli!
#pragma omp parallel for collapse(2) 
for (int i=0; i<n; i++) 
    for (int j=0; j<n; j++) 
        for (int k=0; k<n; k++) 
            C[i][j] += A[i][k] * B[k][j];
```

_Spiegazione sul perché NON collassare tutti e 3 i cicli:_ Se facessimo `collapse(3)`, assegneremmo combinazioni di indici tridimensionali a thread indipendenti. Potremmo avere il Thread 1 che esegue `(i=0, j=0, k=3)` e il Thread 2 che esegue `(i=0, j=0, k=7)`. Entrambi cercherebbero di scrivere e aggiornare simultaneamente la cella `C[0][0]`, causando una **Race Condition**. Collassando solo `i` e `j`, garantiamo che ogni cella `C[i][j]` sia posseduta e aggiornata da un unico thread.

Analizziamo la crescita del lavoro: Il numero totale di iterazioni è $n_p^3$.

- $f(n_p, p) = \frac{n_p^3}{p}$
- Affinché $\frac{n_p^3}{p} = cost$, ricaviamo che $n_p^3 = p \times cost$, ovvero $n_p = \sqrt[3]{p} \times cost'$.
- **Conclusione:** L'input (il lato $n$) deve crescere in modo **proporzionale alla radice cubica** del numero di processori.

---

## Limiti Teorici e Legge di Amdahl

Giudicare un algoritmo guardando unicamente il grafico dello Speedup può essere molto fuorviante. Un algoritmo che "scala" in modo perfettamente lineare su 1000 core potrebbe comunque avere un Wall-Clock Time di ordini di grandezza superiore rispetto a un algoritmo sequenziale super ottimizzato. Valuta sempre il tempo reale ($T_p$) contestualmente allo Speedup.

![[8_performance-models_page_38.png|fix]]
### La Legge di Amdahl

La Legge di Amdahl stabilisce il limite invalicabile di accelerazione di un programma in base alle sue porzioni non parallelizzabili.

Supponiamo che una frazione $\alpha$ del tempo totale del programma seriale **non possa essere parallelizzata**. Questo accade per:

- Limitazioni dell'algoritmo stesso (es. loop-carried dependencies).
- Risorse strettamente condivise (es. scrittura sequenziale su file, I/O).
- Overhead intrinseci di avvio/chiusura thread o calcolo del partizionamento.
- Costi di comunicazione incolmabili.

La restante frazione $(1 - \alpha)$ si assume invece perfettamente parallelizzabile su $p$ processori.

**Formula del Tempo Parallelo:**

$$T_{parallel}(p) = \alpha \cdot T_{serial} + \frac{(1-\alpha) \cdot T_{serial}}{p}$$

**Formula dello Speedup (Legge di Amdahl):**

$$S(p) = \frac{T_{serial}}{T_{parallel}(p)} = \frac{T_{serial}}{\alpha \cdot T_{serial} + \frac{(1-\alpha) \cdot T_{serial}}{p}} = \frac{1}{\alpha + \frac{1-\alpha}{p}}$$

![[8_performance-models_page_46.png|fix]]

![[8_performance-models_page_47.png|fix]]

**Il Limite Asintotico:**

Cosa succede allo speedup se continuiamo a comprare hardware e facciamo tendere i processori a infinito ($p \to \infty$)?

Il termine $\frac{1-\alpha}{p}$ tenderà a zero. Lo Speedup Massimo sarà quindi inchiodato al valore:

$$S_{\max} = \frac{1}{\alpha}$$

**Esempio :** Se solo il **5%** ($\alpha = 0.05$) del tuo programma è seriale e il restante 95% è parallelizzato alla perfezione, non importa se tu abbia un computer quad-core o il supercomputer più potente del pianeta con milioni di core: **il tuo programma non andrà mai più di 20 volte più veloce** ($1 / 0.05 = 20$).

_Nota pratica:_ Dal punto di vista reale, all'aumentare dei processori $p$, i tempi spesi nella comunicazione (overhead) tendono ad aumentare e saturare i bus di sistema. Di conseguenza, la curva empirica dello speedup non solo raggiunge l'asintoto di Amdahl, ma dopo un certo picco inizia addirittura a scendere (degrado delle prestazioni).


