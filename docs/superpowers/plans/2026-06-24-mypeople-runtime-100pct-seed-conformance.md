# Runtime mypeople → 100% conformidade seed — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Trazer o runtime macOS `~/mypeople` a 100% dos contratos do seed, preservando a camada custom multi-projeto, com execução staged no sistema vivo.

**Architecture:** Edits cirúrgicos nos arquivos do runtime (não regen do seed). Cada componente é editado, testado standalone numa porta de teste, verificado contra os gates J*, e só então trocado no launchd via `launchctl kickstart -k`. Backup por componente; rollback = restaurar arquivo + kickstart.

**Tech Stack:** Python 3 (stdlib http.server), HTML/JS vanilla, bash, tmux, launchd. SEM suíte de testes — verificação é `curl`/`grep`/painel real (trust-the-pane).

## Global Constraints

- Spec: `docs/superpowers/specs/2026-06-24-mypeople-runtime-100pct-seed-conformance-design.md` (ler antes de cada tarefa).
- Backup base: `~/mypeople-backup-preconform-20260624-153530`.
- **`boss_ping` SEMPRE roteia para `t["boss"]`** (boss do projeto), nunca `BOSS_AGENT` global.
- **Preservar multi-projeto:** `list_bosses`/`/todo/bosses`, campos `project`/`boss`, routing `mc-<proj>:<tab>`, picker de projeto, supervisor config-driven, HUD pill `projeto`+role, pin §7.3, jump §7.4.
- **macOS:** NÃO portar nada container-only (tailscaled/TUN/fleet/setsid-primário).
- Cada daemon roda via `run-daemon.sh <name>`; swap = editar + `launchctl kickstart -k gui/$(id -u)/com.mypeople.<name>`.
- Porta/secret em `~/.config/mypeople/queue.env` (`QUEUE_PORT`, `TTYD_PORT`, `QUEUE_SECRET`).
- Cookie auth (D2): página NÃO carrega secret; servidor aceita cookie `mp_session` OU header `X-Queue-Secret` (header fica como fallback anti-lockout).

---

### Task 1: Infra — ttyd grouped-session attach-helper (🟢)

**Files:**
- Modify: `~/mypeople/bin/run-daemon.sh:14`
- Modify: `~/mypeople/bin/start-all.sh:31`
- Create: `~/mypeople/bin/attach-helper.sh`

**Interfaces:**
- Produces: `attach-helper.sh <tmux_target>` onde `tmux_target = mc-<sess>:<tab>` → cria sessão de view agrupada e seleciona a janela.

- [ ] **Step 1:** Criar `attach-helper.sh`:
```bash
#!/bin/bash
# §5.7: grouped-session attach — evita o bug "abre janela errada".
# Recebe o alvo via env TTYD_ARG ou $1 (ttyd passa o arg da URL ?arg=).
T="${1:-$TTYD_ARG}"                       # esperado: mc-<sess>:<tab>
SESS="${T%%:*}"; TAB="${T#*:}"
[ -z "$SESS" ] && exec tmux attach        # fallback
U="_v_${TAB}_$$"
exec tmux new-session -t "$SESS" -s "$U" \; select-window -t "$TAB" \; set destroy-unattached on
```
- [ ] **Step 2:** `chmod +x ~/mypeople/bin/attach-helper.sh`
- [ ] **Step 3:** Em `run-daemon.sh:14` e `start-all.sh:31`, trocar `ttyd ... -a -W tmux attach` por `ttyd ... -a -W $INSTALL_DIR/bin/attach-helper.sh` (manter `-W -a -p $TTYD_PORT`). Confirmar como o HUD passa `?arg=-t&arg=mc-...` e ajustar o parse do helper se ttyd repassar `-t` + alvo como dois args.
- [ ] **Step 4 (verify):** Swap: `launchctl kickstart -k gui/$(id -u)/com.mypeople.ttyd`. Abrir `http://localhost:$TTYD_PORT/?arg=-t&arg=mc-main:Boss` em duas abas; confirmar que cada aba mostra a janela Boss e abrir um worker noutra aba NÃO yanka a do Boss. Expected: cada view independente.
- [ ] **Step 5:** Commit no repo de design (não há git no runtime; registrar diff do arquivo no spec/plan dir).

---

### Task 2: Boss doctrine `boss-CLAUDE.md` (🟡)

**Files:**
- Modify: `~/mypeople/boss-CLAUDE.md`

