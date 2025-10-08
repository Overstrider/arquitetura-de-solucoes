# Architecture Decision Records (ADR)

## Contexto inicial
1. Recebi do stakeholder um briefing solicitando uma solução capaz de controlar lançamentos diários (débitos e créditos) e gerar um relatório consolidado às 18h.
2. Durante a reunião de discovery, ouvi que a NAOVON opera porta a porta, pretende suportar 100.000 vendedoras simultâneas, exige operação 24/7 e precisa de verificação ativa de fraudes.
3. As vendedoras utilizam smartphones próprios (Android e iOS); portanto, projetei experiências offline-first com sincronização posterior.
4. Também defini que o aplicativo deve oferecer autenticação, catálogo offline, registro de vendas e despesas, além de sincronização automática quando houver conectividade.
5. Estabeleci requisitos não funcionais: latência média < 100 ms, disponibilidade ≥ 99,9%, auditoria completa, tráfego seguro fim a fim e orçamento mensal controlado.
6. Mapeei os atores principais: gestora central (dashboards e alertas), compliance (auditorias e relatórios), suporte (reconciliação) e squads de produto/dados (minha responsabilidade concentrada).
7. Explorei tecnologias mobile (React Native vs. Kotlin Multiplatform) e redigi o ADR-007 com o racional.
8. Estruturei a arquitetura dividindo o domínio em lançamentos (hot path) e módulos de apoio (Admin, Catálogo, Reports, IA), resultando no arranjo híbrido descrito nos ADR-001 e ADR-003.

---

## ADR-001 - Método de planejamento e definição de módulos
- **Status**: Aceito
- **Contexto**: Eu precisava transformar o briefing em domínios claros antes de escolher tecnologias.
- **Decisão**: Conduzi event storming/domain storytelling, documentei uma matriz de capacidades e explicitei fronteiras entre fluxo crítico (Lançamentos) e núcleo administrativo (Admin, Catálogo, Reports, IA, Auth).
- **Consequências**: Consegui visão única do escopo, reduzi ambiguidade e padronizei revisões trimestrais do plano.

## ADR-002 - Stack de processamento crítico (Rust vs. alternativas)
- **Status**: Aceito
- **Contexto**: O serviço de lançamentos precisava sustentar 100.000 TPS com p95 < 80 ms e memória previsível.
- **Decisão**: Comparei stacks tradicionais (Java/.NET/Node/Python) com Rust/Go e escolhi Rust (Axum + Tokio + SQLx/Diesel) pela performance e safety.
- **Consequências**: Reduzi footprint em GKE, assegurei memory safety e passei a investir em onboarding estruturado para a nova stack.

## ADR-003 - Núcleo híbrido (microserviço crítico + monólito modular)
- **Status**: Aceito
- **Contexto**: Apenas o domínio de lançamentos exige isolamento extremo; demais módulos compartilham modelos e ritmo de evolução.
- **Decisão**: Separei Lançamentos em microserviço Rust e mantive Admin/Catálogo/Reports/IA/Auth em um monólito FastAPI modular.
- **Consequências**: Escalo o hot path de forma independente e mantenho um único pipeline de deploy para módulos administrativos.

## ADR-004 - Implementação do microserviço de lançamentos (Axum + Tokio)
- **Status**: Aceito
- **Contexto**: Precisei consolidar padrões (buffer, WAL, tracing, idempotência) após escolher Rust.
- **Decisão**: Implementei Axum/Tokio com filas lock-free, flush em lote a cada 10 s, WAL com RocksDB e observabilidade via OpenTelemetry.
- **Consequências**: Ganhei previsibilidade de latência e estabeleci práticas de readiness/feature flags para mitigar riscos de rollouts stateful.

## ADR-005 - Monólito modular FastAPI (Admin, Catálogo, Reports, IA, Auth)
- **Status**: Aceito
- **Contexto**: Os módulos compartilham modelos e pipelines; microservices adicionais trariam overhead prematuro.
- **Decisão**: Construí um monólito FastAPI com pacotes isolados por domínio, expus apenas Admin via Apigee e orquestrei jobs internos com Argo/Kafka.
- **Consequências**: Simplifiquei deploys, maximizei o reuso de bibliotecas Python e mantive disciplina de limites internos através de testes de contrato.

## ADR-006 - Persistência central em PostgreSQL Cloud SQL
- **Status**: Aceito
- **Contexto**: Necessitava de ACID, auditoria, suporte a OLTP/OLAP leve e integração com a feature store.
- **Decisão**: Adotei Cloud SQL (PostgreSQL) com primário + réplica, particionamento temporal e CDC para pipelines.
- **Consequências**: Garanti consistência forte, centralizei dados estruturados e estabeleci plano de mitigação para throughput futuro (sharding/caching).

