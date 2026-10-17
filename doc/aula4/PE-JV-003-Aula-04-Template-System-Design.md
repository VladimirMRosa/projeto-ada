# Template de System Design: Aula 04 - Microsserviços & Padrão Saga

> **Contexto do Cenário: O Pedido Pago Sem Produto na Black Friday**
>
> Sua equipe está arquitetando a transação distribuída de compra na Black Friday para o marketplace:
>
> 1. **O Incidente:** Um cliente comprou uma Smart TV de R$ 3.000 via Pix. O gateway de pagamentos debitou o valor imediatamente e confirmou a transação. No milissegundo seguinte, o serviço de Estoque esgotou as unidades e rejeitou a reserva. O dinheiro sumiu da conta do cliente, o produto não existe para entrega e o pedido travou em um estado inconsistente.
> 2. **O Desafio Distribuído:** Não existe transação única (`@Transactional`) entre bancos de dados autônomos de Checkout, Pagamento e Estoque.
> 3. **A Carga:** 5.000 TPS de checkout simultâneo e milhões de clientes consultando o status da compra em "Meus Pedidos".
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Transações Distribuídas: Coordenação e Trade-offs

### 1.1 Volumetria e Pragmatismo: 2PC vs Saga

- **Análise Crítica:** Em quais cenários arquiteturais uma transação ACID local ou Two-Phase Commit (2PC) resolveria o problema com menor custo operacional? Por que, para o cenário de 5.000 TPS na Black Friday com serviços e bancos autônomos, o 2PC é inviável e a consistência eventual se torna necessária?
  > Uma transação ACID local ou até mesmo 2PC seria adequada em ambientes menores , com poucos serviços, baixo volume de transações e forte necessidade de consistência imediata. Nesses cenários, o custo operacional é menor porque todos os participantes conseguem manter uma coordenação centralizada sem impactar significativamente a performance.
  >
  > No cenário da Black Friday, com 5.000 TPS e bancos independentes para Checkout, Pagamento e Estoque, o 2PC torna-se inviável porque exige que todos os serviços permaneçam sincronizados durante a transação. Isso aumenta a latência, reduz a disponibilidade e pode bloquear o fluxo caso um dos participantes fique lento ou indisponível.
  >
  > Por esse motivo, a melhor abordagem é utilizar Saga com consistência eventual. Cada serviço executa sua própria transação local e, caso ocorra uma falha, como pagamento aprovado e estoque indisponível, uma transação compensatória realiza automaticamente o estorno. Dessa forma, a arquitetura mantém escalabilidade, resiliência e autonomia entre os microsserviços sem comprometer o alto throughput exigido pelo negócio.

### 1.2 O Modelo de Coordenação da Saga

- **A Escolha:** ( ) Coreografia (Orientada a Eventos)  (X) Orquestração (Process Manager / Máquina de Estados)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Coreografia:* Como seu design resolve a falta de visibilidade centralizada (saber em tempo real onde o pedido está) e previne dependências cíclicas à medida que novos serviços entrarem no fluxo de compra?
  - *Se você escolheu Orquestração:* Qual componente atua como Orquestrador, como ele se comunica com os participantes (síncrono ou assíncrono) e como você evita que ele se transforme em um "serviço Deus", tirando a inteligência e a autonomia dos outros microsserviços?
  > A Saga será coordenada pelo microsserviço Order Orchestrator Service, responsável por manter a máquina de estados do pedido e coordenar o fluxo entre os serviços de Checkout, Pagamento e Estoque.
  > 
  > As regras de negócio permanecem nos microsserviços de domínio. O serviço de Estoque decide se a reserva pode ser realizada, o serviço de Pagamento aprova ou estorna pagamentos e o serviço de Checkout cria o pedido. O Orchestrator apenas coordena o fluxo e controla os estados da Saga.

A comunicação ocorre de forma assíncrona através do Kafka. O Orchestrator publica comandos para os participantes da Saga e consome os eventos de retorno para decidir o próximo passo da transação ou iniciar uma compensação quando necessário.

