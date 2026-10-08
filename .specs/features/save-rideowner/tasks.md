# Dona da Carona Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/save-rideowner/design.md`
**Spec**: `.specs/features/save-rideowner/spec.md` · **Context**: `.specs/features/save-rideowner/context.md`
**Status**: In Progress

**Repositórios**:
- T1–T5 commitam em `soft-go-ii-api/` (`master`). T6–T10 commitam em `soft-go-II/` (`main`).
- O código vai no commit do submódulo. Logo depois, um commit no repo raiz leva a marcação em `tasks.md`, a rastreabilidade em `spec.md` e o bump do ponteiro.
- Sempre `git add` com caminhos explícitos. Nunca adicionar `tsconfig.build.tsbuildinfo`, `dist/` ou `.env`.

**Python**: neste Windows, use `python` ou `py` (não `python3`).

**Lições aplicadas** (candidatas de auth e vou-junto):
- L-001: manter o teste "Nenhum guard global".
- L-002: metadata de coluna via `getMetadataArgsStorage`.
- L-003: erro de cada campo do formulário.
- L-004: mensagem exata.
- L-005: cada elemento listado no card.
- L-006: rejeição confere o estado persistido.
- L-007: caminho sem `phone` não apaga a sessão.
- L-008: `phone: ""` via HTTP.

**Baseline**: API 96 testes · front 109 testes.

---

## Test Coverage Matrix

> Generated from codebase, project guidelines, and spec - confirm before Execute. Guidelines found: `soft-go-ii-api/CLAUDE.md`, `soft-go-ii-api/.claude/skills/novo-recurso/SKILL.md` (§5), `soft-go-ii-api/vitest.config.ts`, `soft-go-II/vite.config.ts` (vitest + jsdom), AD-002. Os testes da vou-junto (`src/ride*/**/*.spec.ts`, `soft-go-II/src/**/*.test.tsx`) são o piso de estilo.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| ---------- | ------------------ | -------------------- | ---------------- | ----------- |
| API service (`ride.service`, `ride-users.service`) | unit | Todos os ramos; 1:1 com os OWNER da tarefa; cada edge case | `soft-go-ii-api/src/**/*.spec.ts` | `npm test` |
| API controller (HTTP) | integration (Nest testing + supertest + `setupApp`, repositórios fake, sem banco) | Toda rota alterada: feliz + cada erro do spec (401/400/409) + estado persistido (L-006) | `soft-go-ii-api/src/**/*.controller.spec.ts` | `npm test` |
| API entity | unit (metadata TypeORM, L-002) | `ownerId` não nullable, FK `FK_rides_owner`, sem `name`/`phone` | `soft-go-ii-api/src/**/entities/*.spec.ts` | `npm test` |
| API migration / DTO / module | none | build gate. DTO exercitado pelo HTTP. Migration validada com `run → revert → run` em banco temporário + `migration:generate --dryrun` sem diferenças | - | build gate |
| Front lógica (`service/*`) | unit | Todos os ramos; 1:1 com os OWNER | `soft-go-II/src/**/*.test.ts` | `npm test` |
| Front componentes / páginas | unit (jsdom + Testing Library, serviços mockados) | Cada AC de UI: feliz + erro; erro por campo (L-003); cada elemento do card (L-005) | `soft-go-II/src/**/*.test.tsx` | `npm test` |
| Front tipos | none | build gate (`tsc -b`) | - | build gate |

## Gate Check Commands

> Generated from codebase - confirm before Execute.

| Gate Level | When to Use | Command |
| ---------- | ----------- | ------- |
| Quick (API) | Tarefas com unit na API | `cd soft-go-ii-api && npm test` |
| Build (API) | Entity/migration, testes HTTP e fim de fase da API | `cd soft-go-ii-api && npx tsc --noEmit -p tsconfig.build.json && npm run lint && npm test` + `npx tsc --noEmit -p tsconfig.json` tolerando só o erro antigo `test/app.e2e-spec.ts(4,21) TS2307` |
| Quick (Front) | Tarefas com testes no front | `cd soft-go-II && npm test` |
| Build (Front) | Fim de fase do front | `cd soft-go-II && npm run lint && npm run build && npm test` |

