# ⏰ Alterações para o funcionamento total do Alarm Clock

## 💥 Em 'thread.c'
#### 😴 Criação da lista 'sleep_list' 
- sleep_list é a lista que armazena as threads no estado **BLOQUEADO**. Com a implementação dela, as threads agora podem parar de praticar o busy wait e dormir.

#### 🌙 Criação da função 'void thread_sleep(int64_t ticks)'
- essa função pega a thread atual e muda o estado dela para **BLOQUEADO**, depois, desabilitamos as interrupções para evitar condição de corrida e inserimos a thread bloqueada na sleep_list, inserindo ordenadamente ``wakeup_less(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED)`` pela hora que devemos acordar a thread e respeitando a prioridade entre threads que deveriam acordar na mesma hora. Por último, chamamos o escalonador e habilitamos as interrupções novamente.
  - para que fosse possível respeitar essa prioridade, implementamos um comparador que determina a ordem entre os wakeup_tick's das threads e, caso eles sejam iguais, checa a ordem de prioridade entre as threads em si. 
 
#### 🌞 Criação da função 'void thread_interrupt(int64_t actual_time)'    
- essa função, desabilitando as interrupções, analisa a primeira thread de sleep_list e, caso já esteja na hora de acordar aquela thread, nós:
1. removemos ela da sleep_list
2. inserimos ela ordenadamente na ready_list pela ordem de prioridade da thread ('ord_prio (const struct list_elem *a, const struct list_elem *b, void *aux UNUSED)')
3. Mudamos o estado da thread para **PRONTO**
4. mudamos a referência de head para a thread seguinte, que agora ocupa a primeira posição da sleep_list
5. rodamos o while de checagem para a nova head
6. por fim, saímos do while quando não há mais threads para acordar no atual instante e reativamos as interrupções

## 🕓 Em 'timer.c'
#### 🔇 Atualização da função 'void timer_sleep (int64_t ticks)'
- **antes**: a função timer_sleep estava em busy wait, ciclando na ready_list, sem haver escalonamento, até que, finalmente, o tempo que havia sido determinado para que ela ficasse bloqueada acabesse.
- **depois**: foi implementada a lógica de que, caso uma thread deva entrar no estado de bloqueada, chamamos a função thread_sleep, passando para ela o momento que tal thread deve acordar.

#### 🔊 Atualização da função 'static void timer_interrupt(struct intr_frame *args UNUSED)'
- **antes**: a cada tick do timer, ela chamava thread_tick(), função essa que atualiza os ticks do sistema
- **depois**: agora, além das funções anteriores, ela passou a chamar a função thread_interrupt, passando como argumento o tick em que estamos agora (atual_time)

## ✉ Em 'thread.h'
#### 📁 Adição das novas funções de thread implementadas
- 'void thread_sleep(int64_t);'
- 'void thread_interrupt(int64_t);'


# 🧠 Alterações para o funcionamento total do Multi Level Feedback Queue (mlfqs)

## 💥 Em 'thread.c'
#### 🕊 Criação das operações com ponto flutuante

#### 🎠 Criação da variável 'avg', inicializando ela como zero.

#### 🗯 Alteralção na função 'thread_yield()'
- Em vez de apenas darmos push_back na thread para a ready_list, caso o mlfqs esteja ativado, inserimos ordenadamente com base na função ord_prio().
  
#### 💣 Alteralção na função 'void thread_interrupt()'
- Adicionamos a variável booleana need_yield, que verifica se a thread acordada tem prioridade maior do que a atual. Se sim, chamamos intr_yield_on_return(), com a finalidade de forçar a troca de contexto.

#### 🥇 Alteração na função 'void thread_set_priority (int new_priority)'
- Caso o thread_mlfqs não esteja ativado, a prioridade da thread atual será alterada para new_priority.

#### 👍 Alteração na função 'void thread_set_nice (int nice)'
- Garantimos que o valor de nice ∈ [-20, 20].
- Atualizamos o antigo valor de nice da thread atual para o valor passado na função.
- Chamamos update_priority() para a thread atual de modo a concretizar a alteração da nova prioridade.

