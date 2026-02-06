# Documentação Técnica - Schema SQL Server

## 1. Visão Geral

Este documento descreve o schema do banco de dados SQL Server utilizado pelo serviço **Microled.As400.Api**. O banco de dados é responsável por:

- **Persistência de Contrapartes**: Armazenamento de dados de clientes, fornecedores e transportadoras recebidos via API REST
- **Interface de Tickets**: Controle de processamento de tickets de remessa (ShipmentTicket) recebidos do sistema Microled
- **Outboxes para Comunicação Assíncrona**: Implementação do padrão Outbox para garantir entrega eventual de mensagens (Functional ACK e Ticket Return) para sistemas externos
- **Auditoria e Rastreabilidade**: Armazenamento de payloads JSON completos para auditoria, debug e reprocessamento

### Observações Importantes

- **Sem Migrations**: O projeto não utiliza Entity Framework migrations. As tabelas são criadas/alteradas através de scripts SQL idempotentes localizados em `/scripts/sqlserver/`
- **Scripts Idempotentes**: Todos os scripts verificam a existência de objetos antes de criar/alterar, permitindo execução múltipla sem erros
- **Banco de Dados**: Os scripts assumem o banco `REDEXV2` (ajustar conforme necessário)

---

## 2. Convenções

### 2.1 Padrão de Nomenclatura

- **Tabelas**: Prefixo `TB_` seguido do nome em UPPER_CASE (ex: `TB_COUNTERPARTIES`)
- **Chaves Primárias**: Sempre `Id` do tipo `BIGINT IDENTITY(1,1)`
- **Chaves Estrangeiras**: Sufixo `Fk` (ex: `CounterpartyIdFk`)
- **Campos de Data/Hora**: Tipo `DATETIME2` com valores em UTC usando `SYSUTCDATETIME()` ou `GETUTCDATE()`
- **Campos JSON**: Tipo `NVARCHAR(MAX)` para armazenar payloads JSON completos

### 2.2 Chaves Primárias e Estrangeiras

- Todas as tabelas possuem chave primária `Id BIGINT IDENTITY(1,1)`
- Foreign keys utilizam `ON DELETE CASCADE` para garantir integridade referencial
- Índices são criados automaticamente nas foreign keys para otimizar joins

### 2.3 Campos de Auditoria

- **CreatedAt**: `DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()` - Data/hora de criação em UTC
- **UpdatedAt**: `DATETIME2 NULL` - Data/hora da última atualização em UTC (atualizado manualmente)
- **RawPayloadJson / RawPayload**: `NVARCHAR(MAX) NULL` - JSON completo do payload recebido/enviado para auditoria, rastreabilidade e possibilidade de reprocessamento

### 2.4 Estratégia de Idempotência

- **TB_COUNTERPARTIES**: Unique key em `BusinessDocId` - garante que o mesmo documento não seja duplicado
- **TB_TICKET_INTERFACE**: Unique key composta `(BusinessEntity, BusinessDocId, PhaseCode)` - garante idempotência por fase do ticket
- **Outboxes**: Verificação de duplicidade feita no código antes de inserir (ex: `ExistsSentAsync` em `TicketReturnOutboxRepository`)

### 2.5 Máquinas de Estados

As tabelas de outbox e interface de tickets utilizam campos de status para controlar o fluxo de processamento:

- **Status**: `VARCHAR(20)` - Estado atual do registro
- **AttemptCount**: `INT NOT NULL DEFAULT(0)` - Contador de tentativas de processamento
- **LastError**: `NVARCHAR(2000) NULL` - Última mensagem de erro (truncada se necessário)
- **LastAttemptAt**: `DATETIME2 NULL` - Data/hora da última tentativa (usado para backoff exponencial)

---

## 3. Tabelas

### 3.1 TB_COUNTERPARTIES

**Finalidade**: Tabela principal para armazenar dados de contrapartes (clientes, fornecedores, transportadoras) recebidos via endpoint `/api/counterparties`.

**Relacionamentos**:
- **PK**: `Id` (BIGINT IDENTITY)
- **FK**: Nenhuma (tabela raiz)
- **Filhas**: `TB_COUNTERPARTY_ADDRESSES`, `TB_COUNTERPARTY_FISCAL_REFERENCES` (via `CounterpartyIdFk`)

