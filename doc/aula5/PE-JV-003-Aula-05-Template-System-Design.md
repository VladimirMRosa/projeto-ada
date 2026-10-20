# Template de System Design: Aula 05 - Containers, CI/CD & Deploy sem Downtime

> **Contexto do Cenário: O Deploy que Quebrou o Checkout e o Rollback Impossível**
>
> Sua equipe está reestruturando a engenharia de release do serviço de **Checkout** após um incidente severo na Black Friday:
>
> 1. **O Incidente:** Para subir uma otimização urgente na Saga, o pipeline de produção recompilou a branch `main`, sobrescreveu a tag `checkout:latest` e iniciou a atualização. No boot da primeira instância nova, o **Flyway** renomeou a coluna `status_pagamento` para `estado_saga`.
> 2. **O Efeito em Cadeia:** Os 80% de servidores que ainda rodavam a versão anterior passaram a falhar imediatamente (`column "status_pagamento" does not exist`). Ao tentar acionar o rollback puxando `:latest`, a equipe descobriu que a imagem anterior havia sido sobrescrita e, mesmo restaurando um backup do binário antigo, ele continuou falhando porque o banco já estava alterado.
> 3. **A Carga:** 5.000 TPS ativos com múltiplas réplicas de aplicação conectadas ao mesmo banco PostgreSQL.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Empacotamento, Superfície de Ataque e Promoção Imutável

### 1.1 Anatomia da Imagem e Redução de Superfície de Ataque

- **Construção vs Execução:** Como você estruturará o empacotamento do serviço de Checkout para produção? Especifique o que deve ser usado na etapa de compilação, o que deve (e o que não deve) entrar na imagem final de execução, com qual permissão de usuário o processo rodará e qual o trade-off da sua escolha de imagem base para a segurança e para o diagnóstico em produção.
  >  Para produção, escolheria uma imagem Distroless Java ou equivalente minimalista.

### 1.2 Rastreabilidade, Identidade do Artefato e Parametrização

- **Da Homologação à Produção:** Como sua esteira garante que o binário executado em produção é exatamente o mesmo que passou nos testes de homologação? Explique como as configurações mudam entre ambientes (dev, staging, prod) sem recompilar o artefato e como você identifica a versão no registry para que um rollback nunca puxe um binário sobrescrito por engano.
  > Utilizar o padrão Build Once, Deploy Many, configurações externas via variáveis/segredos e versionamento imutável baseado em tags de release e digest SHA256, garantindo rastreabilidade completa e rollback seguro sem dependência da tag latest.

---

## 2. Estratégia de Rollout e Migração de Banco sem Downtime

### 2.1 O Modelo de Rollout em Produção

- **A Escolha:** (X) Canary Release (Liberação Gradual por Percentual)  ( ) Blue-Green Deployment (Alternância de Ambientes)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Canary Release:* Como o tráfego evoluirá sob 5.000 TPS, quais sinais definirão o avanço ou a reversão (rollback) automática e como lidar com o tempo em que duas versões diferentes da API atendem clientes simultaneamente?
  - *Se você escolheu Blue-Green Deployment:* Como você gerenciará o custo de infraestrutura duplicada no pico da Black Friday, o aquecimento das novas instâncias (conexões/cache) antes da virada de chave e o risco de virar 100% do tráfego de uma só vez?
  > Canary Release porque oferece menor risco operacional para um sistema de Checkout com 5.000 TPS. O modelo permite validar a nova versão com tráfego real, identificar regressões rapidamente e executar rollback imediato sem impactar toda a operação nem depender da recuperação de imagens ou da reversão da base de dados.

### 2.2 Evolução de Schema Sem Downtime e Plano de Rollback

- **Passo a Passo da Migração:** Descreva em etapas sequenciais como evoluir a coluna `status_pagamento` para `estado_saga` no PostgreSQL garantindo que as instâncias da versão antiga (`v1`) e da versão nova (`v2`) operem juntas sem erros de SQL.
  > Para permitir que a versão antiga (v1) e a nova (v2) funcionem simultaneamente durante o Canary Release, não devemos renomear diretamente a coluna status_pagamento, pois isso quebraria imediatamente a v1. Em vez disso, utilizamos o padrão Expand and Contract. 
  > 1 - Seguiria a ordem de expandir o shcema, adicionando a nova coluna sem remover a linha antiga. 
  > 2 - Migrar os dados existentes, copiando os valores da coluna antiga para a nova. 
  > 3 - Adaptar a versão v2 e essa deve ser compatível com as duas colunas. 
  > 4 - Implementar Double Write, a v2 deve escrever simultaneamente nas duas colunas 
  > 5 - Iniciar o Canary Release
  > 6 - Concluir a migração
- **Garantia de Rollback Imediato:** Caso a versão `v2` apresente degradação grave de performance com 20% do tráfego na Black Friday, qual é o procedimento exato de rollback e por que o seu desenho de banco de dados garante que a versão `v1` voltará a assumir 100% da carga sem quebrar nem perder dados gravados pela `v2`?
  > O rollback é imediato porque basta retirar a v2 da rota de tráfego. Como a coluna status_pagamento permanece disponível e a v2 grava dados tanto na estrutura antiga quanto na nova, a v1 consegue assumir novamente 100% dos 5.000 TPS, sem erros de SQL, sem perda de dados e sem qualquer intervenção no banco de dados.
   

---

## 3. Guardrails de Segurança, Governança e Validação da Release

### 3.1 Gates de Segurança e Guardrails no Processo de Entrega

- **Guardrails do Commit ao Deploy:** Quais portões automatizados de qualidade e segurança devem proteger a sua esteira ao longo do fluxo de desenvolvimento (desde o Pull Request até a publicação e promoção da imagem)?
  > Para evitar problemas como o incidente da Black Friday, a esteira de CI/CD deve possuir gates automáticos de qualidade, segurança e validação em todas as etapas, desde o Pull Request até o deploy em produção.
- **Governança e Controle em Produção:** Quais *guardrails* de processo e arquitetura impedem que alterações manuais não rastreadas cheguem à produção e como equilibrar segurança rigorosa com agilidade de resposta durante incidentes críticos sem perder a trilha de auditoria?
  > Para impedir alterações manuais não rastreadas, adotaria o modelo GitOps, onde toda mudança de infraestrutura, configuração ou aplicação deve ser realizada através de código versionado.

### 3.2 Estratégia de Testes da Imagem e da Migração

- **Validação Pré-Tráfego:** Passar nos testes unitários durante o build do código não prova que a imagem final do container sobe nem que a migração de banco é retrocompatível.
- **Plano de Testes na Pipeline:** Descreva como você testará automaticamente a imagem construída e validará a coexistência da migração de banco de dados na esteira antes de liberar o deploy para o tráfego real de clientes.
  > Após a construção da imagem Docker, a pipeline Jenkins executará smoke tests para validar a inicialização do container. Em seguida, criará um ambiente efémero com PostgreSQL, aplicará as migrações Flyway e executará testes de compatibilidade entre as versões v1 e v2 da aplicação. Serão validados cenários de leitura, escrita, double-write e rollback para garantir que ambas as versões coexistem sem erros SQL. Apenas após a aprovação desses testes a imagem será promovida para staging e posteriormente liberada por meio de Canary Release em produção.
