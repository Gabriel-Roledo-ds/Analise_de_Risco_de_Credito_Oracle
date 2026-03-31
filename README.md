# Credit Risk Analytics — Give Me Some Credit

> Pipeline end-to-end de análise de risco de crédito construído no OCI Always Free.  
> Dataset: [Give Me Some Credit](https://www.kaggle.com/c/GiveMeSomeCredit) · 150.000 registros · Target: `SeriousDlqin2yrs`

---

## Objetivo

Construir um modelo de classificação de inadimplência — do zero, em produção real — usando Oracle Cloud Infrastructure (Always Free), seguindo a metodologia CRISP-DM.

Este projeto será desenvolvido ao longo da minha formação em Análise e Ciência de Dados e cada decisão técnica e de negócio será documentada aqui.

---

## Arquitetura

```
Kaggle CSV
    │
    ▼
OCI Object Storage          ← Data Lake em camadas
  ├── risco-de-credito-bronze     ← Dados brutos, sem transformação
  ├── risco-de-credito-silver     ← Dados limpos, nulos tratados
  └── risco-de-credito-gold       ← Features engineered, prontas para modelagem
    │
    ▼
Oracle Autonomous Database  ← SQL First: EDA, feature engineering, monitoramento
    │
    ▼
OCI Data Science Notebook   ← Python: imputação, treino, avaliação
    │
    ▼
OCI Model Catalog           ← Modelo versionado com metadados
    │
    ▼
OCI Model Deployment        ← Endpoint HTTP (Fase 6 — planejada)
Oracle APEX Dashboard        ← Monitoramento de score e drift (Fase 6 — planejada)
```

---

## Resultados (atualizados ao fim de cada fase)

| Métrica | Meta mínima | Resultado |
|---------|-------------|-----------|
| AUC-ROC | ≥ 0.75 | — |
| KS | ≥ 0.35 | — |
| Gini | ≥ 0.50 | — |

> Resultados serão preenchidos ao final da Fase 4 (Modeling).

---

## Estrutura do Repositório

```
Analise_de_Risco_de_Credito_Oracle/
├── README.md
├── docs/
│   ├── architecture_diagram.png       ← Diagrama da arquitetura (a adicionar)
│   └── setup_log/
│       └── fase_0.md                  ← Fase 0: infraestrutura OCI ✅
│       └── fase_01.md                 ← Fase 01: Business Understanding ✅
├── notebooks/
│   ├── 01_eda.ipynb                   ← Fase 2: Exploratory Data Analysis
│   ├── 02_data_preparation.ipynb      ← Fase 3: limpeza e feature engineering
│   └── 03_modeling.ipynb              ← Fase 4: treino e avaliação
├── sql/
│   ├── 01_create_tables.sql           ← DDL das tabelas bronze/silver/gold
│   ├── 02_eda_queries.sql             ← Queries exploratórias documentadas
│   └── 03_feature_engineering.sql    ← Features via SQL (CTE + CASE WHEN)
├── src/
│   └── pipeline.py                    ← Script de pipeline reproduzível
└── requirements.txt                   ← Dependências Python
```

---

## Metodologia: CRISP-DM

| Fase | Status | Entregável |
|------|--------|------------|
| 0 — Setup OCI | ✅ Concluído | `docs/setup_log/fase_0.md` |
| 1 — Business Understanding | ✅ Concluído | `docs/setup_log/fase_01.md` |
| 2 — Data Understanding | ⏳ Pendente | `notebooks/01_eda.ipynb` |
| 3 — Data Preparation | ⏳ Pendente | Tabelas silver + gold no ATP |
| 4 — Modeling | ⏳ Pendente | Modelo no OCI Model Catalog |
| 5 — Evaluation | ⏳ Pendente | Relatório de avaliação |
| 6 — Deployment | 📋 Planejado | Endpoint + Dashboard APEX |

---

## Stack Técnica

| Camada | Tecnologia | Tier |
|--------|-----------|------|
| Data Lake | OCI Object Storage | Always Free |
| Banco analítico | Oracle Autonomous Database 19c | Always Free |
| Modelagem | OCI Data Science (VM.Standard.E2.1.Micro) | Always Free |
| Orquestração SQL | Oracle SQL (dialect 19c) | — |
| Python | 3.x · pandas · scikit-learn · LightGBM | — |
| Versionamento | GitHub | Free |

---

## Decisões de Design

**Por que OCI e não Colab?**  
O OCI Always Free entrega Object Storage + ATP + Notebook — Além disso, pretendo me especializar no ambiente Oracle e desenvolver este projeto partiu dessa decisão.

**Por que SQL First?**  
Feature engineering em SQL fica versionado no banco, é auditável e reproduzível sem depender de ambiente Python.

**Por que LightGBM?**  
Roda dentro do limite de 1 OCPU / 1 GB RAM do Always Free. Lida nativamente com desbalanceamento 85/15 via `class_weight='balanced'`. Feature importance nativa facilita explicabilidade para stakeholders de negócio.

**Todas as decisões tomadas nesse início de projeto passarão pelo crivo do teste prático e podem mudar ao longo do desenvolvimento**

---


## Posts no LinkedIn

Cada fase gera um post técnico documentando o processo:

1. [Das Certs Oracle Foundation pra mão na massa: Análise de Risco no Always Free OCI](https://www.linkedin.com/posts/gabriel-roledo_recursos-oci-activity-7441544228934938624-3gzG?utm_source=share&utm_medium=member_desktop&rcm=ACoAADV8pn4B5h_wYfKEMyUD5EjN2KxHtve26gg) *(Fase 0)*✅
2. [Business Understanding, Proxies de Alto Risco e Por Que Evitá-los](https://www.linkedin.com/posts/gabriel-roledo_business-understanding-activity-7444226920428228608-qNDb?utm_source=share&utm_medium=member_desktop&rcm=ACoAADV8pn4B5h_wYfKEMyUD5EjN2KxHtve26gg) *(Fase 1)*✅
3. [150k clientes. O que os dados dizem antes do modelo](#) *(Fase 2)*
4. [Feature engineering com SQL aplicado a crédito](#) *(Fase 3)*
5. [O número que analistas de crédito realmente olham](#) *(Fase 4)*
6. [O modelo passou? Validando contra critérios de negócio](#) *(Fase 5)*

> Links serão adicionados conforme os posts forem publicados.

---

*Projeto em construção · Atualizado semanalmente*
