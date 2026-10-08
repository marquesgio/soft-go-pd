# Vou Junto Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/vou-junto/design.md`
**Spec**: `.specs/features/vou-junto/spec.md` · **Context**: `.specs/features/vou-junto/context.md`
**Status**: Approved (2026-10-07). Execução inline, pausando por fase

**Repositórios**:
- T1–T8 commitam em `soft-go-ii-api/` (`master`). T9–T16 commitam em `soft-go-II/` (`main`).
- O código vai no commit do submódulo. Logo depois, um commit no repo raiz leva a marcação em `tasks.md`, a rastreabilidade em `spec.md` e o bump do ponteiro.
- Sempre `git add` com caminhos explícitos. Nunca adicionar `tsconfig.build.tsbuildinfo`, `dist/` ou `.env`.

**Python**: neste Windows, use `py` (não `python3`).

**Lições aplicadas** (candidatas de `auth`):
- L-001: manter o teste "Nenhum guard global".
- L-002: cobrir as opções críticas de coluna via `getMetadataArgsStorage`.
- L-003: testar o erro de cada campo de formulário.
- L-004: asserções com a mensagem exata.

---

## Test Coverage Matrix

> Generated from codebase, project guidelines, and spec - confirm before Execute. Guidelines found: `soft-go-ii-api/CLAUDE.md`, `soft-go-ii-api/.claude/skills/novo-recurso/SKILL.md` (§5), `soft-go-ii-api/vitest.config.ts`, `soft-go-II/vite.config.ts` (vitest + jsdom), AD-002. Os testes existentes de `auth` (`src/auth/*.spec.ts`, `soft-go-II/src/**/*.test.tsx`) são o piso de estilo.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| ---------- | ------------------ | -------------------- | ---------------- | ----------- |
| API service (`ride-users.service`, `ride.service`, `auth.service`) | unit | Todos os ramos; 1:1 com os JOIN da tarefa; cada edge case | `soft-go-ii-api/src/**/*.spec.ts` | `npm test` |
| API guard | unit | Token ausente, malformado, outro segredo, expirado, válido | `soft-go-ii-api/src/auth/*.spec.ts` | `npm test` |
| API controller (HTTP) | integration (Nest testing + supertest + `setupApp`, repositórios fake, sem banco) | Toda rota alterada: feliz + cada erro do spec (401/400/404/409) | `soft-go-ii-api/src/**/*.controller.spec.ts` | `npm test` |
| API entity | unit (metadata TypeORM, L-002) | Opções críticas: nullable, nomes de constraint/FK | `soft-go-ii-api/src/**/entities/*.spec.ts` | `npm test` |
| API migration / DTO / module | none | build gate. DTOs são exercitados pelo HTTP. Migration validada com `run → revert → run` em banco temporário + `migration:generate --dryrun` sem diferenças | - | build gate |
| Front lógica (`service/*`, `schemas/*`, `context/*`) | unit | Todos os ramos; 1:1 com os JOIN | `soft-go-II/src/**/*.test.ts(x)` | `npm test` |
| Front componentes / páginas | unit (jsdom + Testing Library, serviços mockados) | Cada AC de UI: feliz + erro; erro por campo (L-003) | `soft-go-II/src/**/*.test.tsx` | `npm test` |
| Front tipos | none | build gate (`tsc -b`) | - | build gate |

## Gate Check Commands

> Generated from codebase - confirm before Execute.

| Gate Level | When to Use | Command |
| ---------- | ----------- | ------- |
| Quick (API) | Tarefas com unit na API | `cd soft-go-ii-api && npm test` |
| Build (API) | Entity/migration, testes HTTP e fim de fase da API | `cd soft-go-ii-api && npx tsc --noEmit -p tsconfig.build.json && npm run lint && npm test` + `npx tsc --noEmit -p tsconfig.json` tolerando só o erro antigo `test/app.e2e-spec.ts(4,21) TS2307` |
| Quick (Front) | Tarefas com testes no front | `cd soft-go-II && npm test` |
| Build (Front) | Tipos e fim de fase do front | `cd soft-go-II && npm run lint && npm run build && npm test` |

---

## Execution Plan

Phases are ordered and run sequentially - each phase completes before the next begins, and tasks within a phase execute in order.