**Índices e Constraints**:
- `PK_TB_COUNTERPARTIES` (CLUSTERED) - Chave primária
- `UQ_TB_COUNTERPARTIES_BusinessDocId` (UNIQUE NONCLUSTERED) - Garante idempotência por BusinessDocId
- `IX_TB_COUNTERPARTIES_CounterpartyId` (NONCLUSTERED) - Busca por ID da contraparte
- `IX_TB_COUNTERPARTIES_EntityType` (NONCLUSTERED) - Busca por tipo de entidade

**Uso no Sistema**:
- **Endpoint**: `POST /api/counterparties` - Grava/atualiza contraparte (via `SqlServerCounterpartyRepository.SaveAsync`)
- **Endpoint**: `GET /api/counterparties` - Consulta contraparte por `(businessEntity, businessDocID)` (via `SqlServerCounterpartyRepository.GetAsync`)
- **Endpoint**: `GET /api/counterparties?businessEntity=XXX` - Lista contrapartes paginadas (via `SqlServerCounterpartyRepository.ListByBusinessEntityAsync`)

| Campo | Tipo | Nullable | Default | Chave/Índice | Descrição / Uso |
|------|------|----------|---------|--------------|-----------------|
| Id | BIGINT | NOT NULL | IDENTITY(1,1) | PK (CLUSTERED) | Chave primária auto-incrementada |
| BusinessDocId | VARCHAR(100) | NOT NULL | - | UNIQUE INDEX | Identificador único do documento de negócio; usado como chave de idempotência e correlationId no endpoint |
| BusinessAppId | VARCHAR(20) | NULL | - | - | Código da aplicação de negócio (ex: "CRN") |
| BusinessEntity | VARCHAR(20) | NULL | - | - | Código da entidade de negócio (ex: "TCK") |
| OriginalBusinessDocId | VARCHAR(100) | NULL | - | - | ID original do documento antes de transformação |
| CounterpartyId | VARCHAR(50) | NULL | - | INDEX | ID interno da contraparte no sistema de origem |
| EntityType | VARCHAR(10) | NULL | - | INDEX | Tipo de entidade: "1"=Cliente, "2"=Fornecedor, "5"=Transportadora |
| Name | VARCHAR(200) | NULL | - | - | Nome da contraparte |
| LocalName | VARCHAR(200) | NULL | - | - | Nome local/regionalizado |
| SearchName | VARCHAR(200) | NULL | - | - | Nome para busca/pesquisa |
| ReferenceCode | VARCHAR(50) | NULL | - | - | Código de referência interno |
| IsDeleted | BIT | NOT NULL | 0 | - | Flag de exclusão lógica |
| Source | VARCHAR(20) | NULL | - | - | Origem do cadastro |
| Status | VARCHAR(30) | NOT NULL | 'Received' | - | Status do registro (ex: "Received", "Processed", "Error") |
| CreatedAt | DATETIME2 | NOT NULL | SYSUTCDATETIME() | - | Data/hora de criação em UTC |
| UpdatedAt | DATETIME2 | NULL | - | - | Data/hora da última atualização em UTC |
| RawPayload | NVARCHAR(MAX) | NULL | - | - | JSON completo do payload recebido; usado para auditoria, debug e reprocessamento |

---

### 3.2 TB_COUNTERPARTY_ADDRESSES

**Finalidade**: Armazena endereços associados a uma contraparte. Permite múltiplos endereços por contraparte (principal, entrega, faturamento, etc.).

**Relacionamentos**:
- **PK**: `Id` (BIGINT IDENTITY)
- **FK**: `CounterpartyIdFk` → `TB_COUNTERPARTIES.Id` (ON DELETE CASCADE)

**Índices e Constraints**:
- `PK_TB_COUNTERPARTY_ADDRESSES` (CLUSTERED) - Chave primária
- `FK_TB_COUNTERPARTY_ADDRESSES_Counterparty` (FOREIGN KEY) - Relacionamento com contraparte
- `IX_TB_COUNTERPARTY_ADDRESSES_CounterpartyIdFk` (NONCLUSTERED) - Otimização de joins