**Interfaces:**
- Consumes: estados novos do Task 5 (idle/working/review/done), comment→Boss ping do Task 5.
- Produces: doctrine que o Boss carrega no onboarding (lido por `client.py:215-226` no spawn `--master`).

- [ ] **Step 1:** Preservar verbatim: Rule 0 (hook enforcement), Rule 4-delegation, "Your environment".
- [ ] **Step 2:** Renomear a Rule 4 de board para evitar colisão: manter "Rule 4 — Delegation handoff"; adicionar nova seção "**The board is your queue**" (não numerar 4).
- [ ] **Step 3:** Adicionar (texto do seed §6/§8 pós-18/06, NÃO de `boss-rule4-todo.md` cru): Rule 21 (só CEO marca `done`; AI no máx `review`); board-é-fila (workToDone ON + state≠done, top-first); reconcile pass (idle→working→review→done, verify-against-proof); autonomous spawn-on-ping (J32); durable roster summary com ≥2 keywords (J2c); one-question-per-turn; done-pending-CEO→blocked; chain-of-command (CEO→Boss relay); WhatsApp digest (server-driven); verification authority.
- [ ] **Step 4:** Ajustar as instruções de `/todo/*` POST para usar canal permitido pelo hook do Rule 0 (via `mp`, não Bash cru).
- [ ] **Step 5 (verify):** `grep -c "Rule 0\|Delegation handoff\|Your environment" ~/mypeople/boss-CLAUDE.md` → ≥3 (custom preservado). `grep -ci "only the CEO\|review\|workToDone\|chain of command" ~/mypeople/boss-CLAUDE.md` → ≥4. Confirmar 1 só "Rule 4".
- [ ] **Step 6:** Doctrine só vale em novo onboarding; não força respawn agora (aplica no próximo `--master`).

---

### Task 3: HUD `dashboard.html` (🟡)

**Files:**
- Modify: `~/mypeople/bin/dashboard.html`

- [ ] **Step 1:** REMOVER grade de máquinas: CSS `:24-40` (`.grp/.grid/.mcard/.badge/.b-hydrating/@keyframes pulse`), seção "Máquinas" `:66-67`, `renderMachines` `:104-140`, e o fetch `/clients` em `:191`. Preservar `.pill/.role/projColor` (`:52-54,:87-90`).
- [ ] **Step 2:** Meta line → contagem de AGENTES (de `/agents`), não de máquinas (`:62,:195-196`).
- [ ] **Step 3:** Adicionar pill live/stale (F22) baseado em frescor de `/agents`.
- [ ] **Step 4:** Corrigir ATTACH nas linhas de agente (`:143-144,:159`): URL = `<attach_base>/?arg=-t&arg=<tmux_target>` (J8), usando `attach_url` que o servidor passa a emitir (Task 4 C3). Corrigir Revive (`:165-170`) → `POST /revive {agent_id}`.
- [ ] **Step 5:** Preservar coluna `projeto`/`tipo`/role (superset de F20) — edits column-local em `renderAgents`.
- [ ] **Step 6:** Remover `SECRET`/`__INJECT_SECRET__` do JS (cookie auth, Task 6); fetches same-origin com `credentials:'same-origin'`.
- [ ] **Step 7 (verify):** Swap todo+queue primeiro não necessário; servir dashboard standalone e: `grep -c "@keyframes\|animation:" dashboard.html` → 0 (J29). `grep -c "renderMachines\|Máquinas" dashboard.html` → 0 (§7.1). Abrir HUD: pill live/stale visível, contagem de agentes, ATTACH abre painel certo.

---

### Task 4: Queue layer `queue-server.py` + `queue-client.py` (🔴, sem cookie ainda)

**Files:**
- Modify: `~/mypeople/bin/queue-server.py`, `~/mypeople/bin/queue-client.py`

**Interfaces:**
- Produces: `/agents` rows com `attach_url`; `POST /revive`; task types `answer`/`revive`; `/favicon.ico`.

