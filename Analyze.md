Изначально в репозитории ни thread sanitizer, ни helgrind ошибок не дают


Далее я изменил места блокировок в методе pop, передвинув их непосредственно к местам, где из очереди удаляется 
элемент. push при этом остаётся неизменным. Гонок данных обнаружено не было ни thread sanitizer, ни helgrind.


Если совсем не блокировать ресурсы, то возникает три предупреждения от thread sanitizer, два из которых говорят о том, 
что гонка данных возникает при добавлении элемента, а одно - при удалении. При этом helgrind обнаруживает 132 
потенциальные ошибки.
В целом понятно, почему возникают предупреждения. Это связано с тем, что если несколько потоков будут работать с одними
данными, то возможен сценарий, при котором один поток не успеет закончить выполнение некоторых операций с данными, до
того как придет другой поток и изменит их, вследствии чего может возникнуть непредсказуемое поведение первого потока.
Логи предупреждений thread Sanitizer и helgrind указаны в without_lock_helgrind_log.txt и
without_lock_san.txt


```c
int push (queue*q, void *data) {
	node *ins=malloc(sizeof(node));
	if (!q || !data || !ins) return -1;
	ins->info=data;
	ins->next=NULL;
//	pthread_mutex_lock(&q->lock);
	if (q->size==0) {
		q->first->next=ins;
		q->last=ins;
	} else {
		q->last->next=ins;
		q->last=ins;
	}
	(q->size)++;
	pthread_cond_signal(&q->cond);
//    pthread_mutex_unlock(&q->lock);
	return 0;
}

void* pop (queue*q) {
	if (q==NULL) return NULL;
//	pthread_mutex_lock(&q->lock);
	while (q->size == 0) {
		pthread_cond_wait(&q->cond, &q->lock);
	}
	assert(q->first->next);
	node*n=(node*)q->first;
	void*data=(q->first->next)->info;
	q->first=q->first->next;
	(q->size)--;
	assert(q->size>=0);
//	pthread_mutex_unlock(&q->lock);
	free(n);
	return data;
}
```


Если в pop переместить блокировки как в случае 2, и в push сделать то же самое, то helgrind выругается, 
и даст 6 ошибок, thread sanitizer сообщит об одной, в методе push. Это связано с тем, что в методе push не блокируется
доступ к параметру очереди size, который может быть изменен несколькими потоками, вледствие чего образуется гонка данных.

```c
int push (queue*q, void *data) {
        node *ins=malloc(sizeof(node));
        if (!q || !data || !ins) return -1;
        ins->info=data;
        ins->next=NULL;
//      pthread_mutex_lock(&q->lock);
        if (q->size==0) {
        pthread_mutex_lock(&q->lock);
        q->first->next=ins;
                q->last=ins;
        pthread_mutex_unlock(&q->lock);
    } else {
        pthread_mutex_lock(&q->lock);
        q->last->next=ins;
                q->last=ins;
        pthread_mutex_unlock(&q->lock);
    }
        (q->size)++;
        pthread_cond_signal(&q->cond);
  //  pthread_mutex_unlock(&q->lock);
        return 0;
}

void* pop (queue*q) {
        if (q==NULL) return NULL;
   // pthrzead_mutex_lock(&q->lock);

    while (q->size == 0) {
                pthread_cond_wait(&q->cond, &q->lock);
        }
        assert(q->first->next);
        node*n=(node*)q->first;

        q->first=q->first->next;
    (q->size)--;
//    pthread_mutex_unlock(&q->lock);

    assert(q->size>=0);
        pthread_mutex_unlock(&q->lock);
        pthread_mutex_unlock(&q->lock);
        free(n);
        return data;
}
```



