# Tickets API (ShipmentTicket)

**Base route**: `api/tickets`  
**Auth**: JWT  
**Contexto**: Controle de processamento de tickets de remessa (ShipmentTicket) recebidos do sistema Microled. Suporta fases 15 (dados básicos) e 6 (dados + pesos). Idempotência por `(BusinessEntity, BusinessDocId, PhaseCode)`.

---

### POST `api/tickets`

- **Descrição**: recebe um ticket e grava na interface para processamento assíncrono.
- **Body**: JSON do ShipmentTicket (payload completo armazenado em `RawPayloadJson`).
- **Fluxo**: `POST /api/tickets` → `TicketService` → `SqlServerTicketRepository.UpsertAsync`
- **Tabela**: **TB_TICKET_INTERFACE** – grava/atualiza com idempotência; status inicial `Awaiting`.
- **Respostas**: 200 OK (aceito) ou 4xx/5xx conforme validação e erros.

---

## Fluxos

### 1. Recepção de Ticket

**Fluxo**: `POST /api/tickets` → `TicketService` → `SqlServerTicketRepository`

**Tabelas envolvidas**:
- **TB_TICKET_INTERFACE**: Grava/atualiza ticket (idempotência por `(BusinessEntity, BusinessDocId, PhaseCode)`). Status inicial: `Awaiting`. Campo `RawPayloadJson` armazena o JSON completo do ticket recebido.

---

### 2. Processamento de Ticket (Worker)

**Fluxo**: `TicketProcessingWorker` (background service) → `SqlServerTicketRepository` → Processamento → Atualização de status

**Tabelas envolvidas**:
- **TB_TICKET_INTERFACE**:
  - `DequeueAwaitingAsync`: Busca tickets com `Status = 'Awaiting'` e `AttemptCount < MaxAttempts`
  - Atualiza status para `Processing` → `Processed` ou `Error`
  - Incrementa `AttemptCount` a cada tentativa
  - Após processamento bem-sucedido, enfileira retorno em **TB_TICKET_RETURN_OUTBOX**

---

### 3. Envio de Ticket Return

**Fluxo**: `TicketProcessingWorker` → `TicketReturnOutboxRepository` → `TicketReturnOutboxWorker` (background service)

**Tabelas envolvidas**:
- **TB_TICKET_RETURN_OUTBOX**: `EnqueueAsync` (status `Pending`, com verificação de duplicidade via `ExistsSentAsync`), `DequeuePendingAsync` pelo worker, atualização para `Sent` ou `Error`.
- **TB_TICKET_INTERFACE**: Atualização de `ReturnStatus`, `ReturnAttemptCount`, `ReturnLastError`, `ReturnSentAt`, `ReturnLastAttemptAt`.

---

### 4. Fluxo Gateway (Envio ao Gateway + Callback)

**Fluxo**: `TicketProcessingWorker` → Envio ao Gateway → **TB_TICKET_INTERFACE** (atualiza `GatewaySendStatus`) → Recebimento de Callback → **TB_TICKET_INTERFACE** (atualiza `CallbackStatus`)

**Tabelas envolvidas**:
- **TB_TICKET_INTERFACE**: `GatewaySendStatus` (envio ao gateway externo), `CallbackStatus` (recebimento e processamento do callback assíncrono), `CallbackPayloadJson` (JSON do callback).

---

## Máquinas de estado

### Status principal (TB_TICKET_INTERFACE)

| Status      | Descrição                          |
|------------|-------------------------------------|
| Awaiting   | Ticket aguardando processamento     |
| Processing | Ticket em processamento             |
| Processed  | Ticket processado com sucesso        |
| Error      | Erro no processamento (retentável)  |

### ReturnStatus

| Status   | Descrição                    |
|----------|------------------------------|
| Pending  | Retorno pendente de envio    |
| Sending  | Retorno sendo enviado        |
| Sent     | Retorno enviado com sucesso  |
| Error    | Erro no envio do retorno     |

### GatewaySendStatus

| Status   | Descrição                      |
|----------|---------------------------------|
| Pending  | Envio ao gateway pendente      |
| Sending  | Enviando ao gateway            |
| Sent     | Enviado ao gateway com sucesso |
| Error    | Erro no envio ao gateway       |

### CallbackStatus

| Status    | Descrição                         |
|-----------|-----------------------------------|
| Pending   | Callback pendente de recebimento  |
| Received  | Callback recebido                 |
| Success   | Callback processado com sucesso   |
| Error     | Erro no processamento do callback |

