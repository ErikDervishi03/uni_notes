## 1. Consegna dell'Esercizio

Il programma contiene un messaggio cifrato memorizzato nell'array `enc[]` di lunghezza 64. Il messaggio è stato cifrato usando l'algoritmo _XOR_.

L'algoritmo _XOR_ è simmetrico, il che significa che la stessa chiave viene usata per cifrare e decifrare. La funzione `xorcrypt(in, out, n, key, keylen)` cifra o decifra un messaggio.

Sappiamo che:

- Il testo in chiaro è una stringa ASCII terminata da zero che inizia con `0123456789`.
    
- La chiave di cifratura è una sequenza di 8 caratteri numerici ASCII (una stringa da `"00000000"` a `"99999999"`).
    

**Obiettivo:** Scrivere un programma per forzare la chiave (brute-force) usando OpenMP. Il programma deve provare ogni chiave finché non viene trovato un messaggio valido.

**Vincoli OpenMP:**

Il ciclo principale non può essere parallelizzato con il costrutto `omp for` (perché?). Pertanto, dobbiamo usare `omp parallel` e partizionare manualmente lo spazio delle chiavi tra i thread. Poiché `omp parallel` si applica a un blocco strutturato, il thread che trova la chiave corretta non può uscire dal blocco usando `return` o `goto`. Dobbiamo usare una variabile condivisa `found` (inizializzata a 0) che viene impostata a 1 quando la chiave viene trovata. Questo richiede una corretta gestione del **Modello di Memoria di OpenMP** (uso di `omp flush` o direttive `atomic`) per garantire che tutti i thread leggano il valore aggiornato.

---

## 2. Codice Soluzione

```c
/****************************************************************************
 *
 * omp-brute-force.c - Brute-force password cracking
 *
 ****************************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>
#include <omp.h>

/* Decrypt `enc` of length `n` bytes into buffer `dec` using `key` of
   length `keylen`. The encrypted message, decrypted messages and key
   are treated as binary blobs; hence, they do not need to be
   zero-terminated. */
void xorcrypt(const char* in, char* out, int n, const char* key, int keylen)
{
    for (int i=0; i<n; i++) {
        out[i] = in[i] ^ key[i % keylen];
    }
}

int main( void )
{
    const int KEY_LEN = 8;
    const char enc[] = {
        4, 1, 0, 1, 0, 1, 4, 1,
        12, 9, 115, 18, 71, 64, 64, 87,
        90, 87, 87, 18, 83, 85, 95, 83,
        26, 16, 102, 90, 81, 20, 93, 88,
        88, 73, 18, 69, 93, 90, 92, 95,
        90, 87, 18, 95, 91, 66, 87, 22,
        93, 67, 18, 92, 91, 64, 18, 66,
        91, 16, 66, 94, 85, 77, 28, 54
    };
    const int msglen = sizeof(enc);
    const char check[] = "0123456789"; 
    const int CHECK_LEN = strlen(check);
    const int n = 100000000; /* number of possible keys */
    
    int found = 0; /* Variabile condivisa per la terminazione anticipata */

    #pragma omp parallel default(none) shared(KEY_LEN, CHECK_LEN, n, found, msglen, check, enc)
    {
        /* Variabili private: allocate sullo stack di ciascun thread */
        char key[KEY_LEN+1]; 
        char out[msglen];
        
        /* Partizionamento manuale del carico di lavoro */
        int thread_id = omp_get_thread_num();
        int num_threads = omp_get_num_threads();
        int chunk = n / num_threads;
        
        int start = thread_id * chunk;
        int end = (thread_id == num_threads - 1) ? n : start + chunk;

        for(int k = start; k < end; k++)
        {
            /* Lettura sicura dalla memoria condivisa */
            int local_found;
            #pragma omp atomic read
            local_found = found; 

            if(local_found == 1) {
                break; /* Interruzione immediata e lecita in un ciclo for standard */
            }

            assert(out != NULL);
            snprintf(key, KEY_LEN+1, "%08u", (unsigned)k);
            xorcrypt(enc, out, msglen, key, KEY_LEN);
            
            /* Verifica della password */ 
            if ( 0 == memcmp(out, check, CHECK_LEN) ) {
                printf("Key found: %s\n", key);
                printf("Decrypted message: \"%s\"\n", out);
                
                /* Scrittura sicura per notificare gli altri thread */
                #pragma omp atomic write
                found = 1;
            }
        }
    }

    assert(found); /* Verifica finale che la chiave sia stata trovata */
  
    return EXIT_SUCCESS;
}
```

---

## 3. Approfondimenti e Dubbi Comuni

### Perché partizionare manualmente invece di usare `#pragma omp parallel for`?

Il costrutto `omp for` richiede di conoscere a priori e in modo esatto il numero di iterazioni da eseguire per poterle suddividere internamente. Per questo motivo, **lo standard OpenMP vieta l'uso dell'istruzione `break` all'interno di un `omp for`**.

Se si usasse `omp for` insieme a un'istruzione `continue` per bypassare il divieto, il programma non si fermerebbe una volta trovata la password. I thread continuerebbero a iterare "a vuoto" fino al termine del ciclo (es. per milioni di iterazioni), sprecando inutilmente risorse CPU. Usando il costrutto `#pragma omp parallel` e partizionando matematicamente i blocchi, si ottiene un normale ciclo `for` in C, da cui **è possibile uscire con un `break`**, garantendo una terminazione immediata e super-efficiente.

### Perché non basta dichiarare `volatile int found = 0;`?

La keyword `volatile` in C/C++ istruisce esclusivamente il **compilatore**: gli impedisce di ottimizzare la variabile e forzarlo a leggerla/scriverla sempre dalla memoria, evitando che venga salvata permanentemente in un registro della CPU.

Tuttavia, `volatile` **non istruisce l'hardware**. I processori multi-core moderni sono dotati di memorie cache (L1, L2) separate per ogni core. Se il Core 1 aggiorna `found`, la modifica potrebbe rimanere nella sua cache L1 per ragioni di performance. Il Core 2 continuerebbe a leggere il vecchio valore `0` dalla propria cache L1.

Per forzare i core a sincronizzare le loro cache con la memoria principale è necessaria una vera e propria **Memory Barrier** (o Memory Fence). In OpenMP, direttive come `#pragma omp flush(found)` o `#pragma omp atomic` implementano proprio queste barriere di memoria, garantendo la coerenza del dato a livello hardware.

### L'aumento dei thread riduce sempre il tempo di esecuzione? 

Il motivo per cui aggiungere più thread non sempre riduce il tempo di esecuzione (e a volte lo peggiora) si basa su un concetto fondamentale: questa è una ricerca con **terminazione anticipata** (early exit). Appena un thread trova la password, tutti si fermano.

In questo scenario, il tempo non è dettato dalla "potenza di calcolo" totale, ma dalla **distanza tra il punto di partenza del thread "fortunato" e la posizione effettiva della password**.

Aumentare il numero di thread significa tagliare lo spazio delle chiavi in fette più piccole, spostando di conseguenza i punti di partenza di ogni thread. È come una lotteria.

### Per Compilare

```bash
gcc -std=c99 -Wall -Wpedantic -fopenmp omp-brute-force.c -o omp-brute-force

OMP_NUM_THREADS=2 ./omp-brute-force
```