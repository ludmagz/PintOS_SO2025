# ⏰ Alterações para o funcionamento total do Alarm Clock

### Em 'thread.c'
##### 😴 Criação da lista 'sleep_list' 
- sleep_list é a lista que armazena as threads no estado **BLOQUEADO**. Com a implementação dela, as threads agora podem parar de praticar o busy wait e dormir.

##### 🌙 Criação da função 'void thread_sleep(int64_t ticks)'
- essa função pega a thread atual e muda o estado dela para **BLOQUEADO**, depois, desabilitamos as interrupções para evitar condição de corrida e inserimos a thread bloqueada na sleep_list, inserindo ordenadamente pela hora que devemos acordar a thread e respeitando a prioridade entre threads que deveriam acordar na mesma hora. Por último, chamamos o escalonador e habilitamos as interrupções novamente.
  - para que fosse possível respeitar essa prioridade, implementamos um comparador que determina a ordem entre os wakeup_tick's das threads e, caso eles sejam iguais, checa a ordem de prioridade entre as threads em si. 
 
##### 🌞 Criação da função 'void thread_interrupt(int64_t actual_time)'    
- essa função, desabilitando as interrupções, analisa a primeira thread de sleep_list e, caso já esteja na hora de acordar aquela thread, nós:
1. removemos ela da sleep_list
2. inserimos ela ordenadamente na ready_list pela ordem de prioridade da thread
3. mudamos a referência de head para a thread seguinte, que agora ocupa a primeira posição da sleep_list
4. rodamos o while de checagem para a nova head
5. por fim, saímos do while quando não há mais threads para acordar no atual instante e reativamso as interrupções

### Em 'timer.c'
##### 🔇 Atualização da função 'void timer_sleep (int64_t ticks)'
- **antes**: a função timer_sleep estava em busy wait, ciclando na ready_list, sem haver escalonamento, até que, finalmente, o tempo que foi determinado para que ela ficasse bloqueada acabe.
- **depois**: foi implementada a lógica de que, caso uma thread deva entrar no estado de bloqueada, chamamos a função thread_sleep, passando para ela o momento que tal thread deve acordar.

##### 🔊 Atualização da função 'static void timer_interrupt(struct intr_frame *args UNUSED)'
- **antes**: a cada tick do timer, ela chamava thread_tick(), função essa que atualiza os ticks do sistema
- **depois**: agora, além das funções anteriores, ela passou a chamar a função timer_interrupt, passando como argumento o tick em que estamos agora (atual_time)

### ✉ Em 'thread.h'
##### 📁 Adição das novas funções de thread implementadas
- 'void thread_sleep(int64_t);'
- 'void thread_interrupt(int64_t);'
