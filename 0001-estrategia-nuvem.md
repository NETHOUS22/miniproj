# ADR 0001 — Estratégia de Nuvem e Escalabilidade

| Campo | Valor |
|---|---|
| **Data** | 2025-06 |
| **Status** | Aceito |
| **Decisores** | Equipe de Arquitetura |
| **Contexto do Ciclo** | Fase 4 — Migração para Cloud & Microsserviços |

---

## Contexto

O ArquitetorERP foi desenvolvido nas fases anteriores como um monólito Node.js. Com o crescimento esperado da base de usuários e a necessidade de maior disponibilidade e elasticidade, a equipe precisou definir uma estratégia de implantação em nuvem que suportasse a nova arquitetura de microsserviços.

A decisão envolveu escolher entre os modelos de serviço disponíveis (IaaS, PaaS, SaaS, Serverless) e definir a estratégia de escalabilidade (horizontal vs. vertical).

---

## Decisão

**Adotamos PaaS (Platform as a Service) como modelo principal de implantação, com escalabilidade horizontal para os microsserviços de negócio.**

Os serviços serão implantados em contêineres Docker orquestrados via plataforma de nuvem (ex.: Google Cloud Run, AWS Elastic Beanstalk ou Railway), com banco de dados gerenciado (DBaaS) e escalabilidade automática baseada em carga.

---

## Justificativa Teórica

### Modelos de Serviço em Nuvem

Segundo **Mell e Grance (2011)**, do NIST, os modelos de serviço em nuvem se dividem em:

- **IaaS (Infrastructure as a Service):** fornece recursos computacionais brutos (VMs, armazenamento, redes). Oferece alto controle, porém exige gerenciamento de SO, patches e infraestrutura — custo operacional elevado para equipes pequenas.
- **PaaS (Platform as a Service):** fornece uma plataforma gerenciada onde o desenvolvedor entrega apenas o código/contêiner. O provedor gerencia SO, runtime, escalonamento e patches.
- **SaaS (Software as a Service):** software pronto consumido via API ou interface. Aplicável para serviços auxiliares (e-mail, monitoramento), não para o core do sistema.
- **Serverless (FaaS):** execução orientada a eventos sem gerenciamento de servidor. Adequado para workloads eventuais e muito variáveis.

**Bass, Clements e Kazman (2021)** em *Software Architecture in Practice* destacam que a escolha do modelo de implantação deve considerar o **perfil de carga** do sistema. Para sistemas ERP com uso contínuo ao longo do dia útil, PaaS com contêineres oferece melhor custo-benefício que Serverless (que é otimizado para execuções eventuais e cobra por invocação).

### Escalabilidade Horizontal vs. Vertical

**Escalabilidade vertical** consiste em aumentar os recursos de uma única instância (mais CPU, mais RAM). É simples de implementar, mas possui **limite físico** e gera **ponto único de falha**.

**Escalabilidade horizontal** consiste em adicionar mais instâncias do serviço distribuindo a carga. **Newman (2021)** em *Building Microservices* argumenta que microsserviços são naturalmente projetados para escalar horizontalmente — cada serviço pode ser escalado de forma independente conforme sua demanda específica, sem necessidade de escalar o sistema inteiro.

Para o ArquitetorERP, o `FinanceiroService` e o `ClienteService` tendem a ter picos de carga distintos (fechamento de mês vs. horário comercial). A escalabilidade horizontal permite que cada um responda à sua própria demanda, otimizando custos.

---

## Alternativas Consideradas

### Alternativa 1: IaaS puro (VMs dedicadas)

- **Prós:** controle total da infraestrutura, sem vendor lock-in de plataforma.
- **Contras:** exige gerenciamento de SO, segurança e patches pela equipe; overhead operacional alto para um projeto acadêmico/early-stage; escalabilidade manual.
- **Por que foi rejeitada:** o custo operacional é desproporcional ao tamanho da equipe e ao estágio do projeto.

### Alternativa 2: Serverless / FaaS puro

- **Prós:** custo zero quando ocioso, escala automática sem configuração.
- **Contras:** **cold start** pode comprometer a latência de um ERP usado continuamente; estado difícil de gerenciar entre funções; conexões com banco de dados são problemáticas em ambientes serverless (limites de conexão).
- **Por que foi rejeitada:** o perfil de uso contínuo do ERP torna o cold start um problema real; além disso, a complexidade de gerenciar estado entre funções aumenta a carga cognitiva da equipe (FOWLER, 2018).

### Alternativa 3: Escalabilidade vertical

- **Prós:** simples de implementar, sem necessidade de código distribuído.
- **Contras:** limite físico da máquina; ponto único de falha; não aproveita a independência dos microsserviços.
- **Por que foi rejeitada:** contradiz os princípios de microsserviços e não oferece resiliência.

---

## Trade-offs da Decisão

| Aspecto | Benefício | Custo/Risco |
|---|---|---|
| **PaaS** | Menor overhead operacional; deploy simplificado | Menor controle da infraestrutura; possível vendor lock-in |
| **Escalabilidade Horizontal** | Alta disponibilidade; escala independente por serviço | Complexidade de roteamento; necessidade de serviços stateless |
| **Contêineres (Docker)** | Portabilidade entre provedores; ambiente consistente | Necessidade de conhecimento de Docker/orquestração |

---

## Consequências

- Todos os microsserviços devem ser **stateless** (sem estado em memória entre requisições).
- Sessões e cache devem ser externalizados (Redis).
- O pipeline de CI/CD deve gerar imagens Docker e fazer push para um registry.
- A equipe deve definir `Dockerfile` e variáveis de ambiente para cada serviço.

---

## Referências

- MELL, P.; GRANCE, T. *The NIST Definition of Cloud Computing*. NIST Special Publication 800-145, 2011.
- BASS, L.; CLEMENTS, P.; KAZMAN, R. *Software Architecture in Practice*. 4. ed. Addison-Wesley, 2021.
- NEWMAN, S. *Building Microservices: Designing Fine-Grained Systems*. 2. ed. O'Reilly Media, 2021.
- FOWLER, M. *Serverless Architectures*. martinfowler.com, 2018. Disponível em: https://martinfowler.com/articles/serverless.html
