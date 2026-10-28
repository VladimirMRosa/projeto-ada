## Template de System Design: Aula 02 - APIs Resilientes (Respondido)

**Instruções:** Faça uma cópia deste arquivo e preencha as seções abaixo com as conclusões da sua pesquisa e os debates. O foco é a integração da API de Checkout com o Serviço de Pagamento Externo para suportar 10.000 usuários simultâneos.

### 1. Escolha de Paradigma da API
- **A Escolha:** (X) Spring MVC com Virtual Threads
- **Justificativa:** Considerando o cenário de alta concorrência (10.000 utilizadores simultâneos) e a dependência de uma API externa de pagamentos, a utilização de Virtual Threads permite manter o modelo de programação imperativo do Spring MVC com menor complexidade operacional do que uma arquitetura totalmente reativa. Como as chamadas ao parceiro de pagamentos são operações I/O-bound, as Virtual Threads conseguem escalar milhares de requisições concorrentes sem bloquear recursos valiosos do sistema, reduzindo o consumo de threads do sistema operativo e simplificando a manutenção da aplicação.

### 2. Desenho de Resiliência

Dependemos de uma API terceira instável. Quais recursos do padrão de resiliência você vai utilizar?

- **Proteção contra Falhas e Lentidão:**
  - **Circuit Breaker:** interrompe chamadas para o parceiro quando a taxa de falhas ultrapassa um limite definido, evitando efeito cascata.
  - **TimeLimiter:** define um tempo máximo para resposta do parceiro, impedindo que requisições fiquem presas indefinidamente.
  - **Retry:** realiza novas tentativas apenas para falhas transitórias (timeout, erros temporários de rede), com backoff exponencial.
  - **RateLimiter:** protege a API contra excesso de chamadas simultâneas, evitando sobrecarga interna e externa.
  - **Bulkhead:** isola os recursos utilizados para integração com pagamentos, impedindo que problemas nesse serviço afetem outras funcionalidades do sistema.

- **Proteção contra Duplicidade:**
  A cada requisição de checkout é gerada uma Idempotency-Key única. Antes de processar um pagamento, o sistema verifica se essa chave já existe. Caso exista, devolve o resultado anteriormente registado sem criar uma nova cobrança. O ciclo de vida da chave inicia-se no momento da criação da requisição, permanece armazenado durante o processamento e por um período configurado de retenção (por exemplo, 24 horas) para evitar cobranças duplicadas causadas por retries, refresh do navegador ou falhas de comunicação.

### 3. Os Edge Cases (Engenharia do Caos)

- **E se...** o parceiro de pagamentos começar a responder em 20 segundos devido a degradação de desempenho?
- **Resposta do Sistema:** o TimeLimiter interrompe a chamada após o tempo configurado e o Circuit Breaker contabiliza a falha. Após atingir o limite de erros, o Circuit Breaker abre e passa a rejeitar chamadas temporariamente.

- **E se...** o utilizador submeter o mesmo pedido várias vezes devido a lentidão na interface?
- **Resposta do Sistema:** a mesma Idempotency-Key será utilizada e apenas uma cobrança será criada. As demais requisições receberão o mesmo resultado já registado.

- **E se...** ocorrer um pico acima de 10.000 utilizadores simultâneos durante uma campanha promocional?
- **Resposta do Sistema:** o RateLimiter controla a taxa de entrada, enquanto o Bulkhead impede que a integração de pagamentos consuma todos os recursos da aplicação. O sistema mantém estabilidade mesmo sob elevada pressão.

### 4. Testando a Resiliência
- **Estratégia:**
  - Simular latência de 10 segundos no parceiro de pagamentos utilizando WireMock ou MockServer.
  - Executar testes de carga com JMeter ou Gatling gerando milhares de utilizadores simultâneos.
  - Forçar erros HTTP 500 e timeouts para validar a abertura do Circuit Breaker.
  - Monitorizar métricas do Resilience4j para verificar a transição entre estados Closed, Open e Half-Open.
  - Validar através de testes automatizados que nenhuma thread permanece bloqueada após o timeout configurado.

### Considerações sobre Banco de Dados

Com base no material de apoio, o armazenamento deve considerar:
- Propriedades ACID para operações críticas de pagamento.
- Avaliação de consistência eventual em fluxos assíncronos.
- Estratégias de identificação como UUID ou ULID para melhor escalabilidade.
- Utilização de PostgreSQL com índices apropriados, otimização de queries, controlo de concorrência e locks quando necessário.
- Planeamento de volumetria, expurgo de dados antigos e análise de performance utilizando EXPLAIN ANALYZE.
