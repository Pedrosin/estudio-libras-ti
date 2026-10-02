# Documentação técnica — Estúdio Libras · TI

Como a página funciona por dentro, como mudar os sinais e como testar e publicar. Para o uso, veja o [README](README.md).

## Arquivos

| Arquivo | Para quê |
|---|---|
| `estudio-libras-ti.html` | A aplicação inteira: HTML, CSS e JS num arquivo só, sem dependências e sem build. |
| `index.html` | Redireciona para `estudio-libras-ti.html` (é o que o GitHub Pages abre na raiz). |
| `roteiro-libras-ti.md` | Roteiro de gravação. É a saída de `gerarRoteiro()` com o estado padrão; regenere se mudar a lista. |
| `README.md` / `DOCS.md` | Uso e esta documentação. |
| `_versao-1/` | Backup da primeira versão. Fica só no PC (está no `.gitignore`). |

## Onde roda

- **Local:** dois cliques no `.html` abre por `file://` no Chrome ou Edge. Câmera e gravação funcionam (`file://` conta como contexto seguro).
- **Online:** https://pedrosin.github.io/estudio-libras-ti/ (HTTPS, câmera funciona).
- **Internet obrigatória:** os vídeos dos sinais tocam direto dos servidores das fontes. Nada é baixado nem enviado.
- **Estado salvo por endereço:** o progresso fica no `localStorage` do navegador, e cada endereço (`file://` e o do Pages) tem o seu. O que você marca num não aparece no outro.

## Estrutura do `estudio-libras-ti.html`

Em ordem no arquivo:

1. **Script de tema no `<head>`:** roda antes do primeiro desenho e grava `data-tema="claro|escuro"` no `<html>`. "Automático" segue o Windows (`prefers-color-scheme`) e reage quando ele muda. A escolha fica na chave `estudio-libras-ti-tema`. O clique no seletor (`.tema button[data-tema]`) é tratado aqui mesmo.
2. **CSS:**
   - **Tokens** em `:root` (tema claro) e `:root[data-tema="escuro"]`. Toda cor passa por variável; não há `@media (prefers-color-scheme)` no resto do CSS.
   - **Materiais:**
     - `.vidro`: translúcido, com `backdrop-filter`. Só em navegação e controles.
     - `.painel-mat`: quase opaco. Para tudo que se lê.
   - **Fundo Aero (`.ceu`):** gradiente do céu, SVG com bokeh, `.aurora` com 3 faixas de luz e 12 `.bolha`. Tudo anima só por `transform` e pausa com `body.cam-on` ou `body.rodando`.
   - **Botões de gel** (`.primary`, `.rec`, `.ok` em `button`/`a.botao`): reflexo por `::before` atrás do texto (`isolation: isolate` + `z-index: -1`).
   - **Responsivo:** cortes em 1199, 1099, 979, 859 e 639 px. Abaixo de 980 a lista vira faixa horizontal; abaixo de 640 as abas vão para o rodapé.
   - **Acessibilidade:**
     - `prefers-reduced-transparency`: vidro opaco, sem céu, aurora nem bolhas.
     - `prefers-reduced-motion`: sem animação.
     - Foco por `outline`.
3. **Markup:** `.ceu` (fundo), `.janela` > `header.barra` (seletor de tema, marca, abas, progresso) e `main` com as três abas: `#p-estudar`, `#p-ficha`, `#p-gravar`.
4. **Script principal**, nesta ordem:
   - dados;
   - estado;
   - `Player`;
   - aba Estudar;
   - câmera;
   - aba Ficha;
   - aba Gravar;
   - abas e atalhos;
   - `init()`.

## Dados dos sinais

### `FONTES`

Uma entrada por fonte, com `nome` (longo), `curto` (tabela) e `url`. Hoje são 4:

| Chave | Fonte | Prefixo do vídeo |
|---|---|---|
| `ines` | Dicionário de Libras do INES | `V_INES` = `https://dicionario.ines.gov.br/public/media/palavras/videos/` |
| `sts` | Spread the Sign, Libras do Brasil | `V_STS` = `https://media.spreadthesign.com/video/mp4/14/` (o `14` é o Brasil) |
| `ifba` | Glossário Infolibras, IFBA Seabra | `V_IFBA` = `https://infolibras.ifbaseabra.edu.br/storage/img/` |
| `utfpr` | Sinais na Libras – Área de Tecnologia (UTFPR) | `V_UTFPR` = `https://tecnalibras.github.io/assets/video/` |

### `SINAIS`

Objeto com uma chave por sinal. Campos:

