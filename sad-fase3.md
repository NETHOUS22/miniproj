# SAD — Software Architecture Document
## ArquitetorERP · Fase 4: Cloud & Microsserviços

| Campo | Valor |
|---|---|
| **Versão** | 1.0 |
| **Data** | Junho 2025 |
| **Status** | Aprovado |
| **Autores** | Equipe de Arquitetura |

---

## 1. Introdução

### 1.1 Propósito

Este documento descreve a arquitetura de software do **ArquitetorERP** na sua Fase 4, que representa a evolução da aplicação de um monólito Node.js para uma arquitetura de microsserviços implantada em nuvem. O SAD registra as decisões estruturais, os componentes do sistema, seus relacionamentos e os atributos de qualidade priorizados.

### 1.2 Escopo

O documento abrange a arquitetura da camada de backend (microsserviços), infraestrutura de nuvem, mecanismos de comunicação e padrões de resiliência adotados na Fase 4. O frontend Web (React) é descrito em nível de container, sem detalhamento interno.

### 1.3 Definições e Siglas

| Termo | Definição |
|---|---|
| ADR | Architecture Decision Record — registro de decisão arquitetural |
| API Gateway | Ponto único de entrada para requisições externas |
| Circuit Breaker | Padrão que interrompe chamadas a serviços degradados |
| Bulkhead | Padrão de isolamento de recursos por serviço |
| Broker | Servidor de mensagens para comunicação assíncrona |
| PaaS | Platform as a Service — modelo de serviço em nuvem |
| ERP | Enterprise Resource Planning — sistema de gestão empresarial |
| CRM | Customer Relationship Management — gestão de clientes |

---

## 2. Representação Arquitetural

A arquitetura do ArquitetorERP na Fase 4 é descrita a partir de três visões:

1. **Visão de Contexto (C4 Nível 1):** sistema e seus atores externos
2. **Visão de Containers (C4 Nível 2):** serviços, bancos de dados e tecnologias
3. **Visão de Implantação:** infraestrutura de nuvem

---

## 3. Objetivos e Restrições Arquiteturais

### 3.1 Objetivos

| Atributo de Qualidade | Meta |
|---|---|
| **Disponibilidade** | ≥ 99,5% em horário comercial |
| **Escalabilidade** | Suportar aumento de 10× na carga sem redesenho |
| **Manutenibilidade** | Serviços independentes; deploy sem downtime total |
| **Resiliência** | Falha em um serviço não derruba os demais |
| **Segurança** | Autenticação centralizada; comunicação interna protegida |

### 3.2 Restrições

- Linguagem principal: **Node.js / JavaScript**
- Contêineres: **Docker** (portabilidade entre provedores)
- Sem vendor lock-in exclusivo; preferência por soluções portáteis
- Banco de dados relacional (PostgreSQL) por domínio

---

## 4. Visão de Contexto (C4 — Nível 1)

O ArquitetorERP interage com os seguintes atores e sistemas externos:

| Ator/Sistema | Tipo | Interação |
|---|---|---|
| **Usuário (Gestor/Operador)** | Pessoa | Acessa o sistema via browser |
| **Serviço de E-mail (SendGrid)** | Sistema externo | Recebe notificações disparadas pelo Broker |
| **Provedor de Nuvem (PaaS)** | Infraestrutura | Hospeda e escala os contêineres |

---

## 5. Visão de Containers (C4 — Nível 2)

### 5.1 Componentes Principais

| Container | Tecnologia | Responsabilidade |
|---|---|---|
| **Frontend Web** | React / Node.js | Interface SPA; consome o API Gateway |
| **API Gateway** | Express Gateway / Kong | Roteamento, autenticação, rate limiting |
| **Auth Service** | Node.js + JWT | Login, logout, geração e validação de tokens |
| **Cliente Service** | Node.js + Express | CRUD de clientes, histórico de interações (CRM) |
| **Financeiro Service** | Node.js + Express | Contas a pagar/receber, lançamentos, relatórios |
| **Estoque Service** | Node.js + Express | Produtos, entradas e saídas de estoque |
| **Message Broker** | RabbitMQ / Redis Pub-Sub | Mensageria assíncrona para eventos de domínio |
| **DB Clientes** | PostgreSQL | Persistência do domínio de clientes |
| **DB Financeiro** | PostgreSQL | Persistência do domínio financeiro |
| **DB Estoque** | PostgreSQL | Persistência do domínio de estoque |
| **DB Auth** | PostgreSQL | Usuários, roles e refresh tokens |

### 5.2 Princípio de Banco de Dados por Serviço

Cada microsserviço possui seu próprio banco de dados isolado. **Newman (2021)** denomina este padrão *Database per Service* e o considera essencial para garantir o desacoplamento real entre serviços — um serviço não pode acessar diretamente o banco de outro, apenas via API ou evento.

