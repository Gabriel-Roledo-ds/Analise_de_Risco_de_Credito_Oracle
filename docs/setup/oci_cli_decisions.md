## OCI CLI — Configuração local

**Data:** 2026-03-13
**Versão instalada:** 3.76.0

**Decisão: instalador oficial vs pip:**
 * Instalador oficial (`install.sh`) — cria virtualenv isolado em ~/lib/oracle-cli,
evita conflitos com dependências do projeto Python (scikit-learn, lightgbm).

**Decisão: pacote db (cx_Oracle) descartado:**

 * cx_Oracle==8.3 não compila no Python 3.12 (pkg_resources removido nessa versão).
Substituído por oracledb — será instalado no ambiente do projeto na Fase 4.

**Autenticação: API Key (User Principal):**

 * Chave privada em ~/.oci/oci_api_key.pem (nunca versionada).
 * Config em ~/.oci/config com permissão 600.

**Validação realizada:**
* oci iam region list → tabela de regiões retornada sem erro
* oci iam compartment get → lifecycle-state: ACTIVE, is-accessible: true

**Região:** sa-saopaulo-1
**Compartment:** analise_de_risco_de_credito