| Campo | Obrigatório | Uso |
|---|---|---|
| `termo` | sim | Nome em maiúsculas: lista, Ficha, prompter. |
| `fonte` | sim | Chave de `FONTES`. |
| `verbete` | sim | Como o termo aparece na fonte. No INES, é o que se digita na busca (ex.: `REDE3`). |
| `video` | sim | URL do `.mp4`. Precisa ser mp4 direto (veja "Armadilhas"). |
| `acepcao` | sim | Sentido conferido na fonte; mostrado no Estudar e na Ficha. |
| `pagina` | não | Link "Abrir na fonte". Sem ele, abre a raiz da fonte. |
| `credito` | não | Quem criou o sinal, quando a fonte é um glossário que agrega (IFBA). |
| `nota` | não | Aviso em amarelo no Estudar (homônimo, soletração, variação). |
| `trecho` | não | `[início, fim]` em segundos. O player repete só esse pedaço quando "Só o sinal" está ligado. |
| `legenda` | não | Texto gravado no vídeo, se diferente de `termo`. Ex.: `CIBERSEGURANÇA (segurança da informação)`. O prompter tira o parêntese. |
| `equivale` | não | Termo do roteiro original que este sinal cobre (não conta como troca na Ficha nem no roteiro). |

As letras do alfabeto manual são geradas sob demanda por `letra('V')`, que cria a chave `lV` apontando para o vídeo da letra no INES.

### Listas

- **`PADRAO`:** os 10 da lista sugerida, na ordem do vídeo.
- **`ORIGINAL`:** os 10 termos do roteiro original, na mesma posição. Serve para marcar "no lugar de …" e listar as trocas.
- **`RESERVA`:** opções de troca, nesta ordem no seletor. As que dependem de soletração (`rede`, `senha`) ficam por último.
- **`APRESENTACAO`:** os itens da apresentação no Estudar.
  - **"Frase completa" e "V-I-C-T-O-R":** usam `seq` (lista de chaves tocadas em sequência). `sincronizarNome()` refaz as letras a partir do campo Nome da aba Gravar.
  - **Demais itens:** um sinal cada (`id`).
- **`FRASE_GLOSA`:** a glosa sugerida para a apresentação.
- **`CHECAGEM`:** itens do checklist antes do envio.

## Como trocar ou acrescentar um sinal

1. **Confira na fonte o sinal certo para o sentido de TI.** O sinal segue o sentido, não a palavra: no INES, BANCO é assento ou banco de dinheiro, e DADO é o dado de jogar.
2. **Pegue a URL do mp4:**
   - **INES:** na página, `window.palavras` tem os 5.826 verbetes (campos `palavra`, `descricao`, `libras`, `video`). Monte `V_INES + video`.
   - **Spread the Sign:** abra a página do verbete em `/pt.br/word/...` e copie o `src` do `<video>`. Confira que a página diz "português (Brasil)".
   - **IFBA:** o `src` do `<video>` do cartão na categoria (`/sinais?categoria_id=1..4`).
   - **UTFPR:** `V_UTFPR + nome.mp4`.
3. **Crie a entrada em `SINAIS`** com os campos obrigatórios e uma `acepcao` copiada da fonte.
4. **Se o vídeo soletra a palavra antes do sinal (todo o IFBA faz isso):**
   - meça `trecho` olhando quadros a cada 0,25 s;
   - comece logo depois da última letra e termine quando as mãos voltam ao repouso;
   - se o vídeo repetir o sinal, use só uma execução;
   - ponha `nota: NOTA_IFBA`.
5. **Ponha a chave em `RESERVA`** (aparece para troca) ou em `PADRAO`, na posição certa. Se mexer em `PADRAO`, ajuste também `ORIGINAL` e o cartão "Por que o roteiro mudou" na Ficha.
6. **Fonte nova:** acrescente em `FONTES` e confira `textoFontes()`, que monta a lista "Fontes consultadas" por fonte.
7. **Se a lista padrão mudou:** regenere `roteiro-libras-ti.md` com a saída de `__estudio.gerarRoteiro()` no estado padrão.

> Mudou `PADRAO`? Estados salvos com uma lista que não existe mais caem no padrão sozinhos (`slotsValidos`). Nome, checklist e "já domino" são mantidos, e aparece um aviso.

## Estado salvo

- **Chave `estudio-libras-ti-v2`** (`CHAVE_LS`):
  - **Lista:** `slots` (os 10 escolhidos).
  - **Progresso:**
    - `dom`: já domino.
    - `chk`: 5 parâmetros por sinal.
    - `aula`: "já apareceu nas videoaulas".
    - `envio`: checklist.
  - **Estudo:** `vel`, `trecho`, `atual`, `dicaOk`.
  - **Gravação:** `nome`, `curso`, `avanco` (`auto`/`manual`), `segIntro`, `segTermo`, `segFim`, `pos` (`topo`/`baixo`), `audio`, `canto`.
