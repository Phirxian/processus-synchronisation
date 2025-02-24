<h1 class="text-center" style="position: relative;top: 50%;">Niveau 2</h1>
<p class="text-center" style="position: relative;top: 50%;">Thread, mutex et sémaphore</p>

---
transition: slide-left
---
# Thread, mutex et sémaphore

Limitation de `fork(), exec(), wait() ...`

- Création de Processus : fork crée un nouveau processus en dupliquant le processus parent. <br>
  Cela est coûteux en termes de ressources (cpu et mémoire).
- Mémoire Non Partagée : Les processus créés par fork n'ont pas de mémoire partagée par défaut. <br>
  Nécessite des mécanismes IPC supplémentaires.
- Zombies : Si un parent ne wait pas correctement, les processus enfants peuvent devenir des zombies. <br>
  Occupation des entrées dans la table des processus.
  
Les threads POSIX (Pthreads) sont une bibliothèque pour écrire des programmes multi-threads. <br>
Ces derniers viennent répondre a ces limitations.  

- Pthreads permet de créer des threads dans un processus, partageant la même mémoire et ressources.
- Moins coûteux que fork, communication interne plus facile via la même mémoire (partagée).
- Chaque thread a sa propre stack, le reste est partagé (même le pid)

---
transition: slide-left
---
### Utilisation 

```cpp
#include <pthread.h>
#include <stdio.h>

void* threadFunction(void* arg) {
    printf("Hello from thread!\n");
    return NULL;
}

int main() {
    pthread_t thread;
    pthread_create(&thread, NULL, threadFunction, NULL);
    pthread_join(thread, NULL); // Attendre la fin du thread
    return 0;
}
```

---
transition: slide-left
---
## Passage de variables

```cpp
struct args {
  int a, b;
};

struct results {
  int sum, difference;
};

void * calculator (void *_args) {
  struct args *args = (struct args *) _args;

  struct results *results = malloc (sizeof (struct results));
  results->sum = args->a + args->b;
  results->difference = args->a - args->b;
  
  free (args);
  pthread_exit (results);
}

pthread_t child;
struct args *args = (struct args*)malloc(sizeof (struct args));
pthread_create (&child, NULL, calculator, &args) 
struct results *results;
pthread_join (child[i], (void **)&results);
```

---
transition: slide-left
---
### Quel est l'ordre d'éxécution de ce code ?
<br>

```cpp
int number = 3;

void * thread_A (void* args) {
  int x = 5;
  printf ("A: %d\n", x + number);
}

void * thread_B (void* args) {
  int y = 2;
  printf ("B: %d\n", y + number);
}

int main (int argc, char** argv) {
  /* Create thread A */
  /* Create thread B */
  /* Wait for threads to finish */
  return 0;
}
```

<br>

- Indice Load \& Store

---
transition: slide-left
---
### Quel est l'ordre d'éxécution de ce code ?
<br>

```cpp
int number = 3;

void * thread_A (void* args) {
  number += 5;
  printf ("A: %d\n", number);
}

void * thread_B (void* args) {
  number -= 2;
  printf ("B: %d\n", number);
}

int main (int argc, char** argv) {
  /* Create thread A */
  /* Create thread B */
  /* Wait for threads to finish */
  return 0;
}
```

<br>

- Indice Load \& Store

---
transition: slide-left
---
**Système des jetons ferroviaires :**
Un jeton physique garantit qu'un seul train entre dans une section à voie unique.
En informatique nous avons un jeton numérique qui garantie l'access par un seul processus/thread à la section.

<center>
<iframe width="800" height="350" src="https://www.youtube.com/embed/m7uI7Jd9OUM" title="The Railway token" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</center>

---
transition: slide-left
---

## Section critique

Une `section critique` est une partie d'un programme où des ressources partagées sont accédées sans interférence pour éviter les problèmes de synchronisation (`race condition`) et donc les incohérences de comportement ou de données.

- **Autorisation :** Un processus a besoin d'un "jeton" pour accéder à une section critique, comme un train pour une voie unique.
- **Exclusion Mutuelle :** Un seul processus peut accéder à la ressource partagée, comme un seul train sur la voie.
- **Synchronisation :** Des mécanismes comme les sémaphores garantissent que les processus entrent un par un.
- **Prévention des Collisions :** Les ressources sont accédées séquentiellement pour éviter les problèmes de données.

