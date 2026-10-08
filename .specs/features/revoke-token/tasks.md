# Revogar Sessões ao Sair Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/revoke-token/design.md`
**Spec**: `.specs/features/revoke-token/spec.md`
**Status**: In Progress

**Repositórios**:
- T1–T4 commitam em `soft-go-ii-api/` (`master`). T5 commita em `soft-go-II/` (`main`).
- Depois de cada commit do submódulo, um commit no repo raiz leva `tasks.md`, `spec.md` e o bump do ponteiro.
- Sempre `git add` com caminhos explícitos. Nunca `tsconfig.build.tsbuildinfo`, `dist/`, `.env`.

**Python**: `python` (não `python3`).

**Lições aplicadas**:
- L-004 (confirmada): mensagem exata.
- Candidatas:
  - L-001: guard global;
  - L-002: metadata de coluna;
  - L-006: rejeição confere estado persistido;
  - L-007: caminho sem valor não apaga dado.

**Baseline**: API 130 testes · front 140 testes.

---

## Test Coverage Matrix

> Generated from codebase, project guidelines, and spec - confirm before Execute. Guidelines found: `soft-go-ii-api/CLAUDE.md`, `soft-go-ii-api/vitest.config.ts`, `soft-go-II/vite.config.ts`, AD-002. Piso de estilo: `src/auth/*.spec.ts` e `soft-go-II/src/context/AuthContext.test.tsx`.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| ---------- | ------------------ | -------------------- | ---------------- | ----------- |
| API service (`auth.service`) | unit | Todos os ramos; 1:1 com os REVOKE da tarefa | `soft-go-ii-api/src/**/*.spec.ts` | `npm test` |
| API guards + `verifySession` | unit | Token válido, versão antiga, sem `ver`, conta inexistente, assinatura inválida, expirado | `soft-go-ii-api/src/auth/*.spec.ts` | `npm test` |
| API controller (HTTP) | integration (Nest testing + supertest, repositórios fake) | Rota nova: feliz + 401s + estado persistido; fluxo sair → token antigo recusado → entrar de novo | `soft-go-ii-api/src/**/*.controller.spec.ts` | `npm test` |
| API entity | unit (metadata, L-002) | `tokenVersion`: nome, not null, default 0 | `soft-go-ii-api/src/users/entities/*.spec.ts` | `npm test` |
| API migration / module | none | build gate + `run → revert → run` em banco temporário | - | build gate |
| Front lógica (`service/*`, `context/*`) | unit | Todos os ramos; 1:1 com os REVOKE | `soft-go-II/src/**/*.test.ts(x)` | `npm test` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| ---------- | ----------- | ------- |
| Quick (API) | Tarefas com unit na API | `cd soft-go-ii-api && npm test` |
| Build (API) | Entity/migration, HTTP, fim de fase | `cd soft-go-ii-api && npx tsc --noEmit -p tsconfig.build.json && npm run lint && npm test` + `npx tsc --noEmit -p tsconfig.json` tolerando só `test/app.e2e-spec.ts(4,21) TS2307` |
| Build (Front) | Fim de fase do front | `cd soft-go-II && npm run lint && npm run build && npm test` |

---

## Execution Plan

### Phase 1: API

```
T1 → T2 → T3 → T4
```

### Phase 2: Front

```
T5
```

---

## Task Breakdown

### T1: `users.token_version` (migration + entidade)

**What**: Migration `AddTokenVersionInUsersTable` (`ADD "token_version" integer NOT NULL DEFAULT 0`; `down` = `DROP COLUMN`). `User.tokenVersion: number` (`name: 'token_version'`, `int`, `nullable: false`, `default: 0`).
**Where**: `soft-go-ii-api/src/migrations/<timestamp>-AddTokenVersionInUsersTable.ts`
**Also touches**: `src/users/entities/user.entity.ts`, `src/users/entities/user.entity.spec.ts`
**Depends on**: None
**Reuses**: `1791413863720-AddColumnPhoneInUsersTable.ts`
**Requirement**: REVOKE-14, REVOKE-15

