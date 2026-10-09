# Excluir Carona Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: inline (sem `design.md`: uma rota nova num controller existente, sem migration nem padrão novo)
**Spec**: `.specs/features/delete-rides/spec.md` · **Context**: `.specs/features/delete-rides/context.md`
**Status**: Done (2026-10-08) — T1–T7 concluídas; Verifier PASS (`validation.md`)

**Repositórios**:
- T1–T2 commitam em `soft-go-ii-api/` (`master`). T3–T6 commitam em `soft-go-II/` (`main`).
- O código vai no commit do submódulo. Logo depois, um commit no repo raiz leva a marcação em `tasks.md`, a rastreabilidade em `spec.md` e o bump do ponteiro.
- No fim (junto do último commit raiz): adicionar `deleteRide(id)` → `DELETE /ride/:id` na tabela "Contrato entre front e API" do `CLAUDE.md` raiz.
- Sempre `git add` com caminhos explícitos. Nunca adicionar `tsconfig.build.tsbuildinfo`, `dist/` ou `.env`.

**Python**: neste Windows, use `python` ou `py` (não `python3`).

**Lições aplicadas**:
- L-004 (confirmada): testes conferem a mensagem exata de cada erro.
- L-001: manter o teste "Nenhum guard global".
- L-002: cascade conferido por metadata (`getMetadataArgsStorage`), porque os testes usam repositório fake.
- L-005: testar cada elemento que o modal lista.
- L-006: rejeições (`403`, `401`, `400`) conferem que a carona continua gravada.

**Baseline**: API 153 testes · front 145 testes.

---

## Design inline

**API**
- `RideService.remove(id: number, userId: number): Promise<void>`: `findOneBy({ id })` → `NotFoundException('Carona não encontrada')`; `ride.ownerId !== userId` → `ForbiddenException('Só a dona da carona pode excluí-la')`; depois `rideRepository.delete(id)`. As inscrições saem pelo `ON DELETE CASCADE` do banco, sem apagar `ride_users` no código.
- `RideController.removeRide`: `@Delete(':id') @UseGuards(JwtAuthGuard) @ApiBearerAuth() @HttpCode(HttpStatus.NO_CONTENT)`, `@Param('id', ParseIntPipe)`, passa `request.user.id`. Swagger: `@ApiResponse` para 204/401/403/404.

**Front**
- `deleteRide(id: number): Promise<void>` em `service/ride.service.ts` → `api.delete(`/ride/${id}`)`.
- `Card`: prop `onDeleteRide?: () => void`. Com `showButton && isOwner && onDeleteRide`, mostra o botão "Excluir carona" (`Trash2`) ao lado do selo "Sua carona".
- `ConfirmDeleteRideModal` (novo): props `ride` (`Pick<Ride, "id" | "owner" | "city" | "hour" | "transportType" | "participants">`), `onClose`, `onConfirm: () => Promise<void>`. Mensagens da spec (DEL-10/11/12). O botão "Excluir" fica desabilitado enquanto `onConfirm` não termina.
- `Home`: estado `rideToDelete`; `handleDelete` chama `deleteRide`, trata o erro (DEL-14/15/16) e recarrega com `loadRides()`.

---

## Test Coverage Matrix

