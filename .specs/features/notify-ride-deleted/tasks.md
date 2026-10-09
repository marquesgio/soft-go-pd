# Avisos Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: inline (sem `design.md`: módulo novo no padrão de `src/ride/`)
**Spec**: `.specs/features/notify-ride-deleted/spec.md` · **Context**: `.specs/features/notify-ride-deleted/context.md`
**Status**: Done (2026-10-09) — T1–T8 concluídas. Verifier independente **não rodou** (pedido de rapidez da usuária); validação feita por gates + checagem ponta a ponta no banco local

**Commits**: a pedido da usuária, um commit por repositório para a feature inteira (não um por tarefa).

**Repositórios**:
- T1–T5 commitam em `soft-go-ii-api/` (`master`). T6–T8 commitam em `soft-go-II/` (`main`).
- O código vai no commit do submódulo. Logo depois, um commit no repo raiz leva a marcação em `tasks.md`, a rastreabilidade em `spec.md` e o bump do ponteiro.
- No fim: as duas rotas novas na tabela de contrato do `CLAUDE.md` raiz e uma linha no `HISTORICO.md`.
- Sempre `git add` com caminhos explícitos. Nunca adicionar `tsconfig.build.tsbuildinfo`, `dist/` ou `.env`.

**Python**: neste Windows, use `python` (não `python3`).

**Lições aplicadas**: L-004 (mensagem exata), L-001 ("Nenhum guard global" + `EXPECTED_GUARDS`), L-002 (metadata da entidade), L-006 (rejeição confere que nada foi gravado).

**Baseline**: API 197 testes · front 206 testes.

---

## Design inline

**API (`src/notifications/`)**
- Entidade `Notification` (`@Entity('notifications')`): `id`; `userId` (`user_id`, int, not null) + `user` (`@ManyToOne('User', { onDelete: 'CASCADE' })`, FK `FK_notifications_user`); `type` (varchar 30); `message` (varchar 300); `createdAt` (`@CreateDateColumn`, `created_at`, timestamptz); `readAt` (`read_at`, timestamptz, nullable). Índice `IDX_notifications_user_created` em (`user_id`, `created_at`).
- `NotificationsRepository extends Repository<Notification>`.
- `NotificationsService`:
  - `notifyRideDeleted(ride: { date, hour, city }, ownerName, participantIds[])`
  - `notifyPassengerLeft(ride, passengerName, ownerId)`
  - `notifyPassengerRemoved(ride, ownerName, passengerId)`
  - os três montam a frase (`firstName`, `DD/MM`, `HH:mm`), fazem um `save` em lote e **engolem** erro com `Logger.error`;
  - `listForUser(userId)` → `{ unread, items }` (30 mais recentes, `createdAt DESC`; `read = readAt !== null`);
  - `markAllRead(userId)` → `update({ userId, readAt: IsNull() }, { readAt: new Date() })`.
- `NotificationsController` (`@Controller('notifications')`): `GET` e `POST read` (`@HttpCode(204)`), ambos `JwtAuthGuard`.
- `NotificationsModule` exporta service e repositório; importado por `AppModule`, `RideModule` e `RideUsersModule`.
- `RideService.remove`: busca a carona com `owner` e `rideUser`, apaga, depois `notifyRideDeleted`.
- `RideUsersService.remove`: busca a carona com `owner` e a inscrição com `user`, apaga, depois `notifyPassengerLeft` (quem pede é a passageira) ou `notifyPassengerRemoved` (quem pede é a dona).

**Front**
- Tipos `AppNotification` e `NotificationsResponse`. `service/notification.service.ts`: `getNotifications()` e `markNotificationsRead()`.
- `NotificationBell`: busca ao montar; botão `Bell` com badge; clique abre painel, busca de novo e marca lido se havia não lidos; erro mostra "Não foi possível carregar os avisos.".
- `Header` renderiza `NotificationBell` quando há `user`.

---

## Test Coverage Matrix

