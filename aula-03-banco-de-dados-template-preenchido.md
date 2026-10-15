# Template de System Design: Aula 03 - Banco de Dados em Larga Escala

> **Contexto do Cenário: Checkout na Black Friday**
>
> Sua equipe está preparando a persistência da API de Checkout para o pico de tráfego da Black Friday:
>
> 1. **Armazenamento:** Transações financeiras de pedidos exigem garantias ACID rigorosas e consistência imediata, enquanto dados de catálogo e sessões de usuários demandam alta vazão de leitura e tolerância a partição.
> 2. **Concorrência:** A atualização de status do pedido tem baixa disputa por registro (apenas o comprador interage), mas a **reserva de estoque** sofre altíssima contenção com centenas de clientes disputando as últimas unidades do mesmo SKU promocional.
> 3. **Incidente em Produção:** Em testes de carga, o tempo de resposta p99 subiu para 4.8s e o pool HikariCP saturou devido a consultas lentas de pedidos (`WHERE customer_id = ?` e `WHERE order_status = 'processing'`).
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Adicione este bloco ao repositório contínuo do seu projeto.

---

## 1. Decisão de Armazenamento e Chave Primária (PK)

### 1.1 Motor de Banco de Dados e Paradigma de Persistência

- **A Escolha para o Checkout:** ( ) Relacional Puro (PostgreSQL) (X) Persistência Poliglota / Híbrida
- **Justificativa Arquitetural:** Avalie o Teorema CAP e o modelo de consistência (ACID vs BASE). Por que a sua escolha atende aos requisitos do Checkout sem comprometer a integridade financeira?
  > O cenário descreve dois perfis de dado opostos, e um único motor atenderia mal um dos dois.
  >
  > **Dados transacionais (pedidos, pagamentos, estoque):** exigem ACID e consistência imediata. Pelo Teorema CAP, em caso de partição eu prefiro **CP**: recusar ou atrasar uma operação é aceitável, mas vender o mesmo item duas vezes ou perder um pagamento não é. Por isso essas tabelas ficam no PostgreSQL, com transações, constraints (`CHECK (quantidade >= 0)`, chaves estrangeiras) e isolamento `READ COMMITTED`, que é suficiente quando a concorrência é controlada com lock explícito ou UPDATE condicional.
  >
  > **Dados de catálogo e sessão:** o próprio cenário pede alta vazão de leitura e tolerância a partição, ou seja, **AP / BASE** (disponibilidade e consistência eventual). Um preço de catálogo defasado por alguns segundos tem custo baixo, e a leitura escala horizontalmente com réplicas, cache e índices próprios. Esses dados vão para Redis (sessão e cache) e para um store de leitura (documento ou busca).
  >
  > **Por que não relacional puro:** colocar catálogo e sessões no mesmo PostgreSQL faria a leitura em massa competir por conexões e I/O com as transações financeiras. Esse acoplamento agravaria o tipo de incidente descrito na seção 3 (pool saturado). **Por que não só NoSQL:** abriria mão das garantias ACID justamente onde o negócio não tolera erro.
  >
  > **Custo assumido:** poliglota significa mais componentes para operar e consistência eventual entre stores. Mitigo mantendo uma única fonte da verdade (PostgreSQL) e propagando mudanças por eventos (outbox pattern), de modo que os demais stores são projeções reconstruíveis.

- **Além do Modelo Documental (Ecossistema NoSQL):** A persistência poliglota vai muito além da dicotomia "Relacional vs MongoDB". Considerando as necessidades complementares de uma plataforma de e-commerce (como carrinhos efêmeros, busca textual de produtos, telemetria/logs, auditoria temporal ou análise de conexões/fraude), cite e comente pelo menos outros dois tipos/famílias de bancos NoSQL que poderiam coexistir na arquitetura, indicando para quais casos de uso específicos cada um seria a ferramenta ideal e por quê.
  > 1. **Chave-valor em memória (Redis, ou DynamoDB):** ideal para **carrinhos efêmeros e sessões**. O acesso é sempre por chave, a latência é de sub-milissegundo e o TTL expira carrinhos abandonados sem job de limpeza. Também serve de cache do catálogo e de contador atômico (`DECR`) para proteger o estoque na entrada.
  > 2. **Motor de busca (Elasticsearch / OpenSearch):** ideal para **busca textual de produtos**. Usa índice invertido, relevância, filtros por facetas e tolerância a erros de digitação, coisas que um `LIKE '%termo%'` no relacional não entrega e que travariam o banco transacional.
  > 3. **Colunar / séries temporais (Cassandra, ClickHouse, TimescaleDB):** ideal para **telemetria, logs e auditoria temporal**. Escrita append-only em alto volume e consultas por janela de tempo e agregações, com retenção e compressão por período.
  > 4. **Grafo (Neo4j):** ideal para **análise de conexões e fraude**. Detectar anéis de fraude (vários cartões, dispositivos e endereços ligados ao mesmo grupo) é percorrer relacionamentos, e em grafo isso é uma travessia nativa, enquanto em SQL vira uma cadeia de JOINs recursivos cara.