### Phase 1: API dados e auth

```
T1 → T2 → T3 → T4
```

### Phase 2: API inscrição e listagem

```
T5 → T6 → T7 → T8
```

### Phase 3: Front base

```
T9 → T10 → T11 → T12
```

### Phase 4: Front UI

```
T13 → T14 → T15 → T16
```

---

## Task Breakdown

### T1: `users.phone` (migration + entidade)

**What**: Migration `AddColumnPhoneInUsersTable` (`ADD "phone" character varying(15)`, `down` = `DROP COLUMN`). `User.phone: string | null` (`varchar(15)`, `nullable: true`).
**Where**: `soft-go-ii-api/src/migrations/<timestamp>-AddColumnPhoneInUsersTable.ts`
**Also touches**: `src/users/entities/user.entity.ts`, `src/users/entities/user.entity.spec.ts`
**Depends on**: None
**Reuses**: `src/migrations/1790702705512-AddColumnSpotsInRideTable.ts`
**Requirement**: JOIN-14

**Tools**:
- MCP: NONE
- Skill: `migration` (API)

**Done when**:
- [x] Teste de metadata: coluna `phone` com `nullable: true` e `length: 15`
- [x] No Postgres local: `migrations:run`, depois `migrations:revert`, depois `migrations:run`, sem erro. Não apaga dados
- [x] `migration:generate --dryrun` sem diferenças
- [x] Gate build (API) passa

**Tests**: unit
**Gate**: build
**Commit**: `feat: add phone column to users`
**Status**: ✅ Done — `soft-go-ii-api@1f48a76`

---

### T2: `ride_users` ligado a `users` (migration + entidade)

**What**: Migration `LinkRideUsersToUsers`:
- `up`:
  1. `DELETE FROM "ride_users"`.
  2. Remove `UQ_ride_phone`, `name` e `phone`.
  3. Adiciona `user_id integer NOT NULL`.
  4. Adiciona `FK_ride_users_user` → `users(id)`.
  5. Adiciona `UQ_ride_user (ride_id, user_id)`.
- `down`: `DELETE FROM "ride_users"` e reverte a estrutura.

Entidade `RideUser`: sem `name` e `phone`; `userId` + relação `user` (`foreignKeyConstraintName: 'FK_ride_users_user'`); `@Unique('UQ_ride_user', ['rideId', 'userId'])`. Ajustar o mínimo no `ride-users.service.ts` e no `CreateRideUserDto` só se o build quebrar. A lógica nova vem no T5.
**Where**: `soft-go-ii-api/src/migrations/<timestamp>-LinkRideUsersToUsers.ts`
**Also touches**: `src/ride-users/entities/ride-user.entity.ts`, `src/ride-users/entities/ride-user.entity.spec.ts`
**Depends on**: T1
**Reuses**: migrations e entidades existentes
**Requirement**: JOIN-28, JOIN-11 (constraint)

**Tools**:
- MCP: NONE
- Skill: `migration` (API)

**Done when**:
- [x] Teste de metadata: `userId` não nullable, `UQ_ride_user` em `['rideId','userId']`, FK `FK_ride_users_user`, sem colunas `name`/`phone`
- [x] **Banco temporário** com todas as migrations: `run`, depois `revert` desta, depois `run`, sem erro. Com uma linha antiga semeada, ela some depois do `up`
- [x] **Pedir confirmação da usuária antes** de rodar no banco local, porque apaga as 35 inscrições
- [x] No banco local (após o "sim"): `migrations:run` e `migration:generate --dryrun` sem diferenças
- [x] Gate build (API) passa

**Tests**: unit
**Gate**: build
**Commit**: `feat: link ride users to users`
**Status**: ✅ Done — `soft-go-ii-api@82c062a`

---

### T3: Telefone no cadastro e nas respostas de auth

**What**:
- `SignUpDto.phone?` com os decorators de telefone BR do `CreateRideDto` (vazio vira `undefined`).
- `AuthService.signUp` salva `phone ?? null`.
- `PublicUser` e `toPublicUser` ganham `phone`.
- `@ApiBody` do signup com `phone`.

