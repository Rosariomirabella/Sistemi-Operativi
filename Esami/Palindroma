#include <stdio.h>
#include <stdlib.h>
#include <semaphore.h>
#include <pthread.h>
#include <stdbool.h>
#include "lib-misc.h"
#include <string.h>

#define MAX_DIM 4096

typedef enum {READER, CONTROLLER, WRITER} thread_n;

typedef struct {
    char parole[MAX_DIM];
    sem_t sem[3];
    bool finito;
    
}   shared;

typedef struct {
    pthread_t id;
    thread_n thread_n;
    char *filename;
    shared *shared;
}   thread_data;

void init_sem(sem_t *sem) {

    int err;

    if((err=sem_init(&sem[READER], 0, 1))!=0) exit_with_err("sem_init",err);
    if((err=sem_init(&sem[CONTROLLER], 0, 0))!=0) exit_with_err("sem_init",err);
    if((err=sem_init(&sem[WRITER], 0, 0))!=0) exit_with_err("sem_init",err);
}

void destroy_sem(sem_t *sem) {
    for(int i=0; i<3; i++) {
        sem_destroy(&sem[i]);
    }
}

int is_palindroma (char *s) {
    for(int i=0; i < strlen(s); i++) {
        if(s[i] != s[strlen(s)-i-1]) {
            return 0;
        }
    } 
    return 1;
}


void reader(void *arg) {
    int err;
    thread_data *t = (thread_data*) (arg);
    FILE *file;
    char riga[MAX_DIM];

    file = fopen(t->filename, "r");

    if(!file) {
        exit_with_sys_err("ERRORE\n");
    }
    
    while(fgets(riga,MAX_DIM,file)) {
        if(riga[strlen(riga) - 1] == '\n') {
            riga[strlen(riga) -1] = '\0';
        }
        
        // se c'è qualche riga di testo nel file sveglio il reader per inviare la stringa al controller
        if((err=sem_wait(&t->shared->sem[READER]))!=0)
            exit_with_err("sem_wait",err);

        // inserisco la stringa appena letta nella struttura dati condivisa    
        strncpy(t->shared->parole, riga, MAX_DIM);
        
        // sveglio il controller per fargli verificare se è palindroma
        if((err=sem_post(&t->shared->sem[CONTROLLER]))!=0)
                exit_with_err("sem_post",err);
        }
        
    // se non c'è alcuna riga da leggere perchè il file è vuoto o abbiamo finito di leggere il reader deve bloccarsi
    if((err=sem_wait(&t->shared->sem[READER]))!=0)
        exit_with_err("sem_wait",err);

    t->shared->finito = 1;
        
    // sveglio reader e writer così terminano
    for(int i=0; i<2; i++) {
        if((err=sem_post(&t->shared->sem[i+1]))!=0)
            exit_with_err("sem_post",err);
    }

    fclose(file);
    pthread_exit(NULL);
}

void controller(void *arg) {
    int err;
    thread_data *t = (thread_data *) (arg);
    int counter=0;
    int pal;

    while(1) {
        if((err = sem_wait(&t->shared->sem[CONTROLLER]))!=0) 
            exit_with_err("sem_wait",err);
        
        if(t->shared->finito) {
            printf("[THREAD CONTROLLER]: sono state trovate %d stringhe palindrome \n", counter);
            break;
        }

        pal = is_palindroma(t->shared->parole);

        if(pal) {
            if((err=sem_post(&t->shared->sem[WRITER]))!=0)
                exit_with_err("sem_post",err);
            counter++;
        }
        else{
            if((err=sem_post(&t->shared->sem[READER]))!=0)
                exit_with_err("sem_post",err);
        }
    } 
    pthread_exit(NULL);
}

void writer(void *arg) {
    int err;
    thread_data *t = (thread_data *) (arg);

    while(1) {
        if((err=sem_wait(&t->shared->sem[WRITER]))!=0) 
            exit_with_err("sem_wait",err);

        if(t->shared->finito) {
            break;
        }
            
        printf("[THREAD WRITER]: %s \n", t->shared->parole);
        
        if((err=sem_post(&t->shared->sem[READER]))!=0) {
            exit_with_err("sem_post",err);
        }
        
    }

    pthread_exit(NULL);
}

int main(int argc, char **argv) {

    int err;

    if(argc <=1) {
        printf("%s <input-file>", argv[0]);
        exit(EXIT_FAILURE);
    }

    thread_data t[3];
    shared *sh = malloc(sizeof(shared));
    sh->finito=0;
    init_sem(sh->sem);
    

    for(int i=0; i<3;i++) {
        t[i].thread_n = i;
        t[i].shared = sh;
        t[i].filename = argv[1];
    }

    //t[READER].filename = argv[1];

    if((err=pthread_create(&t[READER].id, NULL, (void*) (reader), (void*)&t[READER])!=0))
        exit_with_err("pthread_create",err);

    if((err=pthread_create(&t[CONTROLLER].id, NULL, (void*) (controller), (void*)&t[CONTROLLER])!=0))
        exit_with_err("pthread_create",err);

    if((err=pthread_create(&t[WRITER].id, NULL, (void*) (writer), (void*)&t[WRITER])!=0))
        exit_with_err("pthread_create",err);

    for(int i=0; i<3; i++) {
        if((err=pthread_join(t[i].id, NULL))!=0)
            exit_with_err("pthread_join",err);
    }

    sem_destroy(sh->sem);
    free(sh);
    exit(EXIT_SUCCESS);

}
