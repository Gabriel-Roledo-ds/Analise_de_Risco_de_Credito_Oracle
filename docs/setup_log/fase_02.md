# Fase 2: Data Understanding — Ingestão Bronze

> CRISP-DM: Fase 2 — Data Understanding  

---

## Passo a Passo desta Fase

- [x] Download do `cs-training.csv` no Kaggle
- [x] Upload do CSV no bucket `risco-de-credito-bronze` via OCI CLI
- [x] Verificação de integridade do objeto no Object Storage
- [x] Confirmação do schema `ANALISEDERISCO` e status dos usuários no ATP
- [x] Confirmação da estrutura da tabela `CREDIT_BRONZE`
- [x] Grant de `EXECUTE` em `DBMS_CLOUD` para o schema `ANALISEDERISCO`
- [x] Criação da credential `OCI_CRED_BRONZE` no ATP
- [x] Carga dos 150.000 registros via `DBMS_CLOUD.COPY_DATA`
- [x] Validação da carga com `COUNT(*)`

---

## Ingestão — Object Storage (Bronze)

**Data:** 2026-03-24  
**Fase:** CRISP-DM Fase 2 — Data Understanding (Segunda)

### O que foi feito

| Ação | Detalhe |
|---|---|
| Upload realizado | OCI CLI — `oci os object put` |
| Destino | `risco-de-credito-bronze/give-me-some-credit/raw/cs-training.csv` |
| Integridade verificada | MD5: `zOUlodQfI09Y07tS89JRaw==` |
| Tamanho | 7.564.965 bytes (~7,2 MB) |
| Timestamp | 2026-03-24T01:55:11 UTC |

### Comando executado

```bash
oci os object put \
  --bucket-name risco-de-credito-bronze \
  --namespace <NAMESPACE> \
  --name give-me-some-credit/raw/cs-training.csv \
  --file <PATH_TO_FILE> \
  --region sa-saopaulo-1
```

### Verificação

```bash
oci os object list \
  --bucket-name risco-de-credito-bronze \
  --namespace <NAMESPACE> \
  --region sa-saopaulo-1
# Resultado: 1 objeto listado — integridade confirmada
```

### Decisões tomadas

**Por que subir apenas o `cs-training.csv` e não o zip completo?**  
O `GiveMeSomeCredit.zip` é o container de transporte do Kaggle. O `cs-training.csv` é o único arquivo com target (`SeriousDlqin2yrs`) — único útil no pipeline. O `cs-test.csv` não tem target e não entra no projeto. Bronze armazena dado bruto útil, não tudo.

**Por que o prefixo `give-me-some-credit/raw/`?**  
Organização dentro do bucket para quando houver múltiplos datasets ou versões futuras. `raw/` sinaliza que é dado original, sem transformação.

---

## Ingestão ATP — COPY_DATA

**Data:** 2026-03-31  
**Fase:** CRISP-DM Fase 2 — Data Understanding (Terça)

### O que foi feito

| Ação | Detalhe |
|---|---|
| Schema verificado | `ANALISEDERISCO` — status `OPEN` |
| Tabela verificada | `CREDIT_BRONZE` — 12 colunas, schema correto |
| Grant concedido | `EXECUTE ON DBMS_CLOUD TO ANALISEDERISCO` |
| Credential criada | `OCI_CRED_BRONZE` — Auth Token OCI |
| Carga executada | `DBMS_CLOUD.COPY_DATA` — CSV do bucket Bronze |
| Registros carregados | 150.000 |

### Queries-chave executadas

```sql
-- Verificação do schema e usuários
SELECT username, account_status
FROM dba_users
WHERE username IN ('ADMIN', 'ANALISEDERISCO');

-- Verificação da tabela e owner
SELECT table_name, owner
FROM dba_tables
WHERE owner IN ('ADMIN', 'ANALISEDERISCO');

-- Verificação da estrutura da tabela
SELECT column_name, data_type, data_length
FROM all_tab_columns
WHERE owner = 'ANALISEDERISCO'
AND table_name = 'CREDIT_BRONZE'
ORDER BY column_id;

-- Grant de execução do pacote DBMS_CLOUD
GRANT EXECUTE ON DBMS_CLOUD TO ANALISEDERISCO;

-- Criação da credential de acesso ao Object Storage
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'OCI_CRED_BRONZE',
    username        => 'gabrielroledo.ds@gmail.com',
    password        => '[REDACTED — Auth Token OCI]'
  );
END;
/

-- Carga do CSV na tabela CREDIT_BRONZE
BEGIN
  DBMS_CLOUD.COPY_DATA(
    table_name      => 'CREDIT_BRONZE',
    schema_name     => 'ANALISEDERISCO',
    credential_name => 'OCI_CRED_BRONZE',
    file_uri_list   => 'https://objectstorage.sa-saopaulo-1.oraclecloud.com/n/grqhsdxamd8m/b/risco-de-credito-bronze/o/give-me-some-credit%2Fraw%2Fcs-training.csv',
    format          => JSON_OBJECT(
      'type'             VALUE 'CSV',
      'skipheaders'      VALUE '1',
      'delimiter'        VALUE ',',
      'blankasnull'      VALUE 'true',
      'conversionerrors' VALUE 'store_null'
    )
  );
END;
/

-- Validação da carga
SELECT COUNT(*) FROM ANALISEDERISCO.CREDIT_BRONZE;
-- Resultado esperado: 150000
```