> Generated from codebase, project guidelines, and spec - confirm before Execute. Guidelines found: `soft-go-ii-api/CLAUDE.md`, `soft-go-ii-api/.claude/skills/novo-recurso/SKILL.md` (§5), `soft-go-ii-api/vitest.config.ts`, `soft-go-II/vite.config.ts` (vitest + jsdom), AD-002. Os testes de save-rideowner (`src/ride/*.spec.ts`, `soft-go-II/src/**/*.test.tsx`) são o piso de estilo.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| ---------- | ------------------ | -------------------- | ---------------- | ----------- |
| API service (`ride.service`) | unit | Todos os ramos; 1:1 com os DEL da tarefa; mensagem exata (L-004) | `soft-go-ii-api/src/**/*.spec.ts` | `npm test` |
| API controller (HTTP) | integration (Nest testing + supertest + `setupApp`, repositórios fake, sem banco) | Rota nova: feliz + cada erro do spec (401/400/403/404) + estado persistido (L-006) + "Nenhum guard global" (L-001) | `soft-go-ii-api/src/**/*.controller.spec.ts` | `npm test` |
| API entity | unit (metadata TypeORM, L-002) | `RideUser.ride` com `onDelete: 'CASCADE'` | `soft-go-ii-api/src/**/entities/*.spec.ts` | `npm test` |
| Front lógica (`service/*`) | unit | Método e URL da chamada | `soft-go-II/src/**/*.test.ts` | `npm test` |
| Front componentes / páginas | unit (jsdom + Testing Library, serviços mockados) | Cada AC de UI: feliz + erro; cada elemento do modal (L-005); singular/plural | `soft-go-II/src/**/*.test.tsx` | `npm test` |

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
T3 → T4 → T5 → T6
```

### Phase 3: Ajuste pedido depois da validação

```
T7
```

---

## Task Breakdown

### T1: `RideService.remove`

**What**: Método `remove(id, userId)` conforme o design inline: `404` se a carona não existe, `403` se `ownerId !== userId`, senão `rideRepository.delete(id)`. Teste de metadata garantindo o cascade de `ride_users`.
**Where**: `soft-go-ii-api/src/ride/ride.service.ts`
**Also touches**: `src/ride/ride.service.spec.ts`, `src/ride-users/entities/ride-user.entity.spec.ts`
**Depends on**: None
**Reuses**: `NotFoundException`/`ConflictException` de `RideUsersService.join`; mocks de repositório do `ride.service.spec.ts`
**Requirement**: DEL-01, DEL-02, DEL-03, DEL-04

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes unit:
  - dona → `delete` chamado com o id e retorno `undefined`;
  - carona inexistente → `404` "Carona não encontrada", `delete` não chamado;
  - outra conta → `403` "Só a dona da carona pode excluí-la", `delete` não chamado;
  - carona com participantes e dona → `delete` chamado (participantes não bloqueiam)
- [x] Teste de metadata: relação `RideUser.ride` com `onDelete: 'CASCADE'` e FK `FK_ride_users_ride` (L-002)
- [x] Gate quick (API) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: allow owners to remove their rides`
**Status**: ✅ Done — `soft-go-ii-api@062bb90` (API: 158 testes)

---

### T2: `DELETE /ride/:id`

**What**: Rota `removeRide` conforme o design inline (`JwtAuthGuard`, `ParseIntPipe`, `204`), com Swagger.
**Where**: `soft-go-ii-api/src/ride/ride.controller.ts`
**Also touches**: `src/ride/ride.controller.spec.ts`, `src/auth/auth.controller.spec.ts` (`removeRide: [JwtAuthGuard]` no mapa `EXPECTED_GUARDS`)
**Depends on**: T1
**Reuses**: `createRide` (guard + `request.user.id`), `POST /auth/signout` (`@HttpCode(HttpStatus.NO_CONTENT)`), setup HTTP do `ride.controller.spec.ts`
**Requirement**: DEL-01, DEL-02, DEL-03, DEL-04, DEL-05, DEL-06, DEL-07

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes HTTP (repositórios fake):
  - token da dona → `204`, corpo vazio, e `GET /ride` não lista mais a carona;
  - token de outra conta → `403` "Só a dona da carona pode excluí-la", e a carona continua no repositório e no `GET /ride` (L-006);
  - id inexistente → `404` "Carona não encontrada";
  - id `abc` → `400`, nada apagado;
  - sem `Authorization` → `401` "Não autenticado", nada apagado;
  - token assinado com outro segredo → `401`, nada apagado;
  - token expirado → `401`, nada apagado;
  - token com `ver` antigo (sessão revogada, AD-005) → `401`, nada apagado;
  - `GET /ride` sem token continua `200`;
  - o teste "Nenhum guard global" continua passando (L-001)
- [x] Verificação manual no banco local (`npm run start:dev`): `DELETE` com o token da dona numa carona com uma inscrição → `204`, e `SELECT count(*) FROM ride_users WHERE ride_id = <id>` retorna 0. **Pedir confirmação da usuária antes**, porque apaga dados do banco local
- [x] Gate build (API) passa (fim da Phase 1). Contagem total da API registrada

**Tests**: integration
**Gate**: build
**Commit**: `feat: add delete ride route`
**Status**: ✅ Done — `soft-go-ii-api@541c0ca` (API: 167 testes; fim da Phase 1). Checagem manual no banco local com contas de teste novas: passageira → `403` com `ride_users` = 1; dona → `204`, `ride_users` = 0 e `rides` = 0; segundo `DELETE` → `404`