**Uso no Sistema**:
- Gravado junto com a contraparte via `SqlServerCounterpartyRepository.SaveAsync` (estratégia REPLACE ALL: deleta todos e reinsere)
- Consultado junto com a contraparte via `SqlServerCounterpartyRepository.GetAsync`

| Campo | Tipo | Nullable | Default | Chave/Índice | Descrição / Uso |
|------|------|----------|---------|--------------|-----------------|
| Id | BIGINT | NOT NULL | IDENTITY(1,1) | PK (CLUSTERED) | Chave primária auto-incrementada |
| CounterpartyIdFk | BIGINT | NOT NULL | - | FK, INDEX | Foreign key para TB_COUNTERPARTIES.Id; ON DELETE CASCADE |
| AddressTypeCode | VARCHAR(10) | NULL | - | - | Código do tipo de endereço (ex: "1") |
| AddressTypeName | VARCHAR(50) | NULL | - | - | Nome do tipo de endereço (ex: "PRINCIPAL", "ENTREGA") |
| Phone | VARCHAR(30) | NULL | - | - | Telefone do endereço |
| Fax | VARCHAR(30) | NULL | - | - | Fax do endereço |
| AddrLine1 | VARCHAR(200) | NULL | - | - | Linha 1 do endereço (logradouro) |
| AddrLine2 | VARCHAR(200) | NULL | - | - | Linha 2 do endereço (complemento) |
| CityCode | VARCHAR(30) | NULL | - | - | Código da cidade (ex: código IBGE) |
| ZipCode | VARCHAR(20) | NULL | - | - | CEP |
| State | VARCHAR(10) | NULL | - | - | Estado/UF |
| District | VARCHAR(100) | NULL | - | - | Bairro/distrito |
| IsMainAddress | BIT | NOT NULL | 0 | - | Flag indicando se é o endereço principal |
| IsDeleted | BIT | NOT NULL | 0 | - | Flag de exclusão lógica |
| CreatedAt | DATETIME2 | NOT NULL | SYSUTCDATETIME() | - | Data/hora de criação em UTC |

---

### 3.3 TB_COUNTERPARTY_FISCAL_REFERENCES

**Finalidade**: Armazena referências fiscais da contraparte (CNPJ/CPF, Inscrição Estadual, etc.). Permite múltiplas referências fiscais por contraparte.

**Relacionamentos**:
- **PK**: `Id` (BIGINT IDENTITY)
- **FK**: `CounterpartyIdFk` → `TB_COUNTERPARTIES.Id` (ON DELETE CASCADE)

**Índices e Constraints**:
- `PK_TB_COUNTERPARTY_FISCAL_REFERENCES` (CLUSTERED) - Chave primária
- `FK_TB_COUNTERPARTY_FISCAL_REFERENCES_Counterparty` (FOREIGN KEY) - Relacionamento com contraparte
- `IX_TB_COUNTERPARTY_FISCAL_REFERENCES_CounterpartyIdFk` (NONCLUSTERED) - Otimização de joins
- `IX_TB_COUNTERPARTY_FISCAL_REFERENCES_FiscalNumber` (NONCLUSTERED) - Busca por CNPJ/CPF/IE

**Uso no Sistema**:
- Gravado junto com a contraparte via `SqlServerCounterpartyRepository.SaveAsync` (estratégia REPLACE ALL: deleta todos e reinsere)
- Consultado junto com a contraparte via `SqlServerCounterpartyRepository.GetAsync`

| Campo | Tipo | Nullable | Default | Chave/Índice | Descrição / Uso |
|------|------|----------|---------|--------------|-----------------|
| Id | BIGINT | NOT NULL | IDENTITY(1,1) | PK (CLUSTERED) | Chave primária auto-incrementada |
| CounterpartyIdFk | BIGINT | NOT NULL | - | FK, INDEX | Foreign key para TB_COUNTERPARTIES.Id; ON DELETE CASCADE |
| FiscalReferenceTypeCode | VARCHAR(10) | NULL | - | - | Código do tipo de referência fiscal (ex: "BR1", "BR3") |
| FiscalReferenceTypeName | VARCHAR(50) | NULL | - | - | Nome do tipo (ex: "CNPJ/CPF", "IE") |
| FiscalNumber | VARCHAR(30) | NULL | - | INDEX | Número fiscal (CNPJ, CPF, IE, etc.) |
| CreatedAt | DATETIME2 | NOT NULL | SYSUTCDATETIME() | - | Data/hora de criação em UTC |

