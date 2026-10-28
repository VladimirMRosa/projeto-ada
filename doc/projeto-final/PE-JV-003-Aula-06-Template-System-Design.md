# Template de System Design: Aula 06 - Observabilidade & Diagnóstico em Produção

> **Contexto do Cenário: O Checkout que Trava sem Dar Erro**
>
> Sua equipe responde pelo serviço de **Checkout** do marketplace, que chama **Pagamento** e **Estoque**. Os três rodam em imagens *Distroless*.
>
> 1. **O Incidente:** Na abertura da Black Friday, com **5.000 TPS**, uma versão nova do Checkout entrou em *Canary*. Minutos depois, os painéis continuavam verdes, com latência média de 120 ms, CPU abaixo de 40% e nenhum pico de HTTP 500.
> 2. **O Impacto:** A conversão despencou. Milhares de reclamações chegaram ao suporte, todas com a mesma cena. O cliente clica em "Finalizar Pedido", a tela fica carregando e o aplicativo desiste por timeout.
> 3. **As Restrições:** A imagem não tem `sh` e não há SSH nos servidores. Cada serviço escreve logs em texto livre, do seu jeito, sem um identificador que ligue a chamada do Checkout à chamada que ele fez ao Pagamento. E dado de cliente que vai parar em log continua sujeito à LGPD.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Logs e Correlação

- **Campos do Log:** Que campos todo log do Checkout precisa ter para você encontrar, no meio de milhares de linhas, as da requisição que travou? Liste os campos e diga para que serve cada um.
  > Todo log do Checkout deve conter, no mínimo, timestamp, service.name, service.version, trace_Id, operation, http.route, http.status_code e message. O trace_Id é o principal identificador para correlacionar uma requisição entre Checkout, Pagamento e Estoque, enquanto os demais campos permitem reconstruir a linha temporal do incidente, identificar a operação executada e localizar o ponto exato onde ocorreu o travamento. Os logs serão estruturados em JSON e não conterão dados pessoais para garantir conformidade com a LGPD.
- **O Caminho do Contexto:** Como o mesmo identificador de rastreamento passa do Checkout para o Pagamento e o Estoque numa chamada HTTP?
  > O contexto distribuído será propagado através do cabeçalho HTTP traceparent, seguindo o padrão W3C Trace Context. O Checkout cria o traceId inicial e envia esse identificador para o Pagamento e posteriormente para o Estoque. Cada serviço cria os seus próprios spanId, mas mantém o mesmo traceId, permitindo correlacionar logs, métricas e traces ao longo de toda a transação. Dessa forma é possível localizar exatamente em qual etapa da jornada do pedido ocorreu o travamento.
- **LGPD nos Logs:** Quais dados de cliente ficam fora de logs e traces? Para os identificadores que precisam aparecer, qual tratamento você aplica e quem pode ler esses registros?
  > Os logs e traces não devem armazenar dados pessoais como CPF, nome, e-mail, telefone, endereço, dados de pagamento ou tokens de autenticação. Para correlação e diagnóstico serão utilizados identificadores técnicos como trace_Id. Quando houver necessidade de relacionar eventos a um pedido ou cliente, serão utilizados identificadores pseudonimizados (hashes ou referências técnicas), evitando a exposição direta de dados pessoais. O acesso aos registros será restrito às equipas de Engenharia, SRE, Observabilidade e Segurança, seguindo o princípio do menor privilégio e os requisitos de segurança da LGPD.

---

## 2. Métricas

- **Visão do Cliente (RED):** Quais métricas do Checkout entram no painel? Explique como você mede a duração para que o travamento de uma parte dos clientes não desapareça na média.
  > O painel RED do Checkout deverá monitorizar Rate (requisições por segundo), Errors (falhas, timeouts e erros HTTP) e Duration (tempo de resposta das operações do Checkout). A duração não será medida apenas pela média, pois uma média de 120 ms pode esconder um subconjunto de clientes enfrentando timeouts. Para evitar esse problema serão utilizados histogramas e percentis (P50, P95, P99 e P99.9), permitindo identificar degradações que afectam apenas parte dos utilizadores e fornecendo uma visão mais fiel da experiência real do cliente.
- **Visão dos Recursos (USE):** Que recursos você monitora? Que sinal acusaria saturação antes do primeiro HTTP 500?
  > No painel USE serão monitorizados recursos como CPU, memória, pool de conexões, threads da aplicação, filas internas e integrações com Pagamento e Estoque. Para cada recurso serão observadas métricas de Utilization, Saturation e Errors. O principal indicador de degradação antes do primeiro HTTP 500 será a saturação, medida através do crescimento de filas, aumento do tempo de espera por conexões e esgotamento de recursos partilhados. Estes sinais permitem detectar gargalos internos antes que as falhas sejam percebidas pelos utilizadores.
- **Labels:** Quais labels você permite nessas métricas e quais proíbe? Diga o efeito de cada escolha no servidor de métricas.
  > Nas métricas do Checkout serão permitidos apenas labels de baixa cardinalidade, como service, environment, version, region, operation e http_status, pois produzem um número controlado de séries temporais e mantêm o Prometheus eficiente. Serão proibidos labels como user_id, cpf, email, telefone, order_id, payment_id, trace_id e session_id, porque geram uma nova série para cada requisição, causando explosão de cardinalidade, aumento de consumo de memória, degradação das consultas e possível sobrecarga do servidor de métricas. Além disso, alguns desses campos contêm dados pessoais protegidos pela LGPD.

---

## 3. Traces e Amostragem

- **A Escolha:** ( ) Head Sampling  ( ) Tail Sampling  (X) Combinação das duas
- **Justificativa:** Guardar 100% dos traces a 5.000 TPS custa caro. Qual é a sua regra de retenção, e como ela evita que justamente as requisições que travaram sejam descartadas?
  > Combinação de Head Sampling e Tail Sampling. O Head Sampling será utilizado para reduzir o volume inicial de telemetria em um ambiente de 5.000 TPS, diminuindo custos de armazenamento e processamento. Em seguida, o Tail Sampling será aplicado no OpenTelemetry Collector para reter 100% dos traces que apresentem timeout, erro, latência elevada ou pertençam à versão Canary. Dessa forma, a plataforma mantém uma amostra representativa do tráfego normal enquanto garante a preservação das requisições anómalas, evitando que exatamente os pedidos que travaram sejam descartados durante a investigação e a RCA.

---

## 4. Estratégia de Testes

- **Qual a nossa estratégia para testar isso e garantir que funciona?** Antes da próxima Black Friday, como você prova que os painéis e os alertas mostrariam este incidente?
  > A estratégia de validação será realizada com testes de carga utilizando k6, simulando o volume esperado da Black Friday e cenários de falha controlados, como lentidão no Pagamento, saturação de pools de conexão e timeouts. Os testes deverão comprovar que os dashboards RED identificam degradação através dos percentis P95 e P99, que os dashboards USE detectam saturação antes do aparecimento de HTTP 500, que os alertas são acionados automaticamente e que os traces problemáticos são preservados pelo Tail Sampling. A observabilidade só será considerada pronta quando for possível reproduzir o incidente e identificar rapidamente a causa através de logs correlacionados, métricas e tracing distribuído
