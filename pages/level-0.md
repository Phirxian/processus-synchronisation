<h1 class="text-center" style="position: relative;top: 50%;">Niveau 0</h1>
<p class="text-center" style="position: relative;top: 50%;">Processus et signaux</p>

---
transition: slide-left
---
# Processus
<br>

**Programme**

- C'est un code compilé contenant des instructions.
- Pour une architecture cible (arm, 64bit, 32bit, fpga, ...)
- Pour un système cible (Linux, BSD, Solaris, ... windows, android, ipomme)
- EXE pour windows, ELF pour les système UNIX, DOS MZ, ...

**Processus**

- Un processus est l'instance d'un programme en cours d'exécution par une unité de calcule.
- Il utilise des ressources système comme la mémoire et le processeur.
- Il a un identifiant `<pid>` pour `process id`.
- Il est dans un état.

---
transition: slide-left
---
## État d'un processus (version simple)
<br>

- Création : Un processus est créé via la fonction fork() ou exec() sur les systèmes Unix.
- Exécution : Le processus s'exécute en utilisant des ressources système.
- Suspension : Un processus peut être suspendu par un signal ou par le système (Ctrl+S).
- Termination : Un processus se termine normalement ou par un signal (Ctrl+C).
- Zombie : Le processus a terminé mais son père n'a pas encore récupéré son statut.

---
transition: slide-left
---
## Cycle de vie d'un processus

<center>
<img src="/snippets/process-state.png" width="100%"/>
</center>

---
transition: slide-left
---
## Systeme uniprogramming
<br>

<center>
<img src="/snippets/CSF-Images.2.0.1.png" width="100%"/>
Chaque processus a un temps d'éxécution géré par le kernel. <br>
Lorsqu'un processus accede a un I/O (ex lecture fichier) il attend simplement. <br>
On perd beaucoup de temps a attendre.
</center>

<br>

> Pour passé d'un programme a un autre, on parle de commutation de context

---
transition: slide-left
---
## Systeme multiprogramming
<br>
  
<center>
  <img src="/snippets/CSF-Images.2.0.2.png" width="100%"/>
  Chaque processus a un temps d'éxécution géré par le kernel. <br>
  En cas d'attente I/O (ex lecture fichier) on donne la main a un autre programme.
</center>

<br>

> Attention, le scheduler (ordonanceur system) peu préempter une tache a n'import quel moment. Pour changer de process.

---
transition: slide-left
---
## Fork et wait

La fonction `fork()` crée un nouveau processus (appelé processus enfant) en dupliquant le processus actuel (appelé processus parent).
La copie contient tout les descripteurs déjà ouvert (fichier, etc).

- Dans le processus parent, fork() retourne l'ID du processus enfant.
- Dans le processus enfant, fork() retourne 0.
- En cas d'erreur, fork() retourne -1.
- Attention, fork() fait une copie de la stack et de la heap !

La fonction `wait(int*status)` fait suspendre le processus parent jusqu'à ce qu'un de ses enfants se termine et enregistre le code de retour dans le pointeur status.
L'ID du processus enfant qui s'est terminé.

- La fonction `waitpid(pid_t, int*status)` permet d'attendre en processus spécifique.
- La fonction `pid_t getpid(void);` Donne le pid du processus actuel
- La fonction `pid_t getppid(void);` Donne le pid du processus parent

---
transition: slide-left
---
### Exemple

```cpp
int main() {
  pid_t pid;
  int status;
  pid = fork();

  if (pid == 0) { // Processus enfant (connait le pid parent via getppid())
    printf("Je suis l'enfant avec PID %d\n", getpid());
    sleep(2); // Attendre 2 secondes
    printf("Enfant terminé\n");
    exit(12); // Terminer avec un statut de sortie
  }
  else if (pid > 0) { // Processus parent (connait le pid de l'enfant)
    printf("Je suis le parent avec PID %d, enfant %d\n", getpid(), pid);
    pid = wait(&status); // Attendre la fin de l'enfant (status => 12)
    printf("Enfant terminé avec statut %d\n", status);
  }
  else {
    perror("fork"); // Erreur lors de fork
    exit(1);
  }

  printf("Fin du processus\n");
  return 0;
}

```

---
transition: slide-left
---
## Commande kill et signaux
La commande `kill` est utilisée pour envoyer des signaux à un processus.

Les signaux courants :
- SIGKILL (9) : Force la fin immédiate d'un processus.
- SIGTERM (15) : Demande poliment à un processus de se terminer.
- SIGSTOP (19) : Suspend un processus.
- SIGCONT (18) : Réactive un processus suspendu.

Exemple:

`kill -9 <pid>`

---
transition: slide-left
---
## Système et signaux
Ils peuvent aussi être générés par le kernel en réponse à des événements système.

- SIGSEGV : Signal généré lors d'un accès invalide à la mémoire (segfault).
  - souvent une erreur de programmation, double free, etc
- SIGFPE : Signale une erreur arithmétique fatale.
  - probablement une div par zero
- SIGILL : Tente d'exécuter une instruction illégale ou malformée.
  - éxécutable corrompu ? hack ?
- SIGBUS : Tentative d'accès à une adresse non alignée ou à une zone mémoire non valide.

---
transition: slide-left
---

```cpp
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void handler(int sig) {
    printf("Signal %d reçu.\n", sig);
}

int main() {
    // Installation du gestionnaire de signal
    signal(SIGILL, handler); // Pour SIGILL
    signal(SIGBUS, handler); // Pour SIGBUS
    signal(SIGTRAP, handler); // Pour SIGTRAP
    signal(SIGABRT, handler); // Pour SIGABRT

    while(1) {
        printf("Processus en cours...\n");
        sleep(1);
    }

    return 0;
}
```