---

## Execution Plan

Phases are ordered and run sequentially - each phase completes before the next begins, and tasks within a phase execute in order.

### Phase 1: API

```
T1 → T2 → T3 → T4 → T5
```

### Phase 2: Front base

```
T6 → T7 → T8
```

### Phase 3: Front páginas

```
T9 → T10
```

---

## Task Breakdown

### T1: `rides.owner_id` (migration + entidade)

**What**: Migration `AddOwnerToRides` conforme o design: `up` apaga as caronas, remove `name`/`phone` e adiciona `owner_id NOT NULL` + `FK_rides_owner`; `down` apaga as caronas e restaura a estrutura. Entidade `Ride` sem `name`/`phone`, com `ownerId` + `owner`. Ajustar o mínimo em `ride.service.ts`, `CreateRideDto` e specs existentes só se o build quebrar. A lógica nova vem no T2/T3.
**Where**: `soft-go-ii-api/src/migrations/<timestamp>-AddOwnerToRides.ts`
**Also touches**: `src/ride/entities/ride.entity.ts`, `src/ride/entities/ride.entity.spec.ts` (novo)
**Depends on**: None
**Reuses**: `src/migrations/1791413956190-LinkRideUsersToUsers.ts`, `src/ride-users/entities/ride-user.entity.ts`
**Requirement**: OWNER-21, OWNER-22, OWNER-23, OWNER-24

**Tools**:
- MCP: NONE
- Skill: `migration` (API)

**Done when**:
- [x] Teste de metadata: `ownerId` com `name: 'owner_id'` e `nullable: false`; relação `owner` com FK `FK_rides_owner`; sem colunas `name` e `phone`
- [x] **Banco temporário** com todas as migrations: `run`, depois `revert` desta, depois `run`, sem erro. Com uma carona e uma inscrição semeadas, as duas somem depois do `up`
- [x] **Pedir confirmação da usuária antes** de rodar no banco local, porque apaga as caronas e inscrições existentes
- [x] No banco local (após o "sim"): `migrations:run` e `migration:generate --dryrun` sem diferenças
- [x] Gate build (API) passa

**Tests**: unit
**Gate**: build
**Commit**: `feat: add owner to rides`
**Status**: ✅ Done — `soft-go-ii-api@df81e71` (API: 99 testes; migration aplicada no banco local com consentimento)

---

### T2: `RideService.create` com dona

**What**: `create(dto, ownerId)`:
- conta inexistente → `UnauthorizedException('Não autenticado')`, sem salvar;
- salva a carona com `ownerId`, sem `phone` no spread;
- depois, se `phone` veio, `usersRepository.update(ownerId, { phone })`.

`CreateRideDto` sem `name` (o `phone` fica com as regras atuais). `RideModule` importa `UsersModule`.
**Where**: `soft-go-ii-api/src/ride/ride.service.ts`
**Also touches**: `src/ride/dto/create-ride.dto.ts`, `src/ride/ride.module.ts`, `src/ride/ride.service.spec.ts`
**Depends on**: T1
**Reuses**: `UsersRepository`, padrão de `RideUsersService.join`
**Requirement**: OWNER-03, OWNER-08, OWNER-26, OWNER-28

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes unit:
  - salva com `ownerId` do argumento;
  - conta inexistente → `401` "Não autenticado", `save` não chamado e `update` não chamado;
  - com `phone` → `update(ownerId, { phone })`, e a carona salva não tem `phone`;
  - sem `phone` → `update` não chamado;
  - tipo de transporte inexistente → `404` (já existente, mantido)