> Generated from codebase, project guidelines, and spec - confirm before Execute. Guidelines found: `soft-go-ii-api/CLAUDE.md`, `soft-go-ii-api/.claude/skills/novo-recurso/SKILL.md` (§5), `soft-go-ii-api/vitest.config.ts`, `soft-go-II/vite.config.ts`, AD-002. Os testes de delete-rides e leave-ride são o piso de estilo.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| ---------- | ------------------ | -------------------- | ---------------- | ----------- |
| API entity | unit (metadata TypeORM, L-002) | Colunas, nulabilidade, FK com CASCADE | `soft-go-ii-api/src/**/entities/*.spec.ts` | `npm test` |
| API migration / module | none | build gate + `run → revert → run` no banco local | - | build gate |
| API service | unit | Todos os ramos; mensagem exata de cada aviso; falha engolida | `soft-go-ii-api/src/**/*.spec.ts` | `npm test` |
| API controller (HTTP) | integration (Nest testing + supertest, repositórios fake) | Rotas novas: feliz + 401 + isolamento entre contas; eventos gravam aviso e falha não muda a resposta | `soft-go-ii-api/src/**/*.controller.spec.ts` | `npm test` |
| Front lógica (`service/*`) | unit | Método, URL, retorno | `soft-go-II/src/**/*.test.ts` | `npm test` |
| Front componentes | unit (jsdom + Testing Library) | Cada AC de UI: feliz + vazio + erro; badge e nome acessível | `soft-go-II/src/**/*.test.tsx` | `npm test` |

## Gate Check Commands

> Generated from codebase - confirm before Execute.

| Gate Level | When to Use | Command |
| ---------- | ----------- | ------- |
| Quick (API) | Tarefas com unit na API | `cd soft-go-ii-api && npm test` |
| Build (API) | Entity/migration, HTTP e fim de fase da API | `cd soft-go-ii-api && npx tsc --noEmit -p tsconfig.build.json && npm run lint && npm test` + `npx tsc --noEmit -p tsconfig.json` tolerando só o erro antigo `test/app.e2e-spec.ts(4,21) TS2307` |
| Quick (Front) | Tarefas com testes no front | `cd soft-go-II && npm test` |
| Build (Front) | Fim de fase do front | `cd soft-go-II && npm run lint && npm run build && npm test` |

---

## Execution Plan

Phases are ordered and run sequentially - each phase completes before the next begins, and tasks within a phase execute in order.

### Phase 1: API

```
T1 → T2 → T3 → T4 → T5
```

### Phase 2: Front

```
T6 → T7 → T8
```

---

## Task Breakdown

### T1: Tabela `notifications` (migration + entidade)

**What**: Entidade `Notification` conforme o design inline, registrada em `postgres.config.service.ts` e `data-source.ts`, e a migration `CreateTableNotifications` (cria tabela, FK e índice; `down` remove tudo).
**Where**: `soft-go-ii-api/src/migrations/<timestamp>-CreateTableNotifications.ts`
**Also touches**: `src/notifications/entities/notification.entity.ts`, `src/notifications/entities/notification.entity.spec.ts` (novo), `src/config/postgres.config.service.ts`, `src/config/data-source.ts`
**Depends on**: None
**Reuses**: `ride-user.entity.ts`, skill `/migration`
**Requirement**: NOTIFY-01, NOTIFY-07

**Tools**:
- MCP: NONE
- Skill: `migration` (API)

**Done when**:
- [x] Teste de metadata: `user_id` not null com FK `FK_notifications_user` e `onDelete: 'CASCADE'`; `type` varchar(30) e `message` varchar(300) not null; `read_at` nullable; `created_at` é coluna de criação
- [x] **Pedir confirmação** e rodar no banco local: `migrations:run`, `migrations:revert`, `migrations:run`, e `migration:generate --dryrun` sem diferenças. A migration só cria tabela; não apaga dados
- [x] Gate build (API) passa

**Tests**: unit
**Gate**: build
**Commit**: `feat: add notifications table`
**Status**: ✅ Done — API: entidade + migration `CreateTableNotifications` aplicada no banco local (`run → revert → run`, `--dryrun` sem diferenças)

---

### T2: `NotificationsService`

