![Context](imgs/context.png)

# Contexto e Arquitetura Corporativa

## 1. Visão do Negócio e Objetivos
- **Missão**: entregar à NAOVON um ecossistema capaz de registrar 100.000 lançamentos/s, consolidar saldos diários e detectar fraudes antes que impactem o negócio.
- **Objetivos estratégicos que defini**:
  - Garantir transparência financeira para a gestora central com relatórios disponíveis às 18h.
  - Permitir que 100.000 vendedoras atuem 24/7 em regime offline-first com sincronização automática.
  - Reduzir perdas operacionais via detecção proativa de anomalias e trilha completa de auditoria.
- **Atores principais**: vendedoras, gestora central, compliance, suporte operacional e eu como responsável pela arquitetura.
- **Restrições que considerei**: orçamento mensal aproximado de R$ 6,63 mil, disponibilidade ≥ 99,9%, p95 < 100 ms, conformidade LGPD e suporte a dispositivos pessoais heterogêneos.

## 2. Arquitetura Corporativa e Domínios
- **Domínios de negócio** que mapeei:
  - **Lançamentos**: ingestão de débitos/créditos, reconciliação, feature store e antifraude.
  - **Catálogo & Campanhas**: governança de SKUs, políticas comerciais e distribuição via CDN.
  - **Governança & Admin**: dashboards em tempo real, aprovações e parametrizações.
  - **Inteligência Analítica**: consolidações (Reports) e IA Observer.
  - **Identidade & Segurança**: Keycloak, MFA, RBAC e postura Zero Trust.
- **Responsabilidades que assumi**:
  - Isolar o fluxo de lançamentos em microserviço Rust (hot path).
  - Centralizar Admin/Catálogo/Reports/IA/Auth em monólito FastAPI modular.
  - Manter Kafka embutido para eventos internos e Cloud SQL como fonte única de dados (instâncias separadas para auth e core financeiro).

## 3. Segurança Corporativa (Visão Macro)
- **Postura**: adotei Zero Trust com sete camadas (Cloud Armor → Apigee → TLS/mTLS → Keycloak → RBAC → Secret Manager/KMS → SIEM/Observabilidade).
- **Zonas de confiança**:
  - Perímetro público protegido por Apigee + Cloud Armor (apps móveis e Admin UI).
  - Zona de serviços (GKE + MIG) com mTLS, cabeçalhos assinados e Kafka privado.
  - Dados sensíveis em Cloud SQL isolados por VPC Service Controls; snapshots CDN servidos com Signed URLs/cookies.
- **Requisitos**: MFA para funções privilegiadas, PKCE + storage criptografado no app, auditorias trimestrais, integração com SIEM (Chronicle) e runbooks PagerDuty.

## 4. Padrões Arquiteturais Macro
- **Escolha estratégica**: arquitetura híbrida composta por microserviço Rust crítico e monólito FastAPI modular.
- **Motivação**: isolar o hot path para escalar independentemente, manter alta velocidade de entrega nos domínios administrativos e reduzir overhead operacional.
- **Trade-offs**: aceitei complexidade multi-runtime e mitiguei com ADRs, contratos bem definidos e pipelines GitLab dedicados.

## 5. Integração Corporativa
- **Fluxos síncronos**: REST/JSON versionado (`/v1`) via Apigee (`/v1/lancamentos`, `/v1/catalogo/sync`, `/v1/admin/*`).
- **Fluxos assíncronos**: eventos Kafka (`ProdutoCriado`, `LancamentoRegistrado`, alertas IA) com retenção de 7 dias e schema versionado.
- **Integrações externas**: Vertex AI (Gemini), Firebase (FCM), SIEM (Chronicle), GitLab CI/CD, CDN assinada para snapshots.

## 6. Requisitos Não-Funcionais Globais
- **Desempenho**: p95 < 100 ms para lançamentos; consolidação de 1M registros < 5 min; alertas IA < 2 min.
- **Disponibilidade**: ≥ 99,9% para APIs críticas; failover automático Cloud SQL/MIG; MTTR alvo < 30 min.
- **Segurança/Conformidade**: LGPD, auditoria completa, MFA, segregação de dados sensíveis.
- **Observabilidade**: OpenTelemetry + Cloud Monitoring, dashboards SLO e MTTD < 5 min.
- **FinOps**: orçamento monitorado via `costs.md` (R$ 6,63 mil), uso de committed/sustained discounts e previsão de custos Vertex AI.

## 7. Mapeamento de Domínios e Capacidades
| Domínio | Capacidades-chave | Responsável | Plataforma |
|---------|------------------|-------------|------------|
| Lançamentos | ingestão débito/crédito, WAL, reconciliação, feature store | Eu | Microserviço Rust (GKE) |
| Catálogo | CRUD SKUs, versionamento incremental, eventos, CSV | Eu | Monólito FastAPI (módulo Catálogo) |
| Admin | Dashboards, approvals, configuração | Eu | FastAPI + Admin React |
| Reports | Consolidação diária, Data Lake, materialized views | Eu | Pipelines Rust (Argo Workflows) |
| IA Observer | Anomalias, alertas, prompts Gemini | Eu | Python + LangChain |
| Identidade | OIDC, MFA, RBAC, tokens | Eu | Keycloak (GKE, DB dedicado) |

## 8. Justificativa das Tecnologias
- **Rust (Axum)**: desempenho e memory safety para 100k TPS com custo de cloud reduzido.
- **Python FastAPI**: produtividade alta, ecossistema maduro e forte compatibilidade com LangChain/analytics.
- **React + TypeScript**: rapidez de entrega para o Admin UI e integração natural com Keycloak.
- **Kotlin Multiplatform**: reutilização de lógica, offline-first nativo e consistência entre Android/iOS.
- **PostgreSQL Cloud SQL**: ACID, réplicas gerenciadas, CDC e suporte a workloads analíticos leves.
- **Kafka embutido (Strimzi)**: eventos internos com replay sem operar cluster dedicado.
- **GitLab CI/CD**: pipelines as code, ambientes efêmeros, segurança (SAST/DAST) e integração com IaC.

## 9. Alinhamento com a Estratégia da NAOVON
- Suporto o crescimento acelerado das vendas porta a porta sem sacrificar governança.
- Entrego flexibilidade para evoluir gradualmente (modularização incremental) mantendo simplicidade inicial.
- Documentei todas as decisões em ADRs, detalhei custos e disponibilizei guias para transferência de conhecimento.
