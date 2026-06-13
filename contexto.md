# Among Cats — Contexto do Projeto

Jogo no estilo Among Us com gatos, feito em HTML5 puro (sem frameworks).
Arquivo principal: `index.html` (tudo em um único arquivo).
Live: https://hellfirespriston-art.github.io/among-cats/
Repo: https://github.com/hellfirespriston-art/among-cats

---

## Estrutura Geral

- **Canvas 2D** para o mapa, personagens e mini-jogos
- **Web Audio API** para todos os sons (nenhum arquivo externo — evita erro `file://`)
- **HTML overlays** (`position:fixed`) para tarefas e telas especiais
- **`drawCat(ctx, x, y, options)`** — função central que desenha qualquer gato em qualquer contexto (mapa, preview, intro, cinemática)
- **`requestAnimationFrame`** com flag `_cinStop` para cancelar loops de animação

---

## Personagens e Mapa

- Jogador principal + impostores + tripulantes controlados por IA
- Gatos com cores customizáveis, nome, chapéus
- Mapa top-down com grid de "walkable tiles"
- Matar: impostor chega perto e pressiona X (ou botão X do controle)
- Condição de vitória:
  - Tripulação: completa todas as tarefas
  - Impostor: elimina todos os tripulantes (`crew.length <= 0`)

---

## Tela de Customização (Skin)

- Preview do personagem numa caixa `.pvbox` (220×220 px)
- Canvas de preview: 210×210 px, `drawCat(pCtx, 105, 130, {scale: 3.2, ...})`
- Função: `updPrev()` — atualiza ao mudar nome/cor/chapéu
- Bug antigo corrigido: o topo das orelhas estava cortado (y muito alto)

---

## Tela de Abertura (Intro)

HTML overlay `#intro-ov` com canvas `#intro-c`.

**Animação:**
- Céu estrelado com estrelas fixas e streaks diagonais passando
- Gatos voando em órbitas aleatórias, girando, com `drawCat()`
- Logo "AMONG CATS" no centro
- Texto "criado por Sarah e Sidney" piscando (animação CSS `@keyframes blink`)
- Texto "Toque para começar" / "Toque novamente para começar"

**Fluxo de interação (2 cliques):**
1. 1º clique → inicia a música, muda o texto para "Toque novamente para começar"
2. 2º clique (400ms+ depois) → pula a intro e vai para o menu

**Função `skipIntro()`:**
```javascript
window.skipIntro = function() {
  if (_done) return;
  if (!_musicStarted) {
    _musicStarted = true;
    _musicStartTime = Date.now();
    startIntroMusic();
    // muda texto do botão
    return;
  }
  if (Date.now() - _musicStartTime < 400) return;
  _doSkip();
};
```

**`go(id)`** — ao entrar no jogo chama `window.stopIntroMusic && window.stopIntroMusic()`

---

## Música da Intro

Sintetizada 100% via Web Audio API (sem arquivo de áudio).
Usa `_ac` separado do AudioContext do jogo.

**Camadas:**
- Drone grave (oscilador senoidal ~55 Hz)
- Pad atmosférico (oscilador suave ~110 Hz)
- Pulso rítmico (burst de ruído a cada ~800ms)
- Arpejo (notas ascendentes e descendentes em loop)

**Para de tocar quando:**
- Intro é pulada (`_doSkip` chama `stopIntroMusic`)
- Janela é fechada (`beforeunload`)
- Aba fica oculta (`visibilitychange`)

```javascript
window.stopIntroMusic = stopIntroMusic;
document.addEventListener('visibilitychange', () => { if (document.hidden) stopIntroMusic(); });
window.addEventListener('beforeunload', () => stopIntroMusic());
```

---

## AudioContext Compartilhado (Sons do Jogo)

Para evitar o erro "too many AudioContexts":

```javascript
let _gac = null;
function getAC() {
  if (!_gac || _gac.state === 'closed') {
    _gac = new (window.AudioContext || window.webkitAudioContext)();
  }
  if (_gac.state === 'suspended') _gac.resume();
  return _gac;
}
// Pré-aquece no primeiro gesto do usuário:
document.addEventListener('pointerdown', () => getAC(), { once: true });
document.addEventListener('keydown',     () => getAC(), { once: true });
document.addEventListener('touchstart',  () => getAC(), { once: true });
```

Todas as funções de som usam `const ac = getAC();` em vez de criar novo contexto.

---

## Sons Sintetizados