**What**: Repositório, service e módulo conforme o design inline.
**Where**: `soft-go-ii-api/src/notifications/notifications.service.ts`
**Also touches**: `src/notifications/notifications.repository.ts`, `src/notifications/notifications.module.ts`, `src/notifications/notifications.service.spec.ts` (novo)
**Depends on**: T1
**Reuses**: `firstName` do front como referência (no back: `name.trim().split(/\s+/)[0]`, igual a `toParticipant`)
**Requirement**: NOTIFY-01, NOTIFY-02, NOTIFY-03, NOTIFY-04, NOTIFY-05, NOTIFY-07, NOTIFY-09

**Tools**:
- MCP: NONE
- Skill: `novo-recurso` (API)

**Done when**:
- [x] Testes unit:
  - `notifyRideDeleted` com 2 participantes → `save` com 2 avisos `ride_deleted`, `userId` de cada uma, mensagem exata "Carla excluiu a carona de 15/10 às 08:00 saindo de Porto Alegre." (nome "Carla Dona Silva", data `2030-10-15`, hora `08:00:00`);
  - `notifyRideDeleted` sem participantes → `save` não chamado;
  - `notifyPassengerLeft` → 1 aviso `passenger_left` para a dona com "Ana saiu da sua carona de 15/10 às 08:00 saindo de Porto Alegre.";
  - `notifyPassengerRemoved` → 1 aviso `passenger_removed` para a passageira com "Carla removeu você da carona de 15/10 às 08:00 saindo de Porto Alegre.";
  - `save` rejeita → os três métodos resolvem sem lançar e o erro vai para `Logger.error`;
  - `listForUser` → `find` com `userId`, `createdAt DESC`, `take: 30`; `items` com `read` derivado de `readAt`; `unread` de `countBy({ userId, readAt: IsNull() })`;
  - `markAllRead` → `update` só com o `userId` e `readAt: IsNull()`