### 1.2 Estratégia da Chave Primária

- **A Escolha:** ( ) Sequence / Identity ( ) UUIDv4 (X) UUIDv7 / TSID
- **Impacto no Armazenamento:** Por que essa estratégia foi escolhida considerando inserções concorrentes e o impacto de fragmentação na árvore B-Tree (page splits)?
  > A PK é indexada em B-Tree, e a ordem dos valores inseridos define o comportamento do índice sob inserção concorrente.
  >
  > - **UUIDv4 (aleatório):** cada inserção cai em uma folha qualquer da árvore. Isso provoca **page splits** frequentes, páginas meio vazias (bloat), baixa taxa de acerto no cache de buffer (a árvore inteira precisa ficar quente) e mais escrita de WAL. Em pico de Black Friday, é o pior caso.
  > - **Sequence / Identity:** é sequencial, ótimo para a B-Tree, mas depende de uma chamada ao banco para gerar o ID, é previsível (permite enumerar pedidos de outros clientes) e dificulta sharding e geração distribuída.
  > - **UUIDv7 / TSID:** os bits mais significativos são um **timestamp**, então os IDs são aproximadamente crescentes. As inserções vão sempre para o lado direito da árvore (append-like), com poucos page splits e cache eficiente, **sem perder** a geração descentralizada na aplicação e a não-enumerabilidade.
  >
  > Escolho **UUIDv7**: ordenável por tempo, gerado na aplicação (sem round-trip) e preparado para particionar ou fazer sharding no futuro. Se o tamanho do índice for crítico, o TSID (64 bits, `bigint`) ocupa metade dos 16 bytes do UUID, com a mesma propriedade de ordenação. O trade-off do UUIDv7 é expor o instante de criação no ID, o que é aceitável para pedidos.

---

## 2. A Dualidade da Concorrência: Locks e Mitigações

Analise e defina a abordagem de concorrência para os dois fluxos críticos do negócio:

### 2.1 Mudança de Status do Pagamento (Baixa contenção por registro)

- **Abordagem de Lock:** (X) Lock Otimista (`@Version`) ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Por que essa abordagem é a mais adequada para este cenário específico?
  > O cenário diz que só o comprador interage com o próprio pedido, então a probabilidade de duas escritas simultâneas no mesmo registro é baixíssima. O lock otimista **não segura lock no banco durante a leitura**: lê a linha com a versão e, no `UPDATE ... WHERE id = ? AND version = ?`, detecta se alguém alterou. Se ninguém alterou (quase sempre), o custo é zero de espera e nenhuma conexão fica presa. O lock pessimista seria pagar um custo (lock + conexão retida) para evitar um conflito que praticamente não acontece.
  >
  > Ele ainda protege contra o caso raro real, como um callback duplicado do gateway de pagamento chegando junto com uma ação do usuário, porque o segundo escritor falha com `OptimisticLockException` em vez de sobrescrever silenciosamente (lost update).

