## Products API (Commodity)

**Base route**: `api/v1/products`  
**Auth**: JWT + policies `MasterDataWrite` (write) / `MasterDataRead` (read)

### POST `api/v1/products`

- **Descrição**: cria ou atualiza um produto (idempotência por `commodity.commodityData.masterDataID`).
- **Body**: JSON exatamente no formato do arquivo de exemplo `Exemplo_Commodity.Json` (root `commodity`).
- **Respostas**:
  - **200 OK**: `ProductResponse` (produto persistido).
  - **400 BadRequest**: `ProblemDetails` com erros de validação.
  - **500 InternalServerError**: `ProblemDetails`.

**Exemplo de body** (trecho simplificado):

```json
{
  "commodity": {
    "functionalDocID": {
      "businessAppID": "ASB",
      "businessEntity": "PRD",
      "businessDocID": "PROD-000000000000000071",
      "originalBusinessDocID": "PROD-000000000000000071",
      "userID": ""
    },
    "functionalAckIsRequired": true,
    "commodityHeader": {
      "targetApplications": {
        "targetApplication": [
          { "applicationCode": "CRN" }
        ]
      }
    },
    "commodityData": {
      "masterDataID": "0003",
      "referenceCode": "15071000",
      "shortName": "OLEO SOJA DEG.",
      "name": "OLEO DE SOJA BRUTO DEGOMADO",
      "platform": "",
      "splitValuation": "PROPRIA",
      "isDeleted": false,
      "productGroup": {
        "productId": "10000107",
        "referenceCode": "",
        "masterDataID": ""
      },
      "packaging": [
        {
          "code": "0001",
          "name": "QUILOGRAMA",
          "shortName": "KG",
          "weight": "0.0000",
          "unitConversionFactor": "1.00000000",
          "weightUnitCode": "KG"
        }
      ]
    }
  }
}
```

### GET `api/v1/products`

- **Descrição**: lista produtos com paginação e filtros.
- **Query params**:
  - `masterDataId` (opcional)
  - `referenceCode` (opcional, NCM)
  - `name` (opcional, like)
  - `shortName` (opcional, like)
  - `isDeleted` (opcional)
  - `page` (opcional, default 1)
  - `pageSize` (opcional, default 20, máx. 200)
- **Resposta**: `PagedResultDto<ProductListItemResponse>`.

### GET `api/v1/products/{masterDataId}`

- **Descrição**: retorna um produto completo + lista de embalagens.
- **Resposta**:
  - **200 OK**: `ProductResponse`.
  - **404 NotFound**: `ProblemDetails` se não encontrado.

---

↑ [Voltar à documentação](index.md)

