

## Diario de Bordo 08/10

**UNICESUMAR**  
**CURSO DE ENGENHARIA DE SOFTWARE**  

**DESAFIO PRÁTICO: ARQUITETURA E CONCORRÊNCIA NA VENDA DE INGRESSOS ONLINE**  
*Estudo de Caso: "O Caos do Show de Rock"*  

**Autor(es):** Roberto Müller  
**Orientador / Professor:** Prof. Leonardo Rocha  

**LONDRINA - PR**  
**2026**  

---

## Sumário

1. **Síntese Teórica (Pesquisa)** 

   1.1 Diferença entre Processo e Thread

   1.2 Criação e Custo Computacional: Threads vs. Processos

   1.3 Threads em Modo Usuário vs. Modo Núcleo (Kernel)

2. **Diagnóstico do Problema (O Gargalo)** 
   
   2.1 A Condição de Corrida (*Race Condition*) 
   
   2.2 Impacto no Sistema de Ingressos sem Tratamento 

3. **A Solução Arquitetural** 
  
   3.1 Exclusão Mútua e Regiões Críticas 
  
   3.2 Aplicação Prática ao Assento A-15

4. **Referências Bibliográficas (Norma ABNT)
---

## 1. Síntese Teórica (Pesquisa)

### 1.1 Diferença entre Processo e Thread
No contexto dos Sistemas Operacionais, um **Processo** representa um programa em execução dotado de um espaço de endereçamento de memória próprio e isolado, além de uma estrutura de controle independente mantida pelo sistema operacional (Bloco de Controle do Processo - BCP). Ele encapsula todos os recursos necessários para a execução, tais como variáveis globais, registradores, sinalizadores e descritores de ficheiros abertos.

Por outro lado, uma **Thread** (ou linha de execução) é a menor unidade de código que pode ser gerenciada e escalonada individualmente pelo processador. Múltiplas threads pertencentes ao mesmo processo partilham o mesmo espaço de endereçamento de memória e os mesmos recursos alocados para o processo pai, mantendo apenas o seu próprio Contador de Programa (*Program Counter* - PC), conjunto de registradores e pilha de execução (*stack*).

### 1.2 Criação e Custo Computacional: Threads vs. Processos
Atender a 100.000 requisições simultâneas através da criação de novos **Processos** geraria uma degradação severa no desempenho do sistema devido ao enorme *overhead* associado.
* **Alocação de Memória e Estruturas:** Criar um processo exige que o sistema operacional aloque uma nova tabela de páginas, uma nova área de dados, BCP e espaço de endereçamento isolado.
* **Troca de Contexto (*Context Switch*):** Trocar a execução entre dois processos diferentes é uma operação custosa. O processador precisa invalidar caches (*TLB flush*), salvar e carregar registradores, além de reconfigurar os ponteiros da memória virtual.

Em contrapartida, criar novas **Threads** é significativamente mais leve (podendo ser até 100 vezes mais rápido que a criação de processos):
* **Compartilhamento de Espaço de Endereçamento:** Como todas as threads de um mesmo processo partilham a mesma memória, não é necessário alocar novas áreas de código ou dados virtuais nem duplicar a estrutura de gerenciamento de recursos.
* **Trocas de Contexto Leves:** A alternância entre threads do mesmo processo preserva os mapeamentos da memória virtual e o conteúdo das caches de instrução e dados, reduzindo drasticamente o tempo consumido pelo despachante (*dispatcher*) e preservando o rendimento do processador.

### 1.3 Threads em Modo Usuário vs. Modo Núcleo (Kernel)
A implementação de threads varia conforme o nível do sistema operacional responsável pelo seu gerenciamento:

* **Threads em Modo Usuário (*User-Level Threads*):**
  * **Funcionamento:** São gerenciadas inteiramente no espaço do usuário por meio de uma biblioteca de rotinas (*runtime*), mantendo uma tabela de threads própria sem o conhecimento do kernel do sistema operacional.
  * **Vantagens:** Criação, destruição e alternância de contexto extremamente rápidas, pois não exigem chamadas de sistema (*system calls*) ou privilégios de kernel.
  * **Desvantagens:** Se uma única thread realizar uma chamada de sistema bloqueante (como leitura/escrita de E/S em disco ou rede), todo o processo é bloqueado pelo kernel, impedindo a execução das demais threads. Além disso, não aproveitam o paralelismo real em arquiteturas multiprocessadas de forma nativa.

