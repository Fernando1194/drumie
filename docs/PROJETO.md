# Drum Practice: documento do projeto

Notas de projeto em português: o que o app faz, como foi construído, cada decisão e as medições
que a justificaram (inclusive o que foi testado e descartado). Para instalar e usar, veja o
[README](../README.md).

O app transforma uma música (link ou arquivo) numa aula de bateria: separa a bateria, transcreve
as batidas, acha padrões e passos, e oferece um player com kit, grade, modo Steps e um "Guitar
Hero" para bateria eletrônica via MIDI. O código também transcreve outros instrumentos, mas essa
parte está congelada e escondida (`MUSIC_PRACTICE_ALL_INSTRUMENTS=1` mostra).

Estado atual: **v39** (22/10/2026), publicado no GitHub. Foco: bateria.

---

## 1. Como rodar

```bash
cd drum-practice && source .venv/bin/activate
drum-practice                      # ou: python -m musicpractice.server; abre http://localhost:7860
git pull && pip install -e .       # atualizar (o pip só é obrigatório quando mudam dependências, como na v32)
```

- Depois de atualizar: **Ctrl+F5** no navegador.
- `music-practice` abre a página antiga em Gradio (todos os instrumentos).
- Para que uma melhoria de transcrição valha numa música já analisada: **Analyze song** de novo.
- Porta 7860 ocupada: `pkill -f musicpractice.server` (ou `drum-practice --port 7861`).
- No Windows, rodar dentro do WSL (Ubuntu): o Smart App Control do Windows bloqueava o executável
  e DLLs (pandas, numba). O navegador e o kit MIDI ficam no Windows normalmente.

Instalação do zero: seção *Install* do README.

---

### Publicação (a partir da v34)
- Código: repositório **privado** `drum-practice`. Vitrine pública: `drumie` (README com imagens,
  CHANGELOG, LICENSE, THIRD_PARTY_NOTICES, este documento e as releases).
- A cada versão: seção nova no CHANGELOG do privado (a release privada sai sozinha pelo
  workflow) e o mesmo CHANGELOG copiado para a vitrine, cujo workflow cria a release pública.

---

## 1b. Onde cada parte roda

Todo o processamento roda no PC do usuário (WSL Ubuntu). Nenhum áudio é enviado para servidor.

| Etapa | Modelo / biblioteca | Onde roda | Internet |
|---|---|---|---|
| Download da música | yt-dlp | local | sim: baixa do YouTube/SoundCloud |
| Separação (6 faixas) | Demucs htdemucs_6s (PyTorch) | local, placa de vídeo se houver | só 1ª vez: pesos ~53 MB |
| Separação da bateria | modelo de bateria do htdemucs_ft | local, placa de vídeo se houver | só 1ª vez: pesos ~80 MB |
| Batidas de bateria | ADTOF (PyTorch) | local, processador | não (pesos vêm no pacote) |
| Notas (guitarra etc.) | Basic Pitch (ONNX) | local, processador | não (modelo vem no pacote) |
| Tempo, acordes, seções, padrões, grade | librosa, numpy, scipy (sem IA) | local | não |
| Interface | servidor FastAPI próprio + navegador | local (localhost:7860) | fontes do Google |

- Pesos do Demucs ficam em `~/.cache/torch/hub/checkpoints/`.
- Os "agentes" são código Python comum, não chamadas a um modelo de linguagem; o app não usa
  Claude nem outra IA de texto.
- A página antiga em Gradio (`music-practice`) envia estatísticas anônimas por padrão; para cortar: `export GRADIO_ANALYTICS_ENABLED=False`.
- Sem internet o app funciona com músicas já baixadas (upload de arquivo), depois que os pesos
  do Demucs foram baixados uma vez; só as fontes da página caem para as do sistema.

## 2. Fluxo

```
Link -> download (yt-dlp) -> separação (Demucs, 6 faixas) -> análise (andamento, compassos,
tom, acordes, seções) -> transcrição (notas/tab ou bateria) -> player na mesma página -> export
```

Tudo fica em cache por música em `~/.music-practice/<id>/artifacts.json`; cada etapa só roda
uma vez, a não ser que fique desatualizada (ver 4.3).

---

## 3. Arquitetura