---

### T3: `deleteRide` no serviço do front

**What**: `deleteRide(id)` → `api.delete(`/ride/${id}`)`.
**Where**: `soft-go-II/src/service/ride.service.ts`
**Also touches**: `src/service/ride.service.test.ts`
**Depends on**: None
**Reuses**: `createRideUser`, mocks de `api` do `ride.service.test.ts`
**Requirement**: DEL-14

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes unit: `deleteRide(7)` chama `api.delete("/ride/7")` uma vez, sem body; erro da API é propagado
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add delete ride service`
**Status**: ✅ Done — `soft-go-II@efbab79` (front: 147 testes)

---

### T4: Botão "Excluir carona" no card

**What**: Token de erro `--color-support-04: #dc2626` no `@theme`. `Card` ganha `onDeleteRide?`. Com `showButton && isOwner && onDeleteRide`, renderiza o botão "Excluir carona" (`Trash2`, cor `support-04`) ao lado de "Sua carona".
**Where**: `soft-go-II/src/components/Card.tsx`
**Also touches**: `src/components/Card.test.tsx`, `src/index.css`
**Depends on**: T3
**Reuses**: selo "Sua carona"
**Requirement**: DEL-08, DEL-09

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes:
  - dona → botão "Excluir carona" visível; clique chama `onDeleteRide` uma vez;
  - dona com carona de ônibus (`spots: null`) e com carona lotada → botão visível;
  - outra usuária logada → sem "Excluir carona";
  - deslogada (`currentUserId` indefinido) → sem "Excluir carona";
  - `showButton={false}` (uso no modal) → sem "Excluir carona"
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: show delete button to ride owner`
**Status**: ✅ Done — `soft-go-II@feb946a` (front: 153 testes)

---

### T5: `ConfirmDeleteRideModal`

**What**: Modal de confirmação conforme o design inline e DEL-10/11/12/13. "Excluir" chama `onConfirm` e fica desabilitado até a promessa terminar.
**Where**: `soft-go-II/src/components/ConfirmDeleteRideModal.tsx`
**Also touches**: `src/components/ConfirmDeleteRideModal.test.tsx` (novo)
**Depends on**: T4
**Reuses**: estrutura de `components/Modal.tsx`, `Card` com `showButton={false}`, `Line`
**Requirement**: DEL-10, DEL-11, DEL-12, DEL-13, DEL-14

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes (L-005):
  - título "Excluir carona?", nome da dona, cidade, hora e transporte, botões "Cancelar" e "Excluir";
  - 0 participantes → "Ninguém confirmou presença ainda.", sem a linha "Sem WhatsApp";
  - 1 participante → "1 pessoa confirmou presença e será removida da carona.";
  - 3 participantes → "3 pessoas confirmaram presença e serão removidas da carona.";
  - participantes Ana (com telefone), Bia e Caio (sem) → "Sem WhatsApp para avisar: Bia, Caio";
  - todos com telefone → sem a linha "Sem WhatsApp";
  - "Cancelar", "X" e clique no overlay → `onClose` chamado, `onConfirm` não;
  - clique dentro do diálogo → `onClose` não chamado;
  - "Excluir" → `onConfirm` chamado uma vez; com `onConfirm` pendente, o botão fica desabilitado e um segundo clique não chama de novo
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add delete ride confirmation modal`
**Status**: ✅ Done — `soft-go-II@429d3f0` (front: 165 testes)

---

### T6: Exclusão no Home

**What**: `Home` passa `onDeleteRide` ao `Card`, abre o `ConfirmDeleteRideModal` com `rideToDelete` e implementa `handleDelete`:
- `204`: toast "Carona excluída", fecha e recarrega;
- `403`/`404`: toast com a mensagem da API, fecha e recarrega;
- `401`: nada além do interceptor;
- outro erro: toast "Não foi possível excluir a carona. Tente novamente.", fecha, mural mantido.
**Where**: `soft-go-II/src/pages/Home.tsx`
**Also touches**: `src/pages/Home.test.tsx`
**Depends on**: T5
**Reuses**: `handleConfirm` (tratamento de erro), `loadRides`, `Toast`
**Requirement**: DEL-08, DEL-10, DEL-13, DEL-14, DEL-15, DEL-16

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes (serviços mockados):
  - logada como dona → card com "Excluir carona"; clique abre "Excluir carona?";
  - logada como outra conta → sem "Excluir carona";
  - "Cancelar" → `deleteRide` não chamado, modal fecha;
  - "Excluir" → `deleteRide(<id da carona>)`, toast "Carona excluída", modal fecha, `getRides` chamado de novo e a carona não aparece mais;
  - `403` com mensagem → toast com a mensagem exata, modal fecha, `getRides` chamado de novo;
  - `404` com mensagem → toast "Carona não encontrada", `getRides` chamado de novo;
  - `500` → toast "Não foi possível excluir a carona. Tente novamente.", a carona continua na lista;
  - `401` → nenhum toast do Home
