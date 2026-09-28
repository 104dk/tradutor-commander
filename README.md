# Deck Tradutor — Commander PT-BR

Ferramenta web (arquivo único, sem build) para montar listas de **Magic: The Gathering Commander** traduzidas para **PT-BR**.

Live: https://tradutor-commander.vercel.app

## O que faz

- Busca cartas por nome no **Scryfall** (exato, fuzzy ou por query de busca).
- Carrega arte, tipos, regras e texto de flavour da carta.
- Traduz os campos para português com 3 provedores em cascata (Google gtx → MyMemory → Google alt), com **cache em `localStorage`** por carta.
- Aplica um dicionário de pós-tradução (fixes de termos que os provedores erram: *ficha*, *campo de batalha*, *desgire*, …).
- **Contador de vida** integrado (20/40/personalizado, press-and-hold, renomear jogadores, importar decks salvos).
- Exporta o deck em **PDF** com imagens e traduções (via `html2canvas` + `jsPDF`).
- Funciona offline no cache, sem backend: tudo roda no navegador.

## Estrutura

```
index.html      app completo (HTML + CSS + JS inline)
og-image.jpg    thumbnail 1200x630 usada no preview de link (WhatsApp/Discord/X)
robots.txt
vercel.json     headers de segurança da Vercel
```

O logo (`LogoMTG.png`) vai embutido no próprio HTML em base64 (favicon 64x64 + logo do cabeçalho), então o arquivo continua **autocontido** — funciona offline e dentro do app Android sem depender de imagem externa.

`index.html` é **autocontido**: só depende de CDN público (Google Fonts, html2canvas, jsPDF) e das APIs do Scryfall + tradutores. Não há etapa de build, bundler ou dependências npm.

## Desenvolvimento local

Abra `index.html` direto no navegador, ou sirva a pasta para testar-resources via HTTP:

```bash
npx serve .
```

## Deploy

Repo ligado à Vercel: qualquer push na branch `main` gera novo deploy automaticamente. Deploy manual:

```bash
vercel --prod
```

## APIs usadas

| Uso | Endpoint |
| --- | --- |
| Carta por nome | `https://api.scryfall.com/cards/named?exact=` / `?fuzzy=` |
| Busca | `https://api.scryfall.com/cards/search?q=` |
| Simbolgia | `https://api.scryfall.com/symbology` |
| Imagens | `https://svgs.scryfall.com/card-symbols/`, `https://cards.scryfall.io/` |
| Tradução | `translate.googleapis.com/translate_a/single?client=gtx` |
| Tradução (fallback) | `api.mymemory.translated.net`, `clients5.google.com/translate_a/t` |

## Observações

- As traduções ficam salvas no `localStorage` do navegador (`deckpt_*`), por isso cada usuário tem seu próprio cache.
- Os provedores públicos de tradução têm limite de requisições; o app cai automaticamente para o próximo provedor e respeita `Retry-After`.
- Sem analytics, sem cookies, sem coleta de dados.