### 2.2 Controle de Estoque Concorrente (Alta contenção no mesmo registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou (X) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Centenas de usuários disputando os últimos itens na Black Friday. Como você garante que o saldo nunca fique negativo sem derrubar o banco?
  > Com centenas de clientes disputando o mesmo SKU, o lock otimista **colapsa**: muitos leem a mesma versão, um vence e os outros falham e refazem, gerando uma tempestade de retries que consome CPU e conexões e frustra o cliente com erros. Nesse cenário, **enfileirar é mais barato que falhar e repetir**, então uso lock pessimista, na forma mais enxuta possível.
  >
  > **Como garanto saldo nunca negativo sem derrubar o banco:**
  > 1. **Decremento atômico condicional**, que adquire o lock de linha implicitamente e faz a verificação e a escrita em uma única instrução: `UPDATE estoque SET qtd = qtd - :n WHERE sku_id = :id AND qtd >= :n`. Se afetou 0 linhas, não há saldo e respondo "esgotado" imediatamente, sem leitura prévia.
  > 2. **Rede de segurança no schema:** `CHECK (qtd >= 0)`, para que mesmo um bug de aplicação não consiga gravar saldo negativo.
  > 3. **Transação curta:** o lock dura só o tempo do UPDATE. **Nunca chamo o gateway de pagamento dentro dessa transação.** A reserva é uma linha de `reserva_estoque` com TTL (ex.: 10 min); se o pagamento não confirmar, um job devolve o saldo.
  > 4. **Proteção do banco na entrada:** um contador atômico no Redis (`DECR`) barra a maioria das tentativas quando o estoque acaba, e só quem passa vai ao PostgreSQL.
  > 5. **Se um SKU for extremamente quente:** dividir o saldo em N "baldes" (sharding lógico do estoque) distribui os locks por várias linhas em vez de uma.

### 2.3 Riscos e Mitigações

- **Análise Crítica:** Quais os riscos operacionais das abordagens de lock que você escolheu (ex: excesso de `OptimisticLockException` frustrando o cliente vs `ConnectionTimeoutException` e *deadlocks* enfileirando threads no Lock Pessimista)? Como sua arquitetura mitiga esses riscos?
  > **Lock otimista (status do pedido):**
  > - *Risco:* `OptimisticLockException` em excesso frustra o cliente. *Por que é baixo aqui:* a contenção por registro é baixa. *Mitigação:* retry limitado (2 a 3 tentativas com backoff curto) para conflitos benignos, combinado com idempotência (a mesma notificação de pagamento processada duas vezes não pode gerar duas transições). Se a taxa de exceções subir, é sinal de que a premissa de baixa contenção deixou de valer e merece revisão.
  >
  > **Lock pessimista (estoque):**
  > - *Risco 1, saturação do pool:* threads esperando lock seguram conexões do HikariCP até acabar o pool e dar `ConnectionTimeoutException` para quem nem toca no estoque. *Mitigação:* transação curta, `lock_timeout` e `statement_timeout` no banco (falhar rápido e liberar a conexão), pool dimensionado com `connectionTimeout` baixo (fail-fast) e um **bulkhead** (pool ou limite de concorrência separado) para o fluxo de estoque, de modo que ele não consuma conexões do resto do sistema.
  > - *Risco 2, deadlock:* ocorre quando duas transações travam recursos em ordens diferentes (ex.: um carrinho com SKU A e B de um lado, B e A do outro). *Mitigação:* **ordem determinística de aquisição** (sempre ordenar os SKUs do pedido por ID antes de travar) e retry ao tratar o erro de deadlock.
  > - *Risco 3, experiência do cliente:* *Mitigação:* resposta rápida e clara ("esgotado") em vez de espera longa, graças ao filtro de Redis e aos timeouts curtos.
  > - *Observabilidade:* monitorar `pg_locks`, tempo de espera por lock, deadlocks (`pg_stat_database`) e a métrica `hikaricp.connections.pending` com alerta.

---

## 3. Troubleshooting do Incidente e Gestão de Índices

### 3.1 Investigação de Incidente em Produção