- [x] Gate build (front) passa (fim da Phase 2). Contagem total do front registrada

**Tests**: unit
**Gate**: build
**Commit**: `feat: let owners delete rides from the board`
**Status**: ✅ Done — `soft-go-II@3f74f33` (front: 173 testes; fim da Phase 2)

---

### T7: Lixeira ao lado do tipo de transporte

**What**: Pedido da usuária depois do PASS. A ação de excluir vira um ícone de lixeira (só ícone, `aria-label` "Excluir carona"), logo à direita da etiqueta do tipo de transporte. Ela sai da linha de ações (WhatsApp, "Sua carona"). DEL-08 foi reescrito na spec.
**Where**: `soft-go-II/src/components/Card.tsx`
**Also touches**: `src/components/Card.test.tsx`
**Depends on**: None (Phase 2 concluída)
**Reuses**: botão "X" do `Modal` (ícone com `aria-label`)
**Requirement**: DEL-08

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: a lixeira é o elemento logo depois da etiqueta "Carro"; o botão não tem texto visível; a linha de "Sua carona" não tem botão. Os testes de ausência (outra conta, deslogada, `showButton={false}`) continuam passando
- [x] Gate build (front) passa

**Tests**: unit
**Gate**: build
**Commit**: `feat: move ride delete action to trash icon beside transport type`
**Status**: ✅ Done — `soft-go-II@44b4e9b` (front: 174 testes)

---

## Phase Execution Map

```
Phase 1 → Phase 2

Phase 1 (API):   T1 → T2
Phase 2 (Front): T3 → T4 → T5 → T6
Phase 3 (Ajuste): T7
```

Execution is strictly sequential. 6 tarefas: cabem num lote só, execução inline sem sub-agentes. O Verifier roda no fim.

---

## Task Granularity Check

| Task | Scope | Status |
| ---- | ----- | ------ |
| T1 | 1 método (+ teste de metadata) | ✅ |
| T2 | 1 rota | ✅ |
| T3 | 1 função | ✅ |
| T4 | 1 componente | ✅ |
| T5 | 1 componente novo | ✅ |
| T6 | 1 página | ✅ |
| T7 | 1 componente | ✅ |

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| ---- | ---------------------- | ------------- | ------ |
| T1 | None | início da Phase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | None | início da Phase 2 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |
| T7 | None (Phase 2 concluída) | início da Phase 3 | ✅ |

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| ---- | --------------------------- | --------------- | --------- | ------ |
| T1 | API service (+ teste de entity) | unit | unit | ✅ |
| T2 | API controller | integration | integration | ✅ |
| T3 | Front lógica | unit | unit | ✅ |
| T4 | Front componente | unit | unit | ✅ |
| T5 | Front componente | unit | unit | ✅ |
| T6 | Front página | unit | unit | ✅ |
| T7 | Front componente | unit | unit | ✅ |

## Requirement Coverage

Todos os 16 requisitos estão mapeados:

| Requisito | Tarefas |
| --------- | ------- |
| DEL-01 | T1, T2 |
| DEL-02 | T1, T2 |
| DEL-03 | T1, T2 |
| DEL-04 | T1, T2 |
| DEL-05 | T2 |
| DEL-06 | T2 |
| DEL-07 | T2 |
| DEL-08 | T4, T6, T7 |
| DEL-09 | T4 |
| DEL-10 | T5, T6 |
| DEL-11 | T5 |
| DEL-12 | T5 |
| DEL-13 | T5, T6 |
| DEL-14 | T3, T5, T6 |
| DEL-15 | T6 |
| DEL-16 | T6 |
