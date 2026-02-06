# Visão Geral – Microled.As400

Este diretório contém a documentação técnica do projeto **Microled.As400**. Os documentos estão organizados por tema; todo o conteúdo provém dos arquivos Markdown existentes.

**Página inicial da documentação:** [index.md](index.md)

---

## Arquitetura

Documentação de convenções, schema do banco de dados e arquitetura genérica dos módulos.

| Documento | Conteúdo |
|-----------|----------|
| [database/SQLServer.Schema.md](database/SQLServer.Schema.md) | Visão geral do banco, convenções e padrões, tabelas, relacionamentos e índices |
| [FUNCTIONAL_ACK.md](FUNCTIONAL_ACK.md) | Arquitetura genérica do Functional Acknowledgement (context, builder, dispatcher) |

---

## Fluxos

Fluxos de negócio e processamento descritos na documentação.

| Documento | Conteúdo |
|-----------|----------|
| [FILEMANAGER.md](FILEMANAGER.md) | Fluxo de upload (validação, armazenamento, opcional persistência em banco) e fluxo de download (GET por path, fileName, fileType) |
| [COUNTERPARTIES_API.md](COUNTERPARTIES_API.md) | Fluxo de contrapartes: cadastro (`POST`), consulta e listagem (`GET`); endereços e referências fiscais (REPLACE ALL); queries |
| [TICKETS_API.md](TICKETS_API.md) | Fluxo de tickets: recepção (`POST /api/tickets`), processamento (worker), retorno (outbox), gateway e callback; máquinas de estado e queries |
| [database/SQLServer.Schema.md](database/SQLServer.Schema.md) | Fluxos e tabelas: referências a contrapartes e tickets (seção 4); envio de Functional ACK; definições de todas as tabelas |
| [FUNCTIONAL_ACK.md](FUNCTIONAL_ACK.md) | Uso do ACK em qualquer fluxo (Product, Counterparty, Ticket), fluxos atualizados |

---

## APIs e Endpoints

Endpoints REST documentados.

| Documento | Conteúdo |
|-----------|----------|
| [FILEMANAGER.md](FILEMANAGER.md) | POST e GET `api/filemanager/files`, body e query params, códigos de resposta |
| [PRODUCTS_API.md](PRODUCTS_API.md) | POST/GET `api/v1/products`, GET por `masterDataId`, filtros e paginação |
| [COUNTERPARTIES_API.md](COUNTERPARTIES_API.md) | POST/GET `api/counterparties` (cadastro, consulta por businessEntity+businessDocID, listagem paginada) |
| [TICKETS_API.md](TICKETS_API.md) | POST `api/tickets`, fluxos (recepção, processamento, retorno, gateway) e queries úteis |

---

## Modelos e Payloads

Estruturas de request/response e exemplos de payloads.

| Documento | Conteúdo |
|-----------|----------|
| [FILEMANAGER.md](FILEMANAGER.md) | `UploadFileRequest`, `UploadFileResponse`, `GetFileResponse`, enum `FileType` |
| [PRODUCTS_API.md](PRODUCTS_API.md) | Payload `commodity` (functionalDocID, commodityHeader, commodityData, packaging), `ProductResponse`, `PagedResultDto` |
| [FUNCTIONAL_ACK.md](FUNCTIONAL_ACK.md) | Payload JSON do Functional ACK (functionalDocID, header com `ackBusinessApplication`, items) |

---

## Integrações

Integrações com sistemas externos e gateways.

| Documento | Conteúdo |
|-----------|----------|
| [FUNCTIONAL_ACK.md](FUNCTIONAL_ACK.md) | Integração Chronos (gateway Functional ACK v3), configuração, headers HTTP, resiliência (retry, backoff), auditoria em `TB_FUNCTIONAL_ACK_LOG` |

---

## Observações Técnicas

Configuração, deploy e observações finais.

| Documento | Conteúdo |
|-----------|----------|
| [GITHUB_PAGES_SETUP.md](GITHUB_PAGES_SETUP.md) | Configuração do GitHub Pages (Jekyll, índice, Wiki) |
| [database/SQLServer.Schema.md](database/SQLServer.Schema.md) | Observações finais: performance, integridade referencial, auditoria, resiliência (seção 6) |
| [FILEMANAGER.md](FILEMANAGER.md) | Configuração `FileStorage:BaseStoragePath`, segurança (path/fileName, Base64, FileType), IIS |

---

## Como contribuir

Ao adicionar novas tabelas ou modificar o schema existente:

1. Atualize os scripts SQL em `/scripts/sqlserver/`
2. Atualize a documentação em [database/SQLServer.Schema.md](database/SQLServer.Schema.md)
3. Mantenha os exemplos de queries atualizados

Para outras alterações, edite o arquivo `.md` correspondente e preserve os links relativos desta página.

---

↑ [Voltar à documentação](index.md)