- [ ] **Step 1 (attach_url join, C3):** No loop `/agents` (`server:254-278`), após montar `tmux_target = f"{mc_sess}:{tab}"`, buscar `base = clients.get(host,{}).get("attach_base","")` e emitir `attach_url = f"{base}/?arg=-t&arg={tmux_target}"` quando `base`. Preservar `mc_sess` custom.
- [ ] **Step 2 (task types, C7/C25):** Adicionar `"answer","revive"` ao set permitido no submit (`server:425-428`); handlers correspondentes em `client.py:302 HANDLERS`.
- [ ] **Step 3 (`POST /revive`, C12):** Rota nova: `retired.pop(agent_id, None)` preservando o `cwd` (não hard-delete a entrada de cwd); refletir no próximo `/roster`.
- [ ] **Step 4 (`/favicon.ico`, C15):** Rota → 204.
- [ ] **Step 5 (reaper, C16):** Thread de sweep: agentes/clients com `last_seen` > `QUEUE_DEAD_AFTER` (≈4 heartbeats) → MOVER para `retired` preservando `cwd` (não hard-delete). Não tocar self-heal/re-announce.
- [ ] **Step 6 (client spawn, C21):** Após `new-window`/`new-session`, emitir `tmux set-option -w automatic-rename off \; set-option -w allow-rename off \; rename-window <tab>`.
- [ ] **Step 7 (client paste, C24):** Em `tmux_send_text` (`client:113-133`), após o Enter, esperar ~0.4s e mandar segundo Enter.
- [ ] **Step 8 (client UTF-8, C26):** No `env_parts` de spawn (`client:187-195`), exportar `LANG=C.UTF-8 LC_ALL=C.UTF-8`.
- [ ] **Step 9 (preservar):** NÃO mexer no routing `mc-<proj>` (`server:20-23,260-276,329-338`; `client:44-67,159-299`), retired+revive-com-cwd, re-announce boss_id/backend.
- [ ] **Step 10 (verify standalone numa porta de teste):** subir queue-server numa `QUEUE_PORT` de teste. `curl :PORT/agents | jq '.[0].attach_url'` → string `<base>/?arg=-t&arg=mc-...`. `curl -X POST :PORT/revive -d '{"agent_id":"..."}'` → ok. `curl -sI :PORT/favicon.ico | head -1` → 204. Esperar reaper expirar um cliente fake e ir pra `/roster` retired.
- [ ] **Step 11:** Swap: `launchctl kickstart -k` queue-server e queue-client. Confirmar que os 6 projetos seguem registrados (`curl :QUEUE_PORT/agents | jq 'length'` estável).

---

### Task 5: Todo board `todo-server.py` + `todos.html` + migração de dados (🔴)

**Files:**
- Modify: `~/mypeople/bin/todo-server.py`, `~/mypeople/bin/todos.html`
- Create: `~/mypeople/bin/migrate-board-states.py`
- Data: `~/mypeople/todos/board.v2.json`

**Interfaces:**
- Consumes: `boss_ping(target,…)` custom existente.
- Produces: board com estados `idle|working|review|done|blocked|cancelled`, `unread`, proofs `{kind,url,body,ts}`, `verified`.