**Where**: `soft-go-ii-api/src/auth/auth.service.ts`
**Also touches**: `dto/sign-up.dto.ts`, `auth.controller.ts` (ApiBody), `auth.service.spec.ts`, `auth.controller.spec.ts`
**Depends on**: T2
**Reuses**: `src/ride/dto/create-ride.dto.ts:47-54`
**Requirement**: JOIN-13, JOIN-14, JOIN-15, JOIN-19

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Unit: signup com telefone salva `phone`. Sem telefone, salva `null`. `user` do retorno tem `phone` em signup, signin e me
- [x] HTTP: signup com `phone` válido → 201 e `user.phone`. `phone: ""` → 201 com `phone: null`. `phone` inválido (`"123"`) → 400 com a mensagem exata. `GET /auth/me` traz `phone`
- [x] Os testes existentes de auth continuam passando, ajustados só no `toEqual` do `user`, que agora inclui `phone`
- [x] Gate quick (API) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: accept phone on sign up`
**Status**: ✅ Done — `soft-go-ii-api@aa398a6`

---

### T4: `OptionalJwtAuthGuard`

**What**: Guard que sempre retorna `true`. Com Bearer válido, preenche `request.user = { id, email }`. Nos outros casos, deixa `request.user` indefinido. É exportado pelo `AuthModule`.
**Where**: `soft-go-ii-api/src/auth/optional-jwt-auth.guard.ts`
**Also touches**: `optional-jwt-auth.guard.spec.ts`, `auth.module.ts`
**Depends on**: T3
**Reuses**: extração do Bearer de `jwt-auth.guard.ts`
**Requirement**: JOIN-21, JOIN-22

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: token válido → `true` e `request.user` exato. Sem header, esquema diferente, malformado, outro segredo e expirado → `true` e `request.user` indefinido
- [x] Gate build (API) passa (fim da Phase 1)

**Tests**: unit
**Gate**: build
**Commit**: `feat: add optional jwt auth guard`
**Status**: ✅ Done — `soft-go-ii-api@ed667cd (API: 63 testes)`

---

### T5: `RideUsersService.join` + `CreateRideUserDto`

**What**:
- `CreateRideUserDto` passa a ser `{ rideId, phone? }`, com os decorators BR.
- `join(userId, dto)`, na ordem:
  1. Carona existe? Se não, 404.
  2. Duplicidade? Se sim, 409 "Você já confirmou presença nesta carona".
  3. Vaga? Se lotada, o 409 atual.
  4. Se veio `phone`, atualiza `users.phone`.
  5. `save({ rideId, userId })`. O erro 23505 vira o 409 de duplicidade.
  6. Retorna `{ id, rideId, userId }`.
- Remove o `create` antigo.
- `RideUsersModule` importa `UsersModule`.

**Where**: `soft-go-ii-api/src/ride-users/ride-users.service.ts`
**Also touches**: `dto/create-ride-user.dto.ts`, `ride-users.module.ts`, `ride-users.service.spec.ts`
**Depends on**: None (Phase 1 concluída)
**Reuses**: padrão 23505 de `auth.service.ts`
**Requirement**: JOIN-03, JOIN-06, JOIN-10, JOIN-11, JOIN-17, JOIN-26

**Tools**:
- MCP: NONE
- Skill: `novo-recurso` (API)

**Done when**:
- [x] Testes: sucesso salva `{ rideId, userId }` e retorna `{ id, rideId, userId }`. Carona inexistente → `NotFoundException` "Corrida não encontrada". Duplicada → 409 com a mensagem exata, sem `save` e sem update de telefone. Lotada → 409 de vagas, sem `save` e sem update. Com `phone` → update de `phone` na conta antes do `save`. Sem `phone` → nenhum update. 23505 → 409 de duplicidade. Outro erro → relançado
- [x] Gate quick (API) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: join ride as logged user`
**Status**: ✅ Done — `soft-go-ii-api@dfac3e1 (o create antigo sai na T6, junto com o controller)`

---

### T6: `POST /ride-users` protegido

**What**:
- `@UseGuards(JwtAuthGuard)` e `@ApiBearerAuth()` no `POST`. Ele chama `join(request.user.id, dto)`.
- `@ApiBody` com `{ rideId, phone? }`.
- `RideUsersModule` importa `AuthModule`.
- Atualizar o teste AUTH-23 em `auth.controller.spec.ts` (AD-003): `RideUsersController.create` agora tem `JwtAuthGuard`, e os outros métodos não.

