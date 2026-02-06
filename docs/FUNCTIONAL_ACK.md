# Functional Acknowledgement (Chronos Integration) - Mecanismo Genérico

## Visão Geral

O módulo de Functional Acknowledgement é um **mecanismo genérico e reutilizável** que envia confirmações de processamento para o gateway Chronos após operações em qualquer fluxo (Produtos, Contrapartes, Tickets, etc.).

### Arquitetura Genérica

O sistema foi projetado para ser **independente do domínio**:

- **`FunctionalAckContext`**: Contexto genérico extraído do request original
- **`FunctionalAckResult`**: Resultado genérico do processamento (sucesso, validação, erro técnico)
- **`IFunctionalAckBuilderGeneric`**: Builder genérico que monta o payload
- **`IFunctionalAckDispatcher`**: Dispatcher genérico que coordena envio e auditoria

### Uso em Qualquer Fluxo

```csharp
// 1. Criar contexto do request original
var context = new FunctionalAckContext
{
    BusinessEntity = functionalDocId.BusinessEntity,
    BusinessDocID = functionalDocId.BusinessDocID,
    OriginalBusinessDocID = functionalDocId.OriginalBusinessDocID,
    BusinessAppID = functionalDocId.BusinessAppID,
    UserID = functionalDocId.UserID,
    FunctionalAckIsRequired = request.FunctionalAckIsRequired,
    Operation = "CREATE", // ou "UPDATE", "DELETE", "PROCESS"
    Source = "Product", // ou "Counterparty", "ShipmentTicket", etc.
    OccurredAt = DateTime.UtcNow
};

// 2. Executar lógica principal
try
{
    await _repository.SaveAsync(...);
    
    // 3. Gerar resultado de sucesso
    var result = new FunctionalAckResult
    {
        Success = true,
        StatusCode = "201",
        Message = "Produto inserido com sucesso"
    };
    
    // 4. Disparar ACK (assíncrono, não bloqueia)
    _ = Task.Run(async () =>
    {
        await _ackDispatcher.DispatchAsync(context, result, cancellationToken);
    });
}
catch (ValidationException ex)
{
    // ACK de erro de validação
    var result = new FunctionalAckResult
    {
        Success = false,
        StatusCode = "400",
        Message = "Validation failed",
        Details = validationErrors
    };
    await _ackDispatcher.DispatchAsync(context, result, cancellationToken);
}
catch (Exception ex)
{
    // ACK de erro técnico
    var result = new FunctionalAckResult
    {
        Success = false,
        StatusCode = "500",
        Message = "Erro interno",
        ExceptionType = ex.GetType().Name,
        ExceptionMessage = ex.Message
    };
    await _ackDispatcher.DispatchAsync(context, result, cancellationToken);
}
```

## Configuração Obrigatória

### Variáveis de Ambiente / App Configuration

```json
{
  "CHRONOS_GATEWAY_APIKEY": "869e7c1e-fd4e-4a91-92c5-7871efcb79a3",
  "CHRONOS_ACK_BUSINESS_APPLICATION": "CRN",
  "ExternalServices:FunctionalAck:BaseUrl": "https://di-api-gw-int.ldc.com/gateway/FunctionalAck/3",
  "ExternalServices:FunctionalAck:TimeoutSeconds": 30
}
```

### Validação Fail-Fast

O serviço **falha ao iniciar** se:
- `CHRONOS_GATEWAY_APIKEY` não estiver configurado
- `CHRONOS_ACK_BUSINESS_APPLICATION` estiver configurado com valor diferente de `"CRN"`

## Comportamento

### Header HTTP Obrigatório

Toda requisição HTTP para o gateway **sempre** inclui:
- `x-Gateway-APIKey`: valor de `CHRONOS_GATEWAY_APIKEY`
- `Content-Type`: `application/json`

### Payload JSON

O campo `functionalAck.header.ackBusinessApplication` **sempre** é `"CRN"`, independentemente de:
- `targetApplications` do request original
- `businessAppID` do request
- Qualquer outro campo do request

**Exemplo de payload:**
```json
{
  "functionalAck": {
    "functionalDocID": {
      "businessAppID": "ASB",
      "businessEntity": "PRD",
      "businessDocID": "PROD-123",
      "originalBusinessDocID": "PROD-123",
      "userID": ""
    },
    "header": [
      {
        "ackBusinessApplication": "CRN",
        "origMessageID": "PROD-123",
        "originalBusinessObject": "AS400_PRODUCT",
        "originalBusinessOperation": "CREATE",
        "status": "201"
      }
    ],
    "items": [...]
  }
}
```

### Múltiplos Headers

Se o request original tiver múltiplos `targetApplications`, o ACK terá múltiplos headers, mas **todos** terão `ackBusinessApplication = "CRN"`.

## Resiliência

- **Retry**: 3 tentativas com backoff exponencial (2s, 4s, 8s)
- **Retry em**: 5xx, timeout, `HttpRequestException`
- **Não bloqueia**: Falhas no envio do ACK não afetam o endpoint principal (fire-and-forget)

## Logs Estruturados

### Sucesso
```
ChronosFunctionalAckSend | Success | BusinessDocID: {BusinessDocID}, HttpStatusCode: {HttpStatusCode}
```

### Envio
```
ChronosFunctionalAckSend | BusinessDocID: {BusinessDocID}, BusinessEntity: {BusinessEntity}, 
AckBusinessApplication: {AckBusinessApplication}, HasApiKeyHeader: {HasApiKeyHeader}, 
Endpoint: {Endpoint}
```

### Falha
```
ChronosFunctionalAckSend | Failed | BusinessDocID: {BusinessDocID}, Endpoint: {Endpoint}, 
HttpStatusCode: {HttpStatusCode}, Error: {Error}
```

## Auditoria

Todos os envios são registrados em `TB_FUNCTIONAL_ACK_LOG`:
- Status: `Pending`, `Sent`, `Failed`
- `HttpStatusCode`, `ErrorMessage` (quando falha)
- `PayloadJson` (opcional, para debug)

## Testes

- **Unit Tests**:
  - `FunctionalAckBuilderTests` - valida que `ackBusinessApplication` sempre é "CRN"
  - `FunctionalAckBuilderGenericTests` - valida builder genérico (context + result → payload)
  - `FunctionalAckDispatcherTests` - valida dispatcher genérico (envio e auditoria)
- **Integration Tests**: `FunctionalAckHttpClientTests` - valida header `x-Gateway-APIKey` e payload

## Fluxos Atualizados

- ✅ **ProductService**: Usa `IFunctionalAckDispatcher` genérico
- ✅ **CounterpartyService**: Usa `IFunctionalAckDispatcher` genérico
- 🔄 **TicketService**: Pode ser atualizado para usar o dispatcher genérico (opcional)