## ADR-007 - Apps mobile em Kotlin Multiplatform
- **Status**: Aceito
- **Contexto**: Avaliei React Native vs. KMP considerando offline-first, performance em dispositivos modestos e compartilhamento de lógica.
- **Decisão**: Escolhi Kotlin Multiplatform com SQLite via SQLDelight, Compose/SwiftUI e outbox para sincronização.
- **Consequências**: Reduzi duplicação entre Android/iOS, entreguei experiência offline robusta e passei a monitorar performance per plataforma.

## ADR-008 - Detecção de anomalias com LangChain + Vertex AI (Gemini)
- **Status**: Aceito
- **Contexto**: O negócio demandava alertas contextuais em tempo quase real usando heurísticas + LLM.
- **Decisão**: Orquestrei agentes LangChain, usei Gemini 2.5 (Flash/Pro) via Vertex AI, conectei MCP ao PostgreSQL e normalizei respostas JSON.
- **Consequências**: Ganhei precisão ≥85%, mantive custos previsíveis (~US$55/mês) e criei fallback para modelos open-source.

## ADR-009 - Plataforma do microserviço (GKE vs. Cloud Run)
- **Status**: Aceito
- **Contexto**: O serviço mantém buffers/WAL e demanda afinidade de pod; Cloud Run não preserva estado.
- **Decisão**: Executei o microserviço em GKE (StatefulSet, node pool dedicado, autoscaling baseado em latência/CPU).
- **Consequências**: Evitei cold starts, ganhei controle de autoscaling/canary e aceitei a responsabilidade de manter o cluster.

## ADR-010 - Hospedagem do monólito em Compute Engine MIG
- **Status**: Aceito
- **Contexto**: O monólito precisa de previsibilidade, WebSockets e upgrades controlados sem overhead de Kubernetes.
- **Decisão**: Empacotei o monólito em container único no Managed Instance Group (2 VMs n2-standard-4, ativo/standby).
- **Consequências**: Assegurei failover rápido, evitei cold starts e assumi patches/IAAC para dimensionamento manual.

## ADR-011 - Barramento interno com Kafka embutido
- **Status**: Aceito
- **Contexto**: Catálogo, IA, Push e Reports precisam de integração assíncrona com replay sem manter cluster Kafka dedicado.
- **Decisão**: Rodei Strimzi leve no mesmo GKE, padronizei eventos JSON versionados e retenção de 7 dias.
- **Consequências**: Ganhei desacoplamento com reprocessamento nativo e mantive custos baixos, monitorando retenção para evitar impacto no microserviço.

## ADR-012 - Autenticação e IAM com Keycloak
- **Status**: Aceito
- **Contexto**: Necessitava de SSO, MFA, RBAC granular e evitar lock-in SaaS.
- **Decisão**: Implementei Keycloak (2 pods HA) com banco dedicado, políticas de senha rígidas e MFA para perfis administrativos.
- **Consequências**: Ampliei flexibilidade de integração, reduzi custos licenciados e instituí rotina de patches e governança de identidade.

## ADR-013 - API Gateway com Apigee
- **Status**: Aceito
- **Contexto**: Preciso de borda única para mobile/Admin/parceiros, com quotas, validação JWT e métricas.
- **Decisão**: Configurei Apigee Standard com proxies `/v1/*`, políticas OAuth, rate limiting e injeção de cabeçalhos de identidade.
- **Consequências**: Centralizei segurança/observabilidade das APIs e mantive plano de fallback via Cloud Endpoints.

## ADR-014 - Distribuição offline-first via CDN assinada
- **Status**: Aceito
- **Contexto**: As vendedoras precisam do catálogo completo offline com atualizações garantidas.
- **Decisão**: Gero snapshot noturno (Parquet/JSON), publico em Cloud Storage + Cloud CDN com Signed URLs/cookies e combino com endpoints delta.
- **Consequências**: Reduzi carga do backend, garanti consistência e implementei governança de expiração/hash.

## ADR-015 - Topologia de bancos (Auth, primário, réplica)
- **Status**: Aceito
- **Contexto**: Keycloak demanda isolamento, o primário recebe alta escrita e relatórios/IA precisam de leitura pesada.
- **Decisão**: Provisiono três instâncias Cloud SQL (Auth dedicado, primário write, réplica read) com monitoramento individual.
- **Consequências**: Reduzi contenção, mantive segregação de dados sensíveis e aceitei custo operacional extra + automação de gestão.