- [ ] **Step 1 (migração — PRIMEIRO, com backup):** Criar `migrate-board-states.py`: backup `board.v2.json` → `.bak-<ts>`; mapear `needs_brainstorm→idle`, `todo→(working se workToDone ON senão idle)`, `in_progress→working`, `blocked/done` mantém; para cada subtask em `subs`, append comment `"[migração] subtarefa: <texto> (done|pending)"`, depois `del t["subs"]`; idempotente (não re-migra estado já novo). Rodar; verificar `jq '[.tasks[].state]|unique' board.v2.json` só contém estados novos.
- [ ] **Step 2 (state enum):** `STATES` (`:24`) → `("idle","working","review","done","blocked","cancelled")`; `add` nasce `idle` (`:212`).
- [ ] **Step 3 (remover negativos):** deletar ops `addsub`/`subtoggle` (`:272-292`) e `reorder` (`:246-252`); remover UI de subtarefas (`todos.html:184-198`).
- [ ] **Step 4 (comment→Boss ping, G3/J32):** No `/todo/comment` (`:295-311`), após append, pingar o Boss DO PROJETO (`boss_ping(t["boss"], tid, title, reason)`), exceto se autor == boss do projeto ou `t.test`; logar em `boss-inbox.log`.
- [ ] **Step 5 (unread, C7):** campo `unread` int default 0; incrementa em comment não-CEO; retornar no `/todo/board`.
- [ ] **Step 6 (proofs, C8):** `/todo/proof` real: armazenar `{kind,url,body,ts}`; classificar `kind` por extensão/content-type (img/video/link/text), nunca default `text`; servir binário com Range.
- [ ] **Step 7 (verified + Rule 21, C13):** campo `verified`; `set state=done` só se `by=="CEO"` (senão erro "only CEO marks done"); AI no máx `review`.
- [ ] **Step 8 (`set` fields):** adicionar `doneCondition`/`workToDone`/`done`.
- [ ] **Step 9 (UI modal):** done-condition + toggle work-to-done + controle de proof (file/URL) + badge verified + badge unread; remapear o kanban custom (5-col) pras novas colunas mantendo picker de projeto; render incremental preservando foco+caret (trocar `innerHTML=''`); poll ≤3s; Esc fecha modal.
- [ ] **Step 10 (preservar):** `boss_ping(target,…)`, `list_bosses`/`/todo/bosses`, `project`/`boss`/`pinned`/`pinRank`, pin+jump.
- [ ] **Step 11 (remover secret do JS):** `todos.html:95` (cookie auth, Task 6).
- [ ] **Step 12 (verify standalone):** porta de teste. `curl -X POST :P/todo -d '{"op":"add","text":"x","project":"...","boss":"..."}'` → estado `idle`. Comentar não-CEO → `unread` incrementa E aparece linha em `boss-inbox.log` com o boss DO PROJETO. `set state=done by!=CEO` → erro. proof de `.png` → `kind:"image"`. `grep -c "addsub\|subtoggle\|reorder" todo-server.py` → 0.
- [ ] **Step 13:** Swap todo-server; abrir board: kanban novo, sem subtarefas, pin/jump intactos, picker de projeto ok.

---

### Task 6: Cookie de sessão §5.12 (🔴 maior risco — por último)

**Files:**
- Modify: `~/mypeople/bin/queue-server.py`, `~/mypeople/bin/todo-server.py`, `~/mypeople/bin/dashboard.html`, `~/mypeople/bin/todos.html`

**Interfaces:**
- Produces: páginas com `Set-Cookie: mp_session=<rand>; HttpOnly`; auth aceita cookie OU header.

- [ ] **Step 1 (server helper):** Função compartilhada `check_auth(headers)`: aceita se cookie `mp_session` ∈ set de sessões válidas OU header `X-Queue-Secret == QUEUE_SECRET`. Manter o header como caminho aceito (fallback anti-lockout).
- [ ] **Step 2 (mint cookie):** Em `GET /`, `/todos`, `/dashboard` (ambos servers), emitir exatamente UM `Set-Cookie: mp_session=<token aleatório>; HttpOnly; Path=/; SameSite=Strict`; registrar token no set de sessões.
- [ ] **Step 3 (de-secret páginas):** remover toda injeção `__INJECT_SECRET__` de `dashboard.html`/`todos.html`/`/attach`; o JS passa a confiar no cookie (`credentials:'same-origin'`).
- [ ] **Step 4 (drag-drop upload):** `/upload` (`server:329-338` área) passa a aceitar o cookie via `check_auth`; testar upload de imagem ponta-a-ponta.
- [ ] **Step 5 (verify J30):** `curl -sI http://localhost:$QUEUE_PORT/dashboard | grep -ic '^set-cookie:'` → `1`. Idem `/todos` no todo-server. Página sem secret: `grep -c "__INJECT_SECRET__" dashboard.html todos.html` → 0. Drag-drop sobe imagem. Board e HUD acessíveis (cookie). Header ainda funciona (fallback): `curl -H "X-Queue-Secret: $QUEUE_SECRET" :P/agents` → ok.
- [ ] **Step 6:** Swap final queue-server + todo-server. Smoke: HUD carrega, board carrega, drag-drop ok, agentes registrados, comment-ping no boss certo.

---

## Verificação final (gates seed)

Rodar o checklist §10 do spec: J3, J5, J6, J7, J8, J20, J21, J22, J23, J24, J25, J29, J30, J31, J32, J33/J34, J37, J38 + drag-drop via cookie + isolamento multi-projeto (ping no boss certo de 2 projetos).

## Rollback

Por componente: `cp ~/mypeople-backup-preconform-20260624-153530/bin/<arquivo> ~/mypeople/bin/ && launchctl kickstart -k gui/$(id -u)/com.mypeople.<svc>`. Migração de dados: restaurar `board.v2.json.bak-<ts>`. Cookie: header fallback evita lockout enquanto se reverte.
