# Auth Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/auth/design.md`
**Spec**: `.specs/features/auth/spec.md` · **Context**: `.specs/features/auth/context.md`
**Status**: Done (2026-10-07) — T1–T20 concluídas; Verifier PASS na iteração 2 (`validation.md`)

**Repositórios**: T1–T9 commitam em `soft-go-ii-api/`; T10–T20 commitam em `soft-go-II/`. `.specs/` vive no repo raiz `soft-go-pd` (os dois projetos são submódulos): o código vai no commit do submódulo; a marcação da tarefa em `tasks.md` + o bump do ponteiro do submódulo vão num commit no repo raiz logo em seguida. Sempre `git add` com caminhos explícitos (nunca `tsconfig.build.tsbuildinfo`, `dist/`, `.env`).

**Python**: neste Windows use `py` (não `python3`) para os scripts da skill.

---

## Test Coverage Matrix

> Generated from codebase, project guidelines, and spec - confirm before Execute. Guidelines found: `soft-go-ii-api/CLAUDE.md`, `soft-go-ii-api/.claude/skills/novo-recurso/SKILL.md` (§5: `<recurso>.service.spec.ts` com vitest, repositório mockado com `vi.fn()`, caminho feliz + cada exceção), `soft-go-ii-api/vitest.config.ts`, `CLAUDE.md` raiz (front sem testes → AD-002 adiciona vitest). Sem testes existentes além do e2e quebrado (`test/app.e2e-spec.ts`), então valem os strong defaults.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| ---------- | ------------------ | -------------------- | ---------------- | ----------- |
| API service (`auth.service.ts`) | unit | Todos os ramos; 1:1 com AUTH-01/02/04/05/09/14/15/16/19/20 | `soft-go-ii-api/src/**/*.spec.ts` | `npm test` |
| API guard / config factory | unit | Todos os ramos (token ausente, malformado, assinatura inválida, expirado, válido; segredo ausente/presente) | `soft-go-ii-api/src/**/*.spec.ts` | `npm test` |
| API controller (HTTP) | integration (Nest testing + supertest, repo fake, sem banco) | Toda rota nova: feliz + cada edge/erro do spec (AUTH-03/06/07/08/21/23/32) | `soft-go-ii-api/src/auth/auth.controller.spec.ts` | `npm test` |
| API entity / DTO / module / migration / setup | none | build gate (DTOs exercitados pelo teste HTTP) | - | build gate |
| Front lógica (`service/*`, `schemas/*`, `utils/*`) | unit | Todos os ramos; 1:1 com AUTH-10/11/24/26/31/33/34 | `soft-go-II/src/**/*.test.ts` | `npm test` |
| Front componentes / páginas / context | unit (jsdom + Testing Library, axios mockado) | Cada AC de UI: feliz + erro (AUTH-12/13/17/18/25/27/28/29/30) | `soft-go-II/src/**/*.test.tsx` | `npm test` |
| Front config (vitest, deps) | none | build gate | - | build gate |

## Gate Check Commands

> Generated from codebase - confirm before Execute.

| Gate Level | When to Use | Command |
| ---------- | ----------- | ------- |
| Quick (API) | Tarefas com unit/integration na API | `cd soft-go-ii-api && npm test` |
| Build (API) | Entity/config/migration e fim de fase da API | `cd soft-go-ii-api && npx tsc --noEmit -p tsconfig.json && npm run lint && npm test` |
| Quick (Front) | Tarefas com testes no front | `cd soft-go-II && npm test` |
| Build (Front) | Config/tooling e fim de fase do front | `cd soft-go-II && npm run lint && npm run build && npm test` |

`npm run test:e2e` da API não é gate (já falha hoje e exige banco).

---

## Execution Plan

Phases are ordered and run sequentially - each phase completes before the next begins, and tasks within a phase execute in order.

### Phase 1: API foundation

```
T1 → T2 → T3 → T4
```

### Phase 2: API auth

```
T5 → T6 → T7 → T8 → T9
```

### Phase 3: Front foundation

```
T10 → T11 → T12 → T13 → T14
```

### Phase 4: Front UI

```
T15 → T16 → T17 → T18 → T19 → T20
```