## ADR-016 - Admin UI em React + TypeScript
- **Status**: Aceito
- **Contexto**: Preciso entregar dashboards dinâmicos, integrados a Keycloak, com time-to-market curto.
- **Decisão**: Construí o Admin UI com React/TS, Vite, integração `@react-keycloak/web`, code splitting, CSP e sanitização centralizada.
- **Consequências**: Reduzi meu time-to-market, facilito colaboração e mantenho monitoramento rígido de bundle size e segurança.

## ADR-017 - Módulo Reports em Rust + Polars/Arrow
- **Status**: Aceito
- **Contexto**: Os relatórios exigem processamento de milhões de registros em segundos e exportação governada.
- **Decisão**: Escrevi pipelines Rust (Polars/Arrow) orquestrados pelo Argo Workflows, atualizando materialized views e data lake.
- **Consequências**: Alcancei performance-alvo (<1 s/milhão), garanti auditoria e assumi operação dos workflows Argo.

## ADR-018 - Módulo Catálogo de Produtos
- **Status**: Aceito
- **Contexto**: O catálogo é a fonte única de SKUs, versões e políticas comerciais para 100.000 vendedoras.
- **Decisão**: Mantive o módulo no monólito (FastAPI) com audit trail, upload/versionamento Cloud Storage e eventos Kafka.
- **Consequências**: Centralizo governança, habilito sincronização incremental e reforço validações para evitar inconsistências.

## ADR-019 - Resiliência Operacional e SLOs
- **Status**: Aceito
- **Contexto**: Preciso sustentar 99,9% de disponibilidade, p95 < 100 ms e recuperação rápida.
- **Decisão**: Estabeleci SLOs (p95, disponibilidade, MTTR), orquestro chaos engineering trimestral, mantenho runbooks e post-mortems obrigatórios.
- **Consequências**: Defini expectativas claras, fortaleci postura resiliente e investi continuamente em observabilidade e automação.

## ADR-020 - Segurança em Camadas
- **Status**: Aceito
- **Contexto**: O ecossistema mistura apps móveis, Admin, IA e integrações externas; adotei abordagem Zero Trust.
- **Decisão**: Implantei defesa em profundidade (Cloud Armor → Apigee → TLS/mTLS → Keycloak → RBAC → Secret Manager/KMS → SIEM) e acesso interno via IAP + MFA.
- **Consequências**: Reduzi superfície de ataque, alinhei compliance (LGPD) e assumi monitoramento constante de segredos/alertas.

## ADR-021 - Observabilidade e Gestão de Incidentes
- **Status**: Aceito
- **Contexto**: Preciso de visibilidade unificada (latência, volume, erros) e alertas acionáveis.
- **Decisão**: Padronizei OpenTelemetry, exportei métricas/logs para Cloud Monitoring/Chronicle e integrei PagerDuty para incidentes.
- **Consequências**: Tenho feedback em tempo real, suporte às metas de SLO e dados completos para post-mortems.

## ADR-022 - CI/CD unificado com GitLab
- **Status**: Aceito
- **Contexto**: A arquitetura multi-runtime exige pipelines consistentes e governança de deploy.
- **Decisão**: Configurei GitLab CI/CD self-hosted com stages padrão (lint/test/security/build/deploy), ambientes efêmeros, IaC (Terraform/Helm) e métricas DORA. Provisionei dois runners dedicados em instâncias n2-standard-4 preemptíveis (autoscaling) para suportar até 6 jobs concorrentes sem saturar o Container Registry.
- **Consequências**: Automatizei qualidade, tornei deploys auditáveis, aproveitei GitLab Container/Package Registry e administro a infraestrutura dos runners (custo previsto em `costs.md` e manutenção via scripts IaC).

---

## Timeline das decisões
1. **Semana 1** – Realizei discovery, domain storytelling e defini limites (ADR-001).
2. **Semana 2** – Comparei stacks, escolhi Rust e validei GKE vs. Cloud Run (ADR-002, ADR-009).
3. **Semana 3** – Detalhei microserviço Rust e decidi pelo monólito FastAPI em MIG (ADR-003, ADR-004, ADR-010).
4. **Semana 4** – Estruturei persistência, topologia de bancos e estratégia offline (ADR-006, ADR-014, ADR-015, ADR-018).
5. **Semana 5** – Formalizei Admin React, pipelines Reports e Kafka interno (ADR-016, ADR-017, ADR-011).
6. **Semana 6** – Consolidei resiliência, observabilidade e CI/CD GitLab (ADR-019, ADR-021, ADR-022).
7. **Semana 7** – Finalizei segurança em camadas, mobile KMP e IA de anomalias (ADR-020, ADR-007, ADR-008).
