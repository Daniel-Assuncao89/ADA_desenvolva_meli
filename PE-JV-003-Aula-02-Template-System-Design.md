# Template de System Design: Aula 02 - APIs Resilientes

> **Instruções:** Faça uma cópia deste arquivo e preencha as seções abaixo com as conclusões da sua pesquisa e os debates. O foco é a integração da API de Checkout com o Serviço de Pagamento Externo para suportar 10.000 usuários simultâneos.

## 1. Escolha de Paradigma da API

- **A Escolha:** (x) Spring MVC com Virtual Threads ou ( ) Spring WebFlux (Reativo)?
- **Justificativa:** Considerando a capacidade de suportar 10k acessos concorrentes, por que você fez essa escolha técnica?
  > Escolhi Spring MVC com Virtual Threads. O gargalo do checkout é espera de I/O (banco e serviço de pagamento), não CPU. Com virtual threads, a thread não fica presa enquanto espera a resposta, então suportamos os 10k acessos concorrentes sem esgotar o pool. Isso me dá a escala do reativo mantendo código imperativo, simples de ler e de depurar, e ainda reaproveito JDBC/JPA. WebFlux resolveria também, mas traz mais complexidade (Mono/Flux, stack todo não bloqueante) sem um ganho claro para esse caso. Eu só trocaria se precisasse de backpressure ou streaming. Um cuidado: virtual threads removem o limite de threads, então o limite de concorrência precisa ser explícito (Bulkhead e pool de conexões dimensionado).

## 2. Desenho de Resiliência

Dependemos de uma API terceira instável. Quais recursos do padrão de resiliência você vai utilizar?

- **Proteção contra Falhas e Lentidão:**
  Quais padrões (Circuit Breaker, TimeLimiter, RateLimiter, Bulkhead, Retry) você implementaria na chamada para o Parceiro de Pagamentos? Por que?
  > Usaria os cinco juntos, porque cada um cobre uma falha diferente:
  > - **TimeLimiter:** timeout de 2s na chamada (junto com timeout de conexão e leitura no HTTP client). Sem ele, uma chamada lenta segura thread e conexão.
  > - **Retry:** 3 tentativas com backoff exponencial e jitter, só para timeout e 5xx. Erro de negócio (cartão recusado) não repete. Só roda porque a chamada é idempotente.
  > - **Circuit Breaker:** abre quando a taxa de falha ou de lentidão passa de 50% e passa a falhar rápido, para não insistir em um parceiro doente. Recusa de cartão não conta como falha.
  > - **Bulkhead:** limita as chamadas simultâneas ao parceiro (semáforo), para a lentidão dele não consumir todos os recursos do checkout.
  > - **RateLimiter:** respeita a cota do parceiro e segura o pico de retries.
  >
  > A ordem é Retry → Circuit Breaker → RateLimiter → TimeLimiter → Bulkhead. Quando o circuito abre, o fallback aceita o pedido como `PAGAMENTO_PENDENTE` e enfileira a cobrança. Os valores são pontos de partida e seriam calibrados com a latência real do parceiro.

- **Proteção contra Duplicidade:**
  Como a `Idempotency-Key` será utilizada na requisição do Checkout para evitar cobranças duplas? Qual o ciclo de vida dessa chave?
  > O cliente gera um UUID por intenção de compra e envia no header `Idempotency-Key`. Em reenvio, usa a mesma chave. Ao receber, o servidor tenta gravar a chave (com hash do payload) com unique constraint e estado `EM_PROCESSAMENTO`. Quem grava primeiro processa, repassando a chave ao parceiro. Ao terminar, guarda a resposta e muda para `CONCLUIDA`.
  >
  > Se a mesma chave chegar de novo, devolvo a resposta guardada, sem cobrar outra vez. Se chegar enquanto ainda processa, retorno `409`. Se chegar com payload diferente, retorno `422`. A chave expira por TTL (24h, por exemplo). Uso o banco como fonte da verdade, porque perder esse estado significa cobrança duplicada.

## 3. Os Edge Cases (Engenharia do Caos)

Cole aqui as 3 perguntas de risco que você formulou e defina como a arquitetura responde a cada uma delas.

1. **E se...** o timeout estourar depois de enviarmos a cobrança, e não soubermos se o parceiro processou?
   - **Resposta do Sistema:** O pedido vai para `INCERTO`, nunca para falha. Respondo ao cliente que o pagamento está em processamento. Consulto o status no parceiro com a mesma chave. Se confirmou, marco como pago. Se não existe cobrança, reenvio com a mesma chave, que é seguro. A conciliação periódica cobre o que escapar.
2. **E se...** o parceiro ficar fora por 30 minutos e aceitarmos os pedidos como pendentes?
   - **Resposta do Sistema:** Limito a exposição com teto de valor, reserva de estoque com expiração e antifraude local. Se a cobrança for recusada depois, uma saga cancela o pedido, libera o estoque e avisa o cliente. Uma feature flag permite voltar a falhar rápido se os pendentes acumularem. O limite aceitável é decisão com produto.
3. **E se...** o parceiro voltar e a fila acumulada for processada de uma vez?
   - **Resposta do Sistema:** Drenagem com taxa controlada (RateLimiter e Bulkhead). O circuito volta por half-open, com poucas chamadas de prova. Retry com backoff e jitter. Mensagens que continuam falhando vão para uma DLQ. Isso evita derrubar o parceiro de novo (retry storm).

## 4. Testando a Resiliência

- **Estratégia:** Como você vai garantir que o Circuit Breaker abre na hora certa e o Timeout não deixa a Thread presa?
  > Em teste de integração, simulo o parceiro com WireMock (500, 503 e 10s de latência) e Toxiproxy. Para o circuito, envio o mínimo de chamadas com 50% de falha, confirmo a transição para OPEN e verifico que as chamadas seguintes não chegam ao parceiro. Depois confirmo HALF_OPEN e o retorno a CLOSED quando o parceiro volta.
  >
  > Para o timeout, simulo 10s de latência com timeout de 2s e confirmo que a chamada falha em cerca de 2s. Disparo muitas chamadas lentas e acompanho threads e conexões do pool, que não podem crescer sem limite (no JFR, olho o evento de pinning). Por fim, rodo um teste de carga no k6 com 10k usuários, injeto a falha no meio e meço p95/p99, taxa de erro e tempo de recuperação. Também testo idempotência com requisições paralelas na mesma chave: só pode existir uma cobrança.