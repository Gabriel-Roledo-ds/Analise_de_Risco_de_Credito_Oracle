# Setup Log — Fase 0: Infraestrutura OCI

> CRISP-DM: Pré-requisito de infraestrutura  
> Duração: 1 semana · Status: ✅ Concluído

---

## Conta OCI

| Campo | Valor |
|-------|-------|
| Data de criação | [PREENCHER] |
| Home Region | sa-saopaulo-1 (Brazil East — São Paulo) |
| Account Name | [PREENCHER] |
| Tenancy OCID | `ocid1.tenancy.oc1..[REDACTED — salvo localmente]` |

**Decisão — Home Region São Paulo:**  
Escolha irreversível. Motivos: (1) latência mínima para operações brasileiras, (2) dados não saem do território nacional — conformidade com LGPD, (3) única região OCI Always Free disponível no Brasil.

---

## IAM — Identidade e Acesso

Estrutura criada seguindo o princípio de **least privilege**: nenhuma operação é feita com o usuário root após o setup inicial.

| Recurso | Nome | Finalidade |
|---------|------|-----------|
| Compartment | `credit-risk` | Isola todos os recursos do projeto |
| Group | `credit-risk-admins` | Agrupa usuários com permissão no compartment |
| User | `[PREENCHER]` | Usuário de trabalho (não-root) |
| Policy | `credit-risk-policy` | Define permissões do grupo no compartment |

**Policy criada:**
```sql
Allow group credit-risk-admins to manage all-resources in compartment credit-risk
```

**Decisão — Compartment isolado:**  
Qualquer recurso criado fora do compartment `credit-risk` não aparece nos relatórios de custo do projeto e não é gerenciado pelas policies. Isolamento é boa prática mesmo no Always Free.

---

## OCI CLI — Configuração Local

```bash
# Arquivo ~/.oci/config
[DEFAULT]
user=ocid1.user.oc1..[REDACTED]
fingerprint=[REDACTED]
tenancy=ocid1.tenancy.oc1..[REDACTED]
region=sa-saopaulo-1
key_file=~/.oci/oci_api_key.pem
```

**Validação:**
```bash
oci iam region list --output table
# Esperado: lista de regiões OCI sem erro de autenticação
```

**Nota de segurança:** O arquivo `~/.oci/oci_api_key.pem` nunca vai para o repositório. O `.gitignore` bloqueia arquivos `.pem` e o diretório `.oci/`.

---

## Networking — VCN

| Componente | Nome | Configuração |
|-----------|------|-------------|
| VCN | `credit-risk-vcn` | CIDR: 10.0.0.0/16 |
| Internet Gateway | `credit-risk-igw` | Acesso público de saída |
| Public Subnet | `credit-risk-subnet-public` | CIDR: 10.0.0.0/24 |
| Security List | `credit-risk-sl` | Entrada: 443 (HTTPS), 1522 (ATP) |

**Criado via:** VCN Wizard (opção "VCN with Internet Connectivity") — gera Internet Gateway e route tables automaticamente.

---

## Object Storage — Data Lake

| Bucket | Finalidade | Visibilidade |
|--------|-----------|-------------|
| `credit-risk-bronze` | Dados brutos do Kaggle (CSV original) | Private |
| `credit-risk-silver` | Dados limpos, nulos tratados | Private |
| `credit-risk-gold` | Features engineered, prontas para ML | Private |

**Decisão — 3 camadas (Medallion Architecture):**  
Padrão adotado por fintechs e bancos digitais (Nubank, Inter, C6). Bronze preserva o dado original — qualquer erro nas transformações permite reprocessar da fonte sem re-download. Silver e Gold são derivados reproduzíveis.

---

## Oracle Autonomous Database (ATP)

| Campo | Valor |
|-------|-------|
| Nome | `credit-risk-atp` |
| Tipo | Autonomous Transaction Processing |
| OCPU | 1 (Always Free) |
| Storage | 20 GB (Always Free) |
| Versão Oracle | 19c |
| Wallet baixada | ✅ |

**Conexão testada via SQL Worksheet:** ✅  
**Schema criado:** `CREDITRISK`

---

## OCI Data Science

| Campo | Valor |
|-------|-------|
| Projeto | `credit-risk-ds` |
| Notebook Session | `credit-risk-nb-01` |
| Shape | VM.Standard.E2.1.Micro (Always Free) |
| Block Volume | 50 GB |

**Validação do ambiente:**
```python
import oci
print(oci.__version__)
# Esperado: versão sem ImportError
```

---

## Budget Alert

**Configurado:** Alerta em $1 USD — notificação por e-mail se qualquer recurso pago for provisionado acidentalmente.

Caminho: Billing → Budgets → Create Budget

---

## Checklist Final da Fase 0

- [ ] Conta OCI ativa com e-mail verificado
- [ ] Home Region definida como São Paulo
- [ ] Compartment `credit-risk` criado
- [ ] IAM: Group + User + Policy configurados
- [ ] OCI CLI instalado e autenticado localmente
- [ ] VCN com Internet Gateway e Subnet pública
- [ ] 3 buckets criados (bronze, silver, gold)
- [ ] ATP provisionado e acessível via SQL Worksheet
- [ ] Wallet do ATP salva localmente (nunca no repositório)
- [ ] Notebook Session ativa com `import oci` sem erro
- [ ] Budget Alert configurado em $1 USD
- [ ] `.gitignore` protegendo credenciais

---

## Erros Encontrados e Soluções

> *Documente aqui qualquer erro que encontrar durante o setup, com a solução. Isso vira documentação valiosa para quem reproduzir o projeto.*

| Erro | Contexto | Solução |
|------|---------|---------|
| — | — | — |

---

## Próxima Fase

**Fase 1 — Business Understanding**  
Antes de qualquer código: definir o problema de negócio, as métricas de sucesso e as restrições do modelo.  
Entregável: `docs/business_understanding.md`