---

### 3.4 TB_TICKET_INTERFACE

**Finalidade**: Tabela de interface para controle de processamento de tickets de remessa (ShipmentTicket) recebidos do sistema Microled. Suporta fases 15 e 6 do fluxo de processamento.

**Relacionamentos**:
- **PK**: `Id` (BIGINT IDENTITY)
- **FK**: Nenhuma (tabela independente)

**Índices e Constraints**:
- `PK_TB_TICKET_INTERFACE` (CLUSTERED) - Chave primária
- `UQ_TB_TICKET_INTERFACE_BusinessEntity_BusinessDocId_PhaseCode` (UNIQUE NONCLUSTERED) - Garante idempotência por (BusinessEntity, BusinessDocId, PhaseCode)
- `IX_TB_TICKET_INTERFACE_Status_CreatedAt` (NONCLUSTERED) - Busca de tickets aguardando processamento
- `IX_TB_TICKET_INTERFACE_ReturnStatus_ReturnLastAttemptAt` (NONCLUSTERED) - Busca de retornos pendentes
- `IX_TB_TICKET_INTERFACE_GatewaySendStatus_GatewaySendLastAttemptAt` (NONCLUSTERED) - Busca de envios ao gateway pendentes
- `IX_TB_TICKET_INTERFACE_CallbackStatus_CallbackReceivedAt` (NONCLUSTERED) - Busca de callbacks

**Uso no Sistema**:
- **Endpoint**: `POST /api/tickets` - Recebe ticket e grava via `SqlServerTicketRepository.UpsertAsync`
- **Worker**: `TicketProcessingWorker` - Processa tickets com status "Awaiting" via `SqlServerTicketRepository.DequeueAwaitingAsync`
- **Worker**: Atualiza status para "Processing", "Processed" ou "Error" conforme resultado do processamento
- **Worker**: `TicketReturnOutboxWorker` - Atualiza campos de retorno (`ReturnStatus`, `ReturnAttemptCount`, etc.)
- **Worker**: Workers de gateway atualizam campos `GatewaySendStatus`, `CallbackStatus`, etc.

**Máquina de Estados - Status Principal**:
- `Awaiting` - Ticket aguardando processamento
- `Processing` - Ticket em processamento
- `Processed` - Ticket processado com sucesso
- `Error` - Erro no processamento (pode ser retentado se `AttemptCount < MaxAttempts`)

**Máquina de Estados - ReturnStatus**:
- `Pending` - Retorno pendente de envio
- `Sending` - Retorno sendo enviado
- `Sent` - Retorno enviado com sucesso
- `Error` - Erro no envio do retorno

**Máquina de Estados - GatewaySendStatus**:
- `Pending` - Envio ao gateway pendente
- `Sending` - Enviando ao gateway
- `Sent` - Enviado ao gateway com sucesso
- `Error` - Erro no envio ao gateway

**Máquina de Estados - CallbackStatus**:
- `Pending` - Callback pendente de recebimento
- `Received` - Callback recebido
- `Success` - Callback processado com sucesso
- `Error` - Erro no processamento do callback

