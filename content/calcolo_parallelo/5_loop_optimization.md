I cicli rappresentano la porzione di codice dove si concentra la maggior parte del tempo di esecuzione di un programma. Ottimizzare e parallelizzare i cicli è quindi l'obiettivo principale della programmazione ad alte prestazioni (High Performance Computing).

## Introduzione

In certi casi, le iterazioni dei cicli sono completamente **indipendenti** l'una dall'altra.

```c
for (i = 0; i < N; i++) { 
    foo(i); 
}
```

**Domanda:** Sarebbe possibile assegnare a unità di esecuzione diverse le iterazioni di questo ciclo?

**Risposta:** In questo caso sì. Essendo le iterazioni indipendenti, possiamo parallelizzare il ciclo facilmente utilizzando direttive come `#pragma omp parallel for`.

---

## Tipi di Dipendenze 
Si ha una _data dependence_ (dipendenza sui dati) tra due accessi alla memoria se **almeno uno dei due è in scrittura** e si riferiscono alla **stessa locazione di memoria**.

Esistono diversi tipi di dipendenze:

1. **Data-Flow (o True dependence) – RAW (Read After Write):** Una variabile viene prima scritta e in seguito letta.
2. **Anti dependence – WAR (Write After Read):** Una variabile viene prima letta e in seguito sovrascritta.
3. **Output dependence – WAW (Write After Write):** Una variabile viene scritta e successivamente sovrascritta.
    

Oltre alle dipendenze sui dati, esiste la:

- **Control dependence:** Si verifica su un'istruzione `S1` se il risultato di `S1` determina se l'istruzione `S2` verrà eseguita o meno. Naturalmente, `S1` e `S2` non possono essere scambiate di ordine. Questo tipo di dipendenza si applica tipicamente alle condizioni di un costrutto `if-then-else` o a un ciclo rispetto al proprio corpo.
  
```c
	if (a > 0) { // S1
		a = 2 * c; // S2
	} else { 
		b = 3; 
	}
```
    

_Nota:_ Ai fini pratici della nostra analisi per la parallelizzazione, tratteremo tutte le dipendenze in modo simile.

**Notazione:** Se c'è una dipendenza dall'istruzione S1 all'istruzione S2, scriveremo **`S1 -> S2`**.

### Teorema fondamentale delle dipendenze

> Qualsiasi trasformazione di riordino che preserva tutte le dipendenze in un programma, preserva il significato e il risultato di quel programma.

Di conseguenza, se vogliamo parallelizzare un loop, dobbiamo verificare se ci sono _data dependence_. Nel caso non ci siano (o se riusciamo a gestirle/eliminarle), possiamo procedere alla parallelizzazione.

---

## Analisi: Esempi di Dipendenze

**Esempio 1: Iterazioni indipendenti**

```c
for (i = 0; i < n; i++) {
    S1: a[i] = b[i] + c[i];
}
```

![[Screenshot_20260519_105113.png]]

In questo caso ogni iterazione è indipendente dalle altre. Il ciclo è **parallelizzabile**.

**Esempio 2: Loop-carried dependence 

```c
for (i = 1; i < n; i++) {
    S1: a[i] = a[i-1] + b[i];
}
```

![[Screenshot_20260519_105138.png]]

Qui il problema è che ogni iterazione dipende dal risultato dell'iterazione precedente (viene letto `a[i-1]` appena scritto). Si tratta di una dipendenza trasportata dal ciclo (_loop-carried dependence_). **Non banalmente parallelizzabile**.

**Esempio 3: Variabile di accumulo 

```c
s = 0; 
for (i = 0; i < n; i++) { 
    S1: s = s + a[i]; 
}
```

![[Screenshot_20260519_105455.png]]

Qui la variabile `s` viene sia letta che scritta ad ogni iterazione, creando una _loop-carried dependence_. Tuttavia, trattandosi di un'operazione di accumulo, in questo caso specifico basta utilizzare una clausola di **riduzione** (`reduction(+:s)`) per parallelizzarlo in sicurezza.

**Esempio 4: Dipendenze incrociate complesse**


```c
for (i = 2; i < n; i++) {
    S1: a[i] = 4 * c[i-1] - 2;
    S2: b[i] = a[i] * 2;
    S3: c[i] = a[i-1] + 3;
    S4: d[i] = b[i] + c[i-2];
}
```

![[Screenshot_20260519_110838.png]]

---

## Tecniche per Eliminare le Dipendenze 

### 1. Allineamento

A volte, traslando gli indici, è possibile rimuovere le dipendenze tra un'iterazione e l'altra.

**Codice originale:**

```c
a[0] = 0; 
for (i = 1; i < n; i++) { 
    S1: a[i] = b[i-1] * c[i]; 
    S2: d[i] = a[i-1] + 2; 
} 
```

Notiamo che `S1` di un'iterazione produce il valore `a[i]` che verrà letto da `S2` nell'iterazione successiva (come `a[i-1]`).

**Codice allineato:**
```c
a[0] = 0; 
d[1] = a[0] + 2; 
for (i = 1; i < n-1; i++) { 
    T1: a[i] = b[i-1] * c[i]; 
    T2: d[i+1] = a[i] + 2; 
} 
a[n-1] = b[n-2] * c[n-1];
```

In questo modo le dipendenze _tra_ iterazioni diverse si sono trasformate in dipendenze _all'interno della stessa_ iterazione, sbloccando potenziali parallelizzazioni.

![[Screenshot_20260519_111656.png]]

### 2. Loop Fission