---
transition: slide-left
---
## Géré des exlusions mutuel (mutex)

Les mutex (mutual exclusion locks) sont des primitives de synchronisation utilisées pour protéger les sections critiques du code, où plusieurs threads ne doivent pas accéder simultanément aux mêmes données.

- **Initialisation :** Un mutex doit être initialisé avant utilisation <br> par exemple avec `PTHREAD_MUTEX_INITIALIZER`.
- **Verrouillage (bloquant) :** Un thread peut verrouiller un mutex avec `pthread_mutex_lock()`.
- **Verrouillage (non-bloquant) :** Essaye de verrouiller un mutex avec `pthread_mutex_trylock()`.
- **Déverrouillage :** Un mutex est déverrouillé avec `pthread_mutex_unlock()` après la section critique.

<!-- https://www.ibm.com/docs/en/zos/2.4.0?topic=files-pthreadh-thread-interfaces#pthrdh -->
<!-- https://w3.cs.jmu.edu/kirkpams/OpenCSF/Books/csf/html/RaceConditions.html -->

---
transition: slide-left
---
### Exemple

```cpp
#include <pthread.h>
#include <stdio.h>

int shared_data = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* threadFunction(void* arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&mutex);
        shared_data++;
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

int main() {
    pthread_t thread1, thread2;
    pthread_create(&thread1, NULL, threadFunction, NULL);
    pthread_create(&thread2, NULL, threadFunction, NULL);
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);
    printf("Final shared data: %d\n", shared_data);
    return 0;
}
```

---
transition: slide-left
---
### Qu'est-ce qui ne va pas avec ce code ?

```cpp
pthread_mutex_t mutex1;
pthread_mutex_t mutex2;

void *thread1_function(void *arg) {
    pthread_mutex_lock(&mutex1);
    sleep(1); // Simulate some work
    pthread_mutex_lock(&mutex2);
    
    pthread_mutex_unlock(&mutex2);
    pthread_mutex_unlock(&mutex1);
    return NULL;
}

void *thread2_function(void *arg) {
    pthread_mutex_lock(&mutex2);
    sleep(1); // Simulate some work;
    pthread_mutex_lock(&mutex1);
    
    pthread_mutex_unlock(&mutex1);
    pthread_mutex_unlock(&mutex2);
    return NULL;
}
```

---
transition: slide-left
---
## Introduction aux Sémaphores

Les sémaphores sont des primitives de synchronisation utilisées pour contrôler l'accès à des ressources partagées entre plusieurs threads ou processus.
Ils sont représentés par un entier qui peut être incrémenté ou décrémenté atomiquement. Par exemple on peu compter le nombre de case vide dans un buffer.

```cpp
#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value);
int sem_destroy(sem_t *sem);
int sem_wait(sem_t *sem);
int sem_post(sem_t *sem);
```

<!-- https://sites.uclouvain.be/SyllabusC/notes/Theorie/Threads/coordination.html -->

---
transition: slide-left
---
## Exemple producteur - consomateur

```cpp
int buffer[BUFFER_SIZE]; // Déclaration du tampon
int in = 0, out = 0; // Index pour le producteur et le consommateur
sem_t empty; // Nombre de places vides dans le tampon
sem_t full; // Nombre d'éléments dans le tampon
sem_t mutex;  // Mutex pour l'accès exclusif au tampon

sem_init(&empty, 0, BUFFER_SIZE); // Tampon vide au début
sem_init(&full, 0, 0); // Aucun élément dans le tampon
sem_init(&mutex, 0, 1); // Mutex pour accès exclusif

void* producer(void* arg) {
    sem_wait(&empty); // Attendre qu'il y ait de la place dans le tampon
    sem_wait(&mutex); // Section critique
    
    int item = rand() % 100;
    buffer[in] = item;
    in = (in + 1) % BUFFER_SIZE;

    sem_post(&mutex);
    sem_post(&full); // Signaler qu'un item est disponible
    return NULL;
}
```


---
transition: slide-left
---
# Multiple producteurs - multiple consommateurs

- A voir en TP (MPMC)