**Tools**:
- MCP: NONE
- Skill: `migration` (API)

**Done when**:
- [x] Metadata: `tokenVersion` com `name: 'token_version'`, `nullable: false`, `default: 0`
- [x] Banco temporário: com uma conta semeada antes, o `up` deixa `token_version = 0` e o resto igual; o `revert` remove só a coluna; o `run` de novo funciona
- [x] Banco local: `migrations:run` + `migration:generate --dryrun` sem diferenças (não apaga dados)
- [x] Gate build (API) passa

**Tests**: unit
**Gate**: build
**Commit**: `feat: add token version to users`
**Status**: ✅ Done — `soft-go-ii-api@cdc8bf0` (API: 131 testes; migration aplicada no banco local)

---

### T2: `ver` no token + `AuthService.signOut`

**What**: `issueToken` inclui `ver: user.tokenVersion`. `signOut(userId)` faz `usersRepository.increment({ id: userId }, 'tokenVersion', 1)`. As respostas continuam sem `tokenVersion`.
**Where**: `soft-go-ii-api/src/auth/auth.service.ts`
**Also touches**: `src/auth/auth.service.spec.ts`
**Depends on**: T1
**Reuses**: `issueToken`, `toPublicUser`
**Requirement**: REVOKE-01, REVOKE-05, REVOKE-13, REVOKE-16

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Unit:
  - signup e signin → token decodificado tem `ver` igual ao `tokenVersion` da conta (ex.: 3);
  - `user` da resposta sem `tokenVersion`;
  - `signOut(7)` → `increment({ id: 7 }, 'tokenVersion', 1)` e nenhum `update`/`save` de outros campos
- [x] Gate quick (API) passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat: carry session version in tokens`
**Status**: ✅ Done — `soft-go-ii-api@2419799` (API: 135 testes)

---

### T3: Guards conferem a versão

**What**: Cria `verifySession`; `JwtAuthGuard` e `OptionalJwtAuthGuard` passam a usá-lo, injetando `UsersRepository`. Os fakes das specs HTTP existentes ganham `findOneBy` (conta com `tokenVersion: 0`) e os tokens de teste passam a levar `ver: 0`, mudança só de setup.
**Where**: `soft-go-ii-api/src/auth/verify-session.ts`
**Also touches**: `src/auth/jwt-auth.guard.ts`, `src/auth/optional-jwt-auth.guard.ts`, `src/auth/jwt-auth.guard.spec.ts`, `src/auth/optional-jwt-auth.guard.spec.ts`, specs HTTP de ride/ride-users/auth (setup)
**Depends on**: T2
**Reuses**: guards atuais
**Requirement**: REVOKE-06, REVOKE-07, REVOKE-08, REVOKE-09, REVOKE-10

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] Unit `JwtAuthGuard`:
  - `ver` igual → libera e `request.user = { id, email }`;
  - `ver` diferente → `401` "Não autenticado";
  - sem `ver` → `401`;
  - conta inexistente → `401`;
  - casos atuais (sem header, outro segredo, expirado) mantidos
- [ ] Unit `OptionalJwtAuthGuard`: os mesmos casos → libera sempre; `request.user` só no caso válido
- [ ] Gate build (API) passa (todas as specs HTTP verdes)

**Tests**: unit
**Gate**: build
**Commit**: `feat: reject tokens from an old session version`

---

### T4: `POST /auth/signout`

**What**: Rota `signout` (`204`, `JwtAuthGuard`, Swagger `ApiBearerAuth`) chamando `authService.signOut(request.user.id)`. Mapa de guards da AD-003 atualizado com `signOut: [JwtAuthGuard]`.
**Where**: `soft-go-ii-api/src/auth/auth.controller.ts`
**Also touches**: `src/auth/auth.controller.spec.ts`
**Depends on**: T3
**Reuses**: `GET /auth/me`
**Requirement**: REVOKE-01, REVOKE-02, REVOKE-06, REVOKE-09, REVOKE-11, REVOKE-12, REVOKE-13

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] HTTP (repositório de usuárias em memória):
  - signin → signout com o token: `204`, corpo vazio, `tokenVersion` da conta 0 → 1;
  - depois: `GET /auth/me` com o token antigo → `401` "Não autenticado";
  - signin de novo → token novo → `GET /auth/me` `200`;
  - conta B logada antes do signout da A → `GET /auth/me` com o token da B `200`;
  - signout sem token / token inválido / token já revogado → `401`, e `tokenVersion` de nenhuma conta muda;
  - signout não muda nome, e-mail, telefone nem `passwordHash`
- [ ] HTTP mural (REVOKE-09): `GET /ride` com token de versão antiga → `owner` e `participants` sem `phone` (em `ride.controller.spec.ts`)
- [ ] Guard map com `signOut: [JwtAuthGuard]`; "Nenhum guard global" verde (L-001)
- [ ] Gate build (API) passa (fim da Phase 1)

**Tests**: integration
**Gate**: build
**Commit**: `feat: add sign out route that revokes sessions`

---

### T5: "Sair" chama a API

**What**:
- `authService.signOut(token)` → `api.post('/auth/signout', null, { headers: { Authorization: 'Bearer <token>' } })`.
- `AuthContext.signOut`:
  1. guarda o token;
  2. limpa a sessão e o usuário e vai para `/login`;
  3. chama `authService.signOut(token)` ignorando erros.
**Where**: `soft-go-II/src/context/AuthContext.tsx`
**Also touches**: `src/service/auth.service.ts`, `src/service/auth.service.test.ts`, `src/context/AuthContext.test.tsx`
**Depends on**: None (Phase 1 concluída)
**Reuses**: `clearSession`, `getSession`
**Requirement**: REVOKE-03, REVOKE-04

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] Unit service: `signOut("abc")` → `post("/auth/signout", null, { headers: { Authorization: "Bearer abc" } })`
- [ ] Unit context:
  - "sair" chama `authService.signOut` com o token da sessão, limpa o `localStorage` e vai para `/login`;
  - com a API rejeitando (`401`, `500`, rede) a sessão também é limpa, vai para `/login` e nenhum toast "Sua sessão expirou" aparece
- [ ] Gate build (front) passa

**Tests**: unit
**Gate**: build
**Commit**: `feat: revoke sessions on sign out`

---

## Phase Execution Map

```
Phase 1 → Phase 2

