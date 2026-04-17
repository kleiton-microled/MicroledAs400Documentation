# Chronos.ApiGateway - Handoff Infra

Documento objetivo para implantação e operação do gateway reverso `Chronos.ApiGateway`.

## 1) O que é

`Chronos.ApiGateway` é um reverse proxy/API Gateway em .NET 8 (YARP), sem regra de negócio.

Responsabilidades:

- receber chamadas externas
- autenticar/autorização (placeholder preparado)
- rotear para App Services internos
- propagar headers técnicos
- gerar logs de rastreabilidade

## 2) Roteamento atual

### Rota configurada

- **Entrada**: `/api/microled/{**catch-all}`
- **Saída**: `https://CHRONOS_APP_SERVICE_URL/{**catch-all}`

### Transformações

- Remove prefixo `/api/microled`
- Define header `X-Forwarded-Host` com host original

Exemplo:

- Entrada: `GET /api/microled/tickets/123`
- Destino: `GET https://CHRONOS_APP_SERVICE_URL/tickets/123`

## 3) Configuração obrigatória (antes de subir)

Arquivo: `src/Chronos.ApiGateway/appsettings.json` (ou variável/override de ambiente)

Substituir placeholder:

- `ReverseProxy:Clusters:chronos-app-services:Destinations:destination1:Address`
- de: `https://CHRONOS_APP_SERVICE_URL/`
- para: URL real do App Service interno (terminando com `/`)

## 4) Autenticação (estado atual)

Já preparado com JWT Bearer:

- `Authority`: `https://AUTH_SERVER/` (placeholder)
- `Audience`: `chronos-api`

> Ainda não está com provider real. Ajustar em conjunto com IAM/segurança quando ativar validação final.

## 5) Observabilidade

### Correlation ID

- Header: `X-Correlation-Id`
- Se vier no request, é reutilizado
- Se não vier, gateway gera GUID
- Mesmo valor vai para response e para `TraceIdentifier`

### Logs por requisição

- método HTTP
- path
- status code
- tempo total (ms)
- correlation id

## 6) Saúde da aplicação

- Endpoint: `GET /health`
- Esperado: `200 OK`

## 7) Timeout do proxy

Configurado no cluster:

- `HttpRequest.ActivityTimeout = 00:00:30`

## 8) Checklist de deploy

- [ ] Configurar URL real do App Service (remover placeholder).
- [ ] Configurar `Authority`/`Audience` reais quando autenticação for ativada em produção.
- [ ] Validar acesso de rede (gateway -> app service).
- [ ] Validar `GET /health` no gateway.
- [ ] Validar rota proxy de ponta a ponta (`/api/microled/...`).
- [ ] Validar logs com `X-Correlation-Id`.

## 9) Teste rápido pós-deploy

```bash
curl -i https://<gateway-host>/health
curl -i https://<gateway-host>/api/microled/health
```

Se o segundo teste falhar, validar:

- URL de destino do cluster
- conectividade/regras de firewall
- DNS do App Service
- certificado TLS no destino
