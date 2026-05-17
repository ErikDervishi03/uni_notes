I cicli rappresentano la porzione di codice dove si concentra la maggior parte del tempo di esecuzione. Ottimizzare e parallelizzare i cicli è quindi l'obiettivo principale della programmazione ad alte prestazioni.

**Indice degli Argomenti:**

- [[#Dipendenze dei Dati (Data Dependence)]]
- [[#Vettori di Distanza e Direzione]]
- [[#Criteri di Parallelizzazione]]
- [[#Trasformazioni dei Cicli (Loop Transformations)]]

---
## Introduzione e Ottimizzazione

L'obiettivo delle trasformazioni dei cicli è preservarne la semantica migliorando al contempo le prestazioni.

- **Sistemi Single-threaded:** Si ottimizza principalmente per la gerarchia di memoria (località).
- **Sistemi Multi-threaded/Vettoriali:** Si punta alla parallelizzazione dei cicli.

Un ciclo è parallelizzabile se le sue iterazioni sono **indipendenti**, ovvero possono essere eseguite in qualsiasi ordine (anche simultaneamente) producendo lo stesso risultato della versione seriale.

---

## Dipendenze dei Dati (Data Dependence)

Esiste una dipendenza tra due istruzioni $S_1$ e $S_2$ se entrambe accedono alla stessa locazione di memoria e almeno una delle due è una scrittura.

### Tipi di Dipendenze

Sia $S_1$ eseguita prima di $S_2$ nell'ordine seriale:

| **Tipo di Dipendenza** | **Nome Comune**         | **Descrizione**                                       | **Simbolo**           |
| ---------------------- | ----------------------- | ----------------------------------------------------- | --------------------- |
| **Flow Dependence**    | Read-After-Write (RAW)  | $S_1$ scrive un valore che $S_2$ legge.               | $S_1 \rightarrow S_2$ |
| **Anti Dependence**    | Write-After-Read (WAR)  | $S_1$ legge un valore prima che $S_2$ lo sovrascriva. | $S_1 \delta^{-1} S_2$ |
| **Output Dependence**  | Write-After-Write (WAW) | $S_1$ e $S_2$ scrivono nella stessa locazione.        | $S_1 \delta^o S_2$    |


### Loop-Carried Dependence (LCD)

Una dipendenza si dice **trasportata dal ciclo** se l'accesso alla memoria avviene in iterazioni differenti. Se la dipendenza esiste solo all'interno della stessa iterazione, è detta _loop-independent_.

- Le LCD impediscono la parallelizzazione banale del ciclo che le trasporta.

---

## Vettori di Distanza e Direzione

Per analizzare le dipendenze in cicli annidati, si utilizzano i vettori.

1. **Vettore di Distanza ($d$):** Rappresenta il numero di iterazioni che intercorrono tra la causa e l'effetto della dipendenza.

    - $d = I_{target} - I_{source}$
    
2. **Vettore di Direzione ($D$):** Indica il segno della distanza per ogni livello del ciclo.
    - `'<'` se la distanza è positiva (dipendenza in iterazioni future).
    - `'='` se la distanza è zero (dipendenza loop-independent per quel livello).
    - `'>'` se la distanza è negativa (impossibile in un ordine di esecuzione sequenziale valido).

**Esempio:**

```c
for (i=1; i<N; i++)
  for (j=1; j<N; j++)
    A[i][j] = A[i-1][j+1] + 1;
```

- Sorgente: `A[i-1][j+1]` (lettura), Destinazione: `A[i][j]` (scrittura).
- Distanza: $d = (i - (i-1), j - (j+1)) = (1, -1)$
- Direzione: $D = (<, >)$

---

## Criteri di Parallelizzazione

- Un ciclo a un determinato livello $k$ può essere parallelizzato se non trasporta alcuna dipendenza (ovvero, per ogni dipendenza, la $k$-esima componente del vettore di distanza è $0$).
- Se un ciclo trasporta una dipendenza, le sue iterazioni devono essere eseguite sequenzialmente per preservare la correttezza.

---

## Trasformazioni dei Cicli (Loop Transformations)

Le trasformazioni cambiano l'ordine di esecuzione delle iterazioni per abilitare la parallelizzazione o migliorare la località.

### 1. Loop Permutation (Interchange)

Scambia l'ordine dei cicli annidati (es. il ciclo `i` diventa quello interno e `j` quello esterno).

- **Validità:** È legale se il nuovo ordine non inverte la direzione di alcuna dipendenza (non deve apparire `>` come prima direzione non nulla).
- **Scopo:** Portare i cicli che trasportano dipendenze all'interno e quelli indipendenti all'esterno.

### 2. Loop Fission (Distribution)

Divide un singolo ciclo in più cicli distinti, ciascuno contenente una parte del corpo originale.

- Utile per isolare istruzioni con dipendenze critiche.

### 3. Loop Fusion

L'opposto della fissione: unisce due cicli adiacenti con gli stessi limiti in un unico ciclo.

- Migliora la località dei dati e riduce l'overhead del ciclo.

### 4. Loop Tiling (Blocking)

Suddivide lo spazio delle iterazioni in "mattonelle" (tiles) per far sì che i dati rimangano nella cache durante l'elaborazione del blocco.

