# FileManager – Fluxo e API

## Visão geral

O **FileManager** permite upload e download de arquivos via API. Os arquivos são enviados em Base64, armazenados em disco em um caminho configurável (`FileStorage:BaseStoragePath`) e, quando há repositório configurado, registrados na tabela de arquivos (metadados, SHA256, path).

- **Rota base**: `api/filemanager/files`
- **Autenticação**: JWT (`[Authorize]`)

---

## Fluxo de upload

1. Cliente envia **POST** com body `UploadFileRequest` (path, fileName, base64Content, fileType, overwrite).
2. **Controller** valida o request (FluentValidation) e chama `IFileManagerService.UploadAsync`.
3. **FileManagerService**:
   - Valida e normaliza `path` e `fileName` (anti path-traversal; caracteres permitidos).
   - Decodifica Base64; rejeita se inválido.
   - Verifica se o arquivo já existe; se existir e `overwrite` for `false`, falha com conflito (409).
   - Calcula SHA256 do conteúdo.
   - Chama **IFileStorage** (ex.: `LocalFileStorage`) para salvar em disco: `BaseStoragePath + path` (a pasta é criada se não existir), depois grava o arquivo.
   - Se `IFileRepository` estiver registrado, persiste um `FileRecord` (fileName, fileType, relativePath, storedPath, sizeInBytes, sha256, createdAt, etc.).
4. Retorna **201 Created** com `UploadFileResponse` (storedPath, fileName, fileType, sizeInBytes, sha256, createdAt).

**Configuração**: em `appsettings` a seção `FileStorage` define `BaseStoragePath` (ex.: `./storage` ou `D:\IIS\microled`). No IIS, a identidade do Application Pool precisa ter permissão de escrita (e criação de subpastas) nesse diretório.

---

## Fluxo de download (GET)

1. Cliente envia **GET** com query params: `path`, `fileName`, `fileType`.
2. **Controller** valida presença de `path` e `fileName` e chama `IFileManagerService.GetAsync`.
3. **FileManagerService**:
   - Normaliza e valida path/fileName (mesmas regras de segurança).
   - Verifica existência do arquivo via `IFileStorage.ExistsAsync`; se não existir, lança `StoredFileNotFoundException` → 404.
   - Lê bytes via `IFileStorage.ReadAsync`, calcula SHA256 e monta `GetFileResponse` com conteúdo em Base64.
4. Retorna **200 OK** com `GetFileResponse` (base64Content, fileName, fileType, sizeInBytes, sha256, storedPath).

---

## Endpoints

### POST `api/filemanager/files`

- **Descrição**: upload de arquivo (conteúdo em Base64). A pasta indicada em `path` é criada se não existir. Opcionalmente registra metadados no banco quando `IFileRepository` está configurado.
- **Body**: `UploadFileRequest` (JSON).

| Campo           | Tipo     | Obrigatório | Descrição                                                                 |
|-----------------|----------|-------------|---------------------------------------------------------------------------|
| base64Content   | string   | Sim         | Conteúdo do arquivo em Base64.                                            |
| fileName        | string   | Sim         | Nome do arquivo (pode incluir extensão). Caracteres: letras, números, `_`, `-`, `.`. |
| path            | string   | Sim         | Pasta relativa (pode ter subpastas, ex. `fevereiro` ou `2025/fevereiro`). Caracteres: letras, números, `_`, `-`, `.`, `/`. |
| fileType        | enum     | Sim         | `XML`, `PDF`, `JPEG` ou `CSV`.                                            |
| overwrite       | boolean  | Não         | Se `true`, sobrescreve arquivo existente; se `false` e existir, retorna 409. |

- **Respostas**:
  - **201 Created**: `UploadFileResponse` (storedPath, fileName, fileType, sizeInBytes, sha256, createdAt).
  - **400 Bad Request**: validação (FluentValidation) ou argumento inválido (ex.: Base64 inválido).
  - **409 Conflict**: arquivo já existe e `overwrite` é `false`.
  - **401 Unauthorized**: token ausente ou inválido.
  - **500 Internal Server Error**: ex.: falha de disco ou permissão (ex. `UnauthorizedAccessException` no diretório).

**Exemplo de body**:

```json
{
  "base64Content": "SGVsbG8gV29ybGQ=",
  "fileName": "exemplo.txt",
  "path": "fevereiro",
  "fileType": 1,
  "overwrite": false
}
```

---

### GET `api/filemanager/files`

- **Descrição**: obtém arquivo por path, nome e tipo. Retorna conteúdo em Base64 e metadados (incl. SHA256).
- **Query params**:
  - `path` (obrigatório): pasta relativa onde o arquivo foi salvo.
  - `fileName` (obrigatório): nome do arquivo.
  - `fileType` (obrigatório): `XML` (0), `PDF` (1), `JPEG` (2), `CSV` (3).
- **Respostas**:
  - **200 OK**: `GetFileResponse` (base64Content, fileName, fileType, sizeInBytes, sha256, storedPath).
  - **400 Bad Request**: `path` ou `fileName` ausentes/inválidos.
  - **404 Not Found**: arquivo não encontrado em disco.
  - **401 Unauthorized**: token ausente ou inválido.
  - **500 Internal Server Error**: erro interno.

**Exemplo**:

```
GET /api/filemanager/files?path=fevereiro&fileName=exemplo.pdf&fileType=PDF
```

---

## Segurança e validação

- **Path e fileName**: não permitem `..`, `:`, e apenas caracteres alfanuméricos, `_`, `-`, `.` e (no path) `/`, evitando path traversal.
- **Base64**: validado antes de decodificar; conteúdo inválido resulta em 400.
- **Tipos de arquivo**: apenas valores do enum `FileType` (XML, PDF, JPEG, CSV).

---

## Configuração

- **FileStorage** (ex.: em `appsettings.json`):

```json
{
  "FileStorage": {
    "BaseStoragePath": "./storage"
  }
}
```

Em produção (ex.: IIS), `BaseStoragePath` pode ser um caminho absoluto (ex.: `D:\IIS\microled`). A conta do Application Pool do IIS deve ter permissão de **Modify** (ou escrita e criação de subpastas) nesse diretório.

---

↑ [Voltar à documentação](index.md)
