# Fase 2: Data Understanding — Ingestão Bronze

> CRISP-DM: Fase 2 — Data Understanding  

---

## Ingestão — Object Storage (Bronze)

**Data:** 2026-03-24  
**Fase:** CRISP-DM Fase 2 — Data Understanding (Segunda)

## O que foi feito

| Ação | Detalhe |
|---|---|
| Upload realizado | OCI CLI — `oci os object put` |
| Destino | `risco-de-credito-bronze/give-me-some-credit/raw/cs-training.csv` |
| Integridade verificada | MD5: `zOUlodQfI09Y07tS89JRaw==` |
| Tamanho | 7.564.965 bytes (~7,2 MB) |
| Timestamp | 2026-03-24T01:55:11 UTC |

## Comando executado
```bash
oci os object put \
  --bucket-name risco-de-credito-bronze \
  --namespace <NAMESPACE> \
  --name give-me-some-credit/raw/cs-training.csv \
  --file <PATH_TO_FILE> \
  --region sa-saopaulo-1
```

## Verificação
```bash
oci os object list \
  --bucket-name risco-de-credito-bronze \
  --namespace <NAMESPACE> \
  --region sa-saopaulo-1
# Resultado: 1 objeto listado — integridade confirmada
```

## Decisões tomadas

**Por que subir apenas o `cs-training.csv` e não o zip completo?**  
O `GiveMeSomeCredit.zip` é o container de transporte do Kaggle. O `cs-training.csv` é o único arquivo com target (`SeriousDlqin2yrs`) — único útil no pipeline. O `cs-test.csv` não tem target e não entra no projeto. Bronze armazena dado bruto útil, não tudo.

**Por que o prefixo `give-me-some-credit/raw/`?**  
Organização dentro do bucket para quando houver múltiplos datasets ou versões futuras. `raw/` sinaliza que é dado original, sem transformação.

---

## Próximo passo

Criar tabela `CREDIT_BRONZE` no ATP e importar o CSV via `DBMS_CLOUD.COPY_DATA`.