- [x] Gate quick (API) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add notifications service`
**Status**: ✅ Done — API: 13 testes do service

---

### T3: Rotas de avisos

**What**: `NotificationsController` (`GET /notifications`, `POST /notifications/read`), registrado no `AppModule`, e as duas rotas no mapa `EXPECTED_GUARDS`.
**Where**: `soft-go-ii-api/src/notifications/notifications.controller.ts`
**Also touches**: `src/notifications/notifications.controller.spec.ts` (novo), `src/notifications/in-memory-notifications.test-spec.ts` (novo, auxiliar de teste fora do build), `src/notifications/notifications.module.ts`, `src/app.module.ts`, `src/auth/auth.controller.spec.ts`
**Depends on**: T2
**Reuses**: `RideUsersController.remove`, setup HTTP com repositório em memória
**Requirement**: NOTIFY-07, NOTIFY-08, NOTIFY-09, NOTIFY-10, NOTIFY-11

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes HTTP (repositório em memória):
  - `GET` com token → `200` com `{ unread, items }` só da conta do token, do mais novo ao mais antigo;
  - conta sem avisos → `{ unread: 0, items: [] }`;
  - 31 avisos → `items` com 30 e `unread: 31`;
  - `POST /notifications/read` → `204`; depois, `GET` traz `unread: 0` e `read: true`; avisos de outra conta continuam não lidos;
  - `POST` sem não lidos → `204`;
  - sem token, outro segredo, expirado e sessão revogada → `401` "Não autenticado" nas duas rotas, e nada é marcado lido;
  - `EXPECTED_GUARDS` e "Nenhum guard global" passando
- [x] Gate build (API) passa

**Tests**: integration
**Gate**: build
**Commit**: `feat: add notifications routes`
**Status**: ✅ Done — API: rotas + `EXPECTED_GUARDS`

---

### T4: Aviso ao excluir carona

**What**: `RideService.remove` busca a carona com `owner` e `rideUser`, apaga, e chama `notifyRideDeleted` com os ids das participantes. `RideModule` importa `NotificationsModule`. Os mocks do `ride.service.spec.ts` e o repositório fake do `ride.controller.spec.ts` passam de `findOneBy` para `findOne` (mudança de implementação, mesmas asserções).
**Where**: `soft-go-ii-api/src/ride/ride.service.ts`
**Also touches**: `src/ride/ride.module.ts`, `src/ride/ride.service.spec.ts`, `src/ride/ride.controller.spec.ts`
**Depends on**: T3
**Reuses**: `NotificationsService`
**Requirement**: NOTIFY-01, NOTIFY-02, NOTIFY-05, NOTIFY-06

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Unit: dona exclui carona com 2 participantes → `notifyRideDeleted` chamado depois do `delete`, com a carona, o nome da dona e os 2 ids; `403`/`404` → `notifyRideDeleted` não chamado
- [x] HTTP: `DELETE /ride/3` pela dona → `204` e o repositório de avisos tem 1 aviso para a participante com a mensagem exata; `403` → nenhum aviso; repositório de avisos que rejeita → continua `204` e a carona some
- [x] Gate build (API) passa

**Tests**: integration
**Gate**: build
**Commit**: `feat: notify passengers when a ride is deleted`
**Status**: ✅ Done — API: aviso ao excluir carona

---

### T5: Aviso ao sair ou ser removida

**What**: `RideUsersService.remove` carrega `owner` da carona e `user` da inscrição, apaga, e chama `notifyPassengerLeft` (quem pede = passageira) ou `notifyPassengerRemoved` (quem pede = dona). `RideUsersModule` importa `NotificationsModule`.
**Where**: `soft-go-ii-api/src/ride-users/ride-users.service.ts`
**Also touches**: `src/ride-users/ride-users.module.ts`, `src/ride-users/ride-users.service.spec.ts`, `src/ride-users/ride-users.controller.spec.ts`
**Depends on**: T4
**Reuses**: `NotificationsService`
**Requirement**: NOTIFY-03, NOTIFY-04, NOTIFY-05, NOTIFY-06

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Unit: passageira sai → `notifyPassengerLeft` com o nome dela e o id da dona, `notifyPassengerRemoved` não chamado; dona remove → `notifyPassengerRemoved` com o nome da dona e o id da passageira, `notifyPassengerLeft` não chamado; `403`/`404` → nenhum dos dois
- [x] HTTP: sair → `204` e aviso `passenger_left` para a dona com a mensagem exata; remover → `204` e aviso `passenger_removed` para a passageira; `403` → nenhum aviso; repositório de avisos que rejeita → continua `204` e a inscrição some
- [x] Verificação manual no banco local (**pedir confirmação**): com contas de teste novas, os três eventos geram os avisos certos em `GET /notifications`
- [x] Gate build (API) passa (fim da Phase 1)

**Tests**: integration
**Gate**: build
**Commit**: `feat: notify on passenger leave and removal`
**Status**: ✅ Done — API: 239 testes; fim da Phase 1. Checagem ponta a ponta no banco local com 3 contas de teste novas: os três avisos chegaram à pessoa certa, `read` zera só a conta do token, sem token → `401`. A checagem achou `UsersModule` faltando no `NotificationsModule` (a API não subia); corrigido

---

### T6: Tipos e serviço de avisos no front

**What**: Tipos `AppNotification` e `NotificationsResponse`; `getNotifications()` → `GET /notifications`; `markNotificationsRead()` → `POST /notifications/read`.
**Where**: `soft-go-II/src/service/notification.service.ts`
**Also touches**: `src/service/notification.service.test.ts` (novo), `src/types/index.ts`
**Depends on**: None
**Reuses**: `ride.service.ts`
**Requirement**: NOTIFY-14

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: `getNotifications` chama `api.get("/notifications")` e devolve `res.data`; `markNotificationsRead` chama `api.post("/notifications/read")`; erros propagam
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add notifications service`
**Status**: ✅ Done — front: serviço e tipos

---

### T7: `NotificationBell`