---

## Task Breakdown

### T1: Extrair `setupApp()` do `main.ts`

**What**: Criar `setupApp(app)` com `enableCors()` + o `ValidationPipe` atual (whitelist, forbidNonWhitelisted, transform) e fazer `main.ts` usá-la.
**Where**: `soft-go-ii-api/src/config/app.setup.ts`
**Also touches**: novo; `main.ts` passa a chamar
**Depends on**: None
**Reuses**: `src/main.ts:10-16`
**Requirement**: AUTH-32 (base para testes HTTP com a config real)

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] `main.ts` não instancia mais o `ValidationPipe` diretamente
- [x] Comportamento idêntico (mesmas opções)
- [x] Gate build (API) passa

**Tests**: none
**Gate**: build
**Commit**: `refactor: extract app setup for reuse in tests`
**Status**: ✅ Done — `soft-go-ii-api@36abd6f`

---

### T2: Entidade `User` + `UsersRepository` + `UsersModule`

**What**: Entidade `users` conforme design (`passwordHash` com `name: 'password_hash'` e `select: false`, `createdAt` `CreateDateColumn`), repositório customizado com `findByEmailWithPassword(email)`, módulo exportando o repositório, e registro de `User` em `postgres.config.service.ts` e `data-source.ts`.
**Where**: `soft-go-ii-api/src/users/`
**Also touches**: novo módulo
**Depends on**: T1
**Reuses**: `src/ride-users/{entities,ride-users.repository,ride-users.module}.ts`
**Requirement**: AUTH-02, AUTH-03 (hash oculto por padrão)

**Tools**:
- MCP: NONE
- Skill: `novo-recurso` (API)

**Done when**:
- [x] Colunas e tamanhos batem com o SQL do PRD
- [x] `User` registrada nos dois arquivos de config
- [x] Gate build (API) passa

**Tests**: none
**Gate**: build
**Commit**: `feat: add user entity and repository`
**Status**: ✅ Done — `soft-go-ii-api@51dda6a`

---

### T3: Migration `CreateTableUsers`

**What**: Migration manual criando `users` (SERIAL id, name VARCHAR(120), email VARCHAR(160) UNIQUE `UQ_users_email`, password_hash VARCHAR(255), created_at TIMESTAMP NOT NULL DEFAULT now()) com `down()` = `DROP TABLE "users"`.
**Where**: `soft-go-ii-api/src/migrations/<timestamp>-CreateTableUsers.ts`
**Depends on**: T2
**Reuses**: `src/migrations/1790702705512-AddColumnSpotsInRideTable.ts`
**Requirement**: AUTH-02, AUTH-05 (constraint UNIQUE)

**Tools**:
- MCP: NONE
- Skill: `migration` (API)

**Done when**:
- [x] Checklist de revisão da skill `migration` ok
- [x] Contra o Postgres **local** (`localhost`): `migrations:run` → `migrations:revert` → `migrations:run` sem erro (se o banco não estiver acessível, registrar e seguir; validar manualmente depois)
- [x] Gate build (API) passa

**Tests**: none
**Gate**: build
**Commit**: `feat: add users table migration`
**Status**: ✅ Done — `soft-go-ii-api@2495e10`

---

### T4: Dependências + `jwtConfigFactory`