**Where**: `soft-go-ii-api/src/ride-users/ride-users.controller.ts`
**Also touches**: `ride-users.module.ts`, `ride-users.controller.spec.ts`, `src/auth/auth.controller.spec.ts`
**Depends on**: T5
**Reuses**: formato de `auth.controller.spec.ts` (módulo real + `overrideModule` + `setupApp`)
**Requirement**: JOIN-02, JOIN-03, JOIN-04, JOIN-07, JOIN-10, JOIN-19

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] HTTP (repos fake):
  - com token → 201 `{ id, rideId, userId }`, com o `userId` do token;
  - sem token, token inválido e token expirado → 401, sem nada criado;
  - body com `name` → 400;
  - `phone` inválido → 400;
  - segunda confirmação → 409 com a mensagem exata.
- [x] Teste "Nenhum guard global" continua passando. O teste das rotas abertas foi atualizado conforme o AD-003
- [x] Gate quick (API) passa

**Tests**: integration
**Gate**: quick
**Commit**: `feat: require login to join a ride`
**Status**: ✅ Done — `soft-go-ii-api@2886d69`

---

### T7: Participantes no `GET /ride` (service)

**What**: `findAllRides(transportType?, date?, city?, viewer?)` carrega `rideUser.user` e devolve `participants: [{ userId, name: primeiro nome }]`. Com `viewer`, cada participante ganha também `phone`. O cálculo de vagas não muda. `findOneRide` repassa o `viewer`.
**Where**: `soft-go-ii-api/src/ride/ride.service.ts`
**Also touches**: `ride.service.spec.ts`
**Depends on**: T6
**Reuses**: lógica atual de vagas
**Requirement**: JOIN-20, JOIN-21, JOIN-22, JOIN-25

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes:
  - `participants` com `userId` e primeiro nome ("Ana Maria Souza" → "Ana");
  - sem `viewer`, nenhum participante tem a chave `phone`;
  - com `viewer`, `phone` traz o valor ou `null`;
  - nenhum campo de senha nem `email` aparece em `participants`;
  - `transportType.spots` desconta as inscrições como antes.
- [x] Gate quick (API) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: list ride participants`
**Status**: ✅ Done — `soft-go-ii-api@134058c`

---

### T8: `GET /ride` com token opcional

**What**: `@UseGuards(OptionalJwtAuthGuard)` em `GET /ride` e `GET /ride/:id`, passando `request.user` como `viewer`. `RideModule` importa `AuthModule`.
**Where**: `soft-go-ii-api/src/ride/ride.controller.ts`
**Also touches**: `ride.module.ts`, `ride.controller.spec.ts`
**Depends on**: T7
**Reuses**: formato dos testes HTTP de auth
**Requirement**: JOIN-09, JOIN-21, JOIN-22

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] HTTP (repos fake): `GET /ride` sem token → 200 e nenhum `phone` em `participants`. Com token válido → 200 com `phone`. Token inválido → 200 sem `phone` (não 401). O mesmo vale para `GET /ride/:id`. `POST /ride` sem token continua 201 ou 400 por validação, nunca 401
- [x] Subir a API compilada contra o Postgres local e fazer um smoke: `GET /ride` 200, `POST /ride-users` sem token 401
- [x] Gate build (API) passa (fim da Phase 2)

**Tests**: integration
**Gate**: build
**Commit**: `feat: show participant phones to logged users`
**Status**: ✅ Done — `soft-go-ii-api@b10541a (API: 95 testes)`

---

### T9: Tipos + sessão tolerante a `phone`

**What**:
- `types/index.ts`:
  - `User.phone: string | null`;
  - `SignUpPayload.phone?`;
  - `Participant`;
  - `Ride.participants`;
  - `JoinRidePayload` substitui `RideUser`.
- `session.ts`: o `isSession` não exige `phone`.

**Where**: `soft-go-II/src/service/session.ts`
**Also touches**: `types/index.ts`, `session.test.ts`
**Depends on**: None (Phase 2 concluída)
**Reuses**: —
**Requirement**: JOIN-15, JOIN-17

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Teste: sessão salva sem `phone`, no formato antigo, continua válida (`getSession()` não é `null`). Sessão com `phone` é devolvida igual
- [x] Os testes de sessão existentes continuam passando
- [x] Gate build (front) passa. `tsc -b` acusa os usos antigos de `RideUser`, que são ajustados no mínimo necessário

**Tests**: unit
**Gate**: build
**Commit**: `feat: add participant and phone types`
**Status**: ✅ Done — `soft-go-II@60defbb`

---

### T10: Serviços: inscrição e cadastro com telefone

**What**: `ride.service.createRideUser(payload: JoinRidePayload)` envia só `{ rideId, phone? }`. `auth.service.signUp` repassa `phone` quando presente.
**Where**: `soft-go-II/src/service/ride.service.ts`
**Also touches**: `ride.service.test.ts`, `auth.service.ts`, `auth.service.test.ts`
**Depends on**: T9
**Reuses**: padrão de `auth.service.ts`
**Requirement**: JOIN-02, JOIN-13

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes (api mockado): `createRideUser({ rideId: 3 })` → `POST /ride-users` com `{ rideId: 3 }` exato. Com `phone` → inclui `phone`. Nunca envia `name`. `signUp` com `phone` → body inclui `phone`. Sem `phone` → body sem a chave
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: send phone on join and sign up`
**Status**: ✅ Done — `soft-go-II@b97995a`