```
musicpractice/
  agents.py          5 agentes (Input, Analysis, Separation, Transcription, Practice) + Conductor
  server.py          app de bateria: servidor FastAPI, API JSON, fila de análises
  app.py             interface antiga em Gradio (uma página, todos os instrumentos)
  core/              Song, Workspace (cache por música), Registry de plugins
  plugins/
    input.py         link -> WAV (yt-dlp; recusa Spotify/Apple Music etc.)
    separation.py    Demucs htdemucs_6s
    rhythm.py        batidas e compassos (librosa + heurística de tempo forte)
    harmony.py       tom (Krumhansl) e acordes (templates + Viterbi)
    structure.py     seções A/B/C (novidade na auto-similaridade)
    transcription.py notas (Basic Pitch via ONNX; TensorFlow escondido)
    tab.py           notas -> corda/casa (Viterbi sobre posições da mão)
    drums_adtof.py   bateria com ADTOF (rede neural, preferido)
    drums.py         detector básico de bateria (NMF + regras), MIDI de bateria
    drum_edit.py     sensibilidade, regra das "três mãos", edições manuais
    drum_tab.py      grade de bateria (x / o por semicolcheia)
  ui/
    shell.html/js/css  página do app: cabeçalho, barra lateral, Biblioteca, rotas #/secao
    player.py        HTML do player (dados da música embutidos em JSON); app=True = em seções
    player.js        player sincronizado, mixer, layout, valores digitáveis
    highway.js       "Guitar Hero" de bateria + Web MIDI + monitor MIDI
    training.js      calibração, desafios, histórico
    editor.js        painel "Fix transcription"
    player.css       estilos (tudo dentro de #mp-wrap; @keyframes no nível de cima)
    kit.py           desenho do kit visto de cima
tests/               testes Python, roteiros de navegador (Playwright), benchmark MDB
```

Os "agentes" são Python comum, não chamadas a LLM: cada etapa tem um procedimento certo, e um
modelo de linguagem só traria custo e aleatoriedade. Nova funcionalidade = um plugin que declara
o que precisa (`requires`) e o que produz (`produces`); o Registry resolve a ordem e o cache.

---

## 4. Decisões importantes (e por quê)

### 4.1 Bateria: ADTOF em vez do detector próprio
Medido no **MDB Drums** (23 gravações reais, ~8.000 batidas anotadas à mão), F1:

| Detector | Bumbo | Caixa | Chimbal | Pratos | Tons |
|---|---|---|---|---|---|
| ADTOF, gravação de bateria limpa (melhor caso) | 0,96 | 0,80 | 0,86 | 0,87 | 0,53 |
| ADTOF, bateria separada pelo Demucs (o que o app faz) | 0,92 | 0,77 | 0,86 | 0,84 | 0,45 |
| ADTOF, mix completo | 0,85 | 0,70 | 0,85 | 0,83 | 0,47 |
| Detector básico, faixa de bateria | 0,93 | 0,59 | 0,70 | 0,10 | 0,02 |

- Tons com limiar 0,5 (o padrão do ADTOF, 0,32, acerta só 1 em 5 tons que reporta).
- Crash x ride decidido por quanto o prato "soa" depois da batida (94% certo).
- Licença do modelo: CC BY-NC-SA 4.0 (ok para uso pessoal).

### 4.2 Transcrição usa a faixa isolada
Sempre que a separação existe. Medido com o Demucs de verdade (htdemucs_6s, mesmo do app,
separando as 23 mixes do MDB): bumbo 0,85 -> 0,92, caixa 0,70 -> 0,77, chimbal 0,85 -> 0,86,
pratos 0,84 -> 0,84, tons 0,47 -> 0,45. (Os 0,96/0,80 de antes eram com a gravação de bateria
limpa original, o melhor caso possível.)

Pós-processamento testado na faixa separada (02/10/2026), F1 nas 23 músicas:
| Variante | Bumbo | Caixa | Chimbal | Pratos | Tons |
|---|---|---|---|---|---|
| faixa separada (app hoje) | 0,922 | 0,766 | 0,860 | 0,839 | 0,453 |
| Demucs com 2 deslocamentos | 0,923 | 0,766 | 0,862 | 0,840 | 0,453 |
| faixa + noise gate | 0,930 | 0,753 | 0,850 | 0,523 | 0,438 |
| média faixa+mix (chimbal/pratos/tons) | 0,922 | 0,766 | 0,873 | 0,845 | 0,488 |
| **htdemucs_ft (modelo de bateria)** | **0,931** | **0,773** | 0,857 | **0,855** | **0,462** |
| gravação limpa (melhor caso) | 0,961 | 0,797 | 0,862 | 0,871 | 0,531 |
Conclusão: 2 deslocamentos não ajuda (dobra o tempo); gate destrói pratos; a média com a mix
ganha na soma mas é instável (8 músicas melhores, 6 piores, até -0,21), contra o objetivo de
consistência. Nada entrou no app. O espaço que sobra (bumbo 0,92 -> 0,96, caixa 0,77 -> 0,80)
é da separação. Testado o htdemucs_ft (só o modelo de bateria dele, f7e0c4bc): melhor em
11 músicas, pior em 4 (Country1 -0,13, a mesma que oscila em todas as variantes). ENTROU (v24):
o app separa tudo com htdemucs_6s e refaz só a bateria com o modelo de bateria do htdemucs_ft
(~10 s a mais por música com placa de vídeo). Músicas antigas trocam a faixa e retranscrevem a
bateria uma vez, sozinhas; edições à mão são mantidas.

### 4.2b Grade de compassos até o fim
O detector de tempo (librosa) descarta as batidas fracas do fim: num fade de 12 s ele parou
~10 s antes do fim. Sem compasso, as notas do fim não tinham onde ser desenhadas. Agora a grade
continua no mesmo espaçamento até onde o som acaba (50 dB abaixo do pico). Músicas antigas
refazem a grade (e acordes/seções) sozinhas no próximo "Analyze song".

