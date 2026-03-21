# ATP — Decisões de Setup

## Instância
- Nome: atp_analise_de_risco
- Workload: Transaction Processing
- Tier: Always Free (1 OCPU, 20GB)
- Região: sa-saopaulo-1
- Autenticação: mTLS obrigatório (fixo no Always Free)

## Schema
- Usuário: ANALISEDERISCO
- Tablespace: DATA
- Quota: UNLIMITED
- ORDS habilitado: sim (necessário para acesso ao Database Actions)
- Privilégio adicional: EXECUTE ON DBMS_CLOUD (importar CSV do Object Storage na Fase 2)

## Decisões
- Workload ATP vs ADW: ATP escolhido por suportar carga mista OLTP+analítica,
  compatível com APEX futuro e endpoints de scoring.
- Schema separado do ADMIN: isola o projeto, princípio do menor privilégio.
- mTLS obrigatório: padrão de segurança para dados financeiros.