| Som | Função | Descrição técnica |
|-----|---------|-------------------|
| Passos | `_playStep()` | Burst de ruído curto, lowpass ~380/320 Hz alternando L/R, cooldown 210ms |
| Morte | `_playDeath(isPlayer)` | Sawtooth descendo em pitch + vibrato + thud (ruído lowpass ~180 Hz) |
| Miado (arranhar) | `_playMeow(tipo)` | 3 tipos rotativos via `_meowIndex % 3` |
| Peito estalado (caixa de areia) | `_playCrack()` | Ruído com envelope rápido, simula osso estalando |
| Passos nas camas/sanduíche | som suave de confirmação |

---

## Tarefas (Mini-jogos)

### 💩 Limpar Caixa de Areia (`#sandbox-ov`)
- Tela preta + caixa de areia com textura
- 5 cocôs espalhados, cada um com 5 cliques para remover
- Cursor personalizado de patinha (`_makePawCursor()`)
- Partículas de areia pretas (`#1a0a00`) voam a cada clique
- Som de "peito estalado" (`_playCrack()`) a cada clique
- Tarefa completada quando todos os 5 cocôs somem

### 🐾 Arranhar Arranhador (`scratch task`)
- Canvas preto com poste de arranhador desenhado no fundo (`drawScratchPost()`)
- Arranhões aparecem dentro da área do poste
- 3 miados diferentes rotativos a cada arranhão
- Botão mobile: `ontouchstart="scratchTap(event)"` → chama `_doScratch()`

### 🥪 Distribuir Sanduíches (Refeitório)
- Botão mobile: `ontouchstart="refeitTap(event)"` → chama `_doRefeit()`

### 🛏️ Arrumar Camas
- Sons de confirmação ao completar

---

## Controle por Gamepad (Xbox)

```javascript
// Mapeamento:
// Analógico esquerdo → mover personagem
// Botão A (index 0) → interagir (tryUse)
// Botão X (index 2) → matar (tryKill)
// Botão B (index 1) → fechar tarefa ativa

function pollGamepad() {
  const gps = navigator.getGamepads ? navigator.getGamepads() : [];
  for (const gp of gps) {
    if (!gp) continue;
    const ax = gp.axes[0], ay = gp.axes[1];
    const dead = 0.18;
    _jvx = Math.abs(ax) > dead ? ax : 0;
    _jvy = Math.abs(ay) > dead ? ay : 0;
    // ... botões A, X, B
    break;
  }
  requestAnimationFrame(pollGamepad);
}
window.addEventListener('gamepadconnected', e => {
  showNotif(`🎮 Controle conectado: ${e.gamepad.id.split('(')[0].trim()}`);
  pollGamepad();
});
```

---

## Cursor Personalizado

Gerado via canvas offscreen, exportado como data URL:

```javascript
function _makePawCursor() {
  // desenha patinha 64×64 em canvas offscreen
  // retorna data URL para usar em canvas.style.cursor
}
```

Usado nas tarefas: caixa de areia e camas.

---

## GitHub Pages

- URL: https://hellfirespriston-art.github.io/among-cats/
- Repo: https://github.com/hellfirespriston-art/among-cats
- Deploy automático pela branch `main`
- Após qualquer edição: `git add index.html && git commit -m "..." && git push`

---

## Bugs Corrigidos

| Bug | Causa | Solução |
|-----|-------|---------|
| `AudioContext not allowed to start` | AC criado antes de gesto do usuário | AC criado dentro de handler de clique (gesto) |
| Música parava imediatamente | 1º clique disparava start E skip ao mesmo tempo | Introduziu fluxo de 2 cliques com delay de 400ms |
| Intro pulava sozinha após 6s | `setTimeout(skipIntro, 6000)` ainda no código | Removido completamente |
| Impostor vencia sem matar todos | `crew.length <= 1` | Corrigido para `crew.length <= 0` |
| `beds-prog` TypeError null | Linha de update de elemento removido | Linha deletada |
| Preview do gato cortado | `y=68` muito alto para escala 2.1 | Ajustado para `y=130, scale=3.2, canvas 210×210` |
| "Too many AudioContexts" | Cada função de som criava novo AC | Shared `getAC()` com instância única `_gac` |
| Música continua após fechar janela | Sem listener `beforeunload` | Adicionado `beforeunload` + `visibilitychange` |

---

## Sistema de Visão "olhinho" + Testemunha corre pro botão 👁️🏃 (13/06/2026, revisado)

**Matar é PERMITIDO mesmo com testemunhas.** O olhinho agora é um AVISO ("se matar aqui, alguém vê e te denuncia"), não um bloqueio. Quem presencia o crime corre pro botão.