### 4.2c Mesma batida, mesmo desenho (grade guiada pela bateria)
Teste de repetibilidade (14 gravações do MDB, mesma música "subida de novo" com 30 ms de atraso
e 3 dB a menos, ou em MP3): a IA ouve as mesmas batidas em 97 a 99% dos casos; o que mudava era
o DESENHO, por causa da grade de tempo:
- batidas do detector ~40 ms atrasadas (todas as músicas): batidas no tempo caíam perto da borda
  entre duas casas e "pulavam" de casa;
- linha do compasso no tempo errado em 4 de 14 músicas; metade do andamento em 1 de 14.
Correção (plugins/drum_grid.py): depois da bateria transcrita, a grade é ajustada pelo próprio
baterista: suaviza as batidas, alinha pelo meio das batidas de bumbo/caixa/chimbal próximas,
dobra o andamento se a caixa cai em todo tempo, e escolhe o tempo 1 por bumbo no 1 e 3 e caixa
no 2 e 4 (acordes desempatam). Resultado contra a grade feita das anotações à mão: 0,50 -> 0,71
de concordância; linhas de compasso certas 65% -> 88%; mesma música re-subida 90% -> 91%.
Também: o Demucs embaralhava o áudio com um deslocamento ALEATÓRIO a cada execução; agora com
semente fixa, a mesma música dá as mesmas faixas.
Testado e descartado: re-rotular batidas pelo timbre (mesmo som = mesma peça) não mudou nada no
MDB (o modelo já é consistente dentro da música).
Não resolvido: músicas em que o detector erra feio o andamento (1 de 14), suingue/tercina.

### 4.2d Bumbo ouvido duas vezes (caso real: Audioslave, "Gasoline")
Introdução = um bumbo repetido, mas alguns saíam também como caixa ou chimbal fracos (logo acima
do limiar). O som dessas batidas era idêntico ao dos bumbos "sozinhos" (±2 dB em todas as
faixas de frequência). Regra (plugins/drum_echo.py): caixa/chimbal fraco (< 2x o limiar) em cima
de um bumbo, que não tem mais energia na sua faixa (caixa 2-6 kHz, chimbal > 6 kHz) do que os
bumbos sozinhos vizinhos, é eco e sai. Na música: as 6 marcas falsas da introdução somem
(31 na música toda). No MDB é neutra (chimbal 0,862 -> 0,869; caixa 0,797 -> 0,796); aplicada a
pratos e tons piorou, então vale só para caixa e chimbal.

### 4.3 Cache que se corrige sozinho
Músicas transcritas antes do Demucs (a partir do mix), com o detector antigo, ou antes de
existirem os "candidatos" (ver 4.4) são refeitas automaticamente no próximo "Analyze song".

### 4.4 Sensibilidade por família (sliders)
O detector guarda todas as batidas com uma nota de confiança ("candidatos"); os sliders só
filtram, sem rodar o modelo de novo. Conferido: filtrar no padrão dá exatamente a saída
original do ADTOF (0 diferenças em 7.716 batidas).
**Expectativa honesta:** escolher a melhor sensibilidade por música melhora o F1 em ~0,01 na
mediana, e 0,1 a 0,3 em poucas músicas.

### 4.5 O que realmente falta (MDB Drums)
| Tipo | Encontradas |
|---|---|
| Caixa normal | ~96% |
| Bumbo | ~97% |
| Ghost notes da caixa | 39% (56% na sensibilidade máxima, e metade vira "tom") |
| Chimbal de pé | 44% |
| Vassourinha | 61% |

Conclusão: o que falta é nuance; o caminho é edição manual (de preferência copiando a correção
para as repetições da seção).

### 4.6 Regra das "três mãos"
Se 3 ou mais peças de mão caem no mesmo instante (30 ms), sai a batida mais fraca. O chimbal
fica (pode ser o pedal), exceto quando um ride ou crash soa junto: aí é o mesmo prato ouvido
duas vezes e sai o mais fraco dos dois (no MDB isso acertou 13 de 16). No MDB, 16 de 16
grupos de "três mãos" eram erros. O painel mostra quantas
foram removidas; dá para devolver à mão.

### 4.7 Padrões e passos (v19)
**Padrões.** Cada compasso vira marcas (peça, semicolcheia) com peso: bumbo e caixa 1, tons
0,7, chimbal e ride 0,5 (é onde o baterista varia e a IA erra mais); o crash é ignorado.
Distância = Jaccard ponderado; agrupamento hierárquico (média) cortado em 0,45.
Medido no MDB Drums (680 compassos, batidas das anotações), concordância entre o agrupamento
feito da transcrição e o feito das anotações à mão (índice de Rand ajustado, 1 = igual):

| Regra | Faixa de bateria | Mix |
|---|---|---|
| v18 (igualdade com 12% de tolerância) | 0,47 | 0,25 |
| v19 (pesos + agrupamento) | 0,73 | 0,70 |