In alcuni casi, le dipendenze possono essere rimosse spezzando il ciclo in due e utilizzando un array temporaneo di appoggio per copiare i dati.

![[Screenshot_20260519_112432.png]]

### 3. Loop Fusion

L'opposto della fissione. A volte è possibile e conveniente unire cicli multipli parallelizzabili per ridurre l'overhead (costo di gestione del ciclo).

**Codice originale (due cicli):**

```c
for (i = 0; i < n; i++) { 
    a[i] = b[i] * c[i]; 
} 
for (i = 0; i < n; i++) { 
    b[i] = f(d[i]) * h[i]; 
} 
```

**Loop Fusion:**

```c
for (i = 0; i < n; i++) { 
    a[i] = b[i] * c[i]; 
    b[i] = f(d[i]) * h[i]; 
}
```

---

## Dipendenze Difficili 

Consideriamo il seguente doppio ciclo annidato:

```c
for (i = 1; i < n; i++) { 
    for (j = 1; j < m; j++) { 
        a[i][j] = a[i-1][j-1] + a[i-1][j] + a[i][j-1]; 
    } 
}
```

- Il ciclo esterno **non è parallelizzabile** a causa delle dipendenze.
    
    ![[Screenshot_20260519_113034.png]]
    
- Anche il ciclo interno **non è parallelizzabile**.
    ![[Screenshot_20260519_113051.png]]

**Soluzione: Scansione Diagonale**

Notiamo che non ci sono frecce di dipendenza lungo le diagonali della matrice. È quindi possibile parallelizzare l'esecuzione attraversando la matrice diagonalmente (onda o _wavefront_).

```c
for (slice = 0; slice < n + m - 1; slice++) { 
    z1 = slice < m ? 0 : slice - m + 1; 
    z2 = slice < n ? 0 : slice - n + 1; 
    
    /* Questo ciclo for interno PUÒ essere parallelizzato */ 
    for (i = slice - z2; i >= z1; i--) { 
        j = slice - i; 
        /* processa a[i][j] … */ 
    } 
}
```
---

## Esercizi e Casi di Studio

### Esercizio 1

![[Pasted image 20260519120137.png]]

_Nota:_ Un modo efficace per calcolare operazioni con dipendenze sequenziali strutturate (come somme prefisse) è usare l'algoritmo di **scan esclusiva**.

### Esercizio 2

```c
#define N some_big_number 
int s[N] = {1, 1, ... 1}; 
double p[N] = { ... }; 
/* Assume that function f() has no side effects */ 

for (int i = 0; i < N; i++) { 
    if (s[i]) { 
        for (int j = 0; j < N; j++) { 
            if (s[j] && f(p[i], p[j])) { 
                s[j] = 0; 
            } 
        } 
    } 
}
```

**Analisi:** * Il **ciclo esterno non è parallelizzabile** perché c'è una potenziale _race condition_ (condizione di corsa) tra la scrittura `s[j] = 0;` (generata in un'iterazione) e la lettura `if (s[i])` (valutata in un'altra iterazione).

- Il **ciclo interno è parallelizzabile** (per un dato `i` fissato, le iterazioni su `j` possono essere svolte in parallelo proteggendo o gestendo le scritture concorrenti su `s[j]`).
    

### Esercizio 3

```c
#define N 10000 
double phi[2][N][N], maxdelta; 
const double EPS = 1.0e-6; 
int cur = 0, next = 1; 
/* ...Initializations not shown... */ 

do { 
    maxdelta = 0.0; 
    for (int i = 1; i < N-1; i++) { 
        for (int j = 1; j < N-1; j++) { 
            phi[next][i][j] = (phi[cur][i+1][j] + phi[cur][i-1][j] + phi[cur][i][j+1] + phi[cur][i][j-1]) / 4; 
            const double delta = fabs(phi[next][i][j] - phi[cur][i][j]); 
            if (delta > maxdelta) { 
                maxdelta = delta; 
            } 
        } 
    } 
    /* exchange “cur” and “next” */ 
    const int tmp = cur; 
    cur = next; 
    next = tmp; 
} while (maxdelta > EPS); 
```

**Analisi:**

In questo frammento stiamo facendo una computazione di tipo [[calcolo parallelo/studio/1_teoria_e_architetture/4_parallel-programming-patterns#Stencil|stencil]]. Una matrice è di sola lettura (`phi[cur]`) e una è di sola scrittura (`phi[next]`). Essendo spazi di memoria separati, **non ci sono race condition** sugli array. L'unico problema è il valore `delta` che deve aggiornare `maxdelta` (variabile globalmente visibile), ma il problema è facilmente risolvibile con una **max reduction** (`reduction(max: maxdelta)`).

**Quale ciclo parallelizzare?**

Teoricamente possiamo parallelizzare qualunque ciclo dei due `for` annidati, ma qual è l'approccio migliore?

- **Ciclo interno:** _Meglio di no._ Ad ogni iterazione del ciclo esterno si creerebbe e distruggerebbe (o sospenderebbe) la regione parallela (fork-join), introducendo un overhead enorme e ingiustificato.
    
- **Ciclo esterno:** _Altamente consigliato._ È perfettamente parallelizzabile e, utilizzando uno _scheduling statico_, non introduciamo overhead continuo e gestiamo bene eventuali sbilanciamenti di carico.
    
- **Collapse (`#pragma omp parallel for collapse(2)`):** _Fattibile._ Si può usare per ampliare lo spazio delle iterazioni, ma il calcolo matematico per mappare l'indice lineare sugli indici originari `i` e `j` introduce un overhead di calcolo non indifferente che potrebbe degradare le prestazioni.