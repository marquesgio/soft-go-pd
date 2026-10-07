# Auth Validation

## Validation: auth - PASS ✅

**Verdict**: PASS

**Date**: 2026-10-07
**Iteration**: 2 of max 3 (re-verification after the iteration-1 fix tasks)
**Spec**: `.specs/features/auth/spec.md` (AUTH-01..AUTH-34); intended behavior from `.specs/features/auth/design.md`
**Diff range**:
- API `soft-go-ii-api` (`master`): `99ebf5b..06b51b4`. Commits `6d61774`, `1e4660c` and `976ec58` are unrelated schema/migration fixes and are excluded from coverage.
- Front `soft-go-II` (`main`): `7399690..f1de23e`
**Verifier**: independent sub-agent (author ≠ verifier). It did not write the code or the fixes, re-derived every row from the spec and current test lines, ran read-only on the real trees, and made mutations only in scratch `git worktree`s.

Why PASS: all 34 ACs have current `file:line` evidence whose assertion targets the spec outcome. Both gates are green (API 46, front 63). All 15 sensor mutants were killed by real test failures, including the three iteration-1 survivors (A1, A11, F10) and the name-field variant of F10. The AUTH-07 spec-precision gap is closed: the test now asserts the exact DTO messages.

---

## Task Completion

| Task | Status | Notes |
| ---- | ------ | ----- |
| T1-T9 (API) | ✅ Done | `36abd6f` … `0d5194d` |
| T10-T20 (Front) | ✅ Done | `ac41549` … `0ab0ff9` |
| Iteration-1 fixes 1-3 + AUTH-07 note | ✅ Done | Test-only commits: API `06b51b4`, front `f1de23e`. Both diffs were reviewed: they touch only spec files, and no production code changed. |

---

## Spec-Anchored Acceptance Criteria

Paths: `api/` = `soft-go-ii-api/src/`, `front/` = `soft-go-II/src/`. Every line below was re-read at the current HEAD. Rows that changed since iteration 1 are marked †.

