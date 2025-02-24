<h1 class="text-center" style="position: relative;top: 50%;">Niveau 1</h1>
<p class="text-center" style="position: relative;top: 50%;">Mémoire partagées</p>

---
transition: slide-left
---
# Mémoire partagées
<br>

> Lorsque fork est appelé, le processus enfant reçoit une copie des pages mémoire du processus parent.
<br> Cependant, ces pages sont marquées comme étant en lecture seule et sont partagées entre les deux processus.

<br>

> Sun processus enfant tente de modifier une page partagée, le système génère une exception de page (page fault).
<br> En réponse, le système crée une copie privée de la page pour le processus qui a tenté la modification.

<br>

> Chaque processus va écrire dans ça propre version / mémoire privé !

Il exist plusieurs façon de partagé des données.

---
transition: slide-left
---
## Communication entre processus
<br>

- Pipes : Canaux unidirectionnels pour l'échange de données.
- Named Pipes (FIFO) : Canaux nommés pour l'échange de données.
- Shared Memory : Mémoire partagée entre processus.
- Message Queues : Files d'attente de messages pour l'échange de données.
  - Peu usité
- Sockets : Communication réseau entre processus.

---
transition: slide-left
---
## Utilisation des pipes
Les pipes sont des canaux unidirectionnels pour l'échange de données entre processus.

```cpp
int pipefd[2];
pipe(pipefd);
pipefd[0] // descripteur de fichier pour la lecture (sortie du tube / read).
pipefd[1] // descripteur de fichier pour l'écriture (entrée du tube / write).
```

En cas de succès, pipe renvoie 0.

```cpp
int pipefd[2];
if (pipe(pipefd) == -1) {
    perror("pipe");
    exit(EXIT_FAILURE);
}
```

Lorsque vous créez un processus fils avec fork, celui-ci hérite des descripteurs de fichiers ouverts par le processus père, y compris ceux du pipe. Pour éviter les problèmes de synchronisation, chaque processus doit fermer l'extrémité du pipe qu'il n'utilise pas.

---
transition: slide-left
---
## Particularité des pipes
<br>

- **Tamponnage :** Les données écrites dans un pipe sont stockées en mémoire tampon par le noyau jusqu'à ce qu'elles soient lues. Cela signifie que si le lecteur ne lit pas les données immédiatement, elles restent dans le tampon jusqu'à ce qu'elles soient traitées.
- **Synchronisation :** Il est important de synchroniser les lectures et écritures sur un pipe pour éviter les problèmes de concurrence. Cela peut être fait en fermant les extrémités inutilisées et en utilisant des mécanismes de synchronisation comme les sémaphores ou les verrous.
- **Limitations de taille :** Les pipes ont une limite de taille pour les données tamponnées. Si cette limite est atteinte, les écritures bloquent jusqu'à ce que des données soient lues.
- **Blocage en cas de pipe vide :** La fonction read met le processus en attente jusqu'à ce que des données soient disponibles ou que l'extrémité d'écriture soit fermée.
- `ls -l | sort -n -k 5 | tail -n 1 | awk '{print $NF}'`
- `./main < test.txt`

---
transition: slide-left
---
## Les pipes sont unidirectionel !
<br>

<center>
<img src="/snippets/CSF-Images.3.1.png" width="80%"/><br>
<img src="/snippets/CSF-Images.3.2.png" width="80%"/>
</center>

<!-- https://w3.cs.jmu.edu/kirkpams/OpenCSF/Books/csf/html/Pipes.html -->

---
transition: slide-left
---
### Exemple

```cpp
#include <stdio.h>
#include <unistd.h>

int main() {
    int pipefd[2];
    pipe(pipefd); // 1 process en écriture, 1 process en lecture

    pid_t pid = fork();
    if (pid == 0) { // Processus enfant
        close(pipefd[1]); // Fermer l'extrémité d'écriture
        char buffer[100];
        read(pipefd[0], buffer, 100);
        printf("Enfant : %s\n", buffer);
        close(pipefd[0]);
    }
    else { // Processus parent
        close(pipefd[0]); // Fermer l'extrémité de lecture
        char *msg = "Hello, Child!";
        write(pipefd[1], msg, strlen(msg) + 1);
        close(pipefd[1]);
    }

    return 0;
}
```

---
transition: slide-left
---
## Utilisation des sockets

Les sockets permettent la communication entre processus en passant par les couches du réseau.
Il est nécessaire de suivre les différentes étapes de connexion réseau du modèle OSI.
Par conséquent, il faut avoir un processus serveur qui accepte les connexions, tandis que le processus fils se connecte à ce serveur.

```cpp
int server_fd = socket(AF_INET, SOCK_STREAM, 0); // création de la socket serveur
bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr); // Liaison du socket à une interface reseau
// communication
recvfrom(server_fd, buffer, BUFFER_SIZE, 0, (struct sockaddr *)&sender_addr, &sender_len);
sendto(server_fd, msg, strlen(msg) + 1, 0, (struct sockaddr *)&sender_addr, sender_len);
// idem sur le client (processus fils)
```

<br>

> Cette solution est parfois utilisée, notamment pour permettre une communication pour des processus externes a votre solution.
Dans les autres cas, elle est peu recommandable. Des processus comme DBUS, X11 et MySQL utilisent cette méthode.

---
transition: slide-left
---
## Utilisation de mmap

mmap permet de mapper un fichier ou une zone mémoire anonyme dans l'espace d'adressage d'un processus.
Avec le flag `MAP_SHARED`, plusieurs processus peuvent accéder à la même zone mémoire.

```cpp
void *ptr = mmap(NULL, taille, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
```

Les mémoires partagées peuvent être nommé, c'est à dire associé a un descripteur de fichier :

```cpp
int fd = shm_open("/example", O_RDWR | O_CREAT, 0666);
void *ptr = mmap(NULL, 1024, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

<br>

> Attention, mmap permet a plusieurs processus d'écrire simultanément dans la mémoire.
Mais `mmap`, ne gère pas ces accès concurrents (race conditions). Dans le cas ou deux processus écrive simultanément dans la mémoire, aucune garantie sur l'ordre d'éxécution n'est possible !
Dans le meilleurs des cas une des deux données est stocké, dans le pire des cas une supperposition des données est possible, donnant un résultat aléatoire.

---
transition: slide-left
---
### Exemple mmap
```cpp
#include <stdio.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd = shm_open("/example", O_RDWR | O_CREAT, 0666);
    // verification erreur d'ouverture @fd
    ftruncate(fd, 1024); // Taille de la mémoire partagée

    void *ptr = mmap(NULL, 1024, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    // verification erreur d'ouverture de @ptr

    // Écriture dans la mémoire partagée
    char *msg = "Hello, World!";
    memcpy(ptr, msg, strlen(msg) + 1);
    // Lecture dans la mémoire partagée
    printf("Données lues : %s\n", (char *)ptr);
      
    munmap(ptr, 1024);
    close(fd);
    shm_unlink("/example");

    return 0;
}
```