**What**: Componente conforme o design inline e NOTIFY-13..17.
**Where**: `soft-go-II/src/components/NotificationBell.tsx`
**Also touches**: `src/components/NotificationBell.test.tsx` (novo)
**Depends on**: T6
**Reuses**: estilo do Header, `date-fns`
**Requirement**: NOTIFY-13, NOTIFY-14, NOTIFY-15, NOTIFY-16, NOTIFY-17

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes (serviço mockado):
  - 2 não lidos → badge "2" e nome acessível "Avisos (2 não lidos)";
  - 12 não lidos → badge "9+";
  - 0 não lidos → sem badge, nome "Avisos";
  - clique → painel "Avisos" com as mensagens na ordem recebida e a data `DD/MM HH:mm` de cada uma; `markNotificationsRead` chamado uma vez; badge some;
  - clique com 0 não lidos → `markNotificationsRead` não chamado;
  - sem avisos → "Nenhum aviso por enquanto.";
  - segundo clique fecha o painel;
  - `getNotifications` rejeita ao abrir → "Não foi possível carregar os avisos." e sem badge
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add notification bell`
**Status**: ✅ Done — front: `NotificationBell`

---

### T8: Sino no Header

**What**: `Header` renderiza `NotificationBell` quando há `user`. `Header.test.tsx` mocka `notification.service`.
**Where**: `soft-go-II/src/components/Header.tsx`
**Also touches**: `src/components/Header.test.tsx`
**Depends on**: T7
**Reuses**: `NotificationBell`
**Requirement**: NOTIFY-12

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: logada → botão "Avisos" (ou "Avisos (N não lidos)") presente; deslogada → ausente; testes atuais do Header passando
- [x] Gate build (front) passa (fim da Phase 2)

**Tests**: unit
**Gate**: build
**Commit**: `feat: show notification bell in header`
**Status**: ✅ Done — front: 220 testes; fim da Phase 2

---

## Phase Execution Map

```
Phase 1 → Phase 2

Phase 1 (API):   T1 → T2 → T3 → T4 → T5
Phase 2 (Front): T6 → T7 → T8
```

Execution is strictly sequential. 8 tarefas: um lote só, execução inline. O Verifier roda no fim.

---

## Task Granularity Check

| Task | Scope | Status |
| ---- | ----- | ------ |
| T1 | 1 migration + 1 entidade (+ registro) | ⚠️ coeso |
| T2 | 1 service (+ repositório e módulo) | ⚠️ coeso |
| T3 | 1 controller | ✅ |
| T4 | 1 método | ✅ |
| T5 | 1 método | ✅ |
| T6 | 1 serviço + tipos | ✅ |
| T7 | 1 componente | ✅ |
| T8 | 1 componente | ✅ |

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| ---- | ---------------------- | ------------- | ------ |
| T1 | None | início da Phase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6 | None | início da Phase 2 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| ---- | --------------------------- | --------------- | --------- | ------ |
| T1 | API entity + migration | unit + none | unit | ✅ |
| T2 | API service | unit | unit | ✅ |
| T3 | API controller | integration | integration | ✅ |
| T4 | API service (+ HTTP) | unit + integration | integration | ✅ |
| T5 | API service (+ HTTP) | unit + integration | integration | ✅ |
| T6 | Front lógica | unit | unit | ✅ |
| T7 | Front componente | unit | unit | ✅ |
| T8 | Front componente | unit | unit | ✅ |

## Requirement Coverage

| Requisito | Tarefas |
| --------- | ------- |
| NOTIFY-01 | T1, T2, T4 |
| NOTIFY-02 | T2, T4 |
| NOTIFY-03 | T2, T5 |
| NOTIFY-04 | T2, T5 |
| NOTIFY-05 | T2, T4, T5 |
| NOTIFY-06 | T4, T5 |
| NOTIFY-07 | T1, T2, T3 |
| NOTIFY-08 | T3 |
| NOTIFY-09 | T2, T3 |
| NOTIFY-10 | T3 |
| NOTIFY-11 | T3 |
| NOTIFY-12 | T8 |
| NOTIFY-13 | T7 |
| NOTIFY-14 | T6, T7 |
| NOTIFY-15 | T7 |
| NOTIFY-16 | T7 |
| NOTIFY-17 | T7 |
