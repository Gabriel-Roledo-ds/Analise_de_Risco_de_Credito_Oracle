# Setup Log — Fase 0: Infraestrutura OCI

> CRISP-DM: Pré-requisito de infraestrutura  


---

## Conta OCI

| Campo | Valor |
|-------|-------|
| Data de criação | 10/03/2026 |
| Home Region | sa-saopaulo-1 (Brazil East — São Paulo) |
| Account Name | analise_de_risco_roledo |
| Tenancy OCID | `ocid1.tenancy.oc1..[REDACTED — salvo localmente]` |

**Decisão — Home Region São Paulo:**  
Motivos: (1) latência mínima para operações brasileiras, (2) dados não saem do território nacional — conformidade com LGPD, (3) única região OCI Always Free disponível no Brasil.

---

# IAM Setup — Decisões e Raciocínio

**Data:** 2026-03-11  
**Fase:** CRISP-DM Fase 0 — Setup OCI (Terça)

## O que foi configurado

| Recurso | Nome | Descrição |
|---|---|---|
| Compartment | `analise_de_risco_de_credito` | Isola todos os recursos do projeto |
| Group | `desenvolvedores_analise_de_risco` | Agrupa usuários com permissão no compartment |
| User | `gabrielroledo.ds@gmail.com` | Usuário de trabalho |
| Policy | `policy_analise_de_risco_de_credito` | Define permissões do grupo no compartment |
| API Key | gerada e configurada localmente | Autenticação CLI/SDK — nunca versionar |

## Decisões tomadas

**Por que Compartment separado do root?**  
Isola billing, escopo de policies e permite deletar o projeto inteiro sem afetar o tenant.

**Por que não criar usuário de serviço separado?**  
Conta Always Free solo — o usuário principal já é o operador do pipeline. Usuário separado seria overhead sem benefício real.

**Por que policy no root e não no compartment?**  
Groups de Identity Domain só são resolvidos corretamente por policies no root. Policy no compartment não consegue validar a sintaxe `'Default'/'desenvolvedores_analise_de_risco'`.

**Por que `manage` e não `use`?**  
Ambiente de desenvolvimento solo. `manage` dá flexibilidade para criar e deletar recursos sem revisitar o IAM a cada nova etapa.

## Statements da Policy
```
Allow group 'Default'/'desenvolvedores_analise_de_risco' to manage object-family in compartment analise_de_risco_de_credito
Allow group 'Default'/'desenvolvedores_analise_de_risco' to manage autonomous-database-family in compartment analise_de_risco_de_credito
Allow group 'Default'/'desenvolvedores_analise_de_risco' to manage data-science-family in compartment analise_de_risco_de_credito
Allow group 'Default'/'desenvolvedores_analise_de_risco' to manage virtual-network-family in compartment analise_de_risco_de_credito
```

## OCI CLI + Autenticação

### Concluído
- [x] OCI CLI 3.76.0 instalada via instalador oficial (`install.sh`)
- [x] Pacote `db` (cx_Oracle) descartado — incompatível com Python 3.12
- [x] Chave privada movida para `~/.oci/oci_api_key.pem`
- [x] `~/.oci/config` criado e configurado
- [x] Permissões aplicadas: `chmod 600` em config e key
- [x] `oci iam region list` validado — autenticação OK
- [x] `oci iam compartment get` validado — policy IAM OK


### Decisões
- `oracledb` substituirá `cx_Oracle` na Fase 4
- Estrutura de documentação definida: `setup_log.md` (visão geral cronológica) + `docs/setup/` (detalhes técnicos por componente)

```
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
| VCN | `vcn_analise_de_risco_de_credito' | CIDR: 10.0.0.0/16 |
| Internet Gateway | `vcn_analise_de_risco_de_credito' | Acesso público de saída |
| Public Subnet | `vcn_analise_de_risco_de_credito` | CIDR: 10.0.0.0/24 |
| Security List | `vcn_analise_de_risco_de_credito` | Entrada: 443 (HTTPS), 1522 (ATP) |

**Criado via:** VCN Wizard (opção "VCN with Internet Connectivity") — gera Internet Gateway e route tables automaticamente.

---

## Object Storage — Data Lake

| Bucket | Finalidade | Visibilidade |
|--------|-----------|-------------|
| `risco-de-credito-bronze` | Dados brutos do Kaggle (CSV original) | Private |
| `risco-de-credito-silver` | Dados limpos, nulos tratados | Private |
| `risco-de-credito-gold` | Features engineered, prontas para ML | Private |

**Decisão — 3 camadas (Medallion Architecture):**  
Padrão adotado por fintechs e bancos digitais (Nubank, Inter, C6). Bronze preserva o dado original — qualquer erro nas transformações permite reprocessar da fonte sem re-download. Silver e Gold são derivados reproduzíveis.

---

## Oracle Autonomous Database (ATP)

| Campo | Valor |
|-------|-------|
| Nome | `atp_analise_de_risco` |
| Tipo | Autonomous Transaction Processing |
| OCPU | 1 (Always Free) |
| Storage | 20 GB (Always Free) |
| Versão Oracle | 19c |
| Wallet baixada | ✅ |

**Conexão testada via SQL Worksheet:** 
**Schema criado:** `ANALISEDERISCO`

---

> ⚠️ **Execução em andamento a partir daqui**


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
