# Business Understanding

> CRISP-DM Fase 1 — Definição do problema de negócio  
> Documento de referência para todas as decisões técnicas das fases seguintes.

---

## Contexto

Dataset: **Give Me Some Credit** (Kaggle)  
Registros: 150.000 clientes  
Target: `SeriousDlqin2yrs`  
Ambiente: OCI Always Free — Object Storage, Autonomous Database (Oracle 19c), OCI Data Science  
Metodologia: CRISP-DM  

---

## 1. Qual o evento de risco?

**Definição:** Inadimplência grave — atraso igual ou superior a 90 dias — nos próximos 24 meses.

**Justificativa:**  
O target `SeriousDlqin2yrs` (*Serious Delinquency in 2 years*) é uma variável binária:

| Valor | Significado |
|-------|-------------|
| `1` | Cliente entrou em inadimplência grave nos 24 meses seguintes à observação |
| `0` | Cliente não entrou em inadimplência grave no mesmo período |

O marco de 90 dias é o padrão adotado pelo Banco Central do Brasil, pelo Acordo de Basileia II e pelos principais bureaus de crédito brasileiros (Serasa, Boa Vista, Quod) para classificar inadimplência grave. Abaixo desse prazo, o atraso é considerado recuperável e tratado como risco moderado.

---

## 2. Quem usa o modelo?

**Definição:** Sistema automatizado de scoring com camada de revisão humana para casos de fronteira.

**Fluxo de uso:**

| Camada | Ator | Função |
|--------|------|---------|
| Automatizada | Sistema | Gera score de probabilidade de default para cada cliente |
| Revisão | Analista de crédito | Revisa casos cujo score cai na zona de fronteira do threshold |
| Monitoramento | Gestor de risco | Acompanha KPIs agregados do portfólio via dashboard |

**Implicações para o modelo:**  
O modelo entrega uma **probabilidade de default** (valor entre 0 e 1), não uma decisão binária direta. A decisão final de aprovar ou recusar é determinada pelo threshold definido pela política de crédito. Casos próximos ao threshold são encaminhados para revisão humana — o que exige que o modelo seja **interpretável e auditável**, não apenas preciso.

---

## 3. Quais métricas definem sucesso?

### Métricas primárias

| Métrica | Definição | Meta mínima | Referência |
|---------|-----------|-------------|------------|
| **AUC-ROC** | Probabilidade de que um cliente inadimplente receba score maior que um cliente adimplente escolhidos aleatoriamente. Avalia o poder de ordenação global do modelo, independente de threshold. Varia de 0,5 (aleatório) a 1,0 (perfeito). | ≥ 0,75 | Abaixo disso o modelo não discrimina de forma útil |
| **KS** (Kolmogorov-Smirnov) | Máxima diferença entre as distribuições acumuladas de bons e maus pagadores ao longo dos decis de score. Padrão de avaliação em bureaus e bancos brasileiros. | ≥ 0,35 | Mínimo de mercado para modelos em produção no Brasil |

### Métrica secundária

| Métrica | Definição | Uso |
|---------|-----------|-----|
| **Gini** | Gini = 2 × AUC − 1. Equivalente ao AUC em outra escala (0 a 1). Linguagem padrão em times de risco de crédito e relatórios regulatórios internacionais. | Comunicação com stakeholders de negócio e risco |

### Referência de benchmark

Projetos públicos com o dataset Give Me Some Credit reportam AUC entre 0,85 e 0,87 com modelos bem calibrados (LightGBM, XGBoost). As metas definidas acima são conservadoras — servem como piso de aprovação, não como alvo.

---

## 4. Assimetria de erro — o que é pior?

**Definição:** Aprovar um mau pagador é o erro mais grave neste modelo.

### Tipos de erro