| AC | Spec-defined outcome | `file:line` + assertion | Result |
| -- | -------------------- | ----------------------- | ------ |
| AUTH-01 | Valid signup → `201` with `{ accessToken, user:{id,name,email} }` | `api/auth/auth.controller.spec.ts:116` `expect(res.status).toBe(201)`; `:117` `typeof accessToken 'string'`; `:118` `res.body.user toEqual({ id: 1, name: 'Ana Maria', email: 'ana@soft.com' })`; `api/auth/auth.service.spec.ts:49` `Object.keys(result).sort() toEqual(['accessToken','user'])` | ✅ |
| AUTH-02 | Only the bcrypt hash is stored; hash ≠ password; `bcrypt.compare === true` | `api/auth/auth.service.spec.ts:64` `saved not.toHaveProperty('password')`; `:65` `passwordHash not.toBe(password)`; `:66` `bcrypt.compare(...) toBe(true)`; design's `select: false` defense: `api/users/entities/user.entity.spec.ts:10` `options.name toBe('password_hash')`, `:11` `options.select toBe(false)` † | ✅ |
| AUTH-03 | No `password`/`passwordHash`/`password_hash` in any response | `api/auth/auth.controller.spec.ts:123` `findPasswordKeys(res.body) toEqual([])` (signup, recursive); `:224` (signin); `:273` `/auth/me` body `toEqual({id,name,email})` (exact); `api/users/entities/user.entity.spec.ts:11` (`select: false`) † | ✅ |
| AUTH-04 | Duplicate email (after trim + lowercase) → `409` `"E-mail já cadastrado"`, no new record | `api/auth/auth.controller.spec.ts:138-139` `status 409`, `message toBe('E-mail já cadastrado')`, sent in uppercase; `api/auth/auth.service.spec.ts:86-89` `ConflictException`, `getStatus() 409`, message, `repository.save not.toHaveBeenCalled()` | ✅ |
| AUTH-05 | UNIQUE violation (23505) → `409` "E-mail já cadastrado", never 500 | `api/auth/auth.service.spec.ts:100-101` `toBeInstanceOf(ConflictException)` + `message toBe('E-mail já cadastrado')`; `:113` other DB errors are rethrown | ✅ |
| AUTH-06 | Password <8 or >72 → `400` "A senha deve ter entre 8 e 72 caracteres" | `api/auth/auth.controller.spec.ts:148-149` (7 and 73 chars) `status 400` + `message toContain('A senha deve ter entre 8 e 72 caracteres')`; boundaries 8/72 → `201` at `:162-163` | ✅ |
| AUTH-07 | Invalid email or >160 chars → `400` with a PT-BR message | `api/auth/auth.controller.spec.ts:176` `status 400`; `:177` `res.body.message toContain(message)` with the exact texts `'Informe um e-mail válido'` / `'O e-mail deve ter no máximo 160 caracteres'` (table at `:166-172`) † | ✅ (spec-precision gap closed) |
| AUTH-08 | Name (trimmed) <2 or >120 → `400` with a PT-BR message | `api/auth/auth.controller.spec.ts:187-188` (1 char, only spaces, 121 chars) `status 400` + `toContain('O nome deve ter entre 2 e 120 caracteres')` | ✅ |
| AUTH-09 | Email is stored trimmed and lowercase | `api/auth/auth.service.spec.ts:76` `existsBy toHaveBeenCalledWith({ email: 'ana@soft.com' })`; `:77` `save.mock.calls[0][0].email toBe('ana@soft.com')`; HTTP `api/auth/auth.controller.spec.ts:130` | ✅ |
| AUTH-10 | Passwords differ → submit blocked, "As senhas não coincidem" under the confirmation field | `front/pages/SignUp.test.tsx:59` `findByText('As senhas não coincidem')`; `:60` `Confirmar senha` `aria-invalid="true"`; `:61` `signUp not.toHaveBeenCalled()`; `front/schemas/auth.schema.test.ts:47` | ✅ |
| AUTH-11 | Field breaks AUTH-06/07/08 rules → submit blocked, PT-BR message under that field | Schema: `front/schemas/auth.schema.test.ts:38` (password), `:61` (email, both messages), `:71` (name). Page: password `front/pages/SignUp.test.tsx:69-72`; email + name † `:83` `findByText(message)`, `:84` `getByLabelText(label) aria-invalid "true"`, `:85` `toHaveAccessibleDescription(message)`, `:86` `signUp not.toHaveBeenCalled()` (table `:75-81`: `'Informe um e-mail válido'`, `'O nome deve ter entre 2 e 120 caracteres'`) | ✅ |
| AUTH-12 | `201` → save the session, toast "Conta criada com sucesso", navigate to `/` | `front/pages/SignUp.test.tsx:95` `página inicial`; `:101` `Toast toHaveBeenCalledWith('success','Conta criada com sucesso')`; `:102` session in localStorage `toEqual({accessToken,user})` | ✅ |
| AUTH-13 | `409` → error toast "E-mail já cadastrado", stay on `/cadastro` | `front/pages/SignUp.test.tsx:117` `Toast toHaveBeenCalledWith('error','E-mail já cadastrado')`; `:119` form still shown; `:120` no session | ✅ |
| AUTH-14 | Correct signin → `200` with `{accessToken,user}` | `api/auth/auth.controller.spec.ts:217` `status 200`; `:219` `user toEqual({...})`; `:224` no password keys; `api/auth/auth.service.spec.ts:150-153` | ✅ |
| AUTH-15 | Email differing only in case or edge spaces still authenticates | `api/auth/auth.controller.spec.ts:233-234` (`'  ANA@Soft.com '`) `status 200`, `user.id 1`; `api/auth/auth.service.spec.ts:164` | ✅ |
| AUTH-16 | Unknown email or wrong password → `401` with an identical body, "E-mail ou senha inválidos" | `api/auth/auth.controller.spec.ts:247-250` both `401`, `message toBe('E-mail ou senha inválidos')`, `wrongPassword.body toEqual(unknownEmail.body)`; `api/auth/auth.service.spec.ts:180-184` | ✅ |
| AUTH-17 | Login `200` → save the session, navigate to `/` | `front/pages/Login.test.tsx:51` `página inicial`; `:52` exact payload `{email,password}`; `:56` session `toEqual({...})` | ✅ |
| AUTH-18 | Login `401` → toast "E-mail ou senha inválidos", stay on `/login` | `front/pages/Login.test.tsx:71` `Toast toHaveBeenCalledWith('error','E-mail ou senha inválidos')`; `:73` `toHaveBeenCalledTimes(1)`; `:74` form still shown | ✅ |
| AUTH-19 | JWT HS256 signed with `JWT_SECRET`, `sub` = id, `email`, expiry `JWT_EXPIRES_IN` (default `7d`) | `api/auth/auth.service.spec.ts:125` `payload.sub 42`; `:126` `payload.email`; `:127` `exp - iat === 7 days` (verified with `algorithms:['HS256']`); `api/auth/jwt.config.spec.ts:18` `secret toBe('segredo')`; `:24` `expiresIn toBe('7d')`; `:32` override `'1h'` | ✅ |
| AUTH-20 | `/auth/me` with a valid Bearer → `200` `{id,name,email}` | `api/auth/auth.controller.spec.ts:272-273` `status 200`, `body toEqual({ id: 1, name: 'Ana Maria', email: 'ana@soft.com' })`; `api/auth/jwt-auth.guard.spec.ts:37-38` `request.user toEqual({id:7,email})` | ✅ |
| AUTH-21 | No token, bad signature, malformed or expired → `401` | HTTP: `api/auth/auth.controller.spec.ts:283` (no token), `:291` (invalid token). Guard: `api/auth/jwt-auth.guard.spec.ts:44` (no header), `:52` (non-Bearer scheme), `:58` (malformed), `:68` (other secret), `:79` (expired); the helper asserts `getStatus() 401` at `:20` | ✅ |
| AUTH-22 | Missing `JWT_SECRET` → startup fails with an explicit error | `api/auth/jwt.config.spec.ts:10` `toThrow('JWT_SECRET não definido')`; wired at boot through `JwtModule.registerAsync({ useFactory: jwtConfigFactory })` in `api/auth/auth.module.ts:13-16` (checked by inspection) | ✅ |
| AUTH-23 | `/ride`, `/ride-users` and `/transport-type` stay open with the same contract | Per controller/method: `api/auth/auth.controller.spec.ts:304` `GUARDS_METADATA` on the controller `toBeUndefined()`, `:306` per method, `:308` methods > 0. No global guard † `:336` `globalGuards toEqual([])` (walks `AppModule` providers and imports recursively); `:335` `toContain(JwtAuthGuard)` proves the walker reaches `AuthModule` | ✅ |
| AUTH-24 | With a saved session, every request sends `Authorization: Bearer <token>` | `front/service/api.test.ts:47` `toBe('Bearer token-abc')`; `:78` adapter of the real `api` instance receives the header | ✅ |
| AUTH-25 | App loads with a saved session → `GET /auth/me`; `200` → shown as logged in | `front/context/AuthContext.test.tsx:71` `logada:Ana Maria`; `:72` `getMe toHaveBeenCalledTimes(1)`; `front/service/auth.service.test.ts:58` `api.get('/auth/me')` | ✅ |
| AUTH-26 | `401` while a session exists → clear the session, toast "Sua sessão expirou. Entre novamente.", navigate to `/login` | Interceptor: `front/service/api.test.ts:91` rejects, `:93` `getSession() toBeNull()`, `:94` `expired toHaveBeenCalledTimes(1)`; `:129` through the `api` instance. Provider: `front/context/AuthContext.test.tsx:118` `Toast('error','Sua sessão expirou. Entre novamente.')`; `:122` `página de login`; `:123` no longer logged in | ✅ |
| AUTH-27 | Logged in → `/login` and `/cadastro` redirect to `/` | `front/components/GuestOnly.test.tsx:38` (`it.each(['/login','/cadastro'])`, real routes) `página inicial`; logged-out controls at `:44`, `:50` | ✅ |
| AUTH-28 | Logged out → "Entrar" (`/login`) and "Criar conta" (`/cadastro`) | `front/components/Header.test.tsx:44` `Entrar href '/login'`; `:45` `Criar conta href '/cadastro'`; `:49` no "Sair" | ✅ |
| AUTH-29 | Logged in → "Olá, {primeiro nome}" and "Sair" | `front/components/Header.test.tsx:57` `getByText('Olá, Ana')` (user "Ana Maria Souza"); `:58` button "Sair"; `front/utils/firstName.test.ts:6` | ✅ |
| AUTH-30 | "Sair" → remove token and user from localStorage, show the logged-out header, navigate to `/login` | `front/components/Header.test.tsx:68` `localStorage.getItem('softgo.session') toBeNull()`; `:69` link "Entrar"; `:70` no "Olá, Ana"; `:71` `página de login`; `front/context/AuthContext.test.tsx:108-109` | ✅ |
| AUTH-31 | After logout or expiry → requests go out without `Authorization` | `front/service/api.test.ts:62` after `saveSession` + `clearSession`, `Authorization toBeUndefined()`; `:53` no session | ✅ |
| AUTH-32 | Undeclared body field → `400` | `api/auth/auth.controller.spec.ts:199-200` signup `confirmPassword` → `400` + `toContain('property confirmPassword should not exist')`; `:260` signin → `400`. The front never sends it: `front/pages/SignUp.test.tsx:96`, `front/service/auth.service.test.ts:30` (exact payload) | ✅ |
| AUTH-33 | Corrupted session JSON → treated as logged out, key removed, no crash | `front/service/session.test.ts:43` `not.toThrow()`; `:44` `toBeNull()`; `:45` `getItem(KEY) toBeNull()`; invalid shapes at `:56-57` | ✅ |
| AUTH-34 | Signin `401` with no session → only "E-mail ou senha inválidos" (no expiry toast, no redirect) | Interceptor: `front/service/api.test.ts:103` rejects, `:105` `expired not.toHaveBeenCalled()`. Page: `front/pages/Login.test.tsx:71` exact toast, `:73` `Toast toHaveBeenCalledTimes(1)`, `:74` stays on `/login` | ✅ |