---

## 2. Máquina de Estados, Compensação e Confiabilidade

### 2.1 Máquina de Estados e Fluxos Ponta a Ponta

- **Estados do Pedido:** Defina a máquina de estados explícita do ciclo de vida da transação no seu modelo (incluindo os estados iniciais, intermediários, de compensação e finais).
  > Descrição de cada item:
  CREATED: pedido criado pelo Checkout.
  >
  > STOCK_RESERVATION_PENDING: solicitação de reserva enviada ao Estoque.
  >
  > STOCK_RESERVED: estoque reservado com sucesso.
  >
  > PAYMENT_PROCESSING: solicitação de cobrança enviada ao Pagamento.
  >
  > PAYMENT_APPROVED: pagamento aprovado.
  >
  > ORDER_CONFIRMED: pedido finalizado com sucesso.
  >
  > STOCK_REJECTED: falha na reserva de estoque.
  >
  > REFUND_PENDING: compensação iniciada, aguardando estorno.
  >
  > REFUND_COMPLETED: estorno realizado com sucesso.
  >
  > ORDER_CANCELLED: pedido cancelado após compensação. 

- **Detalhamento dos Fluxos:**
  - **Caminho Feliz:** Descreva o fluxo de ponta a ponta da transação concluída com sucesso entre Checkout, Pagamento e Estoque.
  - **Caminho de Falha & Compensação:** Descreva o passo a passo a partir da falha na reserva de estoque após o pagamento aprovado. Como a ação compensatória (estorno) é disparada e executada sem intervenção humana?
  > CREATED
       ↓
  >
  > STOCK_RESERVATION_PENDING
           ↓
  >
  > STOCK_RESERVED
           ↓
  >
  > PAYMENT_PROCESSING
           ↓
  >
  > PAYMENT_APPROVED
           ↓
  >
  > ORDER_CONFIRMED (Estado Final de Sucesso)

  >CREATED
      ↓
  >
  > STOCK_RESERVATION_PENDING
          ↓
  >
  > STOCK_RESERVED
          ↓
  >
  > PAYMENT_PROCESSING
          ↓
  >
  > PAYMENT_APPROVED
          ↓
  >
  > STOCK_REJECTED
           ↓
  >
  > REFUND_PENDING
           ↓
  >
  > REFUND_COMPLETED
           ↓
  >
  > ORDER_CANCELLED (Estado Final de Falha)
  >

### 2.2 Resiliência da Compensação e Consistência

- **E se a compensação falhar ou oscilar?** A chamada ou evento de estorno pode sofrer timeout ou instabilidade transitória. Como seu design garante que o cliente não fique no prejuízo se a tentativa de compensação falhar ou for reprocessada?
  > Se a tentativa de estorno falhar por timeout ou indisponibilidade temporária, o pedido não é considerado concluído. O Order Orchestrator Service mantém a Saga no estado REFUND_PENDING e continua tentando executar a compensação até obter sucesso.
  > Para evitar prejuízo ao cliente, as mensagens de compensação são processadas de forma assíncrona e idempotente. Assim, caso o evento de estorno seja reenviado ou reprocessado, o serviço de Pagamento consegue identificar que a operação já foi executada e evita estornos duplicados.
  > Além disso, a arquitetura utiliza retry automático, persistência do estado da Saga e uma DLQ (Dead Letter Queue) para eventos que não puderem ser processados após várias tentativas. Dessa forma, o pedido permanece rastreável e nenhuma compensação é perdida silenciosamente.

Com essa abordagem, garantimos que o cliente nunca fique no prejuízo: o pedido só transita para ORDER_CANCELLED após a confirmação do estorno (REFUND_COMPLETED).
- **Atomicidade entre Estado e Mensageria (Dual Write):** Se a sua arquitetura envolve publicação de eventos ou mensagens entre passos, como você garante que a persistência no banco local e o despacho da mensagem ocorram sem risco de perda silenciosa se a rede ou o broker falharem entre as duas operações?
  > Para evitar o problema de Dual Write, utilizamos o padrão Transactional Outbox. Quando um serviço precisa alterar seu estado e publicar um evento, ambas as operações são gravadas na mesma transação do banco de dados. Assim, o serviço não envia diretamente a mensagem para o Kafka.
  > Primeiro, o microsserviço persiste a alteração de negócio e registra o evento em uma tabela Outbox. Somente após o commit da transação, um processo de publicação lê essa tabela e envia o evento para o Kafka.
  > Dessa forma, mesmo que a rede ou o broker falhem após a gravação no banco, o evento não é perdido, pois continua armazenado na Outbox e poderá ser reenviado posteriormente. Isso garante consistência entre o estado persistido e as mensagens trocadas durante a execução da Saga.

---

## 3. Desacoplamento com CQRS e Estratégia de Testes

### 3.1 Consultas em Alta Vazão com CQRS

- **O Problema de Leitura:** Durante o pico de vendas, milhões de usuários atualizam a página de "Meus Pedidos" a cada poucos segundos.
- **Desenho da Solução:** Como você separa a responsabilidade de escrita (Command) da consulta de status (Query)? Como o modelo de leitura é alimentado e como sua arquitetura lida com a consistência eventual (o delay entre o pagamento e a visualização do status)?
  > A arquitetura utiliza o padrão CQRS (Command Query Responsibility Segregation), separando o fluxo de escrita do fluxo de leitura.
  > No lado Command, os serviços de Checkout, Pagamento, Estoque e o Order Orchestrator Service executam as transações de negócio e publicam eventos no Kafka.
  > No lado Query, um Order Projection Service consome esses eventos e atualiza um banco otimizado para consulta, denominado Read Model, utilizado exclusivamente pela funcionalidade "Meus Pedidos". Dessa forma, as consultas dos usuários não impactam os bancos transacionais dos microsserviços.
  > Como o modelo de leitura é atualizado de forma assíncrona, existe uma pequena janela de consistência eventual entre a aprovação do pagamento e a atualização da tela. Durante esse período, o usuário pode visualizar estados intermediários como PAYMENT_PROCESSING, PAYMENT_APPROVED ou REFUND_PENDING, refletindo o progresso real da Saga. Após o processamento dos eventos, o Read Model é atualizado e passa a exibir o estado final do pedido.
  > Essa abordagem desacopla leitura e escrita, protege os serviços transacionais durante os picos da Black Friday e permite escalar as consultas de forma independente do processamento dos pedidos.

### 3.2 Estratégia de Testes da Transação Distribuída

- **Validação de Resiliência:** Como comprovar em ambiente de testes e pipeline que a Saga funciona de verdade?
- **Plano de Testes:** Descreva como você estruturaria o teste automatizado para simular a rejeição na reserva de estoque após a cobrança e validar que a compensação automática transiciona o pedido para cancelado com sucesso.
  > Para validar a resiliência da solução, seriam executados testes de integração ponta a ponta da Saga no pipeline, utilizando os serviços, banco de dados e Kafka em ambiente controlado (Docker/Testcontainers). O objetivo é comprovar que a máquina de estados evolui corretamente tanto no caminho de sucesso quanto nos cenários de falha.
  > 1. Criar um pedido.
  > 2. Order Orchestrator inicia a Saga.
  > 3. Payment Service aprova o pagamento.
  > 4. Inventory Service simula falha na reserva.
  > 5. Evento STOCK_REJECTED é publicado no Kafka.
  > 6. Order Orchestrator recebe o evento.
  > 7. Order Orchestrator dispara REFUND_REQUESTED.
  > 8. Payment Service executa o estorno.
  > 9. Evento REFUND_COMPLETED é publicado.
  > 10. Order Orchestrator atualiza o pedido para ORDER_CANCELLED.