**What**: Instalar `@nestjs/jwt@^12` e `bcryptjs@^3`; criar `jwtConfigFactory(config: ConfigService)` que lança `Error('JWT_SECRET não definido')` sem segredo e retorna `{ secret, signOptions: { expiresIn: JWT_EXPIRES_IN ?? '7d' } }`; adicionar `JWT_SECRET=` e `JWT_EXPIRES_IN=7d` ao `.env.example`.
**Where**: `soft-go-ii-api/src/auth/jwt.config.ts`
**Also touches**: + `jwt.config.spec.ts`, `package.json`, `.env.example`
**Depends on**: T3
**Reuses**: `ConfigModule` global de `app.module.ts`
**Requirement**: AUTH-19, AUTH-22

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: sem `JWT_SECRET` → lança com a mensagem; com segredo → retorna secret; sem `JWT_EXPIRES_IN` → `'7d'`; com valor → usa o valor
- [x] Gate quick (API) passa; test count: 4 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add jwt config factory`
**Status**: ✅ Done — `soft-go-ii-api@dd70533`

---

### T5: DTOs `SignUpDto` e `SignInDto`

**What**: DTOs com `class-validator` + `@Transform` (trim/lowercase) conforme design; mensagens PT-BR ("A senha deve ter entre 8 e 72 caracteres", etc.).
**Where**: `soft-go-ii-api/src/auth/dto/`
**Also touches**: `sign-up.dto.ts`, `sign-in.dto.ts`
**Depends on**: None (fase anterior concluída)
**Reuses**: `src/ride-users/dto/create-ride-user.dto.ts`
**Requirement**: AUTH-06, AUTH-07, AUTH-08, AUTH-09, AUTH-15

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Tipos TS batem com os validators; nenhum campo sem decorator
- [x] Gate build (API) passa (comportamento coberto pelo teste HTTP de T9)

**Tests**: none
**Gate**: build
**Commit**: `feat: add sign up and sign in dtos`
**Status**: ✅ Done — `soft-go-ii-api@138f367`

---

### T6: `AuthService.signUp`

**What**: `signUp` (pré-checagem de e-mail, `bcrypt.hash` custo 10, `save`, `23505` → `ConflictException('E-mail já cadastrado')`, token via `JwtService`, `toPublicUser`), com `issueToken` e `toPublicUser` privados.
**Where**: `soft-go-ii-api/src/auth/auth.service.ts`
**Also touches**: + `auth.service.spec.ts`
**Depends on**: T5
**Reuses**: padrão de erro de `ride-users.service.ts`
**Requirement**: AUTH-01, AUTH-02, AUTH-04, AUTH-05, AUTH-09, AUTH-19

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Teste: retorno `{ accessToken, user: { id, name, email } }` sem chaves de senha
- [x] Teste: valor passado a `save` tem `passwordHash !== senha` e `bcrypt.compare(senha, passwordHash) === true`
- [x] Teste: e-mail existente → `ConflictException` com "E-mail já cadastrado" e `save` não chamado
- [x] Teste: `save` rejeita `QueryFailedError` código `23505` → `ConflictException` (não 500); outro erro é relançado
- [x] Teste: `accessToken` decodifica com `sub` = id, `email`, e `exp - iat` = 7 dias
- [x] Gate quick (API) passa; test count: ≥ 6 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add sign up to auth service`
**Status**: ✅ Done — `soft-go-ii-api@106dd0f`

---

### T7: `AuthService.signIn` e `me`

**What**: `signIn` (busca com hash, `bcrypt.compare`, mesma `UnauthorizedException('E-mail ou senha inválidos')` nos dois ramos) e `me(userId)`.
**Where**: `soft-go-ii-api/src/auth/auth.service.ts`
**Also touches**: + spec
**Depends on**: T6
**Reuses**: helpers de T6
**Requirement**: AUTH-14, AUTH-15, AUTH-16, AUTH-20

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Teste: credenciais certas → `{ accessToken, user }` sem senha/hash
- [x] Teste: e-mail inexistente e senha errada → `UnauthorizedException` com mensagem e status idênticos
- [x] Teste: `me` devolve `{ id, name, email }`; usuário inexistente → `UnauthorizedException`
- [x] Gate quick (API) passa; test count: ≥ 5 novos, nenhum removido

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add sign in and me to auth service`
**Status**: ✅ Done — `soft-go-ii-api@ab69eff`

---

### T8: `JwtAuthGuard`

**What**: Guard que lê `Authorization: Bearer`, `verifyAsync`, popula `request.user = { id, email }`; qualquer falha → `UnauthorizedException('Não autenticado')`.
**Where**: `soft-go-ii-api/src/auth/jwt-auth.guard.ts`
**Also touches**: + `jwt-auth.guard.spec.ts`
**Depends on**: T7
**Reuses**: `JwtService`
**Requirement**: AUTH-20, AUTH-21

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: sem header; esquema diferente de Bearer; token malformado; assinatura de outro segredo; token expirado → 401; válido → `true` e `request.user` correto
- [x] Gate quick (API) passa; test count: ≥ 6 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add jwt auth guard`
**Status**: ✅ Done — `soft-go-ii-api@1f5564d`