**Status**: ✅ 34/34 ACs covered with spec-anchored assertions. No spec-precision gaps remain.

Payload/conjunction rule: payloads are asserted by value with an exact object (`SignUp.test.tsx:96`, `Login.test.tsx:52`, `auth.service.test.ts:30/46`). Stored state is asserted with `toEqual` on the parsed localStorage. The interceptor tests assert both state (`getSession()`) and the notification. The new AUTH-11 page tests assert the message, `aria-invalid` and the accessible description link together, plus the blocked submit.

---

## Discrimination Sensor

**Depth**: P0 / expanded (auth critical path). This iteration re-ran the 3 iteration-1 survivors plus the name-field variant of F10, then 11 fresh mutations aimed at the areas next to the fixes.

**Method**: `git -C <submodule> worktree add --detach <scratch>/{api,front} HEAD`. Each worktree got `node_modules` through a PowerShell `New-Item -ItemType Junction`. A Python runner applied one textual mutation at a time and ran `npx vitest run <scope>` in the worktree. It then ran `git checkout -- <file>` and confirmed the worktree's porcelain was clean before the next mutant. The unmutated worktrees passed first: API 46/46, front 63/63.

A front run from the 8.3 short path (`GIOVAN~1.MAR`) failed at collection ("no tests"). Those results were discarded rather than counted as kills. The front mutants were re-run from the long path, and the kills below are real assertion failures.