| Campo | Tipo | Nullable | Default | Chave/Índice | Descrição / Uso |
|------|------|----------|---------|--------------|-----------------|
| Id | BIGINT | NOT NULL | IDENTITY(1,1) | PK (CLUSTERED) | Chave primária auto-incrementada |
| BusinessEntity | VARCHAR(100) | NOT NULL | - | UNIQUE (composta) | Código da entidade de negócio (ex: "TCK") |
| BusinessDocId | VARCHAR(100) | NOT NULL | - | UNIQUE (composta) | Identificador único do documento de negócio |
| CompanyCode | VARCHAR(20) | NOT NULL | - | - | Código da empresa |
| BranchCode | VARCHAR(20) | NOT NULL | - | - | Código da filial |
| DestinationCode | VARCHAR(20) | NOT NULL | - | - | Código do destino |
| PhaseCode | INT | NOT NULL | - | UNIQUE (composta) | Código da fase: 15 (dados básicos) ou 6 (dados + pesos) |
| ShipmentEvent | VARCHAR(20) | NOT NULL | - | - | Evento da remessa (ex: "RCD", "DLV") |
| FunctionalAckIsRequired | BIT | NOT NULL | - | - | Flag indicando se Functional ACK é obrigatório |
| TargetApplicationCode | VARCHAR(20) | NOT NULL | - | - | Código da aplicação destino (ex: "ASB") |
| Status | VARCHAR(20) | NOT NULL | 'Awaiting' | INDEX | Status principal: Awaiting, Processing, Processed, Error |
| AttemptCount | INT | NOT NULL | 0 | - | Contador de tentativas de processamento |
| RawPayloadJson | NVARCHAR(MAX) | NOT NULL | - | - | JSON completo do ticket recebido; usado para auditoria e reprocessamento |
| LastError | NVARCHAR(2000) | NULL | - | - | Última mensagem de erro (truncada se > 2000 caracteres) |
| CreatedAt | DATETIME2 | NOT NULL | SYSUTCDATETIME() | - | Data/hora de criação em UTC |
| UpdatedAt | DATETIME2 | NOT NULL | SYSUTCDATETIME() | - | Data/hora da última atualização em UTC |
| ReturnStatus | VARCHAR(20) | NOT NULL | 'Pending' | INDEX | Status do retorno: Pending, Sending, Sent, Error |
| ReturnAttemptCount | INT | NOT NULL | 0 | - | Contador de tentativas de envio do retorno |
| ReturnLastError | NVARCHAR(2000) | NULL | - | - | Última mensagem de erro no envio do retorno |
| ReturnSentAt | DATETIME2 | NULL | - | - | Data/hora em que o retorno foi enviado com sucesso |
| ReturnLastAttemptAt | DATETIME2 | NULL | - | INDEX | Data/hora da última tentativa de envio do retorno |
| GatewaySendStatus | VARCHAR(20) | NOT NULL | 'Pending' | INDEX | Status do envio ao gateway: Pending, Sending, Sent, Error |
| GatewaySendAttemptCount | INT | NOT NULL | 0 | - | Contador de tentativas de envio ao gateway |
| GatewaySendLastError | NVARCHAR(2000) | NULL | - | - | Última mensagem de erro no envio ao gateway |
| GatewaySendLastAttemptAt | DATETIME2 | NULL | - | INDEX | Data/hora da última tentativa de envio ao gateway |
| GatewaySendSentAt | DATETIME2 | NULL | - | - | Data/hora em que o ticket foi enviado ao gateway com sucesso |
| CallbackStatus | VARCHAR(20) | NOT NULL | 'Pending' | INDEX | Status do callback: Pending, Received, Success, Error |
| CallbackReceivedAt | DATETIME2 | NULL | - | INDEX | Data/hora em que o callback foi recebido |
| CallbackPayloadJson | NVARCHAR(MAX) | NULL | - | - | JSON completo do callback recebido |
| CallbackMessage | NVARCHAR(2000) | NULL | - | - | Mensagem do callback (ex: mensagem de erro) |
| CallbackCode | VARCHAR(50) | NULL | - | - | Código do callback (ex: código de erro) |

---

### 3.5 TB_FUNCTIONAL_ACK_OUTBOX

**Finalidade**: Tabela de outbox para envio assíncrono de Functional ACK (v3) para o gateway externo. Implementa padrão Outbox para garantir entrega eventual.

**Relacionamentos**:
- **PK**: `Id` (BIGINT IDENTITY)
- **FK**: Nenhuma (tabela independente)

**Índices e Constraints**:
- `PK_TB_FUNCTIONAL_ACK_OUTBOX` (CLUSTERED) - Chave primária
- `IX_TB_FUNCTIONAL_ACK_OUTBOX_Status_CreatedAt` (NONCLUSTERED) - Busca de mensagens pendentes para envio
- `IX_TB_FUNCTIONAL_ACK_OUTBOX_BusinessDocId` (NONCLUSTERED) - Busca por correlation key

**Uso no Sistema**:
- **Endpoint**: `POST /api/functional-ack` - Enfileira Functional ACK via `FunctionalAckOutboxRepository.EnqueueAsync`
- **Worker**: `FunctionalAckOutboxWorker` - Processa mensagens pendentes via `FunctionalAckOutboxRepository.DequeuePendingAsync`
- **Worker**: Atualiza status para "Sent" ou "Error" conforme resultado do envio