---

### T9: `AuthController` + `AuthModule` + registro

**What**: Controller (`POST /auth/signup` 201, `POST /auth/signin` 200, `GET /auth/me` com guard), `@ApiBody` com `example`, `@ApiBearerAuth`; `AuthModule` (imports `UsersModule`, `JwtModule.registerAsync` com `jwtConfigFactory`); registrar no `AppModule`; `addBearerAuth()` no Swagger.
**Where**: `soft-go-ii-api/src/auth/auth.controller.ts`
**Also touches**: + `auth.module.ts`, `app.module.ts`, `main.ts`, `auth.controller.spec.ts`
**Depends on**: T8
**Reuses**: `setupApp()` (T1), padrão de `ride-users.controller.ts`
**Requirement**: AUTH-01, AUTH-03, AUTH-06, AUTH-07, AUTH-08, AUTH-14, AUTH-16, AUTH-20, AUTH-21, AUTH-23, AUTH-32

**Tools**:
- MCP: NONE
- Skill: `novo-recurso` (API)

**Done when**:
- [x] Teste HTTP (supertest + `setupApp` + `UsersRepository` fake em memória): signup 201 sem `password`/`passwordHash`/`password_hash` em nenhum nível do corpo; signup duplicado (e-mail em maiúsculas) 409; senha 7 e 73 chars → 400; e-mail inválido → 400; nome 1 char/só espaços → 400; campo extra `confirmPassword` → 400; signin 200; signin errado → 401 com corpo idêntico para e-mail inexistente e senha errada; `/auth/me` com token → 200 `{id,name,email}`, sem token → 401
- [x] Teste: `RideController`, `RideUsersController`, `TransportTypeController` não têm metadata de guards (AUTH-23)
- [x] Gate build (API) passa; test count total da API registrado

**Tests**: integration
**Gate**: build
**Commit**: `feat: add auth endpoints`
**Status**: ✅ Done — `soft-go-ii-api@0d5194d (API: 44 testes)`

---

### T10: Ferramental de testes do front

**What**: Instalar `vitest@^5`, `jsdom`, `@testing-library/react`, `@testing-library/dom`, `@testing-library/user-event`, `@testing-library/jest-dom`; `test` em `vite.config.ts` (`environment: 'jsdom'`, setup com jest-dom); script `"test": "vitest run"`; um teste de fumaça que renderiza `Button`.
**Where**: `soft-go-II/vite.config.ts`
**Also touches**: + `package.json`, `src/test/setup.ts`, `src/components/Button.test.tsx`
**Depends on**: None (fase anterior concluída)
**Reuses**: config Vite existente
**Requirement**: AD-002

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] `npm test` roda e passa (1 teste); `npm run build` e `npm run lint` continuam passando

**Tests**: unit
**Gate**: build
**Commit**: `chore: add vitest and testing library`
**Status**: ✅ Done — `soft-go-II@ac41549`

---

### T11: Tipos de auth + `session.ts`

**What**: `User`, `AuthResponse`, `SignUpPayload`, `SignInPayload` em `types/index.ts`; `session.ts` com `getSession`/`saveSession`/`clearSession`/`onSessionExpired`/`notifySessionExpired` (chave `softgo.session`).
**Where**: `soft-go-II/src/service/session.ts`
**Also touches**: + `session.test.ts`, `types/index.ts`
**Depends on**: T10
**Reuses**: —
**Requirement**: AUTH-24, AUTH-30, AUTH-33

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: save → get devolve o mesmo; clear → `null` e chave removida; JSON corrompido e forma inválida → `null` + chave removida sem lançar; `onSessionExpired` chama listener e o unsubscribe para de chamar
- [x] Gate quick (front) passa; test count: ≥ 5 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add session storage module`
**Status**: ✅ Done — `soft-go-II@d285483`

---

### T12: Interceptors do axios

**What**: Exportar e registrar `attachToken(config)` e `handleUnauthorized(error)` em `service/api.ts`.
**Where**: `soft-go-II/src/service/api.ts`
**Also touches**: + `api.test.ts`
**Depends on**: T11
**Reuses**: instância axios existente
**Requirement**: AUTH-24, AUTH-26, AUTH-31, AUTH-34

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: com sessão → header `Authorization: Bearer <token>`; sem sessão (inclusive após `clearSession`) → sem header; 401 com sessão → sessão limpa + `notifySessionExpired` chamado + promise rejeitada; 401 sem sessão → nada limpo/notificado + rejeitada; 409/500 → só rejeita
- [x] Gate quick (front) passa; test count: ≥ 5 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add auth interceptors to api client`
**Status**: ✅ Done — `soft-go-II@6ad321c`