Phase 1 (API):   T1 → T2 → T3 → T4
Phase 2 (Front): T5
```

---

## Task Granularity Check

| Task | Scope | Status |
| ---- | ----- | ------ |
| T1 | 1 migration + 1 coluna | ⚠️ coeso |
| T2 | 2 métodos do mesmo service | ✅ |
| T3 | 1 função + 2 guards que a usam | ⚠️ coeso (mesma regra) |
| T4 | 1 rota | ✅ |
| T5 | 1 função de serviço + 1 função de contexto | ⚠️ coeso |

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| ---- | ---------------------- | ------------- | ------ |
| T1 | None | início da Phase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | None (Phase 1 concluída) | início da Phase 2 | ✅ |

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| ---- | --------------------------- | --------------- | --------- | ------ |
| T1 | API migration + entity | none + unit | unit | ✅ |
| T2 | API service | unit | unit | ✅ |
| T3 | API guards | unit | unit | ✅ |
| T4 | API controller | integration | integration | ✅ |
| T5 | Front service + context | unit | unit | ✅ |

## Requirement Coverage

| Requisito | Tarefas |
| --------- | ------- |
| REVOKE-01 | T2, T4 |
| REVOKE-02 | T4 |
| REVOKE-03 | T5 |
| REVOKE-04 | T5 |
| REVOKE-05 | T2 |
| REVOKE-06 | T3, T4 |
| REVOKE-07 | T3 |
| REVOKE-08 | T3 |
| REVOKE-09 | T3, T4 |
| REVOKE-10 | T3 |
| REVOKE-11 | T4 |
| REVOKE-12 | T4 |
| REVOKE-13 | T2, T4 |
| REVOKE-14 | T1 |
| REVOKE-15 | T1 |
| REVOKE-16 | T2 |
