# bsk-blog

Repositório do meu blog pessoal, construído com **Hugo** e tema baseado no ecossistema do **Hextra**.

A ideia aqui é registrar projetos, estudos e aprendizados. Também funciona como um “backup de conhecimento” para consultas futuras.

## Desenvolvimento local

### Pré-requisitos (instalação local)

- **Hugo Extended**
- **Go**
- **Git**

> Observação: se você estiver usando **Hugo Modules** (ou `_vendor`), o Go é necessário para resolver dependências.

### Rodando localmente

```bash
# clone
git clone https://github.com/bsk-iT/bsk-blog.git
cd bsk-blog

# (opcional) caso use Hugo Modules e queira garantir dependências ok
hugo mod tidy

# rodar servidor
hugo server --logLevel debug --disableFastRender -p 1313
```

Acesse: <http://localhost:1313>

## Publicação (GitHub Pages)

Este repositório publica via **GitHub Actions** usando o workflow:

- `.github/workflows/pages.yaml`

Requisito: em **Settings → Pages**, a fonte de deploy deve estar como **GitHub Actions**.

## Estrutura (visão geral)

- `content/` → posts e páginas (Markdown)
- `layouts/` → overrides de templates
- `assets/` → CSS/JS/imagens
- `hugo.yaml` (ou `config/`) → configuração do Hugo
- `_vendor/` (se aplicável) → dependências vendorizadas do Hugo Modules

## Contribuições

Sugestões, correções e pequenas melhorias são bem-vindas.

- Abra uma issue descrevendo a proposta (recomendado).
- Para PRs: mantenha mudanças pequenas e objetivas.
- Para mudanças maiores: alinhe antes via issue.

## Contato

- Email: [bruno.cubero@proton.me](mailto:bruno.cubero@proton.me)

## Licença

- **Código e configuração do projeto:** MIT License (ver [`LICENSE`](./LICENSE)).
- **Conteúdo autoral do blog (posts/páginas e mídias originais):**  
  [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/)  
  (ver [`LICENSE-CONTENT`](./LICENSE-CONTENT)).

Shield (conteúdo): [![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)
