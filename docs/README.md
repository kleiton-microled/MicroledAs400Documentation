# Documentação do Projeto

Este diretório contém a documentação técnica do projeto Microled.As400.

## Documentos Disponíveis

### APIs e fluxos

- **[FILEMANAGER.md](FILEMANAGER.md)** - FileManager: fluxo de upload/download de arquivos
  - Visão geral e autenticação
  - Fluxo de upload (validação, armazenamento em disco, opcional persistência em banco)
  - Fluxo de download (GET por path, fileName, fileType)
  - Endpoints POST e GET, body e query params, códigos de resposta
  - Segurança (path/fileName, Base64, FileType) e configuração (BaseStoragePath, IIS)

### Banco de Dados

- **[SQLServer.Schema.md](database/SQLServer.Schema.md)** - Documentação técnica completa do schema SQL Server
  - Visão geral do banco de dados
  - Convenções e padrões
  - Documentação detalhada de todas as tabelas
  - Relacionamentos e índices
  - Fluxos do sistema
  - Exemplos de queries úteis

## Como Contribuir

Ao adicionar novas tabelas ou modificar o schema existente:

1. Atualize os scripts SQL em `/scripts/sqlserver/`
2. Atualize a documentação em `database/SQLServer.Schema.md`
3. Mantenha os exemplos de queries atualizados