**Passos.** Frase = um groove repetido + a virada que fecha (nova frase quando muda o groove,
ou depois de uma virada se a frase já tem 4+ compassos). A mesma frase tocada de novo em seguida
é o mesmo passo repetido: "(Groove 2 ×3 + fill) ×6" é 1 passo. MDB: 111 passos da transcrição
contra 111 das anotações (≈ 1 passo a cada 6 compassos); rock/pop de 16 a 20 compassos dá 1 a 3
passos, jazz improvisado continua fragmentado (como deve).
Limite: se a IA confunde chimbal e ride num trecho, um groove de chimbal e um de ride podem
virar o mesmo padrão.

**Na tela.** Faixa de padrões com chips e quadradinhos por compasso (v18) + modo **Steps**:
mostra um compasso só (o groove do passo, desenhado uma vez); a cada repetição o playhead volta
ao início do MESMO compasso com um "varrido" suave, o contador sobe ("2 de 3"), bolinhas da frase
se preenchem, "Passada 1 de 8", lista de passos com ✓ nos concluídos e aviso do próximo.

### 4.7b Figuras de um tempo (v26)
Um nível abaixo dos padrões: cada tempo (4 semicolcheias) de cada compasso é agrupado com os
tempos que soam iguais, e o compasso vira uma sequência de letras: rock simples = "A B C B"
(A = bumbo + chimbal, B = caixa + chimbal, C = bumbo no tempo e no "e" + chimbal). Mesmos pesos
dos padrões, mas aqui o crash conta (0,5): "bumbo + crash" é outro movimento.
Medido nas gravações rock/pop/funk do MDB Drums (batidas das anotações), agrupamento da
transcrição contra o das anotações (Rand ajustado):

| Corte | Faixa de bateria | Mix | Figuras por trecho (anotação / IA, faixa) |
|---|---|---|---|
| 0,2 | 0,83 | 0,63 | 9,3 / 9,3 |
| **0,3 (escolhido)** | **0,84** | **0,72** | 7,7 / 7,5 |
| 0,4 | 0,83 | 0,73 | 6,1 / 5,9 |

0,3 e não 0,4: com 0,4 "bumbo no tempo" e "bumbo no tempo e no e" (distância 0,33) viram a
mesma figura, e essa diferença o baterista tem que tocar; um chimbal perdido (0,25) continua na
figura certa. As 6 figuras mais comuns cobrem 95% dos tempos.
**Na tela:** uma letra colorida sob cada tempo da grade; na faixa de padrões, um chip por figura
com a letra, um desenho de 4 semicolcheias e quantas vezes aparece; clicar acende todos os tempos
daquela figura. "·" = tempo que só aparece uma vez.
Limite [Provável]: em música com muitas variações de chimbal aberto/ghost notes, a IA erra essas
notas e pode criar figuras a mais (mix 0,72 contra 0,84 na faixa isolada).

### 4.7c Espaço em disco (v28)
Medido na pasta do Fernando (56 músicas analisadas durante o desenvolvimento): **25 GB**, sendo
25,7 GB de WAV (523 arquivos: download, 6 faixas separadas, 2 mixes "isolados") e 1,05 GB das
cópias Opus, que são o único áudio que o player toca. Média de ~450 MB por música.
**Regra 1, compactar:** ao fim da análise cada WAV vira a cópia Opus (128 kbit/s), e as pastas
`isolated/`, `practice/`, `export/` e restos do Demucs são apagados: ~20 MB por música. Os
mixes "isolados" passaram a ser feitos só quando pedidos (o player usa o mixer ao vivo).
**Re-transcrever do Opus dá o mesmo resultado** (ADTOF, 23 faixas do MDB, WAV vs Opus 128k, F1):
bumbo 0,961/0,960, caixa 0,797/0,796, chimbal 0,863/0,862, pratos 0,869/0,868, tons 0,322/0,335
(faixa de bateria); no mix completo diferenças de no máximo 0,006.
**Regra 2, limite (5 GB, `--max-gb`):** passando do limite, o áudio das músicas abertas há mais
tempo é apagado; transcrição, sensibilidade e edições ficam. Ao reabrir (link ou o mesmo arquivo),
baixa e separa de novo; se as batidas já vinham do modelo fino de bateria, não transcreve de novo.
Descartado: apagar tudo a cada recarregamento (refaz minutos de separação a cada abertura, e a
página não sabe com segurança quando foi fechada) e streaming (a separação precisa do arquivo inteiro).

### 4.8 Testado e descartado
- **Chimbal aberto x fechado pelo som:** não generaliza entre músicas (a 70% de precisão, quase
  nenhum aberto encontrado). Não entrou no app.