---

### T13: `auth.service.ts` do front

**What**: `signUp`, `signIn`, `getMe` chamando `/auth/signup`, `/auth/signin`, `/auth/me`.
**Where**: `soft-go-II/src/service/auth.service.ts`
**Also touches**: + `auth.service.test.ts`
**Depends on**: T12
**Reuses**: padrão de `ride.service.ts`
**Requirement**: AUTH-01, AUTH-14, AUTH-20, AUTH-32

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes (api mockado): cada função chama o método/URL certo, envia exatamente o payload (sem `confirmPassword`) e devolve `res.data`
- [x] Gate quick (front) passa; test count: ≥ 3 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add auth api service`
**Status**: ✅ Done — `soft-go-II@354f535`

---

### T14: Schemas zod de cadastro e login

**What**: `signUpSchema` (name trim 2–120, email ≤160, password 8–72, `confirmPassword` + refine "As senhas não coincidem") e `signInSchema`.
**Where**: `soft-go-II/src/schemas/auth.schema.ts`
**Also touches**: + `auth.schema.test.ts`
**Depends on**: T13
**Reuses**: padrão zod de `pages/Form.tsx`
**Requirement**: AUTH-10, AUTH-11

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: válido passa; senha 7 e 73 → erro em `password`; senhas diferentes → "As senhas não coincidem" em `confirmPassword`; e-mail inválido; nome com 1 char/só espaços; signin vazio → erros
- [x] Gate quick (front) passa; test count: ≥ 6 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add auth form schemas`
**Status**: ✅ Done — `soft-go-II@2152f5a`

---

### T15: `AuthProvider` / `useAuth` + `App.tsx`

**What**: Context com `user`, `signIn`, `signUp`, `signOut`; estado inicial de `getSession()`; `getMe()` no mount com sessão; assinatura de `onSessionExpired` (toast "Sua sessão expirou. Entre novamente." + `navigate('/login')`); envolver `Header`/`Routes` em `App.tsx` dentro do `BrowserRouter`.
**Where**: `soft-go-II/src/context/AuthContext.tsx`
**Also touches**: + `AuthContext.test.tsx`, `App.tsx`
**Depends on**: None (fase anterior concluída)
**Reuses**: `session.ts`, `auth.service.ts`, `Toast`
**Requirement**: AUTH-12, AUTH-17, AUTH-25, AUTH-26, AUTH-30

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: com sessão salva, `user` disponível no primeiro render e `getMe` chamado; `signIn`/`signUp` salvam sessão e setam `user`; `signOut` limpa storage, `user = null` e navega para `/login`; `notifySessionExpired` → `user = null`, toast e `/login`
- [x] Gate quick (front) passa; test count: ≥ 5 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add auth context provider`
**Status**: ✅ Done — `soft-go-II@25ca652`

---

### T16: Prop `error` no `InputForm`

**What**: Prop opcional `error?: string` exibida abaixo do campo (`text-sm` em vermelho do tema / `text-support-03`), sem mudar o uso atual.
**Where**: `soft-go-II/src/components/InputForm.tsx`
**Also touches**: + `InputForm.test.tsx`
**Depends on**: T15
**Reuses**: componente existente
**Requirement**: AUTH-10, AUTH-11

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: com `error` → texto renderizado e `aria-invalid="true"`; sem `error` → nada extra
- [x] Gate quick (front) passa; test count: ≥ 2 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: show field error in input form`
**Status**: ✅ Done — `soft-go-II@fe5f088`

---

### T17: Página de cadastro