---

## Decisões e Troubleshooting

### 1. Grant de DBMS_CLOUD ausente

**Problema:** Primeira execução do `COPY_DATA` retornou `ORA-20003` por falta de permissão.  
**Causa:** O pacote `DBMS_CLOUD` é de nível DBA. O schema `ANALISEDERISCO` não tinha permissão de execução.  
**Solução:** Conceder `EXECUTE ON DBMS_CLOUD` pelo ADMIN antes de executar o procedure.

```sql
GRANT EXECUTE ON DBMS_CLOUD TO ANALISEDERISCO;
```

**Lição registrada:** Em qualquer novo schema no ATP que precise usar `DBMS_CLOUD`, o grant deve ser o primeiro passo — antes de criar credentials ou executar cargas.

---

### 2. Valores `NA` no CSV rejeitados como número inválido

**Problema:** Segunda execução retornou `ORA-20003 — Reject limit reached`. O log `ADMIN.COPY$2_LOG` indicou `ORA-01722: número inválido` na coluna `MONTHLYINCOME`, linha 8.  
**Causa:** O dataset Give Me Some Credit representa valores ausentes com a string `NA` (padrão R/Kaggle). O Oracle tentou converter `NA` para `NUMBER` e rejeitou a linha.  
**Investigação:** Inspeção direta do CSV confirmou o valor `NA` na coluna `MONTHLYINCOME`:

```
7,0,0.305682465,57,0,5710,NA,8,0,3,0,0
```

**Tentativa intermediária:** Adição de `'blankasnull' VALUE 'true'` — insuficiente, pois trata apenas campos vazios, não strings `NA`.  

**Solução definitiva:** Parâmetro `conversionerrors: store_null` — quando há erro de conversão de tipo, armazena `NULL` em vez de rejeitar a linha. Documentado na [Oracle Autonomous Database Format Options](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/format-options.html).

**Lição registrada:** O parâmetro `nullstring` não existe nessa versão do `DBMS_CLOUD`. O caminho correto para absorver strings não numéricas como NULL é `conversionerrors: store_null`. Nulos representados como `NA` são esperados neste dataset — `MonthlyIncome` e `NumberOfDependents` têm missing values documentados. O tratamento definitivo desses nulos ocorrerá na Fase 3 (Data Preparation).

---

### 3. Lentidão nas views de dicionário de dados

**Observação:** Queries em `dba_users` e `dba_tables` apresentaram latência elevada (carregamento de 30s+) no grupo de consumidores `LOW`.  
**Causa:** Views de dicionário varrem metadados do sistema inteiro. No Always Free com 1 OCPU, o custo é alto.  
**Solução operacional:** Trocar o grupo de consumidores para `HIGH` no SQL Worksheet antes de executar queries administrativas. Queries em tabelas do projeto (`CREDIT_BRONZE`) são rápidas independente do grupo.

---

## Resultado da Fase

| Verificação | Resultado |
|---|---|
| Schema `ANALISEDERISCO` | OPEN |
| Tabela `CREDIT_BRONZE` | 12 colunas, tipos corretos |
| Registros carregados | 150.000 |
| Nulos em `MONTHLYINCOME` | Armazenados como NULL via `store_null` |
| Nulos em `NUMBEROFDEPENDENTS` | Armazenados como NULL via `store_null` |

**Data de conclusão:** 2026-03-31  

Tabela `CREDIT_BRONZE` populada no ATP com 150.000 registros. Pipeline Bronze completo — dados disponíveis para EDA via SQL no notebook.

---

## Próximo passo

EDA Fase 2: exploração da distribuição do target, contagem de nulos por coluna e métricas por segmento de inadimplência — via SQL no OCI Data Science Notebook.