- **Cenário:** Em testes de carga para a Black Friday, o tempo de resposta p99 do Checkout subiu repentinamente para 4.8s e o pool de conexões (HikariCP) saturou. Os logs apontam que a consulta de pedidos mais frequente (`WHERE customer_id = ? AND order_status = 'processing'`) está demorando segundos para responder.
- **Plano de Diagnóstico:** Qual é o seu processo técnico passo a passo para investigar a causa raiz dessa lentidão diretamente no banco de dados? Quais comandos, métricas ou planos você analisaria para comprovar a hipótese antes de propor qualquer mudança?
  > Princípio: **medir antes de mudar**. A saturação do HikariCP é um *sintoma*: consultas lentas seguram conexões por mais tempo (Lei de Little: conexões em uso = taxa de requisições × tempo de cada uma) e o pool esgota. Preciso provar onde está o tempo.
  >
  > 1. **Confirmar o sintoma na aplicação:** métricas do HikariCP (`active`, `pending`, `usage time`) e o p99 por endpoint, para ver que o tempo está no banco e não em outra chamada.
  > 2. **Achar as queries mais caras:** `pg_stat_statements`, ordenando por tempo total e tempo médio, para confirmar que a query de `customer_id` + `order_status` domina.
  > 3. **Ver o que o banco está fazendo agora:** `pg_stat_activity` (queries ativas, estado, `wait_event`) para distinguir lentidão por I/O/CPU de espera por lock, e `pg_locks` para descartar contenção.
  > 4. **Analisar o plano de execução:** `EXPLAIN (ANALYZE, BUFFERS)` da query real, em ambiente com volume semelhante ao de produção. Sinais da hipótese: **`Seq Scan`** na tabela de pedidos, grande número de *Rows Removed by Filter*, muitos blocos lidos e estimativa de linhas muito diferente do real (estatísticas desatualizadas).
  > 5. **Checar a saúde da tabela:** `pg_stat_user_tables` (proporção `seq_scan` x `idx_scan`), bloat, atividade do autovacuum e última execução de `ANALYZE`, pois estatísticas velhas também levam o planner a escolher mal.
  > 6. **Hipótese a comprovar:** não há índice adequado para o filtro composto, então o banco varre a tabela inteira a cada chamada.
  > 7. **Só então propor a mudança** (ver 3.2): índice composto `(customer_id, order_status)`, criado com `CREATE INDEX CONCURRENTLY` para não bloquear escrita, e depois **repetir o `EXPLAIN ANALYZE`** e o teste de carga para comprovar a melhora (de `Seq Scan` para `Index Scan` e p99 de volta ao patamar esperado).

### 3.2 Decisão de Engenharia: A Abordagem de Índices

- **Dilema no Pull Request:** Para solucionar a lentidão, um desenvolvedor da equipe abriu um PR criando índices individuais para todas as colunas da tabela de pedidos que aparecem em filtros de consulta.
- **Seu Posicionamento:** Como responsável técnico, você aprovaria esse PR? Qual é a sua avaliação crítica sobre essa estratégia e quais impactos colaterais sistêmicos devem ser ponderados antes de aprovar novos índices?
  > **Não aprovaria o PR como está.** O problema é real, mas "um índice para cada coluna de filtro" é uma resposta sem evidência e com custo alto.
  >
  > **Avaliação crítica:**
  > - **Índices individuais não resolvem consultas com filtros combinados.** Para `WHERE customer_id = ? AND order_status = 'processing'`, o ideal é **um índice composto** `(customer_id, order_status)`, com a coluna de maior seletividade (`customer_id`) primeiro. Um índice isolado em `order_status` tem **baixa cardinalidade** (poucos valores distintos) e o planner frequentemente o ignora.
  > - **Custo de escrita (write amplification):** cada `INSERT` e `UPDATE` precisa atualizar todos os índices. O status do pedido muda várias vezes no ciclo de vida, então indexar essa coluna também **impede updates HOT** no PostgreSQL, aumentando a escrita e o trabalho de vacuum. Em plena Black Friday, isso piora justamente o caminho de escrita do checkout.
  > - **Custo de espaço e memória:** índices ocupam disco, disputam o cache de buffer com os dados quentes e aumentam o tempo de backup e manutenção.
  >
  > **Impactos sistêmicos a ponderar antes de aprovar qualquer índice:** proporção leitura/escrita da tabela, cardinalidade e seletividade da coluna, uso real (`pg_stat_user_indexes.idx_scan` para identificar índices que ninguém usa), tempo de criação em tabela grande e ordem das colunas conforme as queries reais.
  >
  > **O que eu proporia no lugar:** (1) índice composto guiado pelas queries reais e comprovado com `EXPLAIN ANALYZE`; (2) para a consulta de pedidos `processing`, avaliar **índice parcial** (`WHERE order_status = 'processing'`), pequeno e barato de manter; (3) criação com `CONCURRENTLY`; (4) revisão periódica de índices não usados. Conduziria isso no PR como feedback colaborativo, explicando os motivos e pareando com o autor, porque o objetivo é elevar o critério do time e não só barrar a mudança.
