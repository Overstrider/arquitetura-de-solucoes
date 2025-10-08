# Estimativa de Custos Mensais

**Total estimado revisado:** **R$ 9.925/mês**
Câmbio de referência: **USD 1 = BRL 5,34**. Valores com margens e premissas explicitadas abaixo.

---

## 1) Infraestrutura GCP (**R$ 7.410/mês**)

| Item                              | Valor (R$) | Cálculo / Premissas                                                                |
| --------------------------------- | ---------: | ---------------------------------------------------------------------------------- |
| GKE (microserviço + Kafka)        |        980 | 3 nós n2-standard-2 + 200 GB PD-SSD com CUD 25%                                    |
| Taxa de cluster GKE               |        390 | $0,10/cluster-h × 730h ≈ $73 ⇒ R$ 390                                              |
| Compute Engine (Monólito Modular) |      2.640 | 2× n2-standard-4 (4 vCPU/16GB) em MIG ativo/standby; ~R$ 1.320 por VM após SUD/CUD |
| Cloud SQL (PostgreSQL)            |      2.200 | Primary (8 vCPU/32GB + 500GB SSD) + read replica (4 vCPU/16GB) com CUD 25%         |
| Cloud Storage                     |        150 | 500 GB Standard + 1 TB Archive + operações                                         |
| Cloud Monitoring/Logging          |        200 | Dimensionamento conservador conforme ingestão                                      |
| Network Egress                    |        400 | 500 GB inter‑região + 200 GB internet                                              |
| GitLab CI Runners                 |        450 | 2 VMs n2-standard-4 preemptíveis (~$0,058/h)                                       |

**Subtotal Infra GCP:** **R$ 7.410/mês**

---

## 2) Vertex AI / Gemini 2.5 (**R$ 300/mês**)

Premissas: 5.000 alertas/mês; cada alerta ≈ 3k tokens in + 3k tokens out.
Tabela (out/2025): Flash $0,30/M in e $2,50/M out; Pro $1,25/M in e $10/M out.
Cálculo: 4.500 alertas (Flash) + 500 (Pro) ⇒ ~$54,7 USD/mês ≈ R$ 293 → **R$ 300**.

---

## 3) Licenças e Suporte Operacional (**R$ 2.215/mês**)

| Item                        |       USD/mês | Detalhe                        |       Valor em R$ |
| --------------------------- | ------------: | ------------------------------ | ----------------: |
| PagerDuty Incident Response |       $39 × 5 | Escalonamento 24/7 e runbooks  |            R$ 210 |
| Apigee Pay‑as‑you‑go (Base) | $0,5/h × 730h | Ambiente ativo 24×7 por região | ≈ $365 ⇒ R$ 1.950 |
| MDM / Miscelânea            |           $10 | Custos auxiliares              |             R$ 55 |

**Subtotal Licenças:** **R$ 2.215/mês**

### Origem e Escopo das Licenças/Serviços

* **PagerDuty Incident Response** — SaaS da PagerDuty, Inc., cobrança por usuário/mês.
* **Apigee (Pay‑as‑you‑go)** — Serviço Google Cloud de API Management, cobrança por ambiente/hora por região e por volume de chamadas.
* **MDM / Miscelânea** — Serviços auxiliares de baixo valor (ex.: MDM corporativo, CDN privada, armazenamento de snapshots/backups).

---

## 4) Totais

```
Infraestrutura (GCP) ............ R$ 7.410
Vertex AI (Gemini 2.5) .......... R$   300
Licenças e Suporte .............. R$ 2.215
                                  -------
TOTAL ESTIMADO (revisado) ....... R$ 9.925/mês
```
