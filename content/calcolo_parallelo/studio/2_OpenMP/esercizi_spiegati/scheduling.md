Puoi vedere la Consegna dell'Esercizio : [qua](https://www.moreno.marzolla.name/teaching/parallel-etudes/handouts/omp-schedule.html)
## Codice Soluzione

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <omp.h>

/* Recursive computation of the n-th Fibonacci number, for n=0, 1, 2, ...
   Do not parallelize this function. */
int fib_rec(int n)
{
    if (n<2) {
        return 1;
    } else {
        return fib_rec(n-1) + fib_rec(n-2);
    }
}

/* Iterative computation of the n-th Fibonacci number. This function
   must be used for checking the result only. */
int fib_iter(int n)
{
    if (n<2) {
        return 1;
    } else {
        int fibnm1 = 1;
        int fibnm2 = 1;
        int fibn;
        n = n-1;
        do {
            fibn = fibnm1 + fibnm2;
            fibnm2 = fibnm1;
            fibnm1 = fibn;
            n--;
        } while (n>0);
        return fibn;
    }
}

/* Fill vectors `vin` and `vout` of length `n`; `vin` will contain
   input values; `vout` is initialized with -1 */
void fill(int *vin, int *vout, int n)
{
    for (int i=0; i<n; i++) {
        vin[i] = 25 + (i%10);
        vout[i] = -1;
    }
}

/* Check correctness of `vout[]`. Return 1 if correct, 0 if not */
int is_correct(const int *vin, const int *vout, int n)
{
    for (int i=0; i<n; i++) {
        if ( vout[i] != fib_iter(vin[i]) ) {
            fprintf(stderr,
                    "Test FAILED: vin[%d]=%d, vout[%d]=%d (expected %d)\n",
                    i, vin[i], i, vout[i], fib_iter(vin[i]));
            return 0;
        }
    }
    fprintf(stderr, "Test OK\n");
    return 1;
}


void do_static(const int *vin, int *vout, int n)
{
    const int chunk_size = 1;

#pragma omp parallel default(none) shared(chunk_size,vout,vin,n) 
    {
      const int P = omp_get_num_threads();
      const int stride = P * chunk_size;

      int my_id = omp_get_thread_num();
      int start = my_id * chunk_size;

      for (int my_block_start = start; my_block_start < n; my_block_start += stride) {
         for (int i = my_block_start; (i < my_block_start+chunk_size) && (i < n); i++) {

           vout[i] = fib_rec(vin[i]);

         }
      } 
    }// -- barriera implicita -- 
}

void do_dynamic(const int *vin, int *vout, int n)
{
    const int chunksize = 1;
    int shared_idx = 0;
    #pragma omp parallel default(none) shared(chunksize, vin, vout, n,shared_idx) 
    {
      int my_block_start;
      do {
         #pragma omp critical
         {
           my_block_start = shared_idx;
           shared_idx += chunksize;
         }
         for (int i=my_block_start; i<my_block_start + chunksize && i<n; i++) {
           vout[i] = fib_rec(vin[i]);
         }
       } while (my_block_start < n);
    }

}

int main( int argc, char* argv[] )
{
    int n = 1024;
    const int max_n = 512*1024*1024;
    int *vin, *vout;
    double tstart, elapsed;

    if ( argc > 2 ) {
        fprintf(stderr, "Usage: %s [n]\n", argv[0]);
        return EXIT_FAILURE;
    }

    if ( argc > 1 ) {
        n = atoi(argv[1]);
    }

    if ( n > max_n ) {
        fprintf(stderr, "FATAL: n too large (max value is %d)\n", max_n);
        return EXIT_FAILURE;
    }

    /* initialize the input and output arrays */
    vin = (int*)malloc(n * sizeof(vin[0])); assert(vin != NULL);
    vout = (int*)malloc(n * sizeof(vout[0])); assert(vout != NULL);

    /**
     ** Test static schedule implementation
     **/
    fill(vin, vout, n);
    tstart = omp_get_wtime();
    do_static(vin, vout, n);
    elapsed = omp_get_wtime() - tstart;
    is_correct(vin, vout, n);

    printf("Execution time (static schedule) %.3f\n", elapsed);

    /**
     ** Test dynamic schedule implementation
     **/
    fill(vin, vout, n);
    tstart = omp_get_wtime();
    do_dynamic(vin, vout, n);
    elapsed = omp_get_wtime() - tstart;
    is_correct(vin, vout, n);

    printf("Execution time (dynamic schedule) %.3f\n", elapsed);

    free(vin);
    free(vout);
    return EXIT_SUCCESS;
}
```

### Per Compilare

```bash
gcc -std=c99 -Wall -Wpedantic -fopenmp omp-schedule.c -o omp-schedule

OMP_NUM_THREADS=2 ./omp-schedule
```