---

### T11: Schema de telefone opcional

**What**: `optionalPhone` em `schemas/phone.schema.ts`: vazio, ou 11 dígitos numéricos com "Número inválido, informe 11 dígitos (DDD + número)". `signUpSchema` ganha `phone: optionalPhone`. `joinRideSchema = { phone: optionalPhone }`.
**Where**: `soft-go-II/src/schemas/phone.schema.ts`
**Also touches**: `phone.schema.test.ts`, `auth.schema.ts`, `auth.schema.test.ts`
**Depends on**: T10
**Reuses**: regra de telefone de `pages/Form.tsx`
**Requirement**: JOIN-18

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: `""` válido. `"51999998888"` válido. `"5199999888"` (10 dígitos), `"519999988889"` (12) e `"51a99998888"` → erro com a mensagem exata. `signUpSchema` com `phone` inválido → erro em `phone`. Sem `phone` → válido
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add optional phone schema`
**Status**: ✅ Done — `soft-go-II@9c7971e`

---

### T12: `updateUser` no `AuthContext`

**What**: `updateUser(user)` grava na sessão (`saveSession({ ...session, user })`) e no estado. O `getMe()` do mount passa a usar `updateUser`.
**Where**: `soft-go-II/src/context/AuthContext.tsx`
**Also touches**: `AuthContext.test.tsx`
**Depends on**: T11
**Reuses**: `session.ts`
**Requirement**: JOIN-17

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: `updateUser({ ...user, phone })` → `getSession().user.phone` atualizado, token preservado e o consumidor vê o novo `phone`. `getMe` no mount com `phone` novo → a sessão salva reflete o `phone`
- [x] Gate build (front) passa (fim da Phase 3)

**Tests**: unit
**Gate**: build
**Commit**: `feat: allow updating the logged user`
**Status**: ✅ Done — `soft-go-II@f932703 (front: 81 testes)`

---

### T13: WhatsApp opcional no cadastro

**What**: Campo "WhatsApp (opcional)" (`id="phone"`) com erro por campo. O payload inclui `phone` só quando preenchido.
**Where**: `soft-go-II/src/pages/SignUp.tsx`
**Also touches**: `SignUp.test.tsx`
**Depends on**: None (Phase 3 concluída)
**Reuses**: `InputForm` com `error`
**Requirement**: JOIN-13, JOIN-18

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: campo visível com o rótulo exato. Telefone inválido → mensagem abaixo do campo, `aria-invalid` e serviço não chamado. Com telefone → `signUp` recebe `phone`. Sem telefone → `signUp` sem `phone`. Os testes existentes do cadastro continuam passando
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add optional phone to sign up form`
**Status**: ✅ Done — `soft-go-II@361d912`

---

### T14: Participantes no card