### Outbox – TB_TICKET_RETURN_OUTBOX

| Status   | Descrição                    |
|----------|------------------------------|
| Pending  | Mensagem aguardando envio     |
| Sending  | Mensagem sendo enviada        |
| Sent     | Mensagem enviada com sucesso  |
| Error    | Erro no envio (retentável)    |

---

## Tabelas e schema

Definições completas das tabelas **TB_TICKET_INTERFACE** e **TB_TICKET_RETURN_OUTBOX** (colunas, índices, constraints): [database/SQLServer.Schema.md](database/SQLServer.Schema.md) (seções 3.4 e 3.6).

---

## Queries úteis

### Tickets aguardando processamento

```sql
SELECT 
    Id,
    BusinessEntity,
    BusinessDocId,
    PhaseCode,
    Status,
    AttemptCount,
    LastError,
    CreatedAt,
    UpdatedAt
FROM TB_TICKET_INTERFACE
WHERE Status = 'Awaiting'
  AND AttemptCount < 5  -- MaxAttempts
ORDER BY CreatedAt ASC;
```

### Tickets com erro

```sql
SELECT 
    Id,
    BusinessEntity,
    BusinessDocId,
    PhaseCode,
    Status,
    AttemptCount,
    LastError,
    CreatedAt,
    UpdatedAt
FROM TB_TICKET_INTERFACE
WHERE Status = 'Error'
ORDER BY UpdatedAt DESC;
```

### Últimos erros de processamento

```sql
SELECT TOP 50
    Id,
    BusinessEntity,
    BusinessDocId,
    PhaseCode,
    Status,
    AttemptCount,
    LastError,
    UpdatedAt
FROM TB_TICKET_INTERFACE
WHERE LastError IS NOT NULL
ORDER BY UpdatedAt DESC;
```

### Retornos pendentes

```sql
SELECT 
    Id,
    BusinessEntity,
    BusinessDocId,
    PhaseCode,
    ReturnStatus,
    ReturnAttemptCount,
    ReturnLastError,
    ReturnLastAttemptAt
FROM TB_TICKET_INTERFACE
WHERE ReturnStatus IN ('Pending', 'Error')
  AND ReturnAttemptCount < 5
ORDER BY ReturnLastAttemptAt ASC NULLS FIRST;
```

### Outbox de Ticket Return pendente

```sql
SELECT 
    Id,
    BusinessEntity,
    BusinessDocId,
    PhaseCode,
    Status,
    AttemptCount,
    LastError,
    CreatedAt,
    LastAttemptAt
FROM TB_TICKET_RETURN_OUTBOX
WHERE Status IN ('Pending', 'Error')
  AND AttemptCount < 5
ORDER BY CreatedAt ASC;
```

### Tickets por BusinessDocId (todas as fases)

```sql
SELECT 
    Id,
    BusinessEntity,
    BusinessDocId,
    PhaseCode,
    Status,
    AttemptCount,
    ReturnStatus,
    GatewaySendStatus,
    CallbackStatus,
    CreatedAt,
    UpdatedAt
FROM TB_TICKET_INTERFACE
WHERE BusinessEntity = 'TCK'
  AND BusinessDocId = '01-0107-0257-5200004'
ORDER BY PhaseCode;
```

### Callbacks recebidos

```sql
SELECT 
    Id,
    BusinessEntity,
    BusinessDocId,
    PhaseCode,
    CallbackStatus,
    CallbackCode,
    CallbackMessage,
    CallbackReceivedAt
FROM TB_TICKET_INTERFACE
WHERE CallbackStatus = 'Received'
  OR CallbackStatus = 'Success'
  OR CallbackStatus = 'Error'
ORDER BY CallbackReceivedAt DESC;
```

### Envios ao gateway pendentes

```sql
SELECT 
    Id,
    BusinessEntity,
    BusinessDocId,
    PhaseCode,
    GatewaySendStatus,
    GatewaySendAttemptCount,
    GatewaySendLastError,
    GatewaySendLastAttemptAt
FROM TB_TICKET_INTERFACE
WHERE GatewaySendStatus IN ('Pending', 'Error')
  AND GatewaySendAttemptCount < 5
ORDER BY GatewaySendLastAttemptAt ASC NULLS FIRST;
```

---

↑ [Voltar à documentação](index.md)