- **Separar as peças da bateria antes (drumsep, MDX23C de 6 faixas, out/2026):** medido no MDB
  (23 faixas só de bateria). Melhor variante: maior ativação de chimbal entre o ADTOF na bateria e o
  ADTOF na faixa de chimbal separada: F1 do chimbal 0,863 -> 0,894, mas o ganho vem quase todo de
  UMA faixa (ModalJazz, chimbal de pé: 0 -> 0,90); chimbal de pé 44% -> 79% encontrado, fechado e
  aberto iguais. Faixa a faixa: 5 melhores, 4 piores (Punk -0,04, FunkJazz -0,02, Disco, Britpop),
  11 iguais. Outras variantes (ADTOF só na faixa de chimbal, só nos pratos, filtrar pela energia
  da faixa) ficaram iguais ou piores; pratos pioraram (0,869 -> 0,802). Custo: ~10x a duração da
  música em 2 núcleos sem placa de vídeo, modelo de 418 MB, licença dos pesos indefinida (tratar
  como não comercial). Não entrou: para rock/pop é neutro ou pior. Talvez valha como opção
  "chimbal de pé" para jazz, se um dia houver demanda.
- **Outras percussões (pandeirola, cowbell, shaker):** o modelo só conhece 5 famílias; das 32
  pandeirolas do MDB, 29 viraram "chimbal". Modelos maiores (YourMT3+, Omnizart) não foram
  validados. Além disso, a TDX-N1 não tem pads para essas peças: elas ficam no áudio de
  acompanhamento.

**Separar a bateria em peças (v38).** O drumsep, descartado para melhorar a transcrição (acima),
entrou para outra coisa: praticar uma peça. Sob pedido, uma vez por música (~10x a duração em 2
núcleos), gera 6 faixas Opus em `<música>/parts/`; o artefato `drums:parts` guarda a faixa de
bateria de origem e fica inválido se a música for analisada de novo. Vazamento medido no MDB:
bumbo 96%, caixa 88%, tons 85%, chimbal 79%, ride 76% da energia na faixa certa. Bom para mutar
uma peça e tocar por cima; não é isolamento perfeito. Identificar o "tipo de kit" foi avaliado e
adiado: sem caso real em que mudaria a transcrição.

**Comparar com uma tablatura (v39).** O MDB só tem gravações de bateria isolada; para medir em
músicas reais o gabarito é uma tablatura (MIDI ou Guitar Pro 3-5) que o Fernando confia. A tablatura
fica guardada em compassos (posição = compasso + fração), não em segundos. Alinhamento: desloca a
tablatura um tempo por vez e testa 1, 2 ou 1/2 compasso da música por compasso da tablatura,
ficando com a posição em que mais bumbo, caixa e chimbal caem na mesma semicolcheia; dá para
ajustar à mão. Comparação na grade de semicolcheias do próprio grid, por família (qual tom e
crash x ride não contam: tablaturas divergem aí). `drumie --compare-tabs` soma tudo, versão da IA
(sensibilidade padrão, sem edições) e versão atual. Próximo passo: refazer o teste do drumsep e de
"kit da música" neste gabarito.

**Passos = compasso real (v39).** O modo Passos desenhava o compasso "mais típico" do groove; uma
variação não aparecia (o grid estava certo). Agora desenha o compasso tocado; repetições idênticas
reaproveitam o desenho.

### 4.9 Player
- Página única: o áudio toca no navegador; Python só monta o HTML com os dados.
- Mixer: cada faixa vira uma cópia Opus pequena, baixada inteira para a memória (o Chrome limita
  ~6 conexões por servidor); uma faixa comanda o tempo e as outras seguem ajustando a velocidade.
  Diferença medida entre faixas: normalmente < 10 ms.
- Loops e saltos começam 40 ms antes do compasso, para não cortar o ataque.
- Playhead: segue os tempos e passa por cada marca no momento em que ela soa (corrigia meia
  semicolcheia de atraso; na bateria também compensa a detecção de tempo, que costuma ficar
  20 a 40 ms atrasada).
- Grade: uma pauta contínua por linha, nomes das peças só no início, divisões discretas.
- Altura da partitura: alça embaixo dela; arrastar para baixo mostra mais linhas à frente (1 a 8,
  sempre linhas inteiras), setas ajustam, duplo clique volta a 2; fica salvo no navegador.
- Follow: a página desliza para a próxima linha durante o último tempo da linha (com
  aceleração e freada suaves), calculado pela posição da música a cada quadro; a nova linha
  chega ao topo exatamente quando o primeiro compasso dela começa. Rolar com o mouse pausa o
  follow por 2,5 s. Medido: ~27 quadros de deslize, maior passo 7 px, 60 quadros/s.

**Pulso das notas (v27).** Antes cada nota da grade pulsava no instante do áudio da batida.
Como a nota é desenhada no centro da semicolcheia, uma batida fora da grade pulsava antes ou
depois da linha passar: medido no MDB (faixa de bateria, batidas das anotações), 10% das batidas
ficam a mais de 35 ms da grade, 8% a mais de 40 ms; com a grade do detector de batidas é mais.
Agora a nota pulsa quando a linha cruza o centro dela (a cada quadro, entre a posição anterior e
a atual da linha; um salto, loop ou clique não dispara nada). O kit continua no tempo do som.
O pulso usa Web Animations (transform no compositor) em vez de tirar e pôr a classe com um
layout forçado a cada batida.

