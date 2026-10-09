# Sair da Carona Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: inline (sem `design.md`: uma rota nova e componentes no padrão de delete-rides)
**Spec**: `.specs/features/leave-ride/spec.md` · **Context**: `.specs/features/leave-ride/context.md`
**Status**: In Progress

**Repositórios**:
- T1–T2 commitam em `soft-go-ii-api/` (`master`). T3–T7 commitam em `soft-go-II/` (`main`).
- O código vai no commit do submódulo. Logo depois, um commit no repo raiz leva a marcação em `tasks.md`, a rastreabilidade em `spec.md` e o bump do ponteiro.
- No fim: adicionar `removeRideUser(rideId, userId)` → `DELETE /ride-users/:rideId/users/:userId` na tabela de contrato do `CLAUDE.md` raiz.
- Sempre `git add` com caminhos explícitos. Nunca adicionar `tsconfig.build.tsbuildinfo`, `dist/` ou `.env`.

**Python**: neste Windows, use `python` (não `python3`).

**Lições aplicadas**: L-004 (mensagem exata), L-001 ("Nenhum guard global" + mapa `EXPECTED_GUARDS`), L-005 (cada elemento da lista), L-006 (rejeição confere a inscrição gravada).

**Mudança de requisito em testes existentes**: a lista de passageiros passa a começar fechada (LEAVE-10). Os testes atuais de participantes em `Card.test.tsx` e `Home.test.tsx` esperam os nomes visíveis direto. Eles serão reescritos na T6 para abrir a lista antes, mantendo as mesmas asserções de nomes e links. Isso é mudança de requisito pedida pela usuária, não enfraquecimento.

**Baseline**: API 167 testes · front 174 testes.

---

## Design inline

**API**
- `RideUsersService.remove(rideId, userId, requesterId): Promise<void>`: `rideRepository.findOneBy({ id: rideId })` → `404` "Carona não encontrada"; `requesterId !== userId && requesterId !== ride.ownerId` → `403` "Você só pode sair da carona ou remover passageiras da sua carona"; `rideUsersRepository.findOneBy({ rideId, userId })` → `404` "Esta pessoa não está nesta carona"; senão `rideUsersRepository.delete(rideUser.id)`.
- `RideUsersController.remove`: `@Delete(':rideId/users/:userId') @UseGuards(JwtAuthGuard) @ApiBearerAuth() @HttpCode(HttpStatus.NO_CONTENT)`, os dois params com `ParseIntPipe`, passa `request.user.id`.

**Front**
- `removeRideUser(rideId, userId)` em `service/ride.service.ts` → `api.delete(`/ride-users/${rideId}/users/${userId}`)`.
- `ParticipantsList` (novo): props `participants`, `showPhones`, `currentUserId?`, `isOwner`, `canRemove` (= `showButton`), `onRemove?(participant)`. Estado local `open` (começa `false`). Botão "Ver passageiros (N)" / "Esconder passageiros" com `aria-expanded`. Cada item: nome (link wa.me se `showPhones && phone`) e, se `canRemove && onRemove`, lixeira `Trash2` (`support-04`) com o nome acessível da regra LEAVE-14/15.
- `ConfirmDialog` (novo): props `title`, `confirmLabel`, `onClose`, `onConfirm: () => Promise<void>`. Visual do `ConfirmDeleteRideModal` sem o resumo. Botão de confirmar desabilitado enquanto envia.
- `Card`: troca o bloco "Vão:" por `<ParticipantsList>`. Nova prop `onRemoveParticipant?(participant)`.
- `Home`: estado `removal: { ride, participant } | null`; abre `ConfirmDialog` com o título e o botão conforme `participant.userId === user.id`; `handleRemove` chama `removeRideUser`, trata o erro como `handleDelete` e recarrega.

---

## Test Coverage Matrix

