# Business Understanding — Fase 1

> CRISP-DM: Fase 1  
> Status: ⏳ Pendente  
> **Este documento guia todas as decisões técnicas das fases seguintes.**  
> Preencher antes de escrever qualquer linha de código.

---

## Por que este documento existe

A maioria dos projetos de ML falha não por falta de técnica, mas por falta de clareza sobre o problema que está sendo resolvido. Este documento define o "norte" do projeto — inclusive o threshold do modelo e a escolha de métricas.

Ao final da Fase 5 (Evaluation), este documento será consultado para checar se o modelo atingiu os critérios definidos aqui.

---

## As 6 Perguntas de Negócio

### 1. Qual é o evento de risco?

**Definição técnica:**  
Inadimplência de 90 dias ou mais nos próximos 2 anos — variável `SeriousDlqin2yrs` (binária: 0 = adimplente, 1 = inadimplente).

**O que isso significa na prática:**  
[PREENCHER — ex: cliente que atrasa 3 ou mais parcelas consecutivas ou acumula 90 dias de atraso no período de 24 meses após a concessão]

**Implicação no modelo:**  
Variável target binária → problema de classificação supervisionada.

---

### 2. Quem usa o modelo e como?

**Usuário primário:**  
Sistema de aprovação de crédito — decisão binária automática (aprovar / recusar).

**Contexto de uso:**  
[PREENCHER — ex: modelo roda em batch diário avaliando novas solicitações, ou em tempo real no momento da requisição]

**O que o score representa:**  
Probabilidade de inadimplência. Quanto mais alto, maior o risco. Um threshold converte o score em decisão binária.

**Implicação no modelo:**  
O threshold não é 0.5 por padrão — depende da assimetria de custo definida na pergunta 4.

---

### 3. Quais métricas importam?

**Métricas primárias (linguagem do negócio de crédito):**

| Métrica | Meta mínima | Justificativa |
|---------|-------------|--------------|
| KS (Kolmogorov-Smirnov) | ≥ 0.35 | Padrão de bureaus (Serasa, Boa Vista): mede separação máxima entre bons e maus pagadores |
| AUC-ROC | ≥ 0.75 | Discriminação geral independente de threshold |
| Gini | ≥ 0.50 | = 2×AUC−1; linguagem comum em times de risco de bancos |

**Por que NÃO acurácia:**  
Dataset desbalanceado (≈ 93% adimplentes / 7% inadimplentes). Um modelo que chuta "adimplente" para todos acerta 93% — mas é inútil. Acurácia não discrimina.

**Métricas secundárias (a definir após EDA):**
- Precision@Top20%: eficiência em identificar os clientes de maior risco
- F1-Score no threshold ótimo: balanço precision/recall

---

### 4. Qual a assimetria de erro?

**Tipo I — Falso Negativo (aprovar um mau pagador):**  
Consequência: perda financeira direta — inadimplência, custo de cobrança, provisionamento.

**Tipo II — Falso Positivo (recusar um bom cliente):**  
Consequência: perda de receita — juros não recebidos, cliente vai para concorrente.

**Decisão:**  
[PREENCHER — ex: "Neste projeto, priorizamos minimizar Falsos Negativos. O custo de aprovar um mau pagador é estimado em X vezes o custo de recusar um bom cliente."]

**Implicação no modelo:**  
[PREENCHER — ex: threshold abaixo de 0.5, maior recall para a classe positiva, peso maior na classe 1]

**Referência de mercado:**  
Bancos brasileiros tipicamente operam com razão de custo FN/FP entre 3:1 e 10:1 dependendo do produto (cartão vs. crédito pessoal vs. financiamento). Fintechs de crédito com menor margem tendem a ser mais conservadoras.

---

### 5. Qual a janela de observação?

**Janela disponível no dataset:**  
24 meses de histórico comportamental (variáveis de atraso referem-se a períodos passados).

**Variável target:**  
Evento de inadimplência nos próximos 24 meses a partir da data de observação.

**Implicação:**  
Sem data explícita no dataset — não é possível fazer análise temporal de safra. Para produção real, seria necessário definir data de corte (cut-off date) e janela de desempenho (performance window).

**Risco de data leakage:**  
Verificar na Fase 5 se alguma variável contém informação "do futuro" em relação ao evento de risco. Variáveis de atraso passado são válidas; variáveis que descrevem o estado após o evento não são.

---

### 6. O que o modelo NÃO pode fazer?

**Restrição legal — LGPD (Lei Geral de Proteção de Dados):**  
O modelo não pode usar variáveis sensíveis como proxy para discriminação: raça, gênero, religião, orientação sexual, origem nacional.

**Este dataset:**  
Não contém variáveis sensíveis explícitas. Porém, variáveis como `age` podem funcionar como proxy. Verificar na Fase 5 (fairness check).

**Restrição regulatória — Banco Central / Resolução CMN 4.557:**  
[PREENCHER se aplicável — ex: modelo deve ser explicável para fins de auditoria interna]

**Restrição técnica — Always Free:**  
Modelo deve rodar em 1 OCPU / 1 GB RAM. Soluções que exijam cluster ou GPU estão fora do escopo.

---

## Critérios de Sucesso do Projeto

> Estes critérios serão checados na Fase 5. Se não forem atingidos, o processo retorna à Fase 3.

- [ ] KS ≥ 0.35 no conjunto de teste
- [ ] AUC-ROC ≥ 0.75 no conjunto de teste
- [ ] Gini ≥ 0.50 no conjunto de teste
- [ ] Modelo não discrimina por `age` de forma desproporcional (fairness básica)
- [ ] Nenhuma feature com suspeita de data leakage no top-10 de importância
- [ ] Pipeline reproduzível: notebook executa do início ao fim sem intervenção manual

---

## Glossário de Negócio

| Termo | Definição |
|-------|-----------|
| Inadimplência | Atraso superior a 90 dias no pagamento de obrigação financeira |
| Score de crédito | Probabilidade estimada de um cliente se tornar inadimplente |
| KS | Distância máxima entre as distribuições acumuladas de bons e maus pagadores |
| Gini | Medida de discriminação do modelo; 0 = modelo aleatório, 1 = modelo perfeito |
| Threshold | Ponto de corte do score que define a decisão binária aprovar/recusar |
| Safra | Conjunto de clientes originados em um mesmo período |
| Data leakage | Uso indevido de informação do futuro no treinamento do modelo |
| PSI | Population Stability Index — mede drift da distribuição do score ao longo do tempo |

---

*Documento a ser preenchido na Fase 1 · Revisado na Fase 5*