**Máquina de Estados**:
- `Pending` - Mensagem aguardando envio
- `Sending` - Mensagem sendo enviada
- `Sent` - Mensagem enviada com sucesso
- `Error` - Erro no envio (pode ser retentado se `AttemptCount < MaxAttempts`)

| Campo | Tipo | Nullable | Default | Chave/Índice | Descrição / Uso |
|------|------|----------|---------|--------------|-----------------|
| Id | BIGINT | NOT NULL | IDENTITY(1,1) | PK (CLUSTERED) | Chave primária auto-incrementada |
| BusinessDocId | VARCHAR(100) | NOT NULL | - | INDEX | Identificador do documento de negócio; usado como correlation key |
| PayloadJson | NVARCHAR(MAX) | NOT NULL | - | - | JSON completo do Functional ACK a ser enviado |
| TargetUrl | VARCHAR(500) | NOT NULL | - | - | URL de destino para envio do Functional ACK |
| Status | VARCHAR(20) | NOT NULL | 'Pending' | INDEX | Status: Pending, Sending, Sent, Error |
| AttemptCount | INT | NOT NULL | 0 | - | Contador de tentativas de envio |
| CreatedAt | DATETIME2 | NOT NULL | SYSUTCDATETIME() | INDEX | Data/hora de criação em UTC (usado para ordenação de processamento) |
| LastAttemptAt | DATETIME2 | NULL | - | - | Data/hora da última tentativa de envio (usado para backoff exponencial) |
| SentAt | DATETIME2 | NULL | - | - | Data/hora em que a mensagem foi enviada com sucesso |
| LastError | NVARCHAR(2000) | NULL | - | - | Última mensagem de erro (truncada se > 2000 caracteres) |

---

### 3.6 TB_TICKET_RETURN_OUTBOX

**Finalidade**: Tabela de outbox para envio assíncrono de retorno de processamento de tickets (TicketResult/ReturnTicket) para o gateway externo. Implementa padrão Outbox para garantir entrega eventual.

**Relacionamentos**:
- **PK**: `Id` (BIGINT IDENTITY)
- **FK**: Nenhuma (tabela independente)

**Índices e Constraints**:
- `PK_TB_TICKET_RETURN_OUTBOX` (CLUSTERED) - Chave primária
- `IX_TB_TICKET_RETURN_OUTBOX_Status_CreatedAt` (NONCLUSTERED) - Busca de mensagens pendentes para envio
- `IX_TB_TICKET_RETURN_OUTBOX_TicketKey` (NONCLUSTERED) - Busca por chave do ticket (BusinessEntity, BusinessDocId, PhaseCode)

**Uso no Sistema**:
- **Worker**: `TicketProcessingWorker` - Enfileira retorno após processamento bem-sucedido via `TicketReturnOutboxRepository.EnqueueAsync`
- **Worker**: `TicketReturnOutboxWorker` - Processa mensagens pendentes via `TicketReturnOutboxRepository.DequeuePendingAsync`
- **Worker**: Atualiza status para "Sent" ou "Error" conforme resultado do envio
- **Verificação de Duplicidade**: `TicketReturnOutboxRepository.ExistsSentAsync` verifica se já existe retorno enviado para o mesmo ticket antes de enfileirar

**Máquina de Estados**:
- `Pending` - Mensagem aguardando envio
- `Sending` - Mensagem sendo enviada
- `Sent` - Mensagem enviada com sucesso
- `Error` - Erro no envio (pode ser retentado se `AttemptCount < MaxAttempts`)