#### 😁 Alteração na função 'int thread_get_nice (void)'
- **antes**: a função retornava zero.
- **depois**: Passou a retornar o valor de nice da thread atual.

#### 📥 Criação da função 'int update_load_avg (void)'
- Inicia calculando o valor de ready_threads. Esse valor é computado como sendo o tamanho da ready_list e, caso a thread atual não seja idle, ele soma 1.
- retorna o valor calculado do avg.

#### 📦 Criação da função 'int get_load_avg (void)'
- retorna a multiplicação do avg por 100.

#### 🖥 Criação da função 'void increment_recent_cpu(struct thread *t)'
- Se a thread passada como parâmetro não for idle, incrementamos o recent_cpu: recent_cpu += 1.

#### 💻 Criação da função 'void update_recent_cpu(struct thread *t)'
- Realiza o cálculo com a formula do recent_cpu e passa o resultado dele para o recent_cpu da thread passada como parâmetro.

#### 📢 Criação da função 'void update_recent_cpu_all(void)'
- Iteramos por toda a all_list e, a cada elemento, chamamos update_recent_cpu() para a thread correspondente.
- Ordenamos novamente a ready_list com base em ord_prio().

#### 💾 Criação da função 'int thread_get_recent_cpu(void)'
1. Desabilitamos as interrupções.
2. Pegamos o recent_cpu da thread atual multiplicado por 100.
3. Habilitamos as interrupções.
4. Retornamos tal valor.

#### 📮 Criação da função 'void update_priority(struct thread *t)'
- Caso a thread passada como parâmetro não seja idle, realizamos o cálculo da prioridade com a fórmula.
- Garantimos que o valor calculado ∈ [0, 63].
- Atualizamos o valor da prioridade da thread passada como parâmetro para o valor calculado.

#### 🗂 Criação da função 'void update_priority_all(void)'
1. Desabilitamos as interrupções.
2. Iteramos pela all_list e, a cada elemento, chamamos update_priority() para a thread correspondente.
3. Ordenamos novamente a ready_list com base em ord_prio().
4. Caso a ready_list não esteja vazia, pegamos a thread com mais prioridade e vemos se a prioridade dela é maior do que a prioridade da thread atual.
5. Caso seja, chamamos intr_yield_on_return().
6. Habilitamos as interrupções.

#### 👶 Alteração da 'static void init_thread(struct thread *t, const char *name, int priority)'
- Caso não estejamos no mlfqs, inicializamos o escalonador de forma simples, por prioridade.
- Caso contrário:
  - Se for a thread inicial, o nice e o recent_cpu são setados como zero.
  - Senão, se não for a thread inicial, ela recebe o nice e o recent_cpu da thread atual (pai).
- Chamamos update_priority para a thread passada como parâmetro.

## ✉ Em 'thread.h'
#### 📁 Adição das novas funções de thread implementadas
- 'void increment_recent_cpu(struct thread *t);'
- 'void update_recent_cpu (struct thread *t);'
- 'void update_recent_cpu_all(void);'
- 'void update_priority(struct thread *t);'
- 'void update_priority_all(void);'
- 'int update_load_avg(void);'

#### 🙈 Adição das novas variáveis da struct thread
- 'int nice'
- 'int recent_cpu'

## 🕓 Em 'timer.c'
#### Alteração do 'timer_interrupt(struct intr_frame *args UNUSED)'

## ⏱ Em 'synch.c'
#### Alteração na função 'sema_up (struct semaphore *sema)'
- Para a resolução do teste mlfqs-block. Serve para que sempre que eu desabilitar as interupções para mexer em uma lista, seja dado thread_yield() para que a próxima thread executada seja de fato a de maior prioridade que está na ready_list.
-   if (thread_mlfqs && intr_get_level() == INTR_ON) {
    thread_yield();
    }