- [x] Gate quick (API) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: save ride owner on create`
**Status**: ✅ Done — `soft-go-ii-api@83a6d48` (API: 104 testes; tsc do controller fecha no T4)

---

### T3: `owner` no `GET /ride` (service)

**What**: `findAllRides` carrega `owner` e devolve `owner: toOwner(ride.owner, viewer)` = `{ id, name }` + `phone` só com `viewer`. A resposta sai sem `ownerId`, sem `name` e sem `phone` na raiz.
**Where**: `soft-go-ii-api/src/ride/ride.service.ts`
**Also touches**: `src/ride/ride.service.spec.ts`
**Depends on**: T2
**Reuses**: `toParticipant`
**Requirement**: OWNER-11, OWNER-12, OWNER-13, OWNER-14

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes unit:
  - `owner.id` e `owner.name` completo (ex.: "Ana Maria Souza", não "Ana");
  - com viewer → `owner.phone` igual ao telefone; com viewer e conta sem telefone → `owner.phone === null`;
  - sem viewer → a chave `phone` não existe em `owner`;
  - a raiz da carona não tem as chaves `name`, `phone` e `ownerId`;
  - `relations` inclui `owner`
- [x] Gate quick (API) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: return ride owner in ride list`
**Status**: ✅ Done — `soft-go-ii-api@4db8137` (API: 109 testes)

---

### T4: `POST /ride` protegido

**What**: `@UseGuards(JwtAuthGuard) @ApiBearerAuth()` em `createRide`, que passa `request.user.id`. Swagger sem `name`, com `phone` descrito como "gravado na conta da dona".
**Where**: `soft-go-ii-api/src/ride/ride.controller.ts`
**Also touches**: `src/ride/ride.controller.spec.ts`
**Depends on**: T3
**Reuses**: `RideUsersController.create`, setup HTTP do `ride.controller.spec.ts`
**Requirement**: OWNER-03, OWNER-04, OWNER-06, OWNER-07, OWNER-08, OWNER-12, OWNER-13, OWNER-28, OWNER-30

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes HTTP (repositórios fake):
  - token válido + body válido → `201`, carona gravada com `ownerId` do token;
  - body com `name` → `400`; com `ownerId` → `400`. Nos dois, nada gravado;
  - sem `Authorization` → `401` "Não autenticado", nada gravado;
  - token assinado com outro segredo → `401`, nada gravado;
  - token expirado → `401`, nada gravado;
  - token de conta inexistente → `401` "Não autenticado", nada gravado;
  - `phone: ""` → `201`, e `users.phone` não muda (L-008);
  - `phone` inválido (`"123"`) → `400`, nada gravado, e `users.phone` não muda;
  - `GET /ride` com token → `owner.phone`; sem token → `owner` sem `phone`;
  - o teste "Nenhum guard global" continua passando (L-001)
- [x] Gate build (API) passa

**Tests**: integration
**Gate**: build
**Commit**: `feat: require login to publish rides`
**Status**: ✅ Done — `soft-go-ii-api@4d5ba30` (API: 118 testes)

---

### T5: Dona não se inscreve na própria carona

**What**: em `RideUsersService.join`, depois do 404: `ride.ownerId === userId` → `ConflictException('Você não pode se inscrever na sua própria carona')`.
**Where**: `soft-go-ii-api/src/ride-users/ride-users.service.ts`
**Also touches**: `src/ride-users/ride-users.service.spec.ts`, `src/ride-users/ride-users.controller.spec.ts`
**Depends on**: T4
**Reuses**: `ConflictException` de `join`
**Requirement**: OWNER-20

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Unit: dona → `409` com a mensagem exata, `save` não chamado e `users.update` não chamado (mesmo com `phone`)
- [x] Unit: outra usuária na mesma carona → inscrição criada
- [x] HTTP: `POST /ride-users` com o token da dona → `409` com a mensagem exata, `ride_users` sem registro novo
- [x] Gate build (API) passa (fim da Phase 1). Contagem total da API registrada

**Tests**: integration
**Gate**: build
**Commit**: `feat: block owners from joining their own ride`
**Status**: ✅ Done — `soft-go-ii-api@9589b43` (API: 121 testes; fim da Phase 1)

---

### T6: Tipos + `createRide`