* **Threads em Modo Núcleo (*Kernel-Level Threads*):**
  * **Funcionamento:** O kernel do sistema operacional mantém a tabela de threads e é responsável por criar, agendar e destruir cada linha de execução diretamente.
  * **Vantagens:** Se uma thread for bloqueada por operações de E/S, o escalonador do kernel pode continuar executando outras threads do mesmo processo sem pará-lo por completo. Permite o paralelismo real em CPUs *multicore*.
  * **Desvantagens:** A criação e a troca de contexto são ligeiramente mais custosas em comparação ao modo usuário, pois demandam a transição entre o Modo Usuário e o Modo Kernel.

---

## 2. Diagnóstico do Problema (O Gargalo)

### 2.1 A Condição de Corrida (*Race Condition*)
No cenário em que os Usuários A e B tentam comprar o mesmo assento (Assento A-15) no mesmo milissegundo, ocorre uma **Condição de Corrida**. 

A Condição de Corrida é definida como uma situação assíncrona em que duas ou mais rotinas de execução (threads ou processos) acedem e manipulam uma mesma área de memória ou recurso partilhado concorrentemente, de modo que o resultado final da operação depende da ordem exata em que o escalonador da CPU intercala a execução dos passos de cada rotina.

### 2.2 Impacto no Sistema de Ingressos sem Tratamento
Se o sistema não tratar essa concorrência na base de dados ou na memória central, o seguinte fluxo inconsistente poderá ocorrer:

1. **Leitura Concorrente:** A Thread do Usuário A e a Thread do Usuário B leem o status do Assento A-15 no mesmo instante; ambos veem a propriedade `status = "DISPONÍVEL"`.
2. **Avaliação Concorrente:** Ambas as threads validam que o assento está livre e avançam para a etapa de pagamento e confirmação.
3. **Escrita Concorrente (Sobregravação):** A Thread A atualiza o banco de dados definindo o Assento A-15 como "VENDIDO (Usuário A)". Em seguida, sem tomar conhecimento da alteração da Thread A, a Thread B sobregrava o registro definindo o Assento A-15 como "VENDIDO (Usuário B)".

**Consequência:**
Ocorre o problema de **Venda Dupla (*Overbooking*)**. O sistema emite dois ingressos confirmados para o mesmo assento físico (Assento A-15), gerando prejuízos financeiros, falha crítica na consistência de dados e transtornos legais e operacionais para a empresa na entrada do evento.

---

## 3. A Solução Arquitetural

### 3.1 Exclusão Mútua e Regiões Críticas
Para impedir a ocorrência de condições de corrida, a arquitetura de software deve aplicar o princípio da **Exclusão Mútua** (*Mutual Exclusion*) dentro das **Regiões Críticas** da aplicação.

* **Região Crítica (R.C.):** É a seção de código na qual o sistema realiza leitura, modificação ou gravação em um recurso compartilhado (no caso, a checagem e alteração do status do Assento A-15).
* **Exclusão Mútua:** É a garantia de que, enquanto um processo ou thread estiver executando instruções dentro da sua Região Crítica referente a determinado recurso, nenhuma outra thread/processo poderá entrar em suas próprias Regiões Críticas para aquele mesmo recurso.

### 3.2 Aplicação Prática ao Assento A-15
Para resolver a contenção do Assento A-15, implementa-se um mecanismo de sincronização (como *Mutex/Semáforo* na memória do servidor ou *Distributed Lock/Row-level Locking* na camada de persistência):


1. **Solicitação de Acesso:** Quando o Usuário A clica em "Comprar", a sua thread solicita a trava (*lock*) de exclusão mútua referente ao Assento A-15 antes de iniciar a transação.
2. **Bloqueio da Região Crítica:** A thread do Usuário A adquire o *lock* e entra na Região Crítica. O Assento A-15 fica temporariamente bloqueado para modificações por outras threads.
3. **Fila de Espera / Bloqueio:** Quando a thread do Usuário B tenta processar a compra do Assento A-15 no mesmo milissegundo, ela identifica que a Região Crítica está ocupada. A thread do Usuário B é colocada em estado de espera (*Blocked/Waiting*) ou recebe a notificação de erro informando que o assento está sob processo de compra.
4. **Finalização e Liberação:** A thread do Usuário A conclui a validação, atualiza o status para "VENDIDO (Usuário A)" e libera o *lock*.
5. **Resultado Garantido:** Quando a thread do Usuário B obtém a oportunidade de verificar o Assento A-15, a leitura atualizada indicará que o assento não está mais disponível, impedindo a venda dupla e garantindo a integridade transacional da aplicação.

---

## 4. Referências Bibliográficas

SILBERSCHATZ, Abraham; GALVIN, Peter Baer; GAGNE, Greg. **Fundamentos de Sistemas Operacionais**. 9. ed. Rio de Janeiro: LTC, 2015.

TANENBAUM, Andrew S.; BOS, Herbert. **Sistemas Operacionais Modernos**. 4. ed. São Paulo: Pearson Education do Brasil, 2015.