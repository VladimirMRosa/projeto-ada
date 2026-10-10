# As 3 Perguntas Cruciais para Validar uma Arquitetura de Pagamentos

## Objetivo

Antes de discutir frameworks, MVC, Reativo ou Virtual Threads, uma arquitetura de pagamentos deve responder a três perguntas fundamentais de negócio e resiliência.

Estas perguntas servem como testes de caos para avaliar a capacidade do sistema de sobreviver a falhas reais de produção.

---

# 1. Posso cobrar o cliente duas vezes?

## Cenário

- Cliente inicia um pagamento.
- O gateway processa a transação com sucesso.
- Ocorre timeout na comunicação.
- O cliente tenta novamente.
- Uma segunda transação é enviada.

## Impacto

- Cobrança em duplicidade.
- Chargebacks.
- Perda de confiança.
- Custos operacionais elevados.

## Proteções Esperadas

- Idempotency Key
- Unique Constraint
- Deduplicação de requisições
- Event Store Auditoriável

## Critério de Aprovação

A arquitetura deve ser capaz de receber múltiplas requisições idênticas e processar apenas uma cobrança.

---

# 2. Posso perder a confirmação de um pagamento?

## Cenário

- O banco confirma o pagamento.
- O callback não chega.
- O serviço responsável pela confirmação fica indisponível.
- O sistema registra o pagamento como pendente, embora o cliente tenha sido cobrado.

## Impacto

- Divergência financeira.
- Reconciliação manual.
- Problemas contábeis.
- Falhas de experiência do usuário.

## Proteções Esperadas

- Saga Pattern
- Outbox Pattern
- Inbox Pattern
- Eventual Consistency
- Rotinas de Reconciliação

## Critério de Aprovação

Mesmo após falhas de comunicação, o sistema deve ser capaz de recuperar o estado correto do pagamento.

---

# 3. O que acontece quando uma dependência fica extremamente lenta?

## Cenário

Um dos serviços externos apresenta lentidão:

- Gateway de pagamento
- Banco
- Sistema antifraude
- Serviço de autenticação

A aplicação começa a acumular chamadas pendentes e recursos são consumidos gradualmente.

## Impacto

- Esgotamento de threads
- Aumento de latência
- Falha em cascata
- Indisponibilidade geral

## Proteções Esperadas

### TimeLimiter

Impede que chamadas permaneçam bloqueadas indefinidamente.

### Bulkhead

Impede que uma integração problemática consuma todos os recursos disponíveis.

### Circuit Breaker

Interrompe chamadas para serviços comprovadamente indisponíveis.

### RateLimiter

Controla a quantidade de requisições permitidas.

## Critério de Aprovação

A degradação de um parceiro externo não deve tornar toda a plataforma indisponível.

---

# MVC x Reativo com Virtual Threads

## Antes do Java 21

O modelo reativo era frequentemente escolhido para reduzir o custo de manter milhares de threads bloqueadas durante operações de I/O.

## Com Virtual Threads

O custo de uma thread torna-se significativamente menor, permitindo retomar um modelo mais simples baseado em:

- Spring MVC
- Thread por requisição
- Código imperativo
- Melhor legibilidade
- Depuração simplificada

## Quando utilizar MVC + Virtual Threads

- APIs de Pagamentos
- Serviços de Negócio
- Integrações Bancárias
- Backoffice
- Antifraude

## Quando utilizar Reativo

- Streaming
- Processamento massivo de eventos
- WebSockets em larga escala
- Milhões de conexões simultâneas

---

# Conclusão

Uma arquitetura de pagamentos resiliente deve responder positivamente às seguintes perguntas:

1. Posso evitar cobranças duplicadas?
2. Posso recuperar confirmações perdidas?
3. Posso continuar operando quando uma dependência ficar lenta?

Se a resposta for negativa para qualquer uma delas, existe um risco arquitetural relevante que precisa ser tratado antes da entrada em produção.