> Generated from codebase, project guidelines, and spec - confirm before Execute. Guidelines found: `soft-go-ii-api/CLAUDE.md`, `soft-go-ii-api/.claude/skills/novo-recurso/SKILL.md` (§5), `soft-go-ii-api/vitest.config.ts`, `soft-go-II/vite.config.ts` (vitest + jsdom), AD-002. Os testes de delete-rides são o piso de estilo.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| ---------- | ------------------ | -------------------- | ---------------- | ----------- |
| API service (`ride-users.service`) | unit | Todos os ramos; 1:1 com os LEAVE da tarefa; mensagem exata | `soft-go-ii-api/src/**/*.spec.ts` | `npm test` |
| API controller (HTTP) | integration (Nest testing + supertest + `setupApp`, repositórios fake) | Rota nova: feliz (passageira e dona) + 401/400/403/404 + estado persistido + guards | `soft-go-ii-api/src/**/*.controller.spec.ts` | `npm test` |
| Front lógica (`service/*`) | unit | Método e URL | `soft-go-II/src/**/*.test.ts` | `npm test` |
| Front componentes / páginas | unit (jsdom + Testing Library, serviços mockados) | Cada AC de UI: feliz + erro; cada elemento da lista; regras de visibilidade da lixeira | `soft-go-II/src/**/*.test.tsx` | `npm test` |

## Gate Check Commands

> Generated from codebase - confirm before Execute.

| Gate Level | When to Use | Command |
| ---------- | ----------- | ------- |
| Quick (API) | Tarefas com unit na API | `cd soft-go-ii-api && npm test` |
| Build (API) | Testes HTTP e fim de fase da API | `cd soft-go-ii-api && npx tsc --noEmit -p tsconfig.build.json && npm run lint && npm test` + `npx tsc --noEmit -p tsconfig.json` tolerando só o erro antigo `test/app.e2e-spec.ts(4,21) TS2307` |
| Quick (Front) | Tarefas com testes no front | `cd soft-go-II && npm test` |
| Build (Front) | Fim de fase do front | `cd soft-go-II && npm run lint && npm run build && npm test` |

---

## Execution Plan

Phases are ordered and run sequentially - each phase completes before the next begins, and tasks within a phase execute in order.

### Phase 1: API

```
T1 → T2
```

### Phase 2: Front

```
T3 → T4 → T5 → T6 → T7
```

---

## Task Breakdown

### T1: `RideUsersService.remove`

**What**: Método `remove(rideId, userId, requesterId)` conforme o design inline.
**Where**: `soft-go-ii-api/src/ride-users/ride-users.service.ts`
**Also touches**: `src/ride-users/ride-users.service.spec.ts`
**Depends on**: None
**Reuses**: `RideService.remove`; mocks do `ride-users.service.spec.ts`
**Requirement**: LEAVE-01, LEAVE-02, LEAVE-04, LEAVE-05, LEAVE-06

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes unit:
  - passageira remove a si mesma → `delete` com o id da inscrição;
  - dona remove outra pessoa → `delete` com o id da inscrição;
  - carona inexistente → `404` "Carona não encontrada", sem `delete`;
  - terceira conta → `403` com a mensagem exata, sem `delete` e sem buscar a inscrição;
  - pessoa não inscrita → `404` "Esta pessoa não está nesta carona", sem `delete`;
  - dona como alvo (pela própria dona) → `404` "Esta pessoa não está nesta carona"
- [x] Gate quick (API) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: allow leaving and removing ride passengers`
**Status**: ✅ Done — `soft-go-ii-api@28faf5f` (API: 173 testes)

---

### T2: `DELETE /ride-users/:rideId/users/:userId`

**What**: Rota `remove` conforme o design inline, com Swagger, e `remove: [JwtAuthGuard]` no mapa `EXPECTED_GUARDS`.
**Where**: `soft-go-ii-api/src/ride-users/ride-users.controller.ts`
**Also touches**: `src/ride-users/ride-users.controller.spec.ts`, `src/auth/auth.controller.spec.ts`
**Depends on**: T1
**Reuses**: `RideController.removeRide`, setup HTTP do `ride-users.controller.spec.ts`
**Requirement**: LEAVE-01, LEAVE-02, LEAVE-03, LEAVE-04, LEAVE-05, LEAVE-06, LEAVE-07, LEAVE-08, LEAVE-09

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes HTTP (repositórios fake):
  - token da passageira → `204`, corpo vazio, inscrição apagada;
  - token da dona → `204`, inscrição apagada;
  - depois da saída, a mesma passageira se inscreve de novo (`POST /ride-users` → `201`);
  - terceira conta → `403` com a mensagem exata, inscrição continua gravada;
  - carona inexistente → `404` "Carona não encontrada";
  - não inscrita → `404` "Esta pessoa não está nesta carona";
  - `abc` em `:rideId` e em `:userId` → `400`, nada apagado;
  - sem token, outro segredo, expirado, sessão revogada → `401` "Não autenticado", nada apagado;
  - mapa `EXPECTED_GUARDS` e "Nenhum guard global" passando
- [x] Verificação manual no banco local (**pedir confirmação antes**): com contas de teste novas, terceira conta → `403`; passageira sai → `204` e `GET /ride` devolve a vaga; dona remove → `204`
- [x] Gate build (API) passa (fim da Phase 1)

**Tests**: integration
**Gate**: build
**Commit**: `feat: add route to remove ride passengers`
**Status**: ✅ Done — `soft-go-ii-api@432a706` (API: 185 testes; fim da Phase 1). Checagem manual no banco local com 3 contas de teste novas: terceira → `403`; passageira sai → `204` e vagas 0 → 1; nova inscrição → `201`; dona remove → `204`; repetir → `404`. Carona de teste excluída no fim

---

### T3: `removeRideUser` no serviço do front

**What**: `removeRideUser(rideId, userId)` → `api.delete(`/ride-users/${rideId}/users/${userId}`)`.
**Where**: `soft-go-II/src/service/ride.service.ts`
**Also touches**: `src/service/ride.service.test.ts`
**Depends on**: None
**Reuses**: `deleteRide`
**Requirement**: LEAVE-19

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes unit: `removeRideUser(3, 7)` chama `api.delete("/ride-users/3/users/7")` uma vez; erro da API é propagado
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add remove ride user service`
**Status**: ✅ Done — `soft-go-II@2063e61` (front: 176 testes)

---

### T4: `ParticipantsList`

**What**: Componente da lista recolhível, conforme o design inline e LEAVE-10..16.
**Where**: `soft-go-II/src/components/ParticipantsList.tsx`
**Also touches**: `src/components/ParticipantsList.test.tsx` (novo)
**Depends on**: T3
**Reuses**: bloco "Vão:" do `Card`, lixeira do `Card`
**Requirement**: LEAVE-10, LEAVE-11, LEAVE-12, LEAVE-13, LEAVE-14, LEAVE-15, LEAVE-16

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes:
  - fechada: botão "Ver passageiros (2)" com `aria-expanded="false"`, nomes ausentes;
  - abrir: nomes aparecem, botão vira "Esconder passageiros" com `aria-expanded="true"`; fechar esconde de novo;
  - sem participantes: nem botão nem lista;
  - `showPhones` + telefone → link `https://wa.me/<telefone>`; sem `showPhones` → sem link; participante sem telefone → sem link;
  - participante logada: lixeira "Sair da carona" só no próprio nome (uma lixeira no total); clique chama `onRemove` com o participante;
  - dona: "Remover Ana da carona" e "Remover Bia da carona"; clique chama `onRemove` com o participante certo;
  - outra conta, deslogada, `canRemove=false` (dona ou participante) → nenhuma lixeira
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add collapsible participants list`
**Status**: ✅ Done — `soft-go-II@7a06fcb` (front: 187 testes)

---

### T5: `ConfirmDialog`

**What**: Modal genérico de confirmação conforme o design inline.
**Where**: `soft-go-II/src/components/ConfirmDialog.tsx`
**Also touches**: `src/components/ConfirmDialog.test.tsx` (novo)
**Depends on**: T4
**Reuses**: estrutura do `ConfirmDeleteRideModal`
**Requirement**: LEAVE-17, LEAVE-18, LEAVE-19

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes:
  - título e botão de confirmar com os textos recebidos ("Sair da carona?"/"Sair"), mais "Cancelar";
  - "Cancelar", "X" e overlay → `onClose`, sem `onConfirm`; clique dentro não fecha;
  - confirmar → `onConfirm` uma vez; pendente → botão desabilitado e segundo clique ignorado
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add generic confirm dialog`
**Status**: ✅ Done — `soft-go-II@7a7c674` (front: 194 testes)

---

### T6: `Card` com a lista recolhível

