# Design — Runtime mypeople (macOS) → 100% conformidade com o seed

**Data:** 2026-06-24
**Alvo:** o runtime VIVO em `~/mypeople` (macOS/launchd), orquestrando 6 projetos
(basebjj, gotrade, hecktor, kaxa, main, scout26) agora.
**Autoridade da spec:** `~/mypeople-repo/seeds/mypeople.seed.md` (§4–§8, §A.1–§A.3, gates J*).
**Backup base existente:** `~/mypeople-backup-20260624-13277`.

## 1. Objetivo

Trazer cada componente do runtime a 100% dos contratos do seed, **preservando** a
camada custom multi-projeto (que o seed single-project não tem) e respeitando o fato
de que o runtime roda em macOS/launchd, NÃO num container Debian.

## 2. Decisões travadas (CEO, 2026-06-24)

- **D1 — Conformidade LITERAL ao seed.** Inclui contratos NEGATIVOS: remover
  subtarefas (`addsub`/`subtoggle`), remover `reorder`, e MIGRAR o enum de estados
  (`needs_brainstorm/todo/in_progress/blocked/done` → `idle/working/review/done/blocked/cancelled`).
  Aceita-se perda de subtarefas como recurso.
- **D2 — Cookie de sessão (§5.12) AGORA**, no mesmo lote. Remover o secret injetado
  das páginas; migrar HUD + todo + drag-drop de upload para auth por cookie httpOnly.

## 3. Invariantes que NÃO podem quebrar (camada custom + macOS)

- **Multi-projeto routing.** `boss_ping(target, …)` por projeto; `list_bosses()`;
  `/todo/bosses`; campos `project`/`boss` por task; sessão `mc-<proj>:<tab>`; supervisor
  config-driven (`run/supervised-bosses.json`); HUD pill `projeto` + role 👑/🔧;
  kanban com picker de projeto.
- **`boss_ping` SEMPRE roteia para `t["boss"]`** (o boss do projeto), NUNCA um
  `BOSS_AGENT` global. Toda linha portada do reference single-project é mina.
- **Runtime Boss doctrine custom:** Rule 0 (hook enforcement coordinator-not-engineer),
  Rule 4 (delegation handoff anti-duplicação), seção "Your environment". Preservar verbatim.
- **Pin (§7.3) + jump-to-latest (§7.4):** runtime está NA FRENTE do reference. Manter.
- **drag-drop de upload de imagem:** recurso custom hard-won; não pode regredir na
  migração para cookie.
- **macOS:** launchd (não container). NÃO portar: tailscaled userland/socket, `/dev/net/tun`,
  `TS_AUTHKEY`, `--accept-dns=false`, `setsid` como ciclo primário (launchd KeepAlive substitui),
  fleet/uplink JOIN (standalone). `pbcopy` é real no macOS — manter.

## 4. Trabalho por componente

### 4.1 Infra/scripts (🟢 baixo risco)
Fonte: gap infra. Quase conforme.
- **G2 (portar):** ttyd usa `tmux attach` cru (`run-daemon.sh:14`, `start-all.sh:31`) →
  bug §5.7 "abre janela errada". Trocar por attach-helper de sessão agrupada:
  `tmux new-session -t mc-<sess> -s _v_<tab>_<uniq> \; select-window -t <tab> \; set destroy-unattached on`.
  Confirmar wiring de `?arg=` contra as URLs reais de attach do HUD/mp antes.
- **G1/G3 (opcional, baixa prio):** `nohup`/`setsid` self-guard no supervisor;
  surface de erro mais alto quando Boss não sobe.
- **tmux.conf:** 100% conforme. Nada a fazer.
- **Preservar:** supervisor multi-boss (`boss-supervisor.sh:2-9,14,43-64`).

### 4.2 Boss doctrine `boss-CLAUDE.md` (🟡 médio)
Fonte: gap doctrine. Merge de 3 vias (seed §6/§8 + de-staled + custom runtime).
- **Adicionar (do seed §8 + TODO doctrine, de-staled):** Rule 21 (só CEO marca `done`;
  AI no máx `review`); board-é-a-fila (workToDone ON + state≠done, top-first); reconcile
  pass (idle→working→review→done, verify-against-proof); autonomous spawn-on-ping (J32);
  keyword-bearing durable roster summary (J2c); one-question-per-turn; done-pending-CEO→blocked;
  chain-of-command (CEO→Boss relay); WhatsApp digest (server-driven); verification authority.
- **Preservar verbatim:** Rule 0, Rule 4-delegation, "Your environment".
- ⚠️ **Colisão Rule 4:** runtime Rule 4 = delegation; seed Rule 4 = board-é-fila.
  Renumerar/renomear um (manter delegation como regra nomeada; board-queue em outra seção).
- ⚠️ NÃO importar `boss-rule4-todo.md` cru (ele ainda ensina brainstorm gate que o seed
  cortou em 18/06). Usar a versão pós-18/06 do seed §6.