**What**:
- tipos: `Owner`; `Ride` sem `name`/`phone` e com `owner`; `CreateRidePayload` (sem `name`, `transportType: number`, `phone?`);
- `createRide(payload)` envia `phone` só quando preenchido;
- ajustar o mínimo em `Form.tsx`, `Card.tsx` e fixtures de teste para compilar. O comportamento novo vem no T8–T10.
**Where**: `soft-go-II/src/service/ride.service.ts`
**Also touches**: `src/types/index.ts`, `src/service/ride.service.test.ts`
**Depends on**: None
**Reuses**: `createRideUser`
**Requirement**: OWNER-02, OWNER-27

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes unit:
  - `createRide` com `phone` → body com `phone`;
  - com `phone: ""` ou sem `phone` → body sem a chave `phone`;
  - o body nunca tem `name` nem `ownerId`
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: send ride without owner fields`
**Status**: ✅ Done — `soft-go-II@12e3fdd` (front: 113 testes; tsc de Card/Modal fecha no T8)

---

### T7: `RequireAuth` em `/form`

**What**: Componente `RequireAuth` (deslogada → `<Navigate to="/login" replace />`), envolvendo `/form` em `Routes.tsx`.
**Where**: `soft-go-II/src/components/RequireAuth.tsx`
**Also touches**: `src/components/RequireAuth.test.tsx`, `src/Routes.tsx`
**Depends on**: T6
**Reuses**: `GuestOnly`
**Requirement**: OWNER-10

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] Testes: deslogada em `/form` → `/login`, e o formulário não aparece; logada → renderiza os filhos
- [ ] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: require login to open ride form`

---

### T8: Dona no card

**What**: `Card` recebe `owner`:
- inicial e nome vêm de `owner.name`;
- o WhatsApp aparece com `showPhones && owner.phone`;
- quando `currentUserId === owner.id`, mostra "Sua carona" no lugar de "Vou junto"/"Você vai nesta carona", com prioridade sobre lotada;
- sai `phoneNull`.

No `Modal`, o `Pick` usa `owner`.
**Where**: `soft-go-II/src/components/Card.tsx`
**Also touches**: `src/components/Card.test.tsx`, `src/components/Modal.tsx`, `src/components/Modal.test.tsx`
**Depends on**: T7
**Reuses**: estilo de "Você vai nesta carona"
**Requirement**: OWNER-15, OWNER-16, OWNER-17, OWNER-18, OWNER-19

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] Testes (L-005):
  - nome completo e inicial da dona;
  - logada + `owner.phone` → link `https://wa.me/<phone>`;
  - deslogada → sem "WhatsApp";
  - `owner.phone: null` → sem "WhatsApp";
  - dona → "Sua carona" e sem "Vou junto" (também com a carona lotada e com vagas);
  - outra usuária → "Vou junto"
- [ ] Gate build (front) passa (fim da Phase 2)

**Tests**: unit
**Gate**: build
**Commit**: `feat: show ride owner on card`

---

### T9: Formulário sem nome, com WhatsApp opcional

**What**: `Form`:
- schema sem `name`, com `phone: optionalPhone.optional()`;
- "Seu Nome" sai, e "WhatsApp" só aparece quando `!user?.phone`;
- o payload sai sem `name`;
- em caso de sucesso: toast de sucesso, `updateUser({ ...user, phone })` se veio telefone, `navigate("/")`;
- em caso de erro: toast atual.
**Where**: `soft-go-II/src/pages/Form.tsx`
**Also touches**: `src/pages/Form.test.tsx` (novo)
**Depends on**: None (Phase 2 concluída)
**Reuses**: `optionalPhone`, `useAuth().updateUser`, `Home.handleConfirm`
**Requirement**: OWNER-01, OWNER-02, OWNER-05, OWNER-25, OWNER-26, OWNER-27, OWNER-29

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] Testes (serviços mockados, `MemoryRouter` + `AuthProvider`):
  - logada com telefone → sem "Seu Nome" e sem "WhatsApp";
  - logada sem telefone → "WhatsApp" aparece e "Seu Nome" não;
  - envio válido → `createRide` chamado sem `name`, com `transportType` numérico; toast de sucesso; vai para `/`;
  - com WhatsApp `51999998888` → `createRide` com `phone`, sessão com `phone`;
  - WhatsApp vazio → sem `phone`, sessão mantém `phone: null` (L-007);
  - WhatsApp `123` → "Número inválido, informe 11 dígitos (DDD + número)" e `createRide` não chamado;
  - erro de validação de cada campo obrigatório (data, hora, cidade, transporte) com a mensagem exata (L-003);
  - falha da API → toast "Não foi possível cadastrar a corrida. Tente novamente."