| # | Mutation | File:line | Tests run | Killed? |
| - | -------- | --------- | --------- | ------- |
| A1 (re-run) | Remove `select: false` from `passwordHash` | `soft-go-ii-api/src/users/entities/user.entity.ts:26` | `src` (46) | ✅ Killed: 1 failed, `user.entity.spec.ts` "não carrega password_hash … (select: false)" |
| A11 (re-run) | `providers: [{ provide: APP_GUARD, useClass: JwtAuthGuard }]` in `AppModule` | `soft-go-ii-api/src/app.module.ts:24` | `src` (46) | ✅ Killed: 1 failed, "Nenhum guard global › AppModule e seus imports não registram APP_GUARD" |
| A12 | `select: false` → `select: true` | `soft-go-ii-api/src/users/entities/user.entity.ts:26` | `src` | ✅ Killed (1 failed) |
| A13 | Register `APP_GUARD` inside `AuthModule` providers instead (adjacent to the A11 fix) | `soft-go-ii-api/src/auth/auth.module.ts:19` | `src` | ✅ Killed (18 failed: public auth routes return 401, plus the no-global-guard test) |
| A14 | `forbidNonWhitelisted: true` → `false` in `setupApp` | `soft-go-ii-api/src/config/app.setup.ts:8` | `src` | ✅ Killed (2 failed, AUTH-32 signup and signin) |
| A15 | Guard swallows verify errors (sets a dummy user instead of throwing 401) | `soft-go-ii-api/src/auth/jwt-auth.guard.ts:35-37` | `src` | ✅ Killed (3 failed: malformed, other secret, expired) |
| A16 | `SignUpDto` email `@MaxLength(160)` → `200` (adjacent to the AUTH-07 tightening) | `soft-go-ii-api/src/auth/dto/sign-up.dto.ts:14` | `src` | ✅ Killed (1 failed, ">160 caracteres") |
| A17 | JWT payload drops `email` (`{ sub }` only) | `soft-go-ii-api/src/auth/auth.service.ts:96` | `src` | ✅ Killed (1 failed, "emite JWT HS256 com sub, email …") |
| F10 (re-run) | SignUp email field loses `error={errors.email?.message}` | `soft-go-II/src/pages/SignUp.tsx:75` | `SignUp.test.tsx` (8) | ✅ Killed: 1 failed, "e-mail inválido: mostra a mensagem abaixo do campo e não envia" |
| F10n (re-run variant) | SignUp name field loses `error={errors.name?.message}` | `soft-go-II/src/pages/SignUp.tsx:67` | `SignUp.test.tsx` (8) | ✅ Killed: 1 failed, "nome com 1 caractere: …" |
| F12 | Name field shows the wrong field's error (`errors.password`) | `soft-go-II/src/pages/SignUp.tsx:67` | `SignUp.test.tsx` | ✅ Killed (2 failed) |
| F13 | zod email `.max(160)` → `.max(200)` | `soft-go-II/src/schemas/auth.schema.ts:12` | `src` (63) | ✅ Killed (1 failed) |
| F14 | Header shows the full name instead of the first name | `soft-go-II/src/components/Header.tsx:21` | `src` | ✅ Killed (1 failed, AUTH-29) |
| F15 | `InputForm` drops `aria-describedby` (the error is no longer tied to the field) | `soft-go-II/src/components/InputForm.tsx:20` | `src` | ✅ Killed (3 failed: `InputForm.test` + both new SignUp tests) |
| F16 | `attachToken` sends `Token <jwt>` instead of `Bearer <jwt>` | `soft-go-II/src/service/api.ts:11` | `src` | ✅ Killed (2 failed, AUTH-24) |