### 4.10 Highway (bateria eletrônica via MIDI)
**Sincronia (v29).** O Fernando sentiu a música fora de sincronia com o highway tocando por uma
interface de áudio. Causa: o highway desenhava no `audio.currentTime` (posição do decodificador),
mas o som chega ao ouvido depois, pela latência de saída (Windows + interface). As notas cruzavam
a linha antes do som e as batidas no tempo eram julgadas atrasadas. Agora o highway usa um relógio
atrasado pela latência que o navegador informa (`AudioContext.outputLatency`, 32 ms no Chromium de
teste) e julga no mesmo relógio. A latência só é lida com o highway aberto: abrir o AudioContext no
meio da música fez o áudio engasgar no teste da grade (batidas do kit sem piscar).
"Note delay" desloca só o desenho, para atrasos que o navegador não vê (Bluetooth).
**Dificuldade da música (v37).** Nota 1 a 10 por nível: mãos por segundo nos compassos mais
cheios (percentil 90, peso 35%), bumbo por segundo (20%), bumbo/caixa em semicolcheias fora do tempo
(15%), proporção de compassos de virada (10%), número de grooves e compassos únicos (20%); nota =
1 + 9 x soma^0,75. Conta notas por segundo e não BPM, para não errar com andamento detectado pela
metade ou em dobro. No MDB a ordem faz sentido (country/rock simples 1-2,5; disco/punk ~4; funk e
fusion 5-6; latin e modal jazz 6,4-7,3). Pesos são palpite inicial: calibrar com músicas que o
Fernando conhece. **Drumie no kit (v37):** mão por batida com custo (preferência de destro + custo
de velocidade < 75/130 ms para alternar em rulos e viradas); braço por cinemática inversa de dois
segmentos a partir do ombro; baqueta levantada fica mais curta vista de cima.
**Níveis de dificuldade (v35).** Uma transcrição só (a completa, com as edições); Médio e Fácil
saem dela por regras, então corrigir uma vez vale para todos. Médio: sem ghost notes (batida com
menos da metade da mediana de força da peça), tudo em colcheias. Fácil: chimbal/ride em semínimas,
caixa e bumbo só no tempo, prato só no 1, virada vira o groove anterior (o tempo não para). Sempre
no máximo duas mãos juntas; no Fácil o prato substitui o chimbal no mesmo instante. Shuffle
detectado (colcheia em quiáltera cai na 4ª semicolcheia) mantém o contratempo. Medido em 8 faixas
do MDB: Médio guarda 55 a 98% das notas, Fácil 33 a 69%. Limite: regras não entendem meio-tempo,
condução no ride com chimbal de pé, compassos ímpares; e no jazz muitas "viradas" curtas viram
compassos de groove inteiros (Fácil ficou com 93% das notas numa faixa).
**Identidade Drumie (v33).** O nome vem do mascote: um boneco de testes de batida de carro que
toca bateria, magro e desengonçado, porque aprender bateria é levar pancada. Ele reage ao
resultado (comemora A/S, sua quando o tempo está fora, cai quando a nota fica abaixo de 60%), para o
desconforto virar humor e não punição. Desenhado em código (drumie.js, ângulos por junta) para não
depender de arquivos de imagem e para dar poses novas sem ilustrador. Cores mantidas; o amarelo que
já era o destaque virou a cor da marca, junto com o preto das juntas. Repositório e comando antigo
(`drum-practice`) mantidos para não quebrar links e atalhos.
**Interface em seções (v32).** A página única cresceu até o ponto em que as funções se
escondiam umas atrás das outras (Highway, Training e Fix eram botões que trocavam o painel da
direita). Agora: Biblioteca + uma seção por tarefa numa barra lateral, com a barra de play fixa
embaixo e a música continuando entre seções. O Gradio saiu do app de bateria: ele exigia um
canal oculto (caixa de texto invisível) para salvar correções e redesenhava a página inteira a
cada análise; um servidor FastAPI pequeno com API JSON resolve os dois e permite uma fila de
análises com progresso. A página Gradio continua como `music-practice` para os outros
instrumentos, com o layout antigo (build_player(app=False)). Em telas de até 1500 px a barra
lateral vira um trilho estreito, porque com 232 px a grade caía para 1 compasso por linha em
1400 px (medido no teste). "Copiar este compasso para todas as repetições" usa os mesmos padrões
da faixa de grooves (mesmo padrão e mesmo comprimento) e grava edições manuais, então sobrevive a
reanálise e o Reset desfaz.
**Revisão de sincronia (v31).** Medido: (1) o ADTOF marca cada batida 1 quadro antes do ataque,
mediana -10,0 ms contra as anotações do MDB em todas as peças; corrigido em +10 ms, com migração
única das músicas já analisadas (batidas e grade); (2) as cópias Opus não têm atraso (0 amostras
no ffmpeg e no Chromium); (3) a calibração já corrigia o julgamento, mas o desenho continuava
adiantado pelo atraso de áudio que o navegador não informa: agora a latência calibrada também
desloca as notas; (4) logo após o play, o relógio do highway previa à frente de um áudio que ainda
não tinha começado: segue o áudio cru nos primeiros 250 ms. Não medido no PC do Fernando (as
músicas dele ficam no WSL, fora do alcance desta sessão): a deriva entre as faixas do mixer, que
nos testes fica abaixo de 30 ms com picos ocasionais.
**Calibração por rudimento (v30).** Um compasso de cliques para entrar, depois 8 tempos do
rudimento em qualquer pad. Cada batida é casada com o toque mais próximo (janela de meio toque,
no máximo 150 ms) e a latência é a mediana das diferenças: poucos toques errados não a mudam.
Dispersão (MAD x 1,4826) diz quanto confiar; com mão esquerda e direita separadas dá para ver se
uma mão anda atrás da outra. Limites: semínimas dão o número mais limpo (rudimentos rápidos têm
mais toques mas cada um varia mais), e o hábito do baterista de tocar sempre um pouco atrasado
entra na medida junto com a latência, não dá para separar os dois.
- Web MIDI no Chrome do Windows (nada a configurar no Ubuntu).
- Julgamento em tempo real: Perfect 30 ms, Good 60 ms, OK 100 ms.
- Calibração de latência (8 cliques), "learn" para mapear pads, monitor MIDI (nota e
  velocidade de cada pad, posição do pedal do chimbal em CC4).