- [ ] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: publish rides as the logged user`

---

### T10: Link "Vou para a soft" no Home

**What**: o link aponta para `user ? "/form" : "/login"`. Ajustar as fixtures do `Home.test.tsx` para `owner`.
**Where**: `soft-go-II/src/pages/Home.tsx`
**Also touches**: `src/pages/Home.test.tsx`
**Depends on**: T9
**Reuses**: `useAuth`
**Requirement**: OWNER-09, OWNER-19

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] Testes:
  - deslogada clica "Vou para a soft" → `/login`;
  - logada → `/form`;
  - logada vê a própria carona com "Sua carona" (o Home passa `currentUserId`)
- [ ] Gate build (front) passa (fim da Phase 3). Contagem total do front registrada

**Tests**: unit
**Gate**: build
**Commit**: `feat: send logged-out users to login before publishing`

---

## Phase Execution Map

```
Phase 1 → Phase 2 → Phase 3

Phase 1 (API):   T1 → T2 → T3 → T4 → T5
Phase 2 (Front): T6 → T7 → T8
Phase 3 (Front): T9 → T10
```

Execution is strictly sequential.

---

## Task Granularity Check

| Task | Scope | Status |
| ---- | ----- | ------ |
| T1 | 1 migration + 1 entidade | ⚠️ coeso (migration e entidade precisam bater) |
| T2 | 1 método + DTO + import de módulo | ⚠️ coeso |
| T3 | 1 método | ✅ |
| T4 | 1 rota | ✅ |
| T5 | 1 regra num método | ✅ |
| T6 | tipos + 1 função | ✅ |
| T7 | 1 componente + rota | ✅ |
| T8 | 1 componente (+ tipo do Modal) | ✅ |
| T9 | 1 página | ✅ |
| T10 | 1 link | ✅ |

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
| T9 | None (Phase 2 concluída) | início da Phase 3 | ✅ |
| T10 | T9 | T9 → T10 | ✅ |

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| ---- | --------------------------- | --------------- | --------- | ------ |
| T1 | API migration + entity | none + unit (metadata) | unit | ✅ |
| T2 | API service + DTO + module | unit | unit | ✅ |
| T3 | API service | unit | unit | ✅ |
| T4 | API controller | integration | integration | ✅ |
| T5 | API service (+ HTTP) | unit + integration | integration | ✅ |
| T6 | Front lógica + tipos | unit | unit | ✅ |
| T7 | Front componente | unit | unit | ✅ |
| T8 | Front componente | unit | unit | ✅ |
| T9 | Front página | unit | unit | ✅ |
| T10 | Front página | unit | unit | ✅ |

## Requirement Coverage

Todos os 30 requisitos estão mapeados:

| Requisito | Tarefas |
| --------- | ------- |
| OWNER-01 | T9 |
| OWNER-02 | T6, T9 |
| OWNER-03 | T2, T4 |
| OWNER-04 | T4 |
| OWNER-05 | T9 |
| OWNER-06 | T4 |
| OWNER-07 | T4 |
| OWNER-08 | T2, T4 |
| OWNER-09 | T10 |
| OWNER-10 | T7 |
| OWNER-11 | T3 |
| OWNER-12 | T3, T4 |
| OWNER-13 | T3, T4 |
| OWNER-14 | T3 |
| OWNER-15 | T8 |
| OWNER-16 | T8 |
| OWNER-17 | T8 |
| OWNER-18 | T8 |
| OWNER-19 | T8, T10 |
| OWNER-20 | T5 |
| OWNER-21 | T1 |
| OWNER-22 | T1 |
| OWNER-23 | T1 |
| OWNER-24 | T1 |
| OWNER-25 | T9 |
| OWNER-26 | T2, T9 |
| OWNER-27 | T6, T9 |
| OWNER-28 | T2, T4 |
| OWNER-29 | T9 |
| OWNER-30 | T4 |