- **Migração:** se não houver v2, `carregarEstado()` importa a chave `estudio-libras-ti-v1`. A lista padrão antiga, que tinha SENHA no lugar 10, vira a nova.
- **Tema:** chave separada, `estudio-libras-ti-tema`.
- **Leitura e escrita protegidas:** todo acesso ao `localStorage` está em `try/catch`, então a página funciona sem ele (aba anônima, armazenamento bloqueado).

## Aba Estudar

- **`Player`:** envolve um `<video>`.
  - **Repetição:** um item com `seq` toca as chaves em sequência e recomeça. Um item com `trecho` repete só aquele pedaço. A checagem roda em `requestAnimationFrame` e também em `timeupdate`, para não escapar quando a aba fica em segundo plano.
  - **Velocidade:** é reaplicada a cada `loadedmetadata`, porque o navegador zera a taxa ao trocar de vídeo.
  - **Pausa:** `pausar()` e `retomar()` controlam a flag `ativo`, que impede o vídeo de voltar a tocar sozinho depois de trocar de aba.
  - **Instâncias:** são duas, `refPlayer` (Estudar) e `cantoPlayer` (miniatura na gravação).
- **Webcam:** é um `<video class="camfeed espelho">` espelhado só por CSS (`scaleX(-1)`).
- **"Já domino":** chama `comemorar()`, que solta bolhas (`.festa`) a partir do botão e passa um brilho pela pastilha de progresso. Ao completar os 10, a leva é maior.

## Aba Gravar — como o vídeo é feito

1. **Início:** `comecar(gravar)` liga a câmera (e o microfone, se marcado), monta os passos com `montarPassos()` e faz a contagem de 3 s. Os passos são: apresentação, os 10 termos e o encerramento.
   - Uma trava (`comecando`) evita duas gravações com clique duplo.
2. **Gravador:** `iniciarGravador()` cria um `<canvas>` fora da tela do tamanho da câmera, captura com `captureStream(0)` e grava com `MediaRecorder` (6 Mbps).
   - **Formato:** MP4 (H.264/AAC) quando o navegador suporta; senão WebM.
3. **Cada quadro:** `desenharQuadro()` copia o quadro da câmera sem espelho, desenha a legenda do passo atual (`desenharLegenda`) e empurra o quadro com `requestFrame()`.
   - **Quem dispara:** um Worker com `setInterval` a 30 fps, porque o `requestAnimationFrame` para com a janela minimizada ou coberta e o vídeo congelaria. Se o Worker parar, `animar()` assume pelo rAF.
4. **Prévia na tela:** `#camGravar` espelhado, mais a legenda em HTML (`#legendaPrev`) com o mesmo cálculo de tamanho do canvas (`tamanhoLegenda`). O que se vê bate com o que é gravado.
5. **Avanço:**
   - **Automático:** `setTimeout` com os segundos de cada passo.
   - **Manual:** Espaço, →, PageDown (passador de slides); ← e PageUp voltam.
   - **Parar:** Esc durante a gravação precisa ser apertado duas vezes.
   - **Proteções:** F5, Ctrl+R e tecla segurada são bloqueados durante a rodada.
6. **Fim:** `finalizar()` monta o arquivo, mostra a prévia e já dispara o download.
   - **Aviso ao sair:** enquanto ninguém clica em "Baixar vídeo", fechar a página pede confirmação. O download automático pode ser bloqueado pelo navegador.
   - **Avisos no resultado:** trechos em que a janela ficou oculta, câmera ou microfone perdidos e erro do gravador.
7. **Miniatura de referência:** o vídeo da fonte no canto (`#canto`) só aparece na tela. Nada da fonte é desenhado no canvas: vídeo de outro site "contaminaria" o canvas e a gravação falharia.

## Atalhos

| Onde | Tecla | Ação |
|---|---|---|
| Estudar | ← → | Sinal anterior / próximo |
| Estudar | Espaço | Pausa o vídeo |
| Estudar | 1 2 3 | Velocidade 0,5× / 0,75× / 1× |
| Estudar | D | Marca/desmarca "Já domino" |
| Gravar (rodando) | Espaço → PageDown | Próximo passo |
| Gravar (rodando) | ← PageUp | Passo anterior |
| Gravar (rodando) | Esc (2× ao gravar) | Para |

## Regras de design que valem para qualquer mudança

- **Vidro fora dos vídeos:** o vidro (`backdrop-filter`) nunca fica em cima de vídeo. Por isso:
  - a barra é estática em Estudar e Gravar e só gruda no topo na Ficha;
  - durante a gravação a barra some.