**What**: Props `participants`, `currentUserId?` e `showPhones`. Mostra "Vão: Ana, Bia". Com `showPhones`, quem tem `phone` vira link `https://wa.me/<phone>`. Se `currentUserId` está nos participantes, mostra "Você vai nesta carona" no lugar do "Vou junto".
**Where**: `soft-go-II/src/components/Card.tsx`
**Also touches**: `Card.test.tsx`
**Depends on**: T13
**Reuses**: estilo atual do card e do link de WhatsApp
**Requirement**: JOIN-12, JOIN-23, JOIN-24

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes:
  - nomes listados;
  - sem `showPhones`, nenhum link `wa.me` de participante;
  - com `showPhones`, link com o `href` exato para quem tem telefone e nenhum link para quem não tem;
  - usuária na lista → "Você vai nesta carona", sem o botão "Vou junto";
  - fora da lista e com vaga → botão "Vou junto".
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: show participants on ride card`
**Status**: ✅ Done — `soft-go-II@3a45cc8`

---

### T15: Modal de confirmação

**What**: Reescrever o `Modal` com props `ride`, `onClose`, `onConfirm(phone?)` e `askPhone`. Usa RHF + `joinRideSchema`. O campo "WhatsApp" (`id="phone"`) aparece só com `askPhone`. Botão "Confirmar presença". Não há campo de nome.
**Where**: `soft-go-II/src/components/Modal.tsx`
**Also touches**: `Modal.test.tsx`
**Depends on**: T14
**Reuses**: layout atual do modal, `InputForm`, `Card` (`showButton={false}`)
**Requirement**: JOIN-01, JOIN-16, JOIN-18

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes:
  - sem `askPhone`: nenhum campo de texto, e "Confirmar presença" chama `onConfirm(undefined)`;
  - com `askPhone`: campo visível; vazio chama `onConfirm(undefined)`; válido chama `onConfirm("51999998888")`; inválido mostra a mensagem e não chama `onConfirm`;
  - nunca há campo "Nome";
  - fechar chama `onClose`.
- [x] Gate quick (front) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: turn join modal into a confirmation`
**Status**: ✅ Done — `soft-go-II@25f42c8 (inclui ligação mínima no Home para o build; fluxo completo na T16)`

---

### T16: Fluxo "Vou junto" no Home

**What**: `handleJoin`: deslogada vai para `/login`, logada abre o modal com `askPhone = !user.phone`. `handleConfirm(phone?)`:
- sucesso → toast "Presença confirmada!", `updateUser` com o `phone` (se veio) e `loadRides()`;
- 409 ou 404 → toast com `response.data.message`;
- 401 → nenhum toast extra;
- outros erros → toast "Não foi possível confirmar presença. Tente novamente.";
- fecha o modal em todos os casos.

`loadRides` depende de `user?.id`. O Card recebe `participants`, `currentUserId` e `showPhones = !!user`. O schema antigo de nome e telefone sai do Home.
**Where**: `soft-go-II/src/pages/Home.tsx`
**Also touches**: `Home.test.tsx`
**Depends on**: T15
**Reuses**: `useAuth`, `Toast`, `ride.service`
**Requirement**: JOIN-01, JOIN-02, JOIN-05, JOIN-06, JOIN-08, JOIN-10, JOIN-12, JOIN-16, JOIN-17, JOIN-24, JOIN-27

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes (serviços mockados, `MemoryRouter` + `AuthProvider`):
  - deslogada clica "Vou junto" → `/login`, sem modal;
  - logada com telefone → modal sem campo; confirmar chama `createRideUser({ rideId })`, mostra "Presença confirmada!" e recarrega;
  - logada sem telefone → modal com campo; com o número, sessão com `phone` e próximo modal sem campo;
  - 409 → toast com a mensagem da API;
  - 401 → nenhum toast de sucesso;
  - logada → card recebe telefones; deslogada → não recebe.
- [x] Gate build (front) passa (fim da Phase 4). Contagem total do front registrada

**Tests**: unit
**Gate**: build
**Commit**: `feat: join rides with the logged user`
**Status**: ✅ Done — `soft-go-II@226bf5a (front: 108 testes)`

---

## Phase Execution Map