**What**: `pages/SignUp.tsx` com nome, e-mail, senha, confirmação; `zodResolver(signUpSchema)`; erros por campo; sucesso → toast "Conta criada com sucesso" + `/`; 409 → toast com a mensagem da API; outros → toast genérico; link "Já tem conta? Entrar".
**Where**: `soft-go-II/src/pages/SignUp.tsx`
**Also touches**: + `SignUp.test.tsx`
**Depends on**: T16
**Reuses**: `InputForm`, `Button`, layout de `Form.tsx`, `useAuth`
**Requirement**: AUTH-10, AUTH-11, AUTH-12, AUTH-13, AUTH-32

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: senhas diferentes → "As senhas não coincidem" e serviço não chamado; senha curta → mensagem; sucesso → payload sem `confirmPassword`, toast de sucesso, navegação `/`; 409 → toast "E-mail já cadastrado", continua na página
- [x] Gate quick (front) passa; test count: ≥ 4 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add sign up page`
**Status**: ✅ Done — `soft-go-II@25cc6e8`

---

### T18: Página de login

**What**: `pages/Login.tsx` com e-mail e senha; sucesso → `/`; 401 → toast "E-mail ou senha inválidos"; outros → toast genérico; link "Não tem conta? Criar conta".
**Where**: `soft-go-II/src/pages/Login.tsx`
**Also touches**: + `Login.test.tsx`
**Depends on**: T17
**Reuses**: idem T17
**Requirement**: AUTH-17, AUTH-18, AUTH-34

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: sucesso → navegação `/`; 401 → toast "E-mail ou senha inválidos", sem toast de sessão expirada, continua em `/login`; campos vazios → erros e serviço não chamado
- [x] Gate quick (front) passa; test count: ≥ 3 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add login page`
**Status**: ✅ Done — `soft-go-II@0da0278`

---

### T19: `GuestOnly` + rotas `/login` e `/cadastro`

**What**: Componente que redireciona usuária logada para `/`; registrar as duas rotas em `Routes.tsx`.
**Where**: `soft-go-II/src/components/GuestOnly.tsx`
**Also touches**: + `GuestOnly.test.tsx`, `Routes.tsx`
**Depends on**: T18
**Reuses**: `useAuth`, `Navigate` do react-router
**Requirement**: AUTH-27

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: logada em `/login` e `/cadastro` → renderiza `/`; deslogada → renderiza a página
- [x] Gate quick (front) passa; test count: ≥ 3 novos

**Tests**: unit
**Gate**: quick
**Commit**: `feat: add login and sign up routes`
**Status**: ✅ Done — `soft-go-II@a19ef5b`

---

### T20: Header com estado de auth + logout

**What**: `firstName(name)` em `utils/`; Header deslogado com "Entrar"/"Criar conta", logado com "Olá, {primeiro nome}" e botão "Sair" (`signOut`).
**Where**: `soft-go-II/src/components/Header.tsx`
**Also touches**: + `Header.test.tsx`, `utils/firstName.ts`, `utils/firstName.test.ts`
**Depends on**: T19
**Reuses**: `useAuth`, tokens Tailwind, `lucide-react` (`LogOut`)
**Requirement**: AUTH-28, AUTH-29, AUTH-30, AUTH-31

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [x] Testes: `firstName("  Ana Maria Souza ")` → "Ana"; deslogada → links com `href` `/login` e `/cadastro`, sem "Sair"; logada → "Olá, Ana" e "Sair"; clicar "Sair" → storage sem a chave, header deslogado, rota `/login`
- [x] Gate build (front) passa; test count total do front registrado

**Tests**: unit
**Gate**: build
**Commit**: `feat: add auth actions to header`
**Status**: ✅ Done — `soft-go-II@0ab0ff9 (front: 61 testes)`

---

## Phase Execution Map

```
Phase 1 → Phase 2 → Phase 3 → Phase 4

Phase 1 (API):   T1 → T2 → T3 → T4
Phase 2 (API):   T5 → T6 → T7 → T8 → T9
Phase 3 (Front): T10 → T11 → T12 → T13 → T14
Phase 4 (Front): T15 → T16 → T17 → T18 → T19 → T20
```