---

## 6. Decisões Arquiteturais Relevantes

As decisões estruturais desta fase estão registradas nos ADRs:

| ADR | Tema | Link |
|---|---|---|
| ADR 0001 | Estratégia de Nuvem e Escalabilidade (PaaS + Escalabilidade Horizontal) | [0001-estrategia-nuvem.md](../adrs/0001-estrategia-nuvem.md) |
| ADR 0002 | Padrões de Resiliência (API Gateway + Circuit Breaker + Bulkhead) | [0002-padrao-resiliencia.md](../adrs/0002-padrao-resiliencia.md) |
| ADR 0003 | Modelo de Comunicação Híbrido (REST + Mensageria Assíncrona) | [0003-modelo-comunicacao.md](../adrs/0003-modelo-comunicacao.md) |

---

## 7. Visão de Implantação

### 7.1 Ambiente de Nuvem

O sistema é implantado em **PaaS** com contêineres Docker. O modelo de implantação segue os princípios da **12-Factor App** (Wiggins, 2011):

- Configuração via variáveis de ambiente (sem hardcode)
- Logs como streams de saída padrão
- Processos stateless e descartáveis

### 7.2 Infraestrutura por Serviço

```
┌─────────────────────────────────────────────────────┐
│                   Provedor PaaS (Cloud)              │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ Auth Svc │  │ CRM Svc  │  │ Financeiro Svc   │   │
│  │ (1-2 inst│  │ (1-3 inst│  │ (1-3 instâncias) │   │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
│       │             │                 │              │
│  ┌────┴─────────────┴─────────────────┴─────────┐   │
│  │              API Gateway                      │   │
│  └────────────────────────────────────────────── ┘   │
│                                                      │
│  ┌──────────────────┐   ┌──────────────────────┐     │
│  │  Message Broker  │   │  Managed PostgreSQL  │     │
│  │   (RabbitMQ)     │   │  (DBaaS por serviço) │     │
│  └──────────────────┘   └──────────────────────┘     │
└─────────────────────────────────────────────────────┘
```

### 7.3 Pipeline de CI/CD

```
Push para branch → GitHub Actions → Lint + Testes → Build Docker Image → Push Registry → Deploy PaaS
```

---

## 8. Atributos de Qualidade — Cenários

### 8.1 Disponibilidade

**Cenário:** O `FinanceiroService` fica indisponível por 5 minutos.  
**Resposta esperada:** O Circuit Breaker abre; requisições ao Financeiro falham rapidamente com mensagem amigável; os demais serviços (Clientes, Estoque, Auth) continuam operando normalmente.  
**Tática:** Circuit Breaker + Bulkhead (ADR 0002).

### 8.2 Escalabilidade

**Cenário:** No fechamento do mês, o número de requisições ao `FinanceiroService` aumenta 8×.  
**Resposta esperada:** O PaaS escala horizontalmente o `FinanceiroService` de 1 para 4 instâncias sem afetar os demais serviços.  
**Tática:** Escalabilidade horizontal independente por serviço (ADR 0001).

### 8.3 Consistência de Dados

**Cenário:** Um pagamento é registrado no `FinanceiroService` e deve atualizar o saldo devedor do cliente no `ClienteService`.  
**Resposta esperada:** O `FinanceiroService` publica um evento `pagamento.realizado` no broker; o `ClienteService` consome o evento e atualiza o saldo de forma assíncrona (consistência eventual).  
**Tática:** Comunicação assíncrona via broker (ADR 0003).

---

## 9. Riscos e Débitos Técnicos

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Broker como ponto único de falha | Média | Alto | Configurar broker em modo cluster |
| Consistência eventual gera estado inconsistente temporário | Alta | Médio | Idempotência nos consumidores; dead-letter queue |
| Vendor lock-in no PaaS | Baixa | Médio | Uso de Docker garante portabilidade |
| Overhead de operação de múltiplos bancos | Alta | Baixo | DBaaS gerenciado pelo provedor de nuvem |

---

## 10. Referências

- BASS, L.; CLEMENTS, P.; KAZMAN, R. *Software Architecture in Practice*. 4. ed. Addison-Wesley, 2021.
- NEWMAN, S. *Building Microservices*. 2. ed. O'Reilly Media, 2021.
- RICHARDSON, C. *Microservices Patterns*. Manning Publications, 2018.
- BROWN, S. *The C4 model for visualising software architecture*. Disponível em: https://c4model.com
- WIGGINS, A. *The Twelve-Factor App*. 2011. Disponível em: https://12factor.net
- HOHPE, G.; WOOLF, B. *Enterprise Integration Patterns*. Addison-Wesley, 2003.