- ⚠️ As POSTs `/todo/*` que a doctrine manda o Boss fazer precisam passar por canal
  permitido pelo hook do Rule 0 (verificar que não são Bash não-`mp` bloqueado).

### 4.3 HUD `dashboard.html` (🟡 médio)
Fonte: gap HUD. Aqui "100%" = REMOVER coisa.
- **Remover (FAIL gates):** grade de máquinas inteira (CSS `:24-40`, seção "Máquinas"
  `:66-67`, `renderMachines` `:104-140`, fetch `/clients` `:191`) — proibida §7.1/J11.
  E `@keyframes pulse` + `animation` em `.b-hydrating` (`:35,:37`) — FAIL J29.
  (Remover a grade já mata a animação no mesmo golpe.)
- **Adicionar:** pill live/stale (F22); meta = contagem de AGENTES (não de máquinas).
- **Corrigir:** URL de ATTACH nas linhas de agente → `<attach_base>/?arg=-t&arg=<tmux_target>`
  (J8), hoje é `/attach?target=` relativo; endpoint Revive → `POST /revive{agent_id}`
  (F21/J25), hoje é `/task/submit{action:spawn}`.
- **Preservar (aditivo, superset de F20):** pill `projeto` (`projColor` `:87-90`, `:150-151`),
  coluna `tipo` (`:131`), role 👑/🔧 (`:146-149`). Edits column-local em `renderAgents`/`revive`.
- Secret → ver §5 (cookie).

### 4.4 Queue layer `queue-server.py` + `queue-client.py` (🔴 alto — coração, vivo)
Fonte: gap queue. Edits cirúrgicos (runtime predate o seed).
- **queue-server — portar:** C3 attach_url join no `/agents` (usar o `tmux_target`
  custom `mc-<sess>:<tab>`, não inventar); C7 task types `answer`+`revive` no submit;
  C12 `POST /revive` (limpar do dict `retired` preservando `cwd`); C15 `/favicon.ico`→204;
  C16 reaper/stale-expiry (thread; ao expirar, MOVER p/ `retired` preservando `cwd`,
  não hard-delete — mantém tabela Revive multi-projeto).
- **queue-client — portar:** C21 desabilitar `automatic-rename`/`allow-rename` +
  re-assert `rename-window` pós-create (senão attach-by-name quebra); C24 segundo Enter
  ~0.4s pós bracketed-paste; C26 export `LANG`/`LC_ALL=C.UTF-8` no env de spawn;
  C25 handlers `answer`/`revive`.
- **C22/C23 (dialog dismiss + per-cwd trust):** spec exige, mas não morde neste Mac já
  onboardado. Portar por correção (máquina limpa), prioridade baixa.
- **Preservar:** todo o routing `mc-<proj>` (`server:20-23,260-276,279-284,329-338`;
  `client:44-67,159-299`); retired+revive-com-cwd; re-announce preserva boss_id/backend.

### 4.5 Todo board `todo-server.py` + `todos.html` (🔴 alto)
Fonte: gap todo (autoridade = SEED, não o reference que está stale).
- **Migrar (D1):** `STATES` → `idle|working|review|done|blocked|cancelled`; task nasce
  `idle`. **MIGRAÇÃO DE DADOS** do `board.v2.json` vivo (ver §6). Remapear o kanban custom
  pras novas colunas mantendo picker de projeto.
- **Remover (D1, contratos negativos):** ops `addsub`/`subtoggle`/`reorder` + UI de
  subtarefas (`todos.html:184-198`).
- **Adicionar (backend):** comment→Boss ping em TODA comment (exceto Boss-autor + `{test}`),
  log em `boss-inbox.log` — **roteando p/ `t["boss"]`** (G3, mais crítico); `unread` int
  (incrementa em comment não-CEO, retorna no `/todo/board`); `/todo/proof` real
  `{kind,url,body,ts}` com classificação server-side de `kind` por extensão (nunca default
  `text`) + serve binário; campo `verified` + `done` só-CEO (Rule 21); `set` ganha
  `doneCondition`/`workToDone`/`done`.
- **Adicionar (UI):** modal com done-condition + toggle work-to-done + controle de proof
  (file picker/URL) + badge verified + badge unread; render incremental preservando
  foco+caret+campos sujos (trocar o `innerHTML=''` full rebuild); poll ≤~3s; Esc fecha modal.
- **Preservar:** pin+jump (já tem); `boss_ping(target,…)` por projeto; `list_bosses`/
  `/todo/bosses`; campos `project`/`boss`/`pinned`/`pinRank` no `/todo/board`.
- ⚠️ Ao portar `apply_comment`/watchdog/cron/auto-retire do reference: cada `_mp_send(BOSS_AGENT,…)`
  vira `t["boss"]`; o "exempt Boss-autor" compara contra o boss do PROJETO, não global.