- Tons começam sem pontuar (a transcrição erra muitos).

---

## 5. Equipamento

- Bateria eletrônica **Aroma TDX-N1**: caixa 8" mesh, tons 6", bumbo, chimbal 9" com pedal
  controlador, crash e ride 9" de duas zonas, USB/Bluetooth MIDI.
- O mapa de notas MIDI do TDX-N1 **não foi encontrado publicado**: precisa ser lido no monitor.

---

## 6. To do

1. Com a TDX-N1 conectada, anotar no monitor do Highway ("Last pad:") a nota de cada pad e zona:
   caixa centro e aro; chimbal fechado, aberto e só pedal (e a faixa do CC4); crash borda e
   meio; ride meio e cúpula; cada tom; bumbo.
2. Com esses números: linhas extras no "Fix transcription" só para zonas que o kit toca
   (chimbal aberto, chimbal de pé, cúpula do ride, aro se existir), pistas no Highway, e mapa
   MIDI do export igual ao do módulo.
3. Botão "×2 / ÷2 andamento" para músicas detectadas com metade ou o dobro do tempo
   (aconteceu: uma música saiu com 43 BPM e caixa nos 4 tempos).
4. Depois da bateria: precisão de guitarra e baixo.

Ideias mais distantes: detecção de compasso com modelo (`beat_this`) para músicas com mudança
de andamento; acordes com sétimas; export para Guitar Pro / MusicXML; registro de progresso por
seção; coach com LLM.

---

## 7. Limites conhecidos

- Tablatura é rascunho (bends, slides, hammer-ons não são detectados).
- Acordes só maiores e menores.
- Compassos assumem 4/4; músicas com mudança de andamento derrapam.
- Nomes de seção (Chorus, Solo) são palpites; as letras (A/B) são mais confiáveis.
- Duas guitarras não se separam uma da outra.
- Bateria: tons e ghost notes são o ponto fraco; aberto x fechado do chimbal não é distinguido.

---

## 8. Testes

```bash
python -m pytest -q                                 # 56 testes, ~6 min, músicas sintéticas
python tests/browser_check_app.py  /tmp/out         # seções, Biblioteca, Mixer, Export, Ajustes, celular
python tests/browser_check.py      /tmp/out         # player (guitarra)
python tests/browser_check_drums.py /tmp/out        # kit, grade, cores, pauta contínua
python tests/browser_check_mixer.py /tmp/out        # mixer e sincronia das faixas (página Gradio)
python tests/browser_check_highway.py /tmp/out      # Highway com kit MIDI simulado
python tests/browser_check_fix.py  /tmp/out         # painel de correção, playhead, follow, fluidez
python tests/benchmark_mdb.py drum_only             # precisão real (precisa do MDB Drums, 3 GB)
```

---

## 9. Histórico de versões (resumo)