```
Phase 1 → Phase 2 → Phase 3 → Phase 4

Phase 1 (API):   T1 → T2 → T3 → T4
Phase 2 (API):   T5 → T6 → T7 → T8
Phase 3 (Front): T9 → T10 → T11 → T12
Phase 4 (Front): T13 → T14 → T15 → T16
```

Execution is strictly sequential.

---

## Task Granularity Check

| Task | Scope | Status |
| ---- | ----- | ------ |
| T1 | 1 migration + 1 coluna na entidade | ⚠️ coeso (migration e entidade precisam bater) |
| T2 | 1 migration + 1 entidade | ⚠️ coeso (idem) |
| T3 | telefone no fluxo de signup (DTO + service) | ⚠️ coeso |
| T4 | 1 guard | ✅ |
| T5 | 1 método + DTO | ⚠️ coeso |
| T6 | 1 rota + wiring | ✅ |
| T7 | 1 método | ✅ |
| T8 | 2 rotas de leitura do mesmo controller + wiring | ✅ |
| T9 | tipos + 1 função | ✅ |
| T10 | 2 funções de serviço | ⚠️ coeso |
| T11 | 1 schema | ✅ |
| T12 | 1 função de contexto | ✅ |
| T13 | 1 campo | ✅ |
| T14 | 1 componente | ✅ |
| T15 | 1 componente | ✅ |
| T16 | 1 página | ✅ |

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| ---- | ---------------------- | ------------- | ------ |
| T1 | None | início da Phase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | None (Phase 1 concluída) | início da Phase 2 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |
| T9 | None (Phase 2 concluída) | início da Phase 3 | ✅ |
| T10 | T9 | T9 → T10 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T12 | T11 | T11 → T12 | ✅ |
| T13 | None (Phase 3 concluída) | início da Phase 4 | ✅ |
| T14 | T13 | T13 → T14 | ✅ |
| T15 | T14 | T14 → T15 | ✅ |
| T16 | T15 | T15 → T16 | ✅ |

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| ---- | --------------------------- | --------------- | --------- | ------ |
| T1 | API migration + entity | none + unit (metadata) | unit | ✅ |
| T2 | API migration + entity | none + unit (metadata) | unit | ✅ |
| T3 | API service + DTO | unit (+ HTTP para o DTO) | unit | ✅ |
| T4 | API guard | unit | unit | ✅ |
| T5 | API service + DTO | unit | unit | ✅ |
| T6 | API controller | integration | integration | ✅ |
| T7 | API service | unit | unit | ✅ |
| T8 | API controller | integration | integration | ✅ |
| T9 | Front lógica + tipos | unit | unit | ✅ |
| T10 | Front lógica | unit | unit | ✅ |
| T11 | Front lógica | unit | unit | ✅ |
| T12 | Front context | unit | unit | ✅ |
| T13 | Front página | unit | unit | ✅ |
| T14 | Front componente | unit | unit | ✅ |
| T15 | Front componente | unit | unit | ✅ |
| T16 | Front página | unit | unit | ✅ |

## Requirement Coverage

Todos os 28 requisitos estão mapeados:

| Requisito | Tarefas |
| --------- | ------- |
| JOIN-01 | T15, T16 |
| JOIN-02 | T6, T10, T16 |
| JOIN-03 | T5, T6 |
| JOIN-04 | T6 |
| JOIN-05 | T16 |
| JOIN-06 | T5, T16 |
| JOIN-07 | T6 |
| JOIN-08 | T16 |
| JOIN-09 | T8 |
| JOIN-10 | T5, T6, T16 |
| JOIN-11 | T2, T5 |
| JOIN-12 | T14, T16 |
| JOIN-13 | T3, T10, T13 |
| JOIN-14 | T1, T3 |
| JOIN-15 | T3, T9 |
| JOIN-16 | T15, T16 |
| JOIN-17 | T5, T9, T12, T16 |
| JOIN-18 | T11, T13, T15 |
| JOIN-19 | T3, T6 |
| JOIN-20 | T7 |
| JOIN-21 | T4, T7, T8 |
| JOIN-22 | T4, T7, T8 |
| JOIN-23 | T14 |
| JOIN-24 | T14, T16 |
| JOIN-25 | T7 |
| JOIN-26 | T5 |
| JOIN-27 | T16 |
| JOIN-28 | T2 |