| Campo | Tipo | Nullable | Default | Chave/Índice | Descrição / Uso |
|------|------|----------|---------|--------------|-----------------|
| Id | BIGINT | NOT NULL | IDENTITY(1,1) | PK (CLUSTERED) | Chave primária auto-incrementada |
| BusinessEntity | VARCHAR(100) | NOT NULL | - | INDEX (composta) | Código da entidade de negócio do ticket |
| BusinessDocId | VARCHAR(100) | NOT NULL | - | INDEX (composta) | Identificador do documento de negócio do ticket |
| PhaseCode | INT | NOT NULL | - | INDEX (composta) | Código da fase do ticket (15 ou 6) |
| PayloadJson | NVARCHAR(MAX) | NOT NULL | - | - | JSON completo do retorno (TicketResult/ReturnTicket) a ser enviado |
| TargetUrl | VARCHAR(500) | NOT NULL | - | - | URL de destino para envio do retorno |
| Status | VARCHAR(20) | NOT NULL | 'Pending' | INDEX | Status: Pending, Sending, Sent, Error |
| AttemptCount | INT | NOT NULL | 0 | - | Contador de tentativas de envio |
| CreatedAt | DATETIME2 | NOT NULL | SYSUTCDATETIME() | INDEX | Data/hora de criação em UTC (usado para ordenação de processamento) |
| LastAttemptAt | DATETIME2 | NULL | - | - | Data/hora da última tentativa de envio (usado para backoff exponencial) |
| SentAt | DATETIME2 | NULL | - | - | Data/hora em que a mensagem foi enviada com sucesso |
| LastError | NVARCHAR(2000) | NULL | - | - | Última mensagem de erro (truncada se > 2000 caracteres) |

---

## 4. Fluxos e Tabelas Envolvidas

### 4.1 Fluxo de Contrapartes

Para o fluxo completo de contrapartes (cadastro, consulta e listagem), ver **[COUNTERPARTIES_API.md](../COUNTERPARTIES_API.md)**.

---

### 4.2 Fluxo de Tickets

Para o fluxo completo de tickets (recepção, processamento, retorno e gateway/callback), ver **[TICKETS_API.md](../TICKETS_API.md)**.

---

### 4.3 Envio de Functional ACK

**Fluxo**: `POST /api/functional-ack` → `FunctionalAckService` → `FunctionalAckOutboxRepository` → `FunctionalAckOutboxWorker` (background service)

**Tabelas Envolvidas**:
1. **TB_FUNCTIONAL_ACK_OUTBOX**: 
   - `EnqueueAsync`: Grava mensagem com status `Pending`
   - `DequeuePendingAsync`: Worker busca mensagens pendentes
   - Atualiza status para `Sent` ou `Error` conforme resultado do envio HTTP

---

---

## 5. Exemplos de Queries Úteis

Consultas de contrapartes (por BusinessDocId, listagem paginada, com endereços e referências fiscais): ver **[COUNTERPARTIES_API.md](../COUNTERPARTIES_API.md#queries-úteis)**.

Consultas específicas de tickets: ver **[TICKETS_API.md](../TICKETS_API.md#queries-úteis)**.

### 5.1 Consultar Outbox de Functional ACK Pendente

```sql
SELECT 
    Id,
    BusinessDocId,
    Status,
    AttemptCount,
    LastError,
    CreatedAt,
    LastAttemptAt
FROM TB_FUNCTIONAL_ACK_OUTBOX
WHERE Status IN ('Pending', 'Error')
  AND AttemptCount < 5
ORDER BY CreatedAt ASC;
```

---

## 6. Observações Finais

### 6.1 Performance

- Todos os índices foram criados para otimizar as queries mais comuns (busca por status, foreign keys, unique keys)
- Índices compostos são utilizados para queries que filtram por múltiplas colunas
- Índices filtrados (WHERE) são utilizados quando apropriado (ex: `WHERE CounterpartyId IS NOT NULL`)

### 6.2 Integridade Referencial

- Foreign keys utilizam `ON DELETE CASCADE` para garantir que dados relacionados sejam removidos automaticamente
- Unique constraints garantem idempotência e evitam duplicatas

### 6.3 Auditoria

- Campos `RawPayloadJson` / `RawPayload` armazenam JSON completo para auditoria e reprocessamento
- Campos `CreatedAt` e `UpdatedAt` permitem rastreabilidade temporal
- Campos `LastError` armazenam mensagens de erro para diagnóstico

### 6.4 Resiliência

- Padrão Outbox garante entrega eventual de mensagens mesmo em caso de falhas
- Contadores de tentativas (`AttemptCount`) e backoff exponencial (`LastAttemptAt`) permitem retry automático
- Status de erro permitem identificação e correção manual quando necessário

---

**Última Atualização**: Janeiro 2025  
**Versão do Schema**: 1.0

---

↑ [Voltar à documentação](../index.md)
