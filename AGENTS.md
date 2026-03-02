# AGENTS.md — bsk-blog

Instruções para agentes de IA que operam neste repositório.

---

## Visão Geral do Projeto

Blog pessoal **berserk.iT**, gerado estaticamente com [Hugo](https://gohugo.io/) (Extended).
Tema: [Hextra](https://github.com/imfing/hextra) v0.11.1, vendorizado em `_vendor/`.
Idioma principal do conteúdo: Português Brasileiro (`pt-BR`).

```
_vendor/                  # Tema Hextra (NÃO editar diretamente)
.github/workflows/        # CI/CD — GitHub Actions → GitHub Pages
content/                  # Todo o conteúdo do blog (Markdown)
  _index.md               # Página inicial
  about.md                # Página "About"
  2026/01/10/slug/        # Posts organizados por ano/mês/dia/slug/
layouts/                  # Overrides de templates Hugo (prioridade sobre o tema)
  partials/components/    # Ex.: comments.html (Disqus)
  partials/custom/        # Hooks do Hextra (ex.: head-end.html)
i18n/                     # Sobrescritas de strings de i18n
hugo.yaml                 # Configuração principal do Hugo
go.mod / go.sum           # Dependências do módulo Hugo
netlify.toml              # Deploy secundário (Netlify)
```

---

## Requisitos de Ambiente

- **Hugo Extended** ≥ 0.146.0 (CI usa 0.147.9); **Go** ≥ 1.21 (CI usa 1.24).
- Node/npm **não é necessário** para desenvolver o blog.

```bash
# Instalar Hugo Extended no Linux/WSL
wget -O hugo.deb https://github.com/gohugoio/hugo/releases/download/v0.147.9/hugo_extended_0.147.9_linux-amd64.deb
sudo dpkg -i hugo.deb
```

---

## Comandos de Build e Desenvolvimento

```bash
hugo server --logLevel debug --disableFastRender -p 1313  # servidor local
hugo --gc --minify                                         # build de produção
hugo mod tidy                                              # sincronizar módulos
hugo mod vendor                                            # re-vendorizar tema
```

---

## Testes

**Não há testes automatizados.** Verificação é feita manualmente:

1. `hugo server --disableFastRender -p 1313` — inicia o servidor local.
2. Abre `http://localhost:1313` e inspeciona visualmente.
3. `hugo --gc --minify` deve terminar **sem erros** como verificação de build.

Qualquer alteração que faça o build falhar é considerada uma regressão.

---

## Linting e Formatação

Não há ferramentas de lint configuradas na raiz do projeto. Diretrizes:

- **YAML** (`hugo.yaml`, workflows): indentação de 2 espaços; sem tabs.
- **TOML** (`netlify.toml`): indentação de 2 espaços.
- **Templates Hugo** (`.html` em `layouts/`): indentação de 2 espaços; controle de
  espaço em branco com `{{-` e `-}}`.
- **Markdown** (`content/`): uma linha em branco após o bloco de front matter;
  sem trailing whitespace.
- **CSS** (em `head-end.html`): comentários em português; propriedades CSS modernas
  (`inset`, `aspect-ratio`, `overflow-wrap`).

Prettier está disponível no tema vendorizado mas **não deve ser executado** sobre
os arquivos do blog — apenas sobre os arquivos internos do tema, e somente por
mantenedores do tema.

---

## Criação de Conteúdo (Markdown)

### Estrutura de diretórios

Posts ficam em `content/YYYY/MM/DD/slug/index.md` — cada post é um **diretório**
com um único arquivo `index.md` dentro:

```
content/
  2026/
    01/
      10/
        meu-novo-post/
          index.md
```

O caminho `YYYY/MM/DD/slug/` determina a URL final do post.
Atualize `content/_index.md` para listar o novo post na página inicial.

### Front matter obrigatório

```yaml
---
title: "Título do Post"
slug: "titulo-do-post"
date: 2026-03-01T14:00:00-03:00
draft: false
tags:
  - Linux
  - Homelab
---
```

Campos adicionais suportados pelo Hextra:

```yaml
description: "Resumo breve para SEO e listagens."
draft: true      # Omite o post do build de produção
math: true       # Ativa KaTeX/MathJax
mermaid: true    # Ativa diagramas Mermaid
```

Observações:
- `title` e `slug` entre aspas duplas.
- `date` usa formato ISO 8601 com timezone, sem aspas (`-03:00` para BRT).
- `slug` deve coincidir com o nome do diretório pai do `index.md`.
- `tags` podem ter maiúsculas (ex: `AI`, `Rust`, `Linux`).
- `type: blog` **não deve ser usado** — o lookup de template é feito via `layouts/_default/single.html`, que cobre todos os posts independente da seção.
- `categories` não é utilizado nos posts existentes; preferir `tags`.

### Convenções de conteúdo

- Idioma: Português Brasileiro. Termos técnicos podem permanecer em inglês.
- Tom: direto, técnico, informal sem ser desleixado.
- Imagens: adicionar em `static/images/` e referenciar com caminho absoluto
  (`/images/nome.png`).
- Vídeos incorporados: usar o shortcode `{{< youtube id="..." >}}` ou envolver
  `<iframe>` na classe `.embed-container` (já definida em `head-end.html`).
- Código: usar fenced code blocks com identificador de linguagem:

  ````markdown
  ```bash
  comando aqui
  ```
  ````

---

## Convenções de Templates Hugo

Os overrides do site ficam em `layouts/` e têm **prioridade** sobre os templates
do tema em `_vendor/`.

- Controle de whitespace: usar `{{-` e `-}}` para evitar linhas em branco indesejadas.
- Variáveis locais: `$camelCase` (ex.: `$readMore`, `$pageTitle`).
- Parâmetros para partials: sempre usar `dict` nomeado:
  `{{- partial "components/foo.html" (dict "Page" . "param" $valor) -}}`
- Valores opcionais: preferir `with` em vez de `if` para checar nil/empty.
- Comentários internos: `{{- /* comentário */ -}}`; `<!-- -->` só se deve aparecer no HTML.
- Indentação: 2 espaços.

Antes de criar um novo override, verificar se o Hextra já expõe um hook em
`_vendor/github.com/imfing/hextra/layouts/partials/custom/` — usar o hook é
preferível a reescrever o template inteiro.

---

## CSS e Estilização

O CSS de produção **já está pré-compilado** pelo tema em:
`_vendor/github.com/imfing/hextra/assets/css/compiled/main.css`

**Não editar** esse arquivo nem tentar recompilar o CSS do tema sem necessidade.

Para customizações visuais do blog:
- Adicionar regras em `layouts/partials/custom/head-end.html` dentro do bloco
  `<style>` existente.
- O Hextra usa Tailwind CSS v4 com o prefixo `hx:` — classes `hx:flex`,
  `hx:text-sm`, etc. — visíveis nos templates do tema.
- Não adicionar classes Tailwind diretamente em posts Markdown (o CSS do Tailwind
  não inclui todas as utilitárias; apenas o que o tema usa está disponível).

## Configuração (`hugo.yaml`)

`hugo.yaml` é a fonte única de verdade. Alterações comuns:

| Campo | Finalidade |
|---|---|
| `title` | Nome do site (navbar e `<title>`) |
| `params.description` | Meta description global (SEO) |
| `params.author` | Nome e e-mail do autor |
| `menu.main` | Itens de navegação (ordem via `weight`) |
| `services.disqus.shortname` | Identificador do Disqus |
| `params.theme.default` | Tema padrão: `light` ou `dark` |

Não duplicar chaves em `hugo.yaml` — Hugo usa a última ocorrência, o que gera
comportamento confuso.

---

## Módulos Hugo / Tema

O tema é gerenciado via Hugo Modules (não `git submodule`):

```bash
# Ver módulos em uso
hugo mod graph

# Atualizar Hextra para versão mais recente
# 1. Editar go.mod — alterar a versão de hextra
# 2. hugo mod tidy
# 3. hugo mod vendor   ← re-gera _vendor/
# 4. Testar: hugo server
```

**Nunca editar arquivos dentro de `_vendor/` diretamente.** Alterações serão
sobrescritas na próxima execução de `hugo mod vendor`. Se precisar mudar o
comportamento do tema, criar um override em `layouts/`.

## Deploy

| Alvo | Trigger | Comando |
|---|---|---|
| GitHub Pages (primário) | Push para `main` | `hugo --gc --minify --baseURL <url>` |
| Netlify (secundário) | Push / PR | `hugo --gc --minify -b ${DEPLOY_PRIME_URL}` |

O workflow `.github/workflows/pages.yaml` cuida do build e deploy automaticamente.
Não é necessário fazer push do diretório `public/` — ele está no `.gitignore`.