## 5. Cross-cutting: cookie de sessão §5.12 (D2)

Mudança coordenada em 4 superfícies. Maior risco do projeto.
- **queue-server:** ao servir `GET /` `/todos` `/dashboard`, emitir exatamente UM
  `Set-Cookie: mp_session=<random>; HttpOnly`; endpoints gated aceitam cookie OU header
  `X-Queue-Secret`; remover toda injeção de `__INJECT_SECRET__`. Self-check J30:
  `curl -sI .../dashboard | grep -ic '^set-cookie:'` → `1`.
- **todo-server:** mesma emissão de cookie nas páginas; `/todo/*` aceita cookie.
- **dashboard.html / todos.html:** remover `SECRET`/`__INJECT_SECRET__` do JS; fetches
  passam a depender do cookie (same-origin, credentials).
- ⚠️ **drag-drop upload:** `/upload` HOJE usa o secret injetado. DEVE passar a aceitar o
  cookie senão o recurso regride. Testar upload ponta-a-ponta na validação.
- **Risco de lockout:** se a auth por cookie falhar, o board vivo fica inacessível.
  Mitigação: manter o header `X-Queue-Secret` como caminho de auth ACEITO em paralelo
  (não remover do servidor), só parar de INJETAR na página. Assim há fallback.

## 6. Migração de dados — `board.v2.json` (do D1)

O board vivo tem tasks em estados antigos. Mapa de migração:
- `needs_brainstorm` → `idle`
- `todo` → `working` se `workToDone` ON, senão `idle` (default decidido)
- `in_progress` → `working`
- `blocked` → `blocked` (mantém)
- `done` → `done` (mantém)
- subtarefas existentes (D1 remove o recurso): a migração emite um comentário no card
  `"[migração] subtarefas: <texto> (done/pending)"` por subtarefa, para não perder o
  conteúdo silenciosamente, depois dropa o array `subs`.
- Migração é forward-only, com backup do `board.v2.json` antes. Idempotente (re-rodar
  não re-migra estado já novo).

## 7. Estratégia de execução (sistema VIVO)

Princípio: cada componente é testado FORA do launchd antes de trocar o vivo. Backup +
rollback por componente.
1. **Backup fresco** de `~/mypeople` inteiro antes de começar (além do existente).
2. **Ordem por risco crescente:**
   (a) infra ttyd helper → (b) doctrine → (c) HUD → (d) todo board (+ migração dados) →
   (e) queue layer → (f) cookie auth (cross-file, por último, com fallback header).
3. **Por componente:** editar cópia → rodar standalone numa porta de teste → validar
   contra os gates J* relevantes → `launchctl` swap (unload/load do plist) → verificar vivo →
   só então próximo.
4. **Verificação = trust-the-pane:** não confiar em self-report; provar via curl/pane real
   (ex: J30 cookie count, J8 attach URL, comment-ping aparece em `boss-inbox.log` no boss
   do projeto certo, drag-drop sobe imagem).
5. **Rollback:** restaurar arquivo do backup + `launchctl` reload. Cookie auth tem o
   fallback do header como rede de segurança.

## 8. Riscos principais

- **R1 — cross-routing de boss_ping** (todo/queue): ping vai pro boss errado entre projetos.
  Mitigação: grep por todo `BOSS_AGENT`/`_mp_send` portado; teste com 2 projetos.
- **R2 — lockout por cookie auth**: board vivo inacessível. Mitigação: header fallback.
- **R3 — migração de estados corromper board vivo**: backup + idempotência + dry-run.
- **R4 — drag-drop regredir** na migração cookie. Mitigação: teste E2E de upload.
- **R5 — derrubar agentes ativos** ao swap de launchd. Mitigação: queue-server reaper +
  re-announce reconstrói registro; swap em janela de baixa atividade; supervisor mantém bosses.

## 9. Fora de escopo (container-only, corretamente ausente no macOS)

tailscaled userland/socket, `/dev/net/tun`/`NET_ADMIN`, `TS_AUTHKEY` join,
`--accept-dns=false`, fleet/uplink JOIN (`UPSTREAM_QUEUE_URL`), `setsid` como ciclo
primário, instalação apt de binários. Não portar.

## 10. Verificação final (gates seed relevantes)

J3 (add born idle + ping), J5 (comment), J6 (cross-nav), J7 (attach), J8 (attach_url),
J20 (no brainstorm gate), J21 (unread), J22 (proof kind classify), J23 (no subtasks/deps),
J24 (verified), J25 (revive), J29 (no animation), J30 (one set-cookie), J31 (favicon/modal),
J32 (comment→Boss ping), J33/J34 (hot-reload + focus/caret), J37 (pin max5), J38 (jump-to-latest).
Mais: drag-drop upload via cookie; multi-projeto isolation (ping no boss certo).
