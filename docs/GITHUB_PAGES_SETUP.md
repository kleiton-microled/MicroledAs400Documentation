# Configuração do GitHub Pages (Opcional)

Este guia explica como configurar o GitHub Pages para exibir a documentação do projeto como uma página web.

## Opção 1: Usar o README Principal (Recomendado - Mais Simples)

A forma mais simples é adicionar links no README principal do projeto, que já está configurado. A documentação pode ser acessada diretamente via GitHub:

- Link direto: `https://github.com/SEU-USUARIO/SEU-REPOSITORIO/blob/main/docs/database/SQLServer.Schema.md`
- GitHub renderiza Markdown automaticamente

## Opção 2: GitHub Pages com Jekyll (Página Web Dedicada)

Se você quiser uma página web dedicada para a documentação:

### Passo 1: Habilitar GitHub Pages

1. Vá em **Settings** do repositório no GitHub
2. Role até a seção **Pages**
3. Em **Source**, selecione:
   - **Branch**: `main` (ou `master`)
   - **Folder**: `/docs` ou `/root`
4. Clique em **Save**

### Passo 2: Criar Arquivo de Configuração (Opcional)

Se quiser usar Jekyll para melhor formatação, crie um arquivo `_config.yml` na raiz do projeto:

```yaml
title: Microled.As400 - Documentação Técnica
description: Documentação técnica do projeto Microled.As400
theme: jekyll-theme-minimal
```

### Passo 3: Criar Página Index (Opcional)

Crie um arquivo `docs/index.md` para servir como página inicial:

```markdown
# Documentação Técnica - Microled.As400

Bem-vindo à documentação técnica do projeto.

## Documentos Disponíveis

- [Schema SQL Server](database/SQLServer.Schema.md)

## Links Úteis

- [README Principal](../README.md)
- [Scripts SQL](../scripts/)
```

### Passo 4: Acessar a Documentação

Após configurar, a documentação estará disponível em:
- `https://SEU-USUARIO.github.io/SEU-REPOSITORIO/`

## Opção 3: GitHub Wiki (Alternativa)

Você também pode usar o GitHub Wiki:

1. Vá em **Settings** do repositório
2. Role até **Features**
3. Habilite **Wikis**
4. Acesse a aba **Wiki** no repositório
5. Copie o conteúdo da documentação para as páginas do Wiki

## Recomendação

Para este projeto, a **Opção 1** (links no README) é a mais simples e eficiente, pois:
- ✅ Não requer configuração adicional
- ✅ GitHub renderiza Markdown automaticamente
- ✅ Fácil de manter e atualizar
- ✅ Acessível diretamente via links no README

A documentação já está linkada no README principal do projeto.
