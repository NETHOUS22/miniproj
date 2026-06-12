# ADR 0002 — Padrões de Resiliência

| Campo | Valor |
|---|---|
| **Data** | 2025-06 |
| **Status** | Aceito |
| **Decisores** | Equipe de Arquitetura |
| **Contexto do Ciclo** | Fase 4 — Migração para Cloud & Microsserviços |

---

## Contexto

Em uma arquitetura de microsserviços, falhas parciais são inevitáveis. Um serviço pode ficar lento, indisponível ou retornar erros sob alta carga. Sem mecanismos de resiliência, uma falha em um único serviço pode se propagar em cascata, derrubando o sistema inteiro — fenômeno conhecido como *cascading failure*.

O ArquitetorERP possui múltiplos serviços interdependentes (Auth, Clientes, Financeiro, Estoque) comunicando-se por meio do API Gateway. A equipe precisou definir quais padrões de resiliência seriam adotados para garantir que o sistema continue operacional mesmo em condições de falha parcial.

---

## Decisão

**Adotamos API Gateway com Circuit Breaker e Bulkhead como padrões primários de resiliência.**

- O **API Gateway** (Kong ou Express Gateway) centraliza a entrada, aplicando rate limiting, autenticação e roteamento.
- O **Circuit Breaker** é implementado na comunicação entre o Gateway e os serviços de backend.
- O **Bulkhead** é aplicado via isolamento de thread pools / connection pools por serviço.

---

## Justificativa Teórica

### API Gateway

**Richardson (2018)** em *Microservices Patterns* define o API Gateway como o **ponto único de entrada** para clientes externos. Além de simplificar a interface do cliente (que não precisa conhecer os endereços internos de cada serviço), o Gateway centraliza preocupações transversais: autenticação, autorização, rate limiting, logging e roteamento.

Para o ArquitetorERP, o Gateway também atua como primeira linha de defesa contra sobrecarga, descartando requisições excessivas antes que cheguem aos serviços internos.

### Circuit Breaker

O padrão **Circuit Breaker**, popularizado por **Nygard (2007)** em *Release It!* e detalhado por **Fowler (2014)**, funciona como um "disjuntor" para chamadas remotas:

- **Estado Fechado (normal):** requisições passam normalmente; falhas são contadas.
- **Estado Aberto (falha detectada):** após atingir um limite de falhas, o circuito "abre" e as requisições falham imediatamente sem tentar chamar o serviço — evitando que chamadas lentas ocupem threads.
- **Estado Semi-aberto (recuperação):** após um tempo, algumas requisições de teste passam; se bem-sucedidas, o circuito fecha novamente.

**Richardson (2018)** argumenta que o Circuit Breaker é essencial em microsserviços porque impede que a lentidão de um serviço se propague para seus consumidores, preservando os recursos do sistema.

### Bulkhead

O padrão **Bulkhead** (anteparo de navio) isola recursos para que a falha em um contexto não consuma recursos de outro. **Nygard (2007)** descreve o padrão como a divisão de connection pools, thread pools ou semáforos por serviço consumidor.

No ArquitetorERP, o Bulkhead garante que, por exemplo, uma sobrecarga no `FinanceiroService` não esgote as conexões disponíveis para o `ClienteService` — cada serviço opera dentro de sua própria "comporta".

---

## Alternativas Consideradas

### Alternativa 1: Retry com Backoff Exponencial (sem Circuit Breaker)

- **Prós:** simples de implementar; tenta recuperação automática.
- **Contras:** em caso de falha prolongada, retries repetidos aumentam a carga sobre o serviço degradado (*thundering herd problem*); sem o Circuit Breaker, o sistema continua tentando enquanto o serviço está caído, desperdiçando recursos.
- **Por que foi rejeitada:** Retry é complementar ao Circuit Breaker, não substituto. Sem o disjuntor, o retry pode piorar a situação.

### Alternativa 2: Timeout simples

- **Prós:** evita que chamadas bloqueiem indefinidamente.
- **Contras:** timeouts sozinhos não impedem que chamadas falhas continuem sendo feitas; não há estado acumulado sobre a saúde do serviço.
- **Por que foi rejeitada:** insuficiente como mecanismo de resiliência isolado; será usado em conjunto com o Circuit Breaker, não como alternativa.

### Alternativa 3: Service Mesh (Istio / Linkerd)

- **Prós:** resiliência gerenciada na camada de infraestrutura; sem alteração de código; Circuit Breaker e retry configuráveis via YAML.
- **Contras:** alta complexidade operacional; curva de aprendizado elevada; overhead para um projeto em estágio inicial.
- **Por que foi rejeitada:** o custo de adoção de um Service Mesh supera os benefícios no contexto atual do projeto. É uma evolução natural para fases futuras.

---

## Trade-offs da Decisão

| Aspecto | Benefício | Custo/Risco |
|---|---|---|
| **API Gateway** | Centralização; segurança; observabilidade | Ponto único de falha se não for redundante |
| **Circuit Breaker** | Evita cascading failure; falha rápida | Lógica adicional no código; necessidade de monitoramento do estado do circuito |
| **Bulkhead** | Isolamento de falhas entre domínios | Aumenta o número de recursos provisionados (mais pools) |

---

## Consequências

- O API Gateway deve ter alta disponibilidade (pelo menos 2 instâncias).
- A implementação do Circuit Breaker pode usar bibliotecas como **opossum** (Node.js) ou ser delegada ao Gateway.
- Dashboards de monitoramento devem exibir o estado dos circuitos (aberto/fechado).
- Testes de caos (*chaos engineering*) são recomendados para validar o comportamento em falha.

---

## Referências

- NYGARD, M. T. *Release It!: Design and Deploy Production-Ready Software*. 2. ed. Pragmatic Bookshelf, 2018.
- FOWLER, M. *CircuitBreaker*. martinfowler.com, 2014. Disponível em: https://martinfowler.com/bliki/CircuitBreaker.html
- RICHARDSON, C. *Microservices Patterns: With Examples in Java*. Manning Publications, 2018.
- BASS, L.; CLEMENTS, P.; KAZMAN, R. *Software Architecture in Practice*. 4. ed. Addison-Wesley, 2021.
