# ADR 0003 — Modelo de Comunicação entre Microsserviços

| Campo | Valor |
|---|---|
| **Data** | 2025-06 |
| **Status** | Aceito |
| **Decisores** | Equipe de Arquitetura |
| **Contexto do Ciclo** | Fase 4 — Migração para Cloud & Microsserviços |

---

## Contexto

Em uma arquitetura de microsserviços, a comunicação entre serviços pode seguir dois paradigmas principais: **síncrono** (requisição-resposta imediata) ou **assíncrono** (mensagens e eventos). A escolha afeta diretamente o acoplamento entre serviços, a resiliência do sistema e a complexidade operacional.

No ArquitetorERP, diferentes interações entre serviços possuem naturezas distintas:

- Um cliente fazendo login precisa de resposta imediata do `AuthService`.
- O registro de um pagamento no `FinanceiroService` deve disparar uma atualização no `EstoqueService` — mas o usuário não precisa esperar por isso.

A equipe precisou definir quando usar cada modelo e como estruturar a comunicação híbrida.

---

## Decisão

**Adotamos um modelo de comunicação híbrido: síncrono via REST/HTTP para operações que exigem resposta imediata, e assíncrono via Message Broker (RabbitMQ ou Redis Pub-Sub) para eventos de domínio.**

### Regra de decisão aplicada:

| Tipo de Interação | Modelo | Justificativa |
|---|---|---|
| Autenticação / autorização | Síncrono (REST) | Requer resposta imediata para continuar o fluxo |
| Consulta de dados (GET) | Síncrono (REST) | Resultado necessário antes de retornar ao usuário |
| Criação/atualização de entidades | Síncrono (REST) | Confirmação imediata de sucesso/falha é necessária |
| Eventos de domínio (pagamento realizado, pedido criado) | Assíncrono (Broker) | Outros serviços reagem ao evento sem bloquear o fluxo principal |
| Notificações (e-mail, relatórios) | Assíncrono (Broker) | Baixa urgência; não deve bloquear a transação |

---

## Justificativa Teórica

### Comunicação Síncrona

Na comunicação síncrona, o serviço chamador **aguarda a resposta** antes de prosseguir. **Richardson (2018)** descreve REST sobre HTTP como o mecanismo síncrono mais comum em microsserviços, por sua simplicidade, adoção universal e facilidade de depuração.

**Vantagens:** modelo mental simples; facilidade de rastreamento de erros; resultado imediato.  
**Desvantagens:** acoplamento temporal — se o serviço chamado estiver indisponível, o chamador falha; latência se acumula em chamadas em cadeia (*latency amplification*).

**Bass, Clements e Kazman (2021)** alertam que cadeias longas de chamadas síncronas em microsserviços criam **dependências de disponibilidade** que reduzem a resiliência geral do sistema.

### Comunicação Assíncrona

Na comunicação assíncrona, o serviço **publica um evento** em um broker de mensagens e **não aguarda** a resposta de consumidores. **Hohpe e Woolf (2003)** em *Enterprise Integration Patterns* descrevem esse modelo como a base para arquiteturas orientadas a eventos (*event-driven architecture*).

**Vantagens:** desacoplamento temporal; o publicador não precisa conhecer os consumidores; maior resiliência (o broker retém mensagens se o consumidor estiver indisponível).  
**Desvantagens:** complexidade operacional maior (broker como componente adicional); consistência eventual — os dados entre serviços não estão sincronizados instantaneamente; depuração mais difícil.

**Newman (2021)** recomenda o uso de mensageria assíncrona para **eventos de domínio** — mudanças de estado relevantes para múltiplos serviços — e REST síncrono para **consultas e comandos** que exigem confirmação imediata.

### Por que Híbrido?

**Fowler (2015)** descreve o conceito de *choreography* vs. *orchestration* em microsserviços. A abordagem híbrida equilibra os dois: operações críticas ao usuário usam REST (baixa latência, resposta garantida), enquanto efeitos colaterais de domínio usam eventos (desacoplamento, resiliência).

Forçar comunicação assíncrona em todas as interações aumentaria a complexidade sem benefício real para operações que naturalmente exigem resposta imediata.

---

## Alternativas Consideradas

### Alternativa 1: REST Síncrono para Tudo

- **Prós:** uniformidade; fácil de depurar; sem broker para operar.
- **Contras:** acoplamento temporal entre todos os serviços; falha em um serviço bloqueia o fluxo inteiro; não escala bem para eventos de domínio com múltiplos consumidores.
- **Por que foi rejeitada:** viola o princípio de desacoplamento dos microsserviços para eventos que não precisam de resposta imediata.

### Alternativa 2: Mensageria Assíncrona para Tudo (Event-Driven puro)

- **Prós:** máximo desacoplamento; alta resiliência.
- **Contras:** o modelo de programação reativa é mais complexo para toda a equipe; padrão request-reply assíncrono (para simular síncrono) é muito mais complexo que REST; depuração e rastreamento de erros tornam-se difíceis.
- **Por que foi rejeitada:** a complexidade operacional supera os benefícios para um sistema ERP cujas operações principais requerem confirmação imediata.

### Alternativa 3: gRPC para comunicação síncrona

- **Prós:** mais eficiente que REST (Protocol Buffers, HTTP/2); tipagem forte via Protobuf.
- **Contras:** maior complexidade de configuração; menor suporte em ferramentas de debug padrão; overhead para equipe não familiarizada.
- **Por que foi rejeitada:** o ganho de performance não justifica o custo de aprendizado no contexto atual; REST é suficiente para a carga esperada.

---

## Trade-offs da Decisão

| Aspecto | Benefício | Custo/Risco |
|---|---|---|
| **REST Síncrono** | Simplicidade; resposta imediata; fácil debug | Acoplamento temporal; latência acumulada em cadeias |
| **Broker Assíncrono** | Desacoplamento; resiliência; escala de consumidores | Consistência eventual; broker como ponto adicional de falha |
| **Abordagem Híbrida** | Melhor dos dois mundos por caso de uso | Dois paradigmas para a equipe dominar; necessidade de documentação clara das fronteiras |

---

## Consequências

- A equipe deve documentar quais eventos são publicados por cada serviço (contrato de eventos).
- O broker (RabbitMQ ou Redis) deve ser incluído no `docker-compose.yml` para ambiente local.
- Deve-se implementar **dead-letter queue** para eventos que falham no processamento.
- Logs e rastreamento distribuído (ex.: correlation ID) são necessários para depurar fluxos assíncronos.

---

## Referências

- RICHARDSON, C. *Microservices Patterns: With Examples in Java*. Manning Publications, 2018.
- NEWMAN, S. *Building Microservices: Designing Fine-Grained Systems*. 2. ed. O'Reilly Media, 2021.
- HOHPE, G.; WOOLF, B. *Enterprise Integration Patterns: Designing, Building, and Deploying Messaging Solutions*. Addison-Wesley, 2003.
- BASS, L.; CLEMENTS, P.; KAZMAN, R. *Software Architecture in Practice*. 4. ed. Addison-Wesley, 2021.
- FOWLER, M. *What do you mean by "Event-Driven"?*. martinfowler.com, 2017. Disponível em: https://martinfowler.com/articles/201701-event-driven.html