| Erro | Situação | Consequência |
|------|----------|--------------|
| **Falso Negativo** | Modelo classifica mau pagador como bom → crédito aprovado | Perda financeira direta, provisão contábil, inadimplência no portfólio |
| **Falso Positivo** | Modelo classifica bom cliente como mau → crédito recusado | Perda de receita futura, impacto em NPS, risco regulatório de discriminação |

**Justificativa:**  
O custo do Falso Negativo é imediato e mensurável — aparece no balanço. O custo do Falso Positivo é difuso e de longo prazo. No mercado brasileiro, instituições financeiras supervisionadas pelo Banco Central priorizam a contenção de perdas por inadimplência.

**Contraponto crítico:**  
Modelos excessivamente conservadores geram viés de exclusão — recusam sistematicamente perfis de menor renda, histórico curto de crédito ou idade avançada. Isso configura discriminação indireta, vedada pela LGPD e monitorada pelo Banco Central. O threshold deve ser calibrado para minimizar Falsos Negativos **sem** criar padrão sistemático de exclusão de grupos vulneráveis.

**Implicação prática:**  
Na Fase 4 (Modeling), o modelo será treinado com `class_weight='balanced'` para compensar o desbalanceamento 85/15 do dataset. O threshold final será ajustado na Fase 5 (Evaluation) com base na análise de performance por segmento.

---

## 5. Janela de observação

**Definição:** 24 meses — definida pela construção do dataset.

**Justificativa:**  
O nome da variável target — `SeriousDlqin2yrs` (*in 2 years*) — declara explicitamente o horizonte de predição. As variáveis comportamentais do dataset (`NumberOfTime30-59DaysPastDueNotWorse`, `NumberOfTime60-89DaysPastDueNotWorse`, `NumberOfTimes90DaysLate`) registram o histórico de atrasos do cliente e funcionam como preditores do comportamento futuro dentro dessa janela.

Uma janela de 24 meses é adequada para capturar ciclos econômicos e sazonalidade — horizonte padrão em modelos de crédito de médio prazo no mercado brasileiro.

---

## 6. O que o modelo não pode fazer?

**Definição:** O modelo não pode discriminar clientes diretamente ou por variáveis proxy com base em raça, gênero, religião, origem, classe social, orientação sexual ou qualquer outro atributo protegido.

### Restrições legais

| Legislação | Restrição |
|------------|-----------|
| **LGPD — Art. 20** | Garante ao titular o direito de revisão de decisões automatizadas que afetem seus interesses |
| **Resolução CMN 4.557 (Banco Central)** | Exige que modelos de risco sejam auditáveis e que recusas sejam justificáveis |

### Risco de discriminação indireta por proxy

O dataset não contém variáveis sensíveis explícitas. No entanto, as variáveis abaixo podem atuar como proxies de atributos protegidos e precisam ser monitoradas na Fase 5:

| Variável | Risco de proxy |
|----------|----------------|
| `age` | Discriminação por idade |
| `MonthlyIncome` | Discriminação por classe social |
| `NumberOfDependents` | Proxy indireto de perfil familiar |

### Auditabilidade

O modelo não pode ser uma caixa preta. A escolha do algoritmo (LightGBM) e a análise de importância de features na Fase 4 são decisões diretamente ligadas a este requisito — o analista de crédito precisa ser capaz de justificar cada recusa com base em variáveis objetivas e documentadas.

---

## Critérios de aprovação do modelo (Fase 5)

Este documento será consultado na Fase 5 (Evaluation) para verificar se o modelo atingiu os critérios definidos aqui.

| Critério | Meta | Status |
|----------|------|--------|
| AUC-ROC | ≥ 0,75 | A verificar |
| KS | ≥ 0,35 | A verificar |
| Gini | ≥ 0,50 | A verificar |
| Performance consistente por segmento | Sem padrão sistemático de exclusão | A verificar |
| Auditabilidade | Importância de features documentada | A verificar |

---

*Documento produzido na Fase 1 do CRISP-DM — Business Understanding.*  
*Última atualização: 2026-03-29*