- **Fundo animado:** só por `transform` ou `opacity`. Com a câmera ligada (`body.cam-on`) ou gravando (`body.rodando`), tudo pausa.
- **Gravação não muda:** o tema e os efeitos não mexem nela. A legenda gravada é sempre texto branco sobre faixa escura.
- **Teleprompter:** legível a 1,5 m em 1366×768. Durante a rodada a barra e o painel de configuração somem para dar espaço.
- **Testado e descartado:** lago com reflexo e grama no rodapé. Ficou feio e saiu; não repetir.

## Como testar

O gancho `window.__estudio` expõe:
- `SINAIS` e `estado()`;
- `comecar`, `parar`, `avancar`;
- `tela` (o canvas gravado);
- `montarPassos`, `gerarRoteiro`, `textoFontes`, `trocarSlot`.

1. **Sirva a pasta por HTTP.** Por `file://` muitas ferramentas não executam JS:

   ```bash
   npx --yes http-server -p 8765 -c-1
   ```

   e abra http://localhost:8765/ (ou use o site do Pages).
2. **Simule câmera e microfone no console:**

   ```js
   const c = document.createElement('canvas'); c.width = 1280; c.height = 720;
   const g = c.getContext('2d'); setInterval(() => { g.fillStyle = `hsl(${Date.now() / 20 % 360},55%,40%)`; g.fillRect(0, 0, 1280, 720); }, 100);
   const ac = new AudioContext(), dest = ac.createMediaStreamDestination(); const o = ac.createOscillator(); o.connect(dest); o.start();
   navigator.mediaDevices.getUserMedia = async k => (k.audio && !k.video) ? dest.stream : c.captureStream(30);
   ```

3. **Roteiro de teste:**
   - **Vídeos:** todo vídeo de `SINAIS` carrega (duração > 0) e todo `trecho` cabe na duração.
   - **Estudar:**
     - um sinal com `trecho` fica repetindo dentro do trecho;
     - "V-I-C-T-O-R" passa pelas letras;
     - as teclas 1, 2 e 3 mudam a velocidade.
   - **Ficha:**
     - trocar o slot 1 pelo seletor muda a lista, a gravação e o roteiro;
     - "já apareceu" marca a linha;
     - "Voltar à lista sugerida" restaura.
   - **Gravação automática:** com `Object.assign(__estudio.estado(), {segIntro: 2, segTermo: 1, segFim: 1})` e `__estudio.comecar(true)`, ela passa por todos os passos e gera um arquivo com som. A região da legenda em `__estudio.tela` tem pixels brancos que mudam entre os passos.
   - **Modo manual:** Espaço, → e PageDown avançam, ← volta, nada avança sozinho e Esc para.
   - **Erro de câmera:** simular `NotReadableError` mostra a mensagem e nada trava.
   - **Console e roteiro:** sem erros, e `gerarRoteiro()` no estado padrão é igual ao `roteiro-libras-ti.md`.
4. **Pegadinha do teste:** aba ou janela em segundo plano congela animações e o `requestAnimationFrame`. Para conferir movimento, a página precisa estar visível.

## Publicar

O repositório é da conta pessoal **Pedrosin**; a conta normalmente ativa no `gh` deste PC é a **Victor-Pedroso**, que não tem acesso a ele.

```bash
gh auth switch -u Pedrosin
git add -A && git commit -m "mensagem" && git push
gh auth switch -u Victor-Pedroso
```

- **Identidade dos commits:** "Víctor Pedroso Duma" com o e-mail privado `70385631+Pedrosin@users.noreply.github.com`. Está só no config local deste repositório.
- **Atualização do site:** o GitHub Pages publica a branch `main` (raiz) e atualiza em cerca de um minuto depois do push.

## Armadilhas conhecidas

- **YouTube não serve como fonte embutida.** Em `file://` o player do YouTube falha (sem referrer) e tem anúncio. O canal "Informática em LIBRAS" do IFPB ficou de fora por isso. Use só mp4 direto.
- **SENHA (INES) é S-E-N-H-A soletrado seguido do sinal SEGREDO.** Não é sinal de TI, por isso está no fim da reserva. REDE3 também começa soletrando R-E-D-E.
- **CÓDIGO** (programação) não tem vídeo em Libras em nenhuma das fontes conferidas. DIGITAR ocupa o lugar.
- **SERVIDOR** é o sinal de origem mais fraca: o crédito "Sinalário LSB" do IFBA não foi identificado.
- **Canal regional do AVA (LSBCASMS):** tem 7 vídeos, nenhum de TI. Por isso as fontes são nacionais.
- **WebM do `MediaRecorder` vem sem duração.** A prévia contorna isso pulando para o fim e voltando.