Execution is strictly sequential.

---

## Task Granularity Check

| Task | Scope | Status |
| ---- | ----- | ------ |
| T1 | 1 função (setupApp) + chamada | ✅ |
| T2 | 1 entidade + repo + módulo (padrão coeso do projeto, sem lógica) | ⚠️ coeso |
| T3 | 1 migration | ✅ |
| T4 | 1 função + deps | ✅ |
| T5 | 2 DTOs pequenos do mesmo endpoint-group | ⚠️ coeso |
| T6 | 1 método | ✅ |
| T7 | 2 métodos curtos | ⚠️ coeso |
| T8 | 1 guard | ✅ |
| T9 | 1 controller + wiring (merge-backward para ser testável) | ⚠️ coeso |
| T10 | tooling | ✅ |
| T11 | 1 módulo + tipos | ✅ |
| T12 | 2 interceptors do mesmo arquivo | ✅ |
| T13 | 1 serviço | ✅ |
| T14 | 2 schemas do mesmo arquivo | ✅ |
| T15 | 1 context | ✅ |
| T16 | 1 prop | ✅ |
| T17 | 1 página | ✅ |
| T18 | 1 página | ✅ |
| T19 | 1 componente + 2 rotas | ✅ |
| T20 | 1 componente + 1 util | ⚠️ coeso |

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
| T9 | T8 | T8 → T9 | ✅ |
| T10 | None (Phase 2 concluída) | início da Phase 3 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T12 | T11 | T11 → T12 | ✅ |
| T13 | T12 | T12 → T13 | ✅ |
| T14 | T13 | T13 → T14 | ✅ |
| T15 | None (Phase 3 concluída) | início da Phase 4 | ✅ |
| T16 | T15 | T15 → T16 | ✅ |
| T17 | T16 | T16 → T17 | ✅ |
| T18 | T17 | T17 → T18 | ✅ |
| T19 | T18 | T18 → T19 | ✅ |
| T20 | T19 | T19 → T20 | ✅ |

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| ---- | --------------------------- | --------------- | --------- | ------ |
| T1 | API setup | none | none | ✅ |
| T2 | API entity/repo/module (sem lógica além de query builder) | none | none | ✅ |
| T3 | API migration | none | none | ✅ |
| T4 | API config factory | unit | unit | ✅ |
| T5 | API DTO | none (coberto via HTTP em T9) | none | ✅ |
| T6 | API service | unit | unit | ✅ |
| T7 | API service | unit | unit | ✅ |
| T8 | API guard | unit | unit | ✅ |
| T9 | API controller + module | integration | integration | ✅ |
| T10 | Front config | none (+ fumaça) | unit | ✅ |
| T11 | Front lógica | unit | unit | ✅ |
| T12 | Front lógica | unit | unit | ✅ |
| T13 | Front lógica | unit | unit | ✅ |
| T14 | Front lógica | unit | unit | ✅ |
| T15 | Front context | unit | unit | ✅ |
| T16 | Front componente | unit | unit | ✅ |
| T17 | Front página | unit | unit | ✅ |
| T18 | Front página | unit | unit | ✅ |
| T19 | Front componente | unit | unit | ✅ |
| T20 | Front componente + util | unit | unit | ✅ |

## Requirement Coverage

Todos os 34 requisitos mapeados: AUTH-01 (T6,T9,T13) · 02 (T2,T3,T6) · 03 (T2,T9) · 04 (T6) · 05 (T3,T6) · 06–08 (T5,T9) · 09 (T5,T6) · 10–11 (T14,T16,T17) · 12 (T15,T17) · 13 (T17) · 14 (T7,T9,T13) · 15 (T5,T7) · 16 (T7,T9) · 17 (T15,T18) · 18 (T18) · 19 (T4,T6) · 20 (T7,T8,T9,T13) · 21 (T8,T9) · 22 (T4) · 23 (T9) · 24 (T11,T12) · 25 (T15) · 26 (T12,T15) · 27 (T19) · 28–29 (T20) · 30 (T11,T15,T20) · 31 (T12,T20) · 32 (T1,T9,T13,T17) · 33 (T11) · 34 (T12,T18).