| Versão | O que entrou |
|---|---|
| v1 a v3 | App base: link, separação, análise, transcrição, tab, prática, export; rodar no WSL |
| v4 | Player numa página só, sincronizado com a música |
| v5 a v7 | Bateria: croqui do kit, cores iguais na grade, ADTOF, play/pause corrigido |
| v8 | Tablatura maior em duas linhas; um player por vez; animações que piscam a peça inteira |
| v9 | Layout ajustável (kit/grade), mixer por instrumento ao vivo |
| v10 | Highway estilo Guitar Hero com bateria eletrônica MIDI |
| v11 | Transcrição refeita a partir da faixa isolada quando ela passa a existir |
| v12 | Painel "Fix transcription": sensibilidade por peça e edição na grade |
| v13 | Playhead sincronizado com as notas; valores digitáveis nos sliders |
| v14 | Pauta contínua por linha (nomes só no início) |
| v15 | Monitor MIDI no Highway |
| v16 | Regra das "três mãos"; to do no README |
| v17 | Follow com deslize suave sincronizado; transições mais leves no destaque e no playhead |
| v18 | Padrões repetidos (grooves, viradas, únicos) com cores, faixa da música e loop por padrão; regra das três mãos trata prato duplicado |
| v19 | Padrões mais tolerantes (pesos + agrupamento) e modo Steps: a música em poucos passos que se repetem |
| v20 | Alça para escolher quantas linhas de partitura ver à frente |
| v21 | Alça colada na partitura; grade de compassos vai até o fim da música (antes o detector de tempo parava antes, em fade-outs e finais calmos, e o fim ficava sem transcrição) |
| v22 | Grade guiada pela bateria (alinhamento, tempo 1, andamento dobrado) e Demucs repetível |
| v23 | Bumbo ouvido duas vezes vira um só (caixa/chimbal 'eco' do bumbo) |
| v24 | Bateria separada com o modelo de bateria do htdemucs_ft (melhor que o 6s em bumbo, caixa, pratos e tons) |
| v25 | Página inteira no visual do player (tema escuro único, amarelo de destaque), título e números da música no topo do player, campos desnecessários removidos (instrumento, caixas de seleção, áudios duplicados, explicações) |
| v26 | Figuras de um tempo: cada tempo do compasso recebe uma letra (A = bumbo + chimbal, B = caixa + chimbal...), chips com desenho e contagem, clique acende onde a figura é tocada |
| v27 | Notas da grade pulsam quando a linha passa por elas (não mais no tempo do áudio, que ficava até meia semicolcheia antes/depois); pulsos com Web Animations, sem forçar layout da página a cada batida; o kit continua piscando no tempo do som |
| v28 | Espaço em disco: depois da análise só ficam as cópias Opus e a transcrição (~450 MB -> ~20 MB por música); limite de 5 GB apaga o áudio das músicas abertas há mais tempo, mantendo transcrição e edições; `--clean` para as antigas; cache temporário do Gradio limpo a cada hora |
| v29 | Highway sincronizado com o que se ouve (relógio atrasado pela latência de saída do áudio, que a interface aumenta; acertos julgados no mesmo relógio), controle "Note delay"; highway maior, acertos com anel, faíscas, brilho na pista, textos grandes, contador de combo e faixas a cada 10/25/50/100 |
| v30 | Calibração com rudimentos (semínimas, toques simples, rulo simples, rulo duplo, paradiddle; 70/100/120 BPM), mediana + dispersão + mão esquerda vs direita; botões 0/25/50/75/100 em cada faixa e no master "Song"; no highway: volume da bateria original, volume da música, "Bar back" e pad de play/pause |
| v31 | Revisão de sincronia: ADTOF 10 ms adiantado corrigido (músicas antigas migradas), calibração também move as notas, relógio sem previsão no início; faixas de tempo e encaixes com o formato da nota no highway; aba Training com calibração e 9 desafios com nota final |
| v32 | Interface nova em seções (Biblioteca, Aprender, Tocar, Treinar, Corrigir, Mixer, Ajustes, Exportar) com servidor FastAPI no lugar do Gradio; copiar um compasso para todas as repetições; histórico dos desafios; gravação atômica |
| v33 | Identidade Drumie: nome, logo com o alvo de teste no "e", mascote boneco de testes em 4 poses (Biblioteca vazia, erros, resultado dos desafios), favicon e indicador de carregando; comando `drumie` |
| v34 | Licença: grátis e código fechado (v0.24 a v0.33 seguem MIT); THIRD_PARTY_NOTICES.md; código vai para repositório privado e uma vitrine pública mostra README, CHANGELOG e releases |
| v35 | Níveis Completo / Médio / Fácil derivados da transcrição por regras (plugins/drum_simplify.py); grade, steps, kit e highway seguem o nível; Fix sempre no Completo |
| v36 | Seção Mixer substituída pelo botão Tracks na barra de play (mixer em todas as seções); CSS achatado (sem aninhamento) para navegadores antigos |
| v37 | Dificuldade 1-10 por música e nível (mãos/s, bumbo/s, síncope, viradas, variedade), coluna e ordenação na Biblioteca; Drumie toca no desenho do kit (antebraços, mãos R/L, baquetas, pés) com sticking de destro |
| v37.1 | Drumie no kit só com antebraços, mãos, baquetas e pés flutuantes (sem corpo); batida mais visível |
| v37.2 | Desenho do kit centrado no baterista: caixa no meio (entre as pernas), tons à frente, bumbo à direita |
| v38 | Separar a bateria em peças (DrumSep, opcional, sob pedido) e "Praticar uma peça": muta a peça, o resto toca, a grade destaca a linha; volume por peça em Tracks; membros do Drumie movidos pelo compositor |
| v38.1 | Tracks com duas abas (Instrumentos, Kit de bateria); corrige Drumie desenhado fora do kit na v38 |
| v39 | Comparar com uma tablatura (MIDI, Guitar Pro 3-5): alinhamento automático, concordância por peça e por compasso, marcas no grid; `drumie --compare-tabs`; Passos mostram o compasso real (não o "típico") |
