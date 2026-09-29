# Deck Tradutor — Commander PT-BR

Ferramenta web para montar listas de **Magic: The Gathering Commander** traduzidas para **português do Brasil**.
Cole a lista, o app busca cada carta no Scryfall, monta a versão oficial em PT quando existe, traduz automaticamente quando não existe, e exporta o deck em PDF pronto para jogar.

**No ar:** https://tradutor-commander.vercel.app
**Repositório:** https://github.com/104dk/tradutor-commander

---

## Sumário

1. [O que o app faz](#o-que-o-app-faz)
2. [Como o app funciona](#como-o-app-funciona)
3. [Detalhamento técnico](#detalhamento-técnico)
4. [Os arquivos do repositório](#os-arquivos-do-repositório)
   - [Por que `robots.txt` está aqui](#por-que-robotstxt-está-aqui)
   - [Por que `.gitattributes` está aqui](#por-que-gitattributes-está-aqui)
   - [Por que `.gitignore` está aqui](#por-que-gitignore-está-aqui)
   - [Por que `vercel.json` está aqui](#por-que-verceljson-está-aqui)
   - [Por que `og-image.jpg` está aqui](#por-que-og-imagejpg-está-aqui)
5. [Cache no navegador](#5-cache-por-navegador)
6. [App Android](#app-android)
7. [APIs externas](#apis-externas)
8. [Desenvolvimento local](#desenvolvimento-local)
9. [Deploy](#deploy)
10. [Limitações conhecidas](#limitações-conhecidas)

---

## O que o app faz

- **Lista → deck**: cole a lista no formato de qualquer exportador (Moxfield, Archidekt, planilha). Aceita `1x Carta`, `2x Carta`, `× Carta`, `# comentário`, e limpa o ruído de exportador (`Forest (UNF)`, `Sol Ring (CMM) 40`, `Swiftfoot Boots (M21) 245/280`, `(Borderless)`, `(foil)`).
- **Busca no Scryfall**: cada nome é resolvido com escada de tentativas (exato → fuzzy → busca por frase → busca sem acento → busca por token distintivo), com comparação por similaridade e suporte a cartas de duas faces (`Frente // Verso`).
- **Versão PT em duas vias**:
  - **Impressão oficial** — quando a carta tem impressão em português no Scryfall, usa nome, tipo e arte oficiais;
  - **Tradução automática** — quando não tem impressão PT, traduz o texto mantendo as palavras-chave oficiais de Magic.
- **Duas abas (Inglês / Português)** com *swipe* entre cartas no mobile.
- **PDF**: 9 cartas por página A4, com arte, tipos, regras e texto PT. Preview em modal antes de baixar. O botão **Salvar** pede o **nome do deck** antes de gerar: esse nome vira o subtítulo do PDF e o nome do arquivo em `Downloads/decks`.
- **Contador de vida**: 20 / 40 / personalizado, multiplayer, press-and-hold para somar rápido, renomear jogadores, importar ícone de mana do deck salvo, e botão de defesa (duas vidas).
- **Ranking do grupo**: placar por jogador e por deck (vitória 3, empate 1, derrota 0), com taxa de vitória e melhor deck. Registra a partida à mão ou monta a mesa direto do contador de vida (quem já levou 21 chega com a derrota marcada), e o grupo inteiro se troca entre aparelhos por um código de texto. Detalhes em [Ranking do grupo](#8-ranking-do-grupo).
- **Estatísticas do deck**: cores, tipos e curva de mana. O chip **N nomes** é clicável e abre a lista de cartas do deck com nome original, tradução disponível e a origem dela (oficial do Scryfall ou automática).
- **Decks salvos**: guarda listas no navegador para carregar rápido, e o botão **+ deck** importa um arquivo `.txt`/`.csv`/`.md` do dispositivo, usando o nome do arquivo como nome do deck.
- **App Android**: o mesmo `index.html` roda dentro de um app WebView empacotado, disponível para download aqui em cima. O botão *Baixar app* some dentro do próprio app. Detalhes em [App Android](#app-android).
- **Sem backend, sem conta, sem analytics.** Tudo roda no cliente.

---

## Como o app funciona

### Visão geral do fluxo

```
lista colada
    │
    ├─ parse da lista ────────────► itens { nome, qtd }  (deduplicado)
    │
    ├─ por item: buscar no Scryfall
    │      ├─oracle (EN): nome, tipo, mana, regras, flavour, arte
    │      └─ impressão PT: nome oficial, tipo oficial, arte
    │
    ├─ montar painel EN  (sempre o texto oficial em inglês)
    └─ montar painel PT
           ├─ tem impressão PT?  ──► nome/tipo/arte oficiais
           │                          + regras PT oficiais quando existem
           │                          senão regras traduzidas do texto EN
           └─ não tem impressão PT? ─► tudo traduzido automaticamente
                                          (com glossário de Magic preservado)
    │
    ├─ cache em localStorage (cartas e traduções separadas)
    └─ PDF / contador de vida / estatísticas / ranking
```

### 1. Parse da lista

`parseList()` (seção `// ===== Parse da lista =====`) normaliza cada linha:

- remove comentários (`#`, `//`) e linhas vazias;
- extrai a quantidade (`4x`, `× 4`, `4 Carta`) e o nome;
- **`cleanCardName()`** remove o sufixo de exportador. O cuidado é não mutilar cartas que terminam em número: `Pain 101`, `Pip-Boy 3000` e `Naturalize 2` continuam inteiras porque a remoção de dígitos só acontece **depois** de um parêntese de set code ter sido visto;
- junta linhas repetidas somando a quantidade.

### 2. Busca no Scryfall

Cada nome passa por `cardByName()` → `fetchByPtName()`, que é a parte mais delicada do app, porque o buscador do Scryfall é inconsistente com nomes acentuados em português e com cartas de token/nomes compostos. A escada é:

1. `!"Frase exata" lang:pt` — frase exata;
2. `"Frase sem acentos" lang:pt` — se a carta tem acento;
3. todos os tokens + `lang:pt`;
4. o token distintivo mais longo (pulando stop words) + `lang:pt`.

Depois os candidatos são comparados por **similaridade** (`nameSim`: interseção de palavras + distância de Levenshtein, com corte em `0.8`). `unique=cards` é usado em vez de `unique=prints` de propósito: para achar *o* nome basta 1 carta por `oracle_id`, o que deixa a resposta ~10× menor e evita baixar todas as reimpressões PT só para comparar.

Cartas de duas faces (split, transform, MDFC, adventure) chegam como `Frente // Verso`; o casamento aceita o nome inteiro **ou** qualquer uma das metades.

### 3. Montagem do painel em português

`ptViewFor()` é a **fonte única de verdade** do painel PT (é o mesmo usado pela visualização, pelo botão *Verificar* e pela exportação de PDF). A regra:

- **Tem impressão oficial PT** → nome, tipo e arte oficiais. As regras vêm do texto PT oficial quando ele existe; quando a impressão é *textless* (terrenos básicos, full-art etc., em que o Scryfall devolve só símbolos ou nada), as regras vêm da tradução automática do texto EN, e o card recebe a nota explicando isso;
- **Não tem impressão PT** → nome, tipo, regras e flavour traduzidos, com o selo `Tradução automática` e uma nota avisando que as palavras-chave oficiais foram preservadas pelo glossário e que o restante pode ter imprecisões — por isso a aba Inglês existe.

### 4. Tradução automática

A tradução é a parte que mais degrada com o tempo, então o app não confia em um único serviço:

**Cascata de provedores** (`PROVIDERS`), na ordem em que são tentados:

| Ordem | Provedor | Endpoint |
| --- | --- | --- |
| 1 | Google (gtx) | `translate.googleapis.com/translate_a/single?client=gtx` |
| 2 | MyMemory | `api.mymemory.translated.net/get` |
| 3 | Google (2ª via) | `clients5.google.com/translate_a/t?client=dict-chrome-ex` |

Cada providers só é acionado se o anterior falhar ou devolver texto vazio. Respostas `429` respeitam o `Retry-After`; erros de rede/CORS viram mensagem clara em vez de travar a tela.

**Proteções antes de enviar o texto para o tradutor** (o que evita os erros clássicos):

- **Símbolos de mana** — `{T}`, `{2/U}`, `{W/U/P}` etc. são extraídos e protegidos, traduzidos e reinseridos. Sem isso o tradutor transforma `{2/U}` em `{2/0}` ou inventa texto;
- **Glossário oficial de Magic em PT-BR** (`GLOSSARY_RAW`) — pares de palavras-chave em inglês → português oficial (*flying → voar*, *trample → atropelar*, *deathtouch → toque mortal*, …). São substituídos por marcadores antes da chamada e restaurados depois, então o tradutor nunca "melhora" um termo oficial;
- **Type line determinístico** — o tipo da carta é montado por tokenização (não pelo tradutor): tipos base, supertipos, subtipos e a classe de criatura/terreno. Só o nome do tipo passa pelo tradutor;
- **Batching** — nome e tipo saem numa única chamada quando possível, reduzindo requisições (e o consumo de cota dos provedores);
- **Terreno básico resolvido localmente** — `localReminderText()` monta o texto-lembrete em PT sem nenhuma chamada de rede; só o nome vai ao tradutor. O provedor é registrado como `glossário local`.

**Pós-tradução** (`fixupPt`): um dicionário de correções para os erros recorrentes dos provedores — *um ficha → uma ficha*, *crie um token → crie uma ficha*, *em jogo → em campo de batalha*, *custo de mana convertido → valor de mana*, *descomprima → desgire*.

**Verificação e tradução em lote**: o botão *Traduzir pendentes* processa o deck inteiro em paralelo (pool de 4) sempre preferring o cache, e o *Verificar* força a revalidação das cartas que ainda não têm tradução.

### 5. Cache (por navegador)

Cada usuário tem seu próprio cache, por isso a segunda visita ao mesmo deck é instantânea e não gasta cota de tradução.

| Chave | Conteúdo | TTL |
| --- | --- | --- |
| `deckpt_v1_<nome>` | resposta do Scryfall (EN + impressão PT) | sem expiração |
| `deckpt_auto_v4_<nome>` | tradução automática (nome, tipo, regras, flavour) + provedor usado | sem expiração |
| `deckpt_nf_v1_<nome>` | cache **negativo**: nome que o Scryfall respondeu 404 | 7 dias |
| `deckpt_symb_v1` | simbologia do Scryfall (mapa de símbolos de mana) | sem expiração |
| `deckpt_saved_decks_v1` | decks salvos | — |
| `deckpt_lc_cfg_v1` | vida inicial do contador | — |
| `deckpt_rank_v1` | ranking do grupo (jogadores, decks, histórico por carimbo de tempo) | — |

Duas decisões importantes aí dentro:

- **Somente 404 entra no cache negativo.** Falha transitória (rate limit, timeout, rede) **não** é memorizada — senão uma carta válida ficava travada por 7 dias;
- **A versão da chave é versionada** (`v1`, `auto_v4`). Quando a lógica de tradução muda, o sufixo muda e o cache antigo simplesmente é ignorado, sem migração.

### 6. PDF

`exportPdf()` monta a paginação (9 cartas por página A4) num container fora da tela, converte cada `.pdf-page` com **html2canvas** e monta o arquivo com **jsPDF** (A4 retrato, com folga de segurança e escala conforme a quantidade de páginas). As duas bibliotecas entram com `defer` — não bloqueiam a renderização da página; e como são carregadas em paralelo, o `exportPdf()` espera até 6 s por elas antes de desistir (evita o falso "biblioteca bloqueada" num clique rápido logo após o load).

### 7. Contador de vida

Modal inicial, vida inicial 20/40/personalizado, adição/subtração por toque com **press-and-hold** (repete a cada 85 ms), renomear jogador, escolher deck salvo para puxar o ícone de mana e modo de duas vidas (derrota). Estado em `deckpt_lc_cfg_v1`.

### 8. Ranking do grupo

Placar do grupo, guardado em `deckpt_rank_v1` — só no aparelho, como todo o resto do app.

**Três abas dentro do modal:**

| Aba | O que faz |
| --- | --- |
| `Ranking` | Tabela por pontos (vitória 3, empate 1, derrota 0) com V-D-E, taxa de vitória e o melhor deck. Tocar na linha abre os decks daquele jogador com o V-D-E de cada um; dá para corrigir à mão um resultado velho. |
| `Registrar partida` | Uma linha por pessoa: nome, deck (com autocompletar nos decks salvos) e o resultado — **Vitória** / **Empate** / **Derrota**. Quem não jogou a partida inteira é só desmarcar o checkbox. **Salvar resultado** grava tudo de uma vez e volta para a tabela. |
| `Grupo` | Cadastra e edita jogadores, define a **cor do deck** de cada um (as bolinhas W/U/B/R/G), e mostra o código de texto para passar o ranking inteiro para outro aparelho. |

**Entradas para o mesmo lugar:**

- o botão **Ranking** na barra de baixo;
- dentro do contador de vida, pelo menu do ≡ e pelo botão que aparece quando alguém chega a 0 de vida;
- quem vem da calculadora entra com a **mesa já montada**: os jogadores que estavam na tela vêm marcados, e quem já levou 21 de dano chega com a **derrota pré-marcada** (`rkSeedFromLifeCalc`).

**Sem servidor, então o compartilhamento é por texto.** O botão *Gerar código* produz um texto `DTR1:` com jogadores, decks e o histórico completo; o botão *Importar* cola o código de outro aparelho e **mescla** por carimbo de tempo (`t`): o mesmo jogo não é contado duas vezes quando o código é importado outra vez, e o aparelho com o histórico mais novo sobrescreve só o que é mais recente. Tudo em `localStorage`, com `try/catch` como o resto do app.

**Cores.** A cor do jogador é a cor do deck dele, e o app preenche sozinho quando o deck informado é um deck já salvo (a identidade vem do cache do Scryfall).

---

## Detalhamento técnico

### O arquivo `index.html` por dentro

O app inteiro vive em um único arquivo de ~3.470 linhas, dividido em seções marcadas no código:

| Linha | Seção | O que faz |
| --- | --- | --- |
| 205 | `/* 3a linha: Ranking ocupa 1 celula ... */` | posição do botão novo na barra de ações do mobile |
| 314 | `/* ===== Calculadora de Vida ===== */` | CSS do contador e do drawer mobile |
| 337 | `/* ===== Ranking do grupo ===== */` | CSS do modal do ranking |
| 707 | `// ===== Parse da lista =====` | normalização de linhas e limpeza de sufixo de exportador |
| 746 | `// ===== Rede (Scryfall + tradução) =====` | timeout, retry, `Retry-After`, parsing de erros |
| 1027 | `// ===== Tradução automática =====` | glossário, proteção de mana, type line, cascata de provedores |
| 1440 | `// ===== Render helpers =====` | simbologia, meta da carta, faces, escaping |
| 1593 | `// ===== Monta o painel PT =====` | impressão oficial **ou** tradução automática |
| 1626 | `// ===== Lista =====` | render da lista, filtro, seleção |
| 1702 | `// ===== Seleção (EN + PT) =====` | cards das duas abas |
| 1778 | `// ===== Verificar Tradução =====` | revalidação forçada |
| 1838 | `// ===== Traduzir cartas pendentes =====` | lote em paralelo, cache-first |
| 2001 | `// ===== Estatísticas do deck =====` | cores, tipos, curva e a lista de cartas |
| 2190 | `// ===== Salvar Deck (PDF) =====` | modal do nome, paginação, html2canvas, jsPDF |
| 2367 | `// ===== Prévia do PDF =====` | modal de preview |
| 2399 | `// ===== Decks salvos =====` | persistência e menu rápido |
| 2458 | `// ===== Importar deck de um arquivo =====` | leitura do arquivo, nome do deck, carga automática |
| 2504 | `// ===== Mobile: abas EN/PT + swipe =====` | navegação por toque |
| 2531 | `// ===== Events =====` | listeners |
| 2668 | `// ===== Calculadora de Vida =====` | lógica do contador |
| 2974 | `// ===== Ranking do grupo =====` | placar do grupo: estado, tabelas, código de troca |

### Decisões de arquitetura

- **Sem build, sem bundler, sem `package.json`.** Um HTML, três CDNs. Isso mantém o arquivo compartilhável: dá para mandar o `index.html` no WhatsApp, abrir do `file://` ou embutir no app Android sem pipeline.
- **Logo embutido em base64** (`LogoMTG.png`): favicon 64×64 e logo do cabeçalho. Arquivo externo quebraria o uso offline e o app Android.
- **Todas as texturas de string passam por `esc()`** — nomes de carta vêm de API externa e são injetados via `innerHTML`.
- **Toda chamada de rede passa por `fetchWithTimeout()`** (18 s) + `fetchJson()` (4 tentativas com backoff).
- **Escapamento de nome em atributos HTML** ao montar as linhas da lista (`data-raw`).
- **Acessibilidade**: `lang="pt-BR"`, foco visível, `aria-expanded` no burger do contador, alvos de toque ≥ 44 px no mobile, `viewport-fit=cover` com `env(safe-area-inset-*)`.
- **Tema escuro fixo** com as variáveis em `:root` (`--gold`, `--ink-*`, `--parchment`…).

---

## Os arquivos do repositório

| Arquivo | Tamanho | Papel |
| --- | --- | --- |
| `index.html` | 273 KB | **o app inteiro** — HTML, CSS e JS inline |
| `DeckTradutor.apk` | 684 KB | app Android assinado, servido direto pelo link *Baixar app* |
| `og-image.jpg` | 62 KB | thumbnail 1200×630 do preview de link |
| `vercel.json` | 576 B | headers de segurança do site e do APK |
| `README.md` | — | esta documentação |
| `robots.txt` | 23 B | política de rastreamento |
| `.gitignore` | 226 B | não versiona artefatos locais |
| `.gitattributes` | 202 B | normaliza finais de linha e marca binários |

### Por que `robots.txt` está aqui

É a **política de rastreamento** do site, no formato padrão de um arquivo por diretório na raiz do domínio.

O conteúdo atual é o mais permissivo possível:

```
User-agent: *
Allow: /
```

Ou seja: libera explicitamente todos os robôs em todas as páginas. O motivo de o arquivo existir mesmo assim é:

1. **Declaração de intenção.** Sem ele, o padrão de qualquer buscador já é permitir tudo — o arquivo não muda o comportamento *hoje*. Ele existe para deixar isso **explícito** e para ser o ponto único onde a política vai ser editada no dia em que for preciso liberar tudo menos alguma coisa;
2. **Ponto de extensão.** É o lugar natural para receber um `Sitemap:` quando houver uma página de listagem por carta, o que faria o site aparecer em busca muito mais rápido;
3. **Zero custo e zero risco.** 23 bytes, servido com `text/plain`, não afeta a performance nem pode quebrar o site.

Ele é referenciado por **nada** no `index.html` — é lido direto por buscadores no momento do rastreio, por isso a remoção seria inofensiva, e é justamente por ser barato e útil mais adiante que ele fica versionado.

### Por que `.gitattributes` está aqui

O arquivo mais curto do repo resolve dois problemas de quem desenvolve no Windows:

```
* text=auto eol=lf
*.html text eol=lf
*.json text eol=lf
*.txt text eol=lf
*.md text eol=lf

*.apk     binary
*.jks     binary
*.keystore binary
```

A primeira parte é sobre **texto**: sem ela, o Git no Windows converte as quebras de linha para **CRLF** ao tocar o arquivo, e produz o aviso `LF will be replaced by CRLF` em cada `git add`. Isso importa por dois motivos:

1. **Ruído constante.** O `index.html` é um arquivo gigante de uma linha por elemento; qualquer diff fica mais difícil de ler quando o fim de linha muda junto;
2. **Previsibilidade entre máquinas.** Quem edita no Linux/macOS e quem edita no Windows passam a ver o mesmo conteúdo, com LF no repositório, o que evita conflitos de linha inteira em commits seguintes.

Efeito colateral útil: como a regra é `eol=lf`, o HTML é gravado com LF no disco também — que é o formato esperado pelos navegadores e pelo GitHub.

A segunda parte é o **exato oposto**, e é obrigatória: `*.apk binary` desliga a normalização para o APK. Um APK é um ZIP com assinatura — se o Git normalizar alguma quebra de linha dentro dele, o arquivo sobe com checksum diferente do que a máquina gerou, a assinatura deixa de validar e o Android recusa a instalação. As regras `*.jks` e `*.keystore` existem pelo mesmo motivo, caso a chave de assinatura um dia seja versionada (a deste projeto **não** é: ela fica no `.gitignore` do projeto Android).

### Por que `.gitignore` está aqui

O projeto **não tem** `package.json`, build nem dependências — então não há `node_modules` para ignorar. O que ele protege é o que a CLI da Vercel cria na pasta quando alguém roda um deploy local:

```
node_modules/   .vercel/   dist/   build/
*.log           .DS_Store  Thumbs.db  .vscode/  .idea/
.env  .env.local  .env*.local
```

O item crítico é **`.vercel/`**: ele já existe nesta pasta (criado pelo `vercel link`), e sem a regra um `git add -A` poderia versionar `.vercel/project.json`. A lista de ambiente (`.env*`) é precaução padrão: se um dia alguém testar uma chave de API no navegador, ela não entra no repo por acidente.

### Por que `vercel.json` está aqui

É o único lugar do projeto que configura o **servidor** — e ele não é decoração. Publica três headers de segurança em todas as respostas:

```json
{ "key": "X-Content-Type-Options", "value": "nosniff" }
{ "key": "Referrer-Policy",      "value": "strict-origin-when-cross-origin" }
{ "key": "X-Frame-Options",      "value": "SAMEORIGIN" }
```

| Header | O que protege |
| --- | --- |
| `X-Content-Type-Options: nosniff` | impede o navegador de adivinhar o tipo de uma resposta e executá-la como script |
| `Referrer-Policy: strict-origin-when-cross-origin` | ao clicar num link externo, só o domínio (não o caminho da URL) é enviado |
| `X-Frame-Options: SAMEORIGIN` | impede que o site seja embutido num `<iframe>` em outro domínio (clickjacking) |

Isso foi verificado: outros projetos Vercel sem `vercel.json` **não** devolvem nenhum desses headers. Ou seja, se o arquivo sair do repo, os três headers somem junto com ele.

O mesmo arquivo define `"cleanUrls": true` e `"trailingSlash": false`, para `/index.html` responder como `/`.

E tem uma **quarta regra, só para o APK** (`/DeckTradutor.apk`), que precisa de headers próprios porque o arquivo é binário e baixado, não exibido:

```json
{ "key": "Content-Type",        "value": "application/vnd.android.package-archive" }
{ "key": "Content-Disposition", "value": "attachment; filename=\"DeckTradutor.apk\"" }
{ "key": "Cache-Control",       "value": "public, max-age=3600, must-revalidate" }
```

O `Content-Type` explícito evita depender da detecção por extensão, e o `Content-Disposition: attachment` garante o download em vez de qualquer tentativa de exibição inline. O `Cache-Control` é propositalmente curto e com `must-revalidate`: o link já carrega `?v=<versão>` para furar o cache, então não há ganho em guardar o APK por um ano — e um binário republicado com a mesma versão ainda chega rápido.

### Por que `og-image.jpg` está aqui

Quando alguém cola o link do site no WhatsApp, Discord, X ou Telegram, esses apps leem as meta tags `og:*` para montar o cartão de preview. Este arquivo é a imagem que aparece no cartão:

```
og:image          → https://tradutor-commander.vercel.app/og-image.jpg
og:image:width    → 1200
og:image:height   → 630
twitter:card      → summary_large_image
```

**Por que precisa ser um arquivo real e não base64 inline:** os scrapers de preview são servidores externos que abrem a URL e baixam a imagem. Um `data:` URI dentro do HTML não é acessível para eles. Já o favicon e o logo do cabeçalho são base64 porque só precisam ser lidos pelo navegador, que já tem o HTML em mãos.

A imagem (1200×630, JPEG, 62 KB) é gerada a partir do `LogoMTG.png` com a mesma identidade visual do site: fundo escuro em gradiente, barra de cores MTG no topo, logo à esquerda e os textos *Deck Tradutor*, *Commander | EN → PT-BR* e a descrição das funcionalidades.

---

## App Android

O `DeckTradutor.apk` deste repositório é o **mesmo `index.html` do site**, empacotado num app Android com WebView. Não existe uma segunda cópia do frontend: o Gradle copia o `index.html` para `app/src/main/assets/www/` antes de cada build e falha se a pasta `tradutor-commander-web` não estiver no lugar.

O projeto Android fica **fora** deste repositório, na pasta `Magic/tradutor-apk` do monorepo.

| Item | Valor |
| --- | --- |
| Package | `br.com.tradutordeck.app` |
| Versão | 1.2.0 (`versionCode` 4) |
| Requisitos | Android 6.0 (API 23) ou superior |
| Permissões | apenas `INTERNET` |
| Assinatura | v1 + v2 + v3 |

**Por que o app não carrega `file://`:** a partir do Android 11 (`targetSdk >= 30`) o WebView bloqueia `fetch`/XHR de páginas `file://` para origem externa, e o app depende de `fetch` para o Scryfall e para os provedores de tradução. O `WebViewAssetLoader` serve os assets em `https://appassets.androidplatform.net`, que é uma origem HTTPS de verdade — mesmo comportamento do site, `localStorage` funcionando e o canvas do PDF sem *tainting*.

**O que o Java acrescenta** (o resto é o HTML puro):

- **Importar deck** — o WebView ignora `<input type="file">` sem `onShowFileChooser`, então o app implementa o callback e abre o seletor do sistema já em *Downloads*;
- **Salvar o PDF** — o jsPDF gera um `blob:`, que o `DownloadManager` não entende. O app lê o blob dentro da página, recebe em base64 pelo bridge `DeckNative` e grava em **`Downloads/decks`** via `MediaStore` (Android 10+) ou na pasta externa do app (Android 9 e anteriores);
- **Nome do arquivo** — o nome escolhido no modal do site chega em `window.__deckPdfName` e é sanitizado com a mesma regra do `pdfFileName()` do HTML, então o arquivo sai igual nos dois lugares;
- **Botão *Baixar app*** — escondido dentro do app pelo User-Agent `DeckTradutor/`, que o próprio HTML detecta;
- **Pastas do app** — `decks` e `deckpdf` são criadas em `getExternalFilesDir(null)` na primeira execução: somem ao desinstalar e não pedem permissão;
- **Botão voltar** — fecha o modal do nome, recolhe a lista de nomes, e só então fecha a pré-visualização, as estatísticas e o app, nessa ordem.

**Publicar uma versão nova:** subir o `versionName` em `app/build.gradle`, atualizar o `?v=` no `href` do botão *Baixar app* (o build falha se os dois divergirem), rodar `gradlew assembleRelease` e substituir o `DeckTradutor.apk` deste repositório.

---

## APIs externas

O app não tem backend: consome APIs públicas direto do navegador.

| Uso | Endpoint |
| --- | --- |
| Carta por nome (EN) | `https://api.scryfall.com/cards/named?exact=` · `?fuzzy=` |
| Busca (inclusive impressões PT) | `https://api.scryfall.com/cards/search?q=` |
| Simbolgia de mana | `https://api.scryfall.com/symbology` |
| Símbolos de mana em SVG | `https://svgs.scryfall.com/card-symbols/` |
| Imagens de carta | `https://cards.scryfall.io/` |
| Tradução (1ª opção) | `https://translate.googleapis.com/translate_a/single?client=gtx` |
| Tradução (2ª opção) | `https://api.mymemory.translated.net/get` |
| Tradução (3ª opção) | `https://clients5.google.com/translate_a/t?client=dict-chrome-ex` |
| Bibliotecas de PDF | `cdnjs.cloudflare.com` (html2canvas 1.4.1, jsPDF 2.5.1) |
| Fontes | Fraunces + Inter, via `fonts.googleapis.com` |

---

## Desenvolvimento local

Não há build. Para desenvolver, basta abrir o arquivo:

```bash
# abrir direto
start index.html

# ou servir por HTTP (recomendado: evita diferença de file:// em some APIs)
npx serve .
```

Antes de publicar, o arquivo passa por estas verificações:

```bash
node --check      # sintaxe do JS inline
```

## Deploy

O repositório está ligado à Vercel: **qualquer push na branch `main` gera deploy de produção automaticamente** em `https://tradutor-commander.vercel.app`.

```bash
git add -A
git commit -m "..."
git push            # deploy automático

vercel --prod        # deploy manual, se necessário
```

### Ao mexer no app, cuidado com

- **`esc()` em qualquer texto vindo de API** — nomes de carta são injetados via `innerHTML`;
- **ids**: todo `$("id")` no JS precisa de um `id="..."` correspondente no HTML (checagem automatizada: extrair os ids usados e comparar com os definidos);
- **tag `<script>` inline**: o JS é um bloco só, no fim do `<body>`; erro de sintaxe quebra a página inteira — rode `node --check`;
- **cache**: ao mudar a lógica de tradução, suba o sufixo de `autoKey` (`deckpt_auto_v4_` → `v5`) para invalidar o cache antigo;
- **delegação de evento no modal do ranking**: um único listener no `#rkModal` trata tudo por `data-act`, e a linha do jogador (`.rk-row`, que abre os decks) é testada **antes** do bloco de ações — depois do `if(!act) return` ela nunca chegaria a ser tratada;
- **grade da barra de ações no mobile**: `grid-area` não é herdável, então cada variação (`#dlApp`, `#rankBtn`, `calcLifeBtn` sem o link do APK) declara a sua;
- **localStorage**: `setItem` sempre dentro de `try/catch` (Safari em modo privado e quotas cheias lançam exceção).

---

## Limitações conhecidas

- **Tradução automática é automática mesmo.** As palavras-chave oficiais são preservadas pelo glossário e o texto legal de cartas complexas pode soar estranho. A aba Inglês está sempre disponível para conferir, e o badge no card avisa quando o texto é traduzido;
- **Provedores públicos de tradução têm cota.** O app respeita `Retry-After` e cai para o próximo provedor, mas em uso intenso pode ser preciso recarregar;
- **Nomes em português podem falhar no Scryfall.** A busca em PT é uma escada de tentativas; nomes muito(token) incomuns às vezes exigem o nome em inglês;
- **Cache é por navegador.** Trocar de máquina ou limpar dados do site significa traduzir de novo;
- **O ranking também é por aparelho.** Não há servidor: para levar o placar do grupo para outro celular é preciso exportar e importar o código de texto da aba *Grupo* (ou registrar as mesmas partidas nos dois).
- **Sem imagem offline para as cartas** além do cache: para uso 100% offline, o PDF já gerado é a melhor saída.
