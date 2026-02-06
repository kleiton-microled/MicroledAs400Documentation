---
layout: default
title: Documentação Técnica – Microled.As400
---

# Documentação Técnica – Microled.As400

Bem-vindo à documentação técnica do projeto **Microled.As400**. Este site reúne a documentação existente em Markdown de forma organizada e navegável.

## Propósito da documentação

- Centralizar a documentação técnica da API e dos fluxos do sistema
- Facilitar a consulta por desenvolvedores e integradores
- Manter uma única fonte de verdade (arquivos `.md`) sem duplicar conteúdo

## Navegação principal

| Seção | Descrição | Documentos |
|-------|-----------|------------|
| [**Visão Geral**](README.md) | Apresentação do projeto e índice dos documentos | [README.md](README.md) |
| [**Arquitetura**](README.md#arquitetura) | Convenções, schema do banco e arquitetura genérica | [database/SQLServer.Schema.md](database/SQLServer.Schema.md), [FUNCTIONAL_ACK.md](FUNCTIONAL_ACK.md) |
| [**Fluxos**](README.md#fluxos) | Fluxos de upload/download, tickets, contrapartes e ACK | [FILEMANAGER.md](FILEMANAGER.md), [database/SQLServer.Schema.md](database/SQLServer.Schema.md), [FUNCTIONAL_ACK.md](FUNCTIONAL_ACK.md) |
| [**APIs / Endpoints**](README.md#apis-e-endpoints) | Endpoints da API (FileManager, Products, etc.) | [FILEMANAGER.md](FILEMANAGER.md), [PRODUCTS_API.md](PRODUCTS_API.md) |
| [**Modelos / Payloads**](README.md#modelos-e-payloads) | Estruturas de request/response e payloads JSON | [FILEMANAGER.md](FILEMANAGER.md), [PRODUCTS_API.md](PRODUCTS_API.md), [FUNCTIONAL_ACK.md](FUNCTIONAL_ACK.md) |
| [**Integrações**](README.md#integrações) | Integração Chronos (Functional ACK) e gateways | [FUNCTIONAL_ACK.md](FUNCTIONAL_ACK.md) |
| [**Observações Técnicas**](README.md#observações-técnicas) | Configuração, deploy e observações finais | [GITHUB_PAGES_SETUP.md](GITHUB_PAGES_SETUP.md), [database/SQLServer.Schema.md](database/SQLServer.Schema.md), [FILEMANAGER.md](FILEMANAGER.md) |

## Acesso rápido por documento

- [FileManager – Fluxo e API](FILEMANAGER.md)
- [Products API (Commodity)](PRODUCTS_API.md)
- [Functional Acknowledgement (Chronos)](FUNCTIONAL_ACK.md)
- [Schema SQL Server](database/SQLServer.Schema.md)
- [Configuração GitHub Pages](GITHUB_PAGES_SETUP.md)

---

*Todo o conteúdo desta documentação provém dos arquivos Markdown existentes no repositório.*