**What**: `Card` usa `ParticipantsList` no lugar do bloco "Vão:" e repassa `onRemoveParticipant`. Testes de participantes existentes no `Card` e no `Home` passam a abrir a lista antes de conferir nomes e links (mudança de requisito, ver nota no topo).
**Where**: `soft-go-II/src/components/Card.tsx`
**Also touches**: `src/components/Card.test.tsx`, `src/pages/Home.test.tsx`
**Depends on**: T5
**Reuses**: `ParticipantsList`
**Requirement**: LEAVE-14, LEAVE-15, LEAVE-16

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] Testes do Card: lista fechada por padrão; aberta mostra os nomes e links (asserções atuais mantidas); dona vê "Remover Ana da carona"; participante vê "Sair da carona"; `showButton={false}` sem lixeiras; clique chama `onRemoveParticipant` com o participante
- [ ] Testes do Home de participantes abrem a lista e mantêm as asserções atuais
- [ ] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: collapse participants list on ride card`

---

### T7: Sair/remover no Home

**What**: `Home` liga `onRemoveParticipant` ao `ConfirmDialog` e implementa `handleRemove` (LEAVE-17..21).
**Where**: `soft-go-II/src/pages/Home.tsx`
**Also touches**: `src/pages/Home.test.tsx`
**Depends on**: T6
**Reuses**: `handleDelete`, `loadRides`, `Toast`
**Requirement**: LEAVE-10, LEAVE-13, LEAVE-17, LEAVE-18, LEAVE-19, LEAVE-20, LEAVE-21

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] Testes (serviços mockados):
  - mural: card com "Ver passageiros (N)" e nomes escondidos;
  - passageira: "Sair da carona" abre "Sair da carona?"; "Sair" → `removeRideUser(3, <id dela>)`, toast "Você saiu da carona", modal fecha, recarrega e o card volta a mostrar "Vou junto";
  - dona: "Remover Bia da carona" abre "Remover Bia da carona?"; "Remover" → `removeRideUser(3, 2)`, toast "Bia foi removida da carona", recarrega;
  - "Cancelar" → sem chamada;
  - `403`/`404` → toast com a mensagem da API, fecha, recarrega;
  - `500` → toast "Não foi possível remover da carona. Tente novamente.";
  - `401` → nenhum toast do Home
- [ ] Gate build (front) passa (fim da Phase 2)

**Tests**: unit
**Gate**: build
**Commit**: `feat: let passengers leave and owners remove passengers`

---

## Phase Execution Map

```
Phase 1 → Phase 2

Phase 1 (API):   T1 → T2
Phase 2 (Front): T3 → T4 → T5 → T6 → T7
```

Execution is strictly sequential. 7 tarefas: um lote só, execução inline. O Verifier roda no fim.

---

## Task Granularity Check

| Task | Scope | Status |
| ---- | ----- | ------ |
| T1 | 1 método | ✅ |
| T2 | 1 rota | ✅ |
| T3 | 1 função | ✅ |
| T4 | 1 componente novo | ✅ |
| T5 | 1 componente novo | ✅ |
| T6 | 1 componente (+ ajuste de testes existentes) | ⚠️ coeso |
| T7 | 1 página | ✅ |

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| ---- | ---------------------- | ------------- | ------ |
| T1 | None | início da Phase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | None | início da Phase 2 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| ---- | --------------------------- | --------------- | --------- | ------ |
| T1 | API service | unit | unit | ✅ |
| T2 | API controller | integration | integration | ✅ |
| T3 | Front lógica | unit | unit | ✅ |
| T4 | Front componente | unit | unit | ✅ |
| T5 | Front componente | unit | unit | ✅ |
| T6 | Front componente | unit | unit | ✅ |
| T7 | Front página | unit | unit | ✅ |

## Requirement Coverage

| Requisito | Tarefas |
| --------- | ------- |
| LEAVE-01 | T1, T2 |
| LEAVE-02 | T1, T2 |
| LEAVE-03 | T2 |
| LEAVE-04 | T1, T2 |
| LEAVE-05 | T1, T2 |
| LEAVE-06 | T1, T2 |
| LEAVE-07 | T2 |
| LEAVE-08 | T2 |
| LEAVE-09 | T2 |
| LEAVE-10 | T4, T7 |
| LEAVE-11 | T4 |
| LEAVE-12 | T4 |
| LEAVE-13 | T4, T7 |
| LEAVE-14 | T4, T6 |
| LEAVE-15 | T4, T6 |
| LEAVE-16 | T4, T6 |
| LEAVE-17 | T5, T7 |
| LEAVE-18 | T5, T7 |
| LEAVE-19 | T3, T5, T7 |
| LEAVE-20 | T7 |
| LEAVE-21 | T7 |