- `VIS` (dinâmico) raio de visão; `_losClear(ax,ay,bx,by)` faz raycast usando `walkable()` (parede bloqueia a linha de visão).
- `sees(o,target)` / `watched(target,ignore)` — usados só pro indicador `_drawEye` (👁️) e pro aviso "👁️ Mas alguém viu...". **Não bloqueiam mais o kill.**
- `kill(t,by)` captura as testemunhas (`wits`) **antes** de matar (pois a vítima morta deixa de ser "vista") e chama `_startRush(wits,by)`.
- `_startRush(wits,killer)`: marca `G.rushKiller=killer.id`; cada testemunha NPC ganha `toBtn=true`+`panic=true`; player testemunha vê aviso pra correr e denunciar.
- No `updBots`, bot com `toBtn` ignora a IA normal: corre 1.5× pro `EMG`, **recruta por contato** (qualquer vivo a <28px, exceto o assassino, vira `toBtn`), e ao chegar (<46px) chama `_btnPressed()`.
- `_btnPressed()`: conta `crowd` = bots vivos com `toBtn`. **1 testemunha → `meeting('accuse',…,killerId)`** → `_accuseEject` ejeta o assassino direto. **Turba (≥2) → `meeting('chaos')`** → `russianRoulette()` (sorteio).
- Player perto do 🚨 com `G.rushKiller` setado: botão vira "🚨 Denunciar!"; aperta sozinho → `accuse`; se há gente correndo (`crowd≥1`) → `chaos`.
- `_drawPanic(sx,sy)` desenha ❗ pulsante acima de quem está correndo. Flags zeradas em `meeting()` e `endMeeting()`.

## Roleta Russa — quando o PLAYER é o impostor / turba / NPC convoca (13/06/2026)

- Bots tripulantes encontram corpos (`dst(b,bo)<62`) e chamam reunião automaticamente — passam `byNpc=true` pro `meeting()`.
- Em `meeting()`, se `type==='chaos' || (G.player.isImp && alive) || byNpc` → chama `russianRoulette()` em vez da votação normal. Ou seja: turba no botão, player impostor, ou reunião convocada por NPC → sorteio. Reunião convocada pelo player inocente (sem testemunho) = votação normal. `type==='accuse'` (testemunha sozinha) → `_accuseEject` ejeta o assassino sem votar.
- `russianRoulette()` monta grade de cards de todos os vivos (sem onclick), gira um destaque que desacelera (`stops=alive.length*3+pick`) e para num sorteado → `endMeeting(alive[idx].id)` após 1400ms.
- Sons: `_playRRTick()` (blip 900Hz quadrado) e `_playRRBang()` (ruído+sawtooth = tiro).
- CSS: `.vcard.roulette-on` (destaque amarelo) e `.vcard.roulette-pick` (vermelho pulsante).

## Salas Repaginadas — estilo Among Us / The Skeld (13/06/2026)

`drawFurniture(r,sx,sy)` reescrita com mobília detalhada por sala:
- **dormit**: camas + travesseiros · **refeit**: balcão + mesa oval + tigelas
- **abrigo**: árvore de gato + casinhas + postes + novelos
- **caixa**: caixas de areia + pá · **reunioes**: anel de chão + 8 banquinhos + mesa hexagonal metálica (cafeteria)
- **lab**: balcão de frascos + maca + anel de scanner + monitores de parede · **jardim**: jardineiras + arbusto + flores + tufos de grama + regador
- `drawMap()`: parede com stroke azulado + sombra interna recortada.

## Lobby = Foguete (13/06/2026)

Tela `menu` (`#lobby-c`) — gatinhos logando/deslogando dentro de um **foguete**.
- `_lFloor()` retorna geometria do foguete (corpo, nariz, turbinas) escalada por `s=min(_lW,_lH)/680`.
- `_drawLRoom()`: fundo espacial + `_drawLStars()` (70 streaks subindo) + 2 turbinas com `_drawFlame()` (uma de cada lado em `cx±bodyW*.26`, na base) + bicos metálicos + aletas e cone vermelhos + corpo metálico com painéis/rebites + porta central com luz verde piscando + 2 vigias + assentos + caixotes.
- `_drawFlame(ctx,fx,fy,fw,t)`: chama em camadas (halo, laranja externo, núcleo branco-amarelo, base azul, faíscas) com flicker.

## Estado Atual (última sessão — 13/06/2026)

- 4 features novas acima implementadas e verificadas por amostragem de pixel/lógica.
- Tudo funcionando no GitHub Pages (após deploy manual — repo NÃO tem auto-push).
- Preview de skin ampliado (220×220), controle Xbox, música da intro 2 cliques, sons via Web Audio API.
- ⚠️ Deploy do Among Cats é **manual** — confirmar com o Sidney antes de `git push`.