**Sensor depth**: P0-expanded (15 mutations: 8 API, 7 front; 4 re-runs and 11 fresh)
**Result**: 15/15 killed - PASS

**Isolation**: `git status --porcelain` was recorded before the sensor and was identical after cleanup. Junctions were removed with `cmd /c rmdir` (the targets' `node_modules` were confirmed intact). The worktrees were removed with `git worktree remove --force` and then pruned. `git worktree list` shows only the main trees.
- root: ` M soft-go-ii-api`, `?? .specs/features/auth/validation.md`, `?? prds/`
- API: ` M tsconfig.build.tsbuildinfo`
- front: clean

**Residual note (non-blocking)**: a guard registered imperatively with `app.useGlobalGuards(...)` inside `main.ts` would not be caught, because `main.ts` is a bootstrap with top-level `await` and no test imports it. The same call inside `setupApp` *would* be caught, since the HTTP spec boots through `setupApp` and the public auth routes would return 401 (the same failure pattern as A13). It was not injected: `main.ts` holds no feature logic after T1's extraction. It is noted for awareness only.

---

## Gate Check

Both gates were run on the real trees at HEAD (API `06b51b4`, front `f1de23e`) before the sensor:

| Gate | Command | Outcome |
| ---- | ------- | ------- |
| API type-check (build cfg) | `npx tsc --noEmit -p tsconfig.build.json` | exit 0 |
| API type-check (full cfg) | `npx tsc --noEmit -p tsconfig.json` | exit 1, only the known pre-existing `test/app.e2e-spec.ts(4,21) TS2307` (tolerated) |
| API lint | `npm run lint` (oxlint) | exit 0 (4 pre-existing warnings in `transport-type`, `ride-users` and `ride` DTO; none in `auth/` or `users/`) |
| API tests | `npm test` (vitest) | exit 0: 5 files, **46 passed**, 0 failed, 0 skipped |
| Front lint | `npm run lint` (eslint) | exit 0 |
| Front build | `npm run build` (`tsc -b && vite build`) | exit 0 |
| Front tests | `npm test` (vitest) | exit 0: 12 files, **63 passed**, 0 failed, 0 skipped |

- Test count before the feature: API 0 unit tests (only the broken e2e), front 0 (no runner)
- After iteration 1: API 44, front 61
- Now: API 46 (+2: no-global-guard test, `user.entity.spec.ts`), front 63 (+2: email and name page tests). The AUTH-07 test was tightened, not added.
- No tests were deleted or weakened, and there are no `.only`/`.skip`. `npm run test:e2e` is not a gate and was not run. No database was touched.

---

## Code Quality

| Principle | Status |
| --------- | ------ |
| Minimum code / no scope creep | ✅ The fix commits are test-only. Production code is unchanged since iteration 1, and only `/auth/me` is guarded. |
| Surgical changes | ✅ `06b51b4` touches only `auth.controller.spec.ts` and the new `user.entity.spec.ts`. `f1de23e` touches only `SignUp.test.tsx`. |
| Matches patterns | ✅ ESM `.js` imports in API specs, `it.each` tables like the existing ones, Testing Library role/label queries |
| Spec-anchored outcome check | ✅ AUTH-07 now asserts the exact messages |
| Per-layer coverage (domain 1:1; routes happy + edge + error) | ✅ Every new route covers its happy, edge and error paths. AUTH-11 is now covered per field at page level. |
| Every test maps to a requirement | ✅ The new tests map to AUTH-23, AUTH-02/03 (design "Ocultar hash") and AUTH-11 |
| Documented guidelines | ✅ `CLAUDE.md` (root), `soft-go-ii-api/CLAUDE.md` |

---

## Edge Cases

- [x] AUTH-32 undeclared field → 400 (signup and signin; mutant A14 killed)
- [x] AUTH-33 corrupted or invalid-shape storage → null and the key is removed
- [x] AUTH-34 signin 401 with no session → only the credentials toast
- [x] Generic network/5xx on login/signup → generic toast (`front/pages/Login.test.tsx:84`, `front/pages/SignUp.test.tsx:130`)

---

## Requirement Traceability Update

| Requirement | New Status |
| ----------- | ---------- |
| AUTH-01..AUTH-34 | ✅ Verified |

(The Verifier does not edit `spec.md`; the orchestrator applies these statuses.)

---

## Iteration history

| Iteration | Verdict | Key findings | What changed |
| --------- | ------- | ------------ | ------------ |
| 1 | FAIL ❌ | 19/22 mutants killed. Survivors: A1 (`select: false` removed), A11 (global `APP_GUARD`), F10 (SignUp email field error unwired). Spec-precision gap on AUTH-07 (loose `/e-mail/i` regex). | - |
| 2 | PASS ✅ | 15/15 mutants killed, including the A1, A11, F10 and F10-name re-runs. 34/34 ACs spec-anchored. | API `06b51b4`: `user.entity.spec.ts` asserts `select: false` and the `password_hash` column name. A recursive `AppModule` provider/import walk asserts no `APP_GUARD`. AUTH-07 asserts the exact DTO messages. Front `f1de23e`: SignUp page tests for invalid email and 1-char name (message, `aria-invalid`, accessible description, no submit). |

---

## Summary

**Overall**: ✅ Ready

- **Spec-anchored check**: 34/34 ACs match the spec outcome; 0 spec-precision gaps
- **Sensor**: 15/15 mutations killed
- **Gate**: API 46 passed, front 63 passed, 0 failed

**What works**: hashing with the hash hidden at entity level and absent from responses; duplicate and race-condition 409; identical 401 on signin; email normalization; JWT HS256 with 7d expiry; the guard's 401 cases; open legacy routes with no global guard; the `JWT_SECRET` startup check; Bearer attach and detach; 401 handling with and without a session; the corrupted-storage fallback; GuestOnly; header states and logout; per-field validation messages on the signup page.

**Next steps**: the orchestrator marks AUTH-01..34 Verified. No fix tasks.

## validate_state output

`py .claude/skills/tlc-spec-driven/scripts/validate_state.py auth` (from the repo root), exit 0:

```
validate_state: 0 error(s) across [auth]
```
