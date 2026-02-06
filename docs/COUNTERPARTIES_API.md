# Counterparties API (Contrapartes)

**Base route**: `api/counterparties`  
**Auth**: JWT  
**Contexto**: Persistência de clientes, fornecedores e transportadoras recebidos via API REST. Idempotência por `BusinessDocId`. Todas as operações de cadastro são executadas em uma única transação (commit ou rollback completo).

**EntityType**: `"1"` = Cliente, `"2"` = Fornecedor, `"5"` = Transportadora.

---

## Endpoints

### POST `api/counterparties`

- **Descrição**: grava ou atualiza uma contraparte (idempotência por `BusinessDocId`). Inclui endereços e referências fiscais (estratégia REPLACE ALL: remove os existentes e reinsere os enviados).
- **Body**: JSON do payload de contraparte (armazenado em `RawPayload` para auditoria).
- **Fluxo**: `POST /api/counterparties` → `CounterpartyService` → `SqlServerCounterpartyRepository.SaveAsync`
- **Tabelas**: TB_COUNTERPARTIES (principal), TB_COUNTERPARTY_ADDRESSES, TB_COUNTERPARTY_FISCAL_REFERENCES.
- **Respostas**: 200 OK (ou conforme contrato) ou 4xx/5xx.

---

### GET `api/counterparties?businessEntity=XXX&businessDocID=YYY`

- **Descrição**: consulta uma contraparte por `(businessEntity, businessDocID)`.
- **Fluxo**: `CounterpartyQueryService` → `SqlServerCounterpartyRepository.GetAsync`
- **Tabelas**: TB_COUNTERPARTIES, TB_COUNTERPARTY_ADDRESSES (JOIN), TB_COUNTERPARTY_FISCAL_REFERENCES (JOIN).
- **Resposta**: contraparte com endereços e referências fiscais.

---

### GET `api/counterparties?businessEntity=XXX`

- **Descrição**: lista contrapartes paginadas por `BusinessEntity` (ordenado por `CreatedAt DESC`).
- **Fluxo**: `CounterpartyQueryService` → `SqlServerCounterpartyRepository.ListByBusinessEntityAsync`
- **Tabelas**: TB_COUNTERPARTIES.

---

## Fluxos

### 1. Cadastro de Contraparte

**Fluxo**: `POST /api/counterparties` → `CounterpartyService` → `SqlServerCounterpartyRepository`

**Tabelas envolvidas**:
1. **TB_COUNTERPARTIES**: Grava/atualiza contraparte principal (idempotência por `BusinessDocId`)
2. **TB_COUNTERPARTY_ADDRESSES**: Grava endereços (estratégia REPLACE ALL: deleta todos e reinsere)
3. **TB_COUNTERPARTY_FISCAL_REFERENCES**: Grava referências fiscais (estratégia REPLACE ALL: deleta todos e reinsere)

**Transação**: Todas as operações em uma única transação (commit ou rollback completo).

---

### 2. Consulta de Contraparte

**Fluxo**: `GET /api/counterparties?businessEntity=XXX&businessDocID=YYY` → `CounterpartyQueryService` → `SqlServerCounterpartyRepository`

**Tabelas envolvidas**:
1. **TB_COUNTERPARTIES**: Busca por `(BusinessEntity, BusinessDocId)`
2. **TB_COUNTERPARTY_ADDRESSES**: Endereços relacionados (JOIN via `CounterpartyIdFk`)
3. **TB_COUNTERPARTY_FISCAL_REFERENCES**: Referências fiscais relacionadas (JOIN via `CounterpartyIdFk`)

---

### 3. Listagem de Contrapartes

**Fluxo**: `GET /api/counterparties?businessEntity=XXX` → `CounterpartyQueryService` → `SqlServerCounterpartyRepository`

**Tabelas envolvidas**:
1. **TB_COUNTERPARTIES**: Lista paginada por `BusinessEntity` (ORDER BY `CreatedAt DESC`)

---

## Tabelas e schema

Definições completas das tabelas **TB_COUNTERPARTIES**, **TB_COUNTERPARTY_ADDRESSES** e **TB_COUNTERPARTY_FISCAL_REFERENCES** (colunas, índices, constraints): [database/SQLServer.Schema.md](database/SQLServer.Schema.md) (seções 3.1, 3.2 e 3.3).

---

## Queries úteis

### Buscar contraparte por BusinessDocId

```sql
SELECT 
    c.Id,
    c.BusinessDocId,
    c.BusinessEntity,
    c.Name,
    c.EntityType,
    c.Status,
    c.CreatedAt,
    c.UpdatedAt
FROM TB_COUNTERPARTIES c
WHERE c.BusinessDocId = '01-0107-0257-5200004';
```

### Listar contrapartes por BusinessEntity (paginado)

```sql
-- Total de registros
SELECT COUNT(1) AS Total
FROM TB_COUNTERPARTIES
WHERE BusinessEntity = 'TCK';

-- Página de registros (exemplo: página 1, 10 itens por página)
SELECT
    BusinessDocId,
    BusinessAppId,
    BusinessEntity,
    Name,
    EntityType,
    Status,
    CreatedAt
FROM TB_COUNTERPARTIES
WHERE BusinessEntity = 'TCK'
ORDER BY CreatedAt DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
```

### Buscar contraparte com endereços e referências fiscais

```sql
-- Contraparte principal
SELECT * FROM TB_COUNTERPARTIES WHERE BusinessDocId = '01-0107-0257-5200004';

-- Endereços
SELECT * FROM TB_COUNTERPARTY_ADDRESSES 
WHERE CounterpartyIdFk = (SELECT Id FROM TB_COUNTERPARTIES WHERE BusinessDocId = '01-0107-0257-5200004');

-- Referências Fiscais
SELECT * FROM TB_COUNTERPARTY_FISCAL_REFERENCES 
WHERE CounterpartyIdFk = (SELECT Id FROM TB_COUNTERPARTIES WHERE BusinessDocId = '01-0107-0257-5200004');
```

---

↑ [Voltar à documentação](index.md)
