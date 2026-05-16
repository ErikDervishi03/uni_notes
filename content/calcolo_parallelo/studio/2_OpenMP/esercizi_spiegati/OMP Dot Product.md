Link alla consegna : [qua](https://www.moreno.marzolla.name/teaching/parallel-etudes/handouts/omp-dot.html) 
## 1. Consegna dell'Esercizio

Il programma calcola il prodotto scalare di due array `v1[]` e `v2[]` di lunghezza $n$. La lunghezza $n$ viene passata come parametro da riga di comando. Gli array sono inizializzati in modo deterministico, così da conoscere il risultato atteso senza doverlo calcolare esplicitamente.

Il prodotto scalare è definito come:

$$\sum_{i = 0}^{n-1} v1[i] \times v2[i]$$

**Obiettivo:** Parallelizzare il programma seriale.

L'esercizio suggerisce due approcci:

1. **Metodo didattico (Partizionamento manuale):** Iniziare senza usare `omp parallel for`. Partizionare gli array in $P$ blocchi (dove $P$ è il numero di thread). Ogni thread calcola il prodotto scalare parziale della sua porzione e lo salva in un array condiviso `partial_p[]` di dimensione $P$. Alla fine, il master thread somma gli elementi di `partial_p[]`.
2. **Metodo efficiente (Reduction):** Usare la direttiva `omp parallel for` combinata con la clausola `reduction()` per far gestire l'accumulo in modo automatico ed efficiente al compilatore.

---

## 2. Codice Soluzione

Di seguito il codice completo. 

l'uso di `atomic` dentro un ciclo iterato milioni di volte crea un "collo di bottiglia" spaventoso sulle prestazioni, rendendo il programma parallelo probabilmente molto più lento di quello seriale. La soluzione con `reduction` è quella corretta.


```c
/****************************************************************************
 *
 * omp-dot.c - Dot product
 *
 ****************************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <omp.h>

void fill( int *v1, int *v2, size_t n )
{
    const int seq1[3] = { 3, 7, 18};
    const int seq2[3] = {12, 0, -2};
    for (size_t i=0; i<n; i++) {
        v1[i] = seq1[i%3];
        v2[i] = seq2[i%3];
    }
}

int dot(const int *v1, const int *v2, size_t n)
{
    /* This version uses neither "parallel for" nor "reduction"
       directives; although this solution should not be used in
       practice, it is instructive to try it. */
    const int P = omp_get_max_threads();
    int partial_p[P];
#pragma omp parallel default(none) shared(P, v1, v2, n, partial_p)
    {
        const int my_id = omp_get_thread_num();
        const size_t my_start = (n * my_id) / P;
        const size_t my_end = (n * (my_id + 1)) / P;
        int my_p = 0;
        for (size_t j=my_start; j<my_end; j++) {
            my_p += v1[j] * v2[j];
        }
        partial_p[my_id] = my_p;
    } /* implicit barrier here */

    /* we are outside a parallel region, so what follows is done by
       the master only */
    int result = 0;
    for (int i=0; i<P; i++) {
        result += partial_p[i];
    }
    return result;
}

/*versione efficiente*/ 
int dot_reduction(const int *v1, const int *v2, size_t n)
{
    int result = 0;
    #pragma omp parallel for default(none) shared(n, v1, v2) reduction(+:result)
    for (size_t i=0; i<n; i++) {
        result = result + (v1[i] * v2[i]);
    }
    return result;
}

int main( int argc, char *argv[] )
{
    size_t n = 10*1024*1024l; /* array length */
    const size_t n_max = 512*1024*1024l; /* max length */
    int *v1, *v2;

    if ( argc > 2 ) {
        fprintf(stderr, "Usage: %s [n]\n", argv[0]);
        return EXIT_FAILURE;
    }

    if ( argc > 1 ) {
        n = atol(argv[1]);
    }

    if ( n > n_max ) {
        fprintf(stderr, "FATAL: Array too long (requested length=%lu, maximum length=%lu\n", (unsigned long)n, (unsigned long)n_max);
        return EXIT_FAILURE;
    }

    printf("Initializing array of length %lu\n", (unsigned long)n);
    v1 = (int*)malloc( n*sizeof(v1[0])); assert(v1 != NULL);
    v2 = (int*)malloc( n*sizeof(v2[0])); assert(v2 != NULL);
    fill(v1, v2, n);

    const int expect = (n % 3 == 0 ? 0 : 36);

    const double tstart = omp_get_wtime();
    const int result = dot(v1, v2, n);
    const double elapsed = omp_get_wtime() - tstart;

    if ( result == expect ) {
        printf("Test OK\n");
    } else {
        printf("Test FAILED: expected %d, got %d\n", expect, result);
    }
    printf("Execution time %.3f\n", elapsed);
    free(v1);
    free(v2);

    return EXIT_SUCCESS;
}
```

---

## 3. Approfondimenti Teorici

### Perché `atomic` è lento per il calcolo di una somma su un array?

Quando applichi la direttiva `#pragma omp atomic update` (o `critical`) all'interno di un ciclo molto lungo per aggiornare una variabile globale `result`, stai dicendo all'hardware: _"Fai in modo che solo un thread alla volta possa leggere, sommare e scrivere su questa locazione di memoria"_.

Questo crea una grave **contesa delle risorse (contentions)**. I thread calcolano il prodotto molto velocemente, ma poi devono mettersi "in fila indiana" per aggiornare la variabile globale `result`. Il tempo perso ad aspettare il proprio turno per accedere alla memoria rende il codice parallelo spesso più lento dell'equivalente sequenziale (dove un solo core aggiorna la propria cache senza barriere o locking).

### Per Compilare

```bash
gcc -fopenmp -std=c99 -Wall -Wpedantic omp-dot.c -o omp-dot

OMP_NUM_THREADS=2 ./omp-dot 1000000
```