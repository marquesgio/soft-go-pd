# Revogar Sessões ao Sair (revoke-token) — Validation Report

## Validation: revoke-token - FAIL ❌

**Date**: 2026-10-08
**Spec**: `.specs/features/revoke-token/spec.md` (REVOKE-01..REVOKE-16)
**Diff range**: API `soft-go-ii-api` `64cdad0..cb3fc00` (4 commits: cdc8bf0, 2419799, bc59ca9, cb3fc00) · Front `soft-go-II` `4dfcc68..755bc40` (1 commit)
**Verifier**: independent sub-agent (author ≠ verifier)

**Why FAIL**: every AC has spec-anchored evidence and 15/16 mutants were killed, but the front Build gate (`npm test` in `soft-go-II`) **exits 1**: the feature commit introduced 2 unhandled errors in tests that mock `auth.service` without a usable `signOut`. At the pre-feature commit `4dfcc68` the same suite exits 0 (140 tests, no errors). Per validate.md §4, a non-zero gate exit blocks the feature.

---

## Task Completion

| Task | Status | Notes |
| ---- | ------ | ----- |
| T1 | ✅ Done | `cdc8bf0` migration + entity |
| T2 | ✅ Done | `2419799` `ver` in token + `AuthService.signOut` |
| T3 | ✅ Done | `bc59ca9` `verifySession` + both guards |
| T4 | ✅ Done | `cb3fc00` `POST /auth/signout` |
| T5 | ⚠️ Done with gate defect | `755bc40`: code correct, but "Gate build (front) passa" is not true — `npm test` exits 1 (see Gate Check) |

---

## Spec-Anchored Acceptance Criteria

API paths are relative to `soft-go-ii-api/`, front paths to `soft-go-II/`.

| AC | Spec-defined outcome | `file:line` + assertion | Result |
| -- | -------------------- | ----------------------- | ------ |
| REVOKE-01 | signout with valid token → `token_version` +1 (only that account), `204`, empty body | `src/auth/auth.controller.spec.ts:357` `expect(res.status).toBe(204)`; `:358` `expect(res.text).toBe('')`; `:359` `expect(users().users[0].tokenVersion).toBe(1)`; `src/auth/auth.service.spec.ts:273-274` `increment` called once with `({ id: 7 }, 'tokenVersion', 1)` | ✅ PASS |
| REVOKE-02 | no token / invalid / revoked → `401` "Não autenticado", no version changes | `src/auth/auth.controller.spec.ts:423-425` (it.each sem token, token inválido) `status 401`, `message 'Não autenticado'`, `tokenVersion` stays `0`; revoked: `:434` `401`, `:435` `tokenVersion` stays `1` (message for revoked token anchored by `src/auth/jwt-auth.guard.spec.ts:31,96`) | ✅ PASS |
| REVOKE-03 | "Sair" → POST /auth/signout with session token, local session cleared, `/login` | `src/service/auth.service.test.ts:87` `api.post` called with `("/auth/signout", null, { headers: { Authorization: "Bearer token-abc" } })`; `src/context/AuthContext.test.tsx:154` `signOut` called with `"token-abc"`; `:156` `sessionAtCall` `toBeNull()` (cleared before the call); `:157` "página de login"; `:140` storage cleared | ✅ PASS |
| REVOKE-04 | API fails (network, 401, 500) → still clears session + `/login`, no "Sua sessão expirou" toast | `src/context/AuthContext.test.tsx:160-174` it.each 401/500/rede: `:172` `localStorage.getItem(KEY)` `toBeNull()`, `:173` "página de login", `:174` `expect(Toast).not.toHaveBeenCalled()`; the interceptor side is anchored by `:156` (no session at call time) + existing `src/service/api.test.ts:98` ("401 sem sessão: não avisa expiração") | ✅ PASS |
| REVOKE-05 | token from signup/signin carries `ver` = current `token_version` | `src/auth/auth.service.spec.ts:156` signup `ver` `toBe(0)` (+ `:155` saved with `tokenVersion: 0`); `:198` signin with stored version 3 → `payload.ver` `toBe(3)` | ✅ PASS |
| REVOKE-06 | `JwtAuthGuard`, `ver` ≠ current → `401` "Não autenticado" | `src/auth/jwt-auth.guard.spec.ts:92-97` (account 8 at v1, token v0) `expectUnauthorized` (`:30` status 401, `:31` message `'Não autenticado'`) + `request.user` undefined; HTTP: `src/auth/auth.controller.spec.ts:368-369` `401` + `'Não autenticado'`; `:380` other device's token `401` | ✅ PASS |
| REVOKE-07 | `JwtAuthGuard`, token without `ver` → `401` "Não autenticado" | `src/auth/jwt-auth.guard.spec.ts:108-112` `expectUnauthorized` | ✅ PASS |
| REVOKE-08 | `JwtAuthGuard`, valid signature, account missing → `401` "Não autenticado" | `src/auth/jwt-auth.guard.spec.ts:115-119` `expectUnauthorized`; HTTP `src/ride/ride.controller.spec.ts` "token de conta inexistente: 401 'Não autenticado' sem gravar carona" (`sub: 99, ver: 0`) | ✅ PASS |
| REVOKE-09 | `OptionalJwtAuthGuard` revoked / no `ver` / missing account → anonymous; `GET /ride` without `phone` in `owner` and `participants` | unit `src/auth/optional-jwt-auth.guard.spec.ts:52-69,90-91` (versão antiga, sem ver, conta inexistente → `resolves.toBe(true)` + `request.user` undefined); HTTP `src/ride/ride.controller.spec.ts:131-133` `status 200`, `participants` `toEqual([{ userId: 1, name: 'Ana' }])`, `owner` `toEqual({ id: 2, name: 'Carla Dona' })` (JSON body, so a `phone` key with any value would fail `toEqual`) | ✅ PASS |
| REVOKE-10 | `ver` = current → accepted (`JwtAuthGuard` passes; Optional fills user) | `src/auth/jwt-auth.guard.spec.ts:47-48` and `:104-105` (account already at v1, token v1) `resolves.toBe(true)`, `request.user` `toEqual({ id: 8, email: 'bia@soft.com' })`; `src/auth/optional-jwt-auth.guard.spec.ts:40-41` | ✅ PASS |
| REVOKE-11 | sign in again after signout → new token accepted | `src/auth/auth.controller.spec.ts:392-393` `GET /auth/me` with the new token `200`, email `'ana@soft.com'` | ✅ PASS |
| REVOKE-12 | account A signs out → B's tokens still accepted | `src/auth/auth.controller.spec.ts:402` B `me` `200`; `:403` B `tokenVersion` `toBe(0)` | ✅ PASS |
| REVOKE-13 | signout changes only `token_version` (name, email, phone, hash, rides, joins unchanged) | `src/auth/auth.controller.spec.ts:412` `users().users[0]` `toEqual({ ...before, tokenVersion: before.tokenVersion + 1 })` (persisted row incl. `passwordHash`, `phone`); `src/auth/auth.service.spec.ts:282-283` no `update`/`save`. Rides/joins: `AuthService` has no ride/ride-user repository dependency (structural) | ✅ PASS |
| REVOKE-14 | `up()` adds `token_version integer NOT NULL DEFAULT 0`; existing rows = 0 | SQL `src/migrations/1791483753603-AddTokenVersionInUsersTable.ts:7` `ALTER TABLE "users" ADD "token_version" integer NOT NULL DEFAULT 0`; entity metadata `src/users/entities/user.entity.spec.ts:28-30` name `'token_version'`, nullable `false`, default `0`. Existing-row behaviour: Postgres `ADD ... NOT NULL DEFAULT 0` backfills 0; author's temp-DB run (not re-run by verifier, per instructions) | ✅ PASS (read + metadata) |
| REVOKE-15 | `down()` drops only the column, no accounts deleted | `src/migrations/1791483753603-AddTokenVersionInUsersTable.ts:11` `ALTER TABLE "users" DROP COLUMN "token_version"` (only statement) | ✅ PASS (read) |
| REVOKE-16 | me / signup / signin never expose `token_version` | `src/auth/auth.controller.spec.ts:127` signup, `:254` signin, `:309` me — exact `toEqual({ id, name, email, phone })`; `src/auth/auth.service.spec.ts:157,199` `not.toHaveProperty('tokenVersion')` | ✅ PASS |

**Status**: ✅ 16/16 ACs covered with spec-anchored assertions; 0 spec-precision gaps.

Minor observations (not gaps): REVOKE-02 "nenhuma conta" is asserted with a single seeded account in the no-token/invalid cases; REVOKE-13 rides/joins invariance is structural, not asserted.

---

## Discrimination Sensor

Run in temporary `git worktree`s (`scratchpad/wt-api` at `cb3fc00`, `scratchpad/wt-front` at `755bc40`) with junctioned `node_modules`, from the long path. Each mutant applied, targeted tests run, file restored and byte-checked.

| # | File:line | Mutation | Killed? (by) |
| - | --------- | -------- | ------------ |
| A1 | `soft-go-ii-api/src/auth/verify-session.ts:28` | drop version comparison (`if (!user) return null`) | ✅ 5 fails (jwt guard v-old, optional guard v-old, controller signout ×3) |
| A2 | `soft-go-ii-api/src/auth/verify-session.ts:25-28` | accept tokens without `ver` (`payload.ver ?? user.tokenVersion`) | ✅ 2 fails (both guards "sem ver") |
| A3 | `soft-go-ii-api/src/auth/optional-jwt-auth.guard.ts:22-27` | Optional guard back to signature-only (ignores version/account) | ✅ 5 fails (optional guard ×3, ride.controller old-version ×2) |
| A4 | `soft-go-ii-api/src/auth/auth.service.ts:100` | increment by 2 | ✅ 4 fails |
| A5 | `soft-go-ii-api/src/auth/auth.service.ts:100` | increment wrong account (`userId + 1`) | ✅ 7 fails |
| A6 | `soft-go-ii-api/src/auth/auth.controller.ts:74` | signout route without `@UseGuards(JwtAuthGuard)` | ✅ 8 fails (+ guard map) |
| A7 | `soft-go-ii-api/src/auth/auth.service.ts:107` | `issueToken` omits `ver` | ✅ 9 fails |
| A8 | `soft-go-ii-api/src/auth/auth.controller.ts:73` | `204` → `200` | ✅ 1 fail (`:357`) |
| A9 | `soft-go-ii-api/src/auth/auth.service.ts:116` | `toPublicUser` leaks `tokenVersion` | ✅ 6 fails |
| A10 | `soft-go-ii-api/src/auth/auth.controller.ts:77` | signout handler skips `authService.signOut` | ✅ 5 fails |
| F1 | `soft-go-II/src/context/AuthContext.tsx:82-85` | call API before clearing the session | ✅ 2 fails (`:156` sessionAtCall) |
| F2 | `soft-go-II/src/context/AuthContext.tsx:82-85` | `await` API first (uncaught), then clear | ✅ 4 fails (failure it.each + call test) |
| F3 | `soft-go-II/src/context/AuthContext.tsx:85` | never call the API | ✅ 4 fails |
| F4 | `soft-go-II/src/service/auth.service.ts:26` | header without `Bearer ` | ✅ 1 fail (`auth.service.test.ts:87`) |
| F5 | `soft-go-II/src/context/AuthContext.tsx:79` | send wrong value as token | ✅ 1 fail (`:154`) |
| F6 | `soft-go-II/src/context/AuthContext.tsx:85` | drop only `.catch(() => {})` (rejection unhandled) | ❌ Survived (12/12 pass, exit 0) — spec-equivalent: session is already cleared and `/login` reached, only an unhandled rejection remains |

**Sensor depth**: P0 (auth) — 16 manual behavior-level mutations covering all new branches.
**Sensor outcome**: 15/16 killed; F6 survived (low severity, spec-level equivalent).

---

## Code Quality

| Principle | Status |
| --------- | ------ |
| Minimum code | ✅ (`verifySession` shared by both guards, 31 lines) |
| Surgical changes | ✅ (spec setup changes in ride/ride-users specs only add `findOneBy` + `ver: 0`) |
| No scope creep | ✅ |
| Matches patterns | ✅ (ESM `.js` imports, custom repository, Swagger `ApiBearerAuth`, PT-BR messages) |
| Spec-anchored outcome check | ✅ |
| Per-layer Coverage Expectation | ✅ (guards: valid/old/no-ver/missing/invalid/expired; HTTP: happy + 401s + persisted state + flow) |
| Every test maps to a requirement | ✅ |
| Documented guidelines followed (`CLAUDE.md`, `soft-go-ii-api/CLAUDE.md`, AD-002/003/005) | ❌ front test gate not clean (see Gate Check) |

---

## Edge Cases

- [x] Double "Sair" (two tabs): second call `401`, no extra increment (`src/auth/auth.controller.spec.ts:434-435`); front exits regardless (`src/context/AuthContext.test.tsx:160-174`).
- [x] Revoked token hits a logged route from the front → existing interceptor handles as expired (no new code; `src/service/api.test.ts:85`).
- [x] Login × signout race: accepted per Assumptions (not tested, by design).

---

## Gate Check

| Gate | Command | Outcome |
| ---- | ------- | ------- |
| Build (API) | `npx tsc --noEmit -p tsconfig.build.json` | exit 0 |
| Build (API) | `npm run lint` | exit 0 (5 pre-existing warnings, none in feature files) |
| Build (API) | `npm test` | **153 passed**, 0 failed, 12 files, exit 0 |
| Build (Front) | `npm run lint` | exit 0 (1 pre-existing warning `Home.tsx:45`) |
| Build (Front) | `npm run build` | exit 0 |
| Build (Front) | `npm test` | 145 passed, 0 failed, **2 unhandled errors, exit 1** ❌ |

- **Test count before feature**: API 130 · front 140 (front at `4dfcc68`: 140 passed, exit 0, re-run by verifier)
- **Test count after feature**: API 153 (+23) · front 145 (+5)
- **Skipped tests**: none
- **Failures (front, unhandled errors introduced by `755bc40`)**:
  1. `TypeError: Cannot read properties of undefined (reading 'catch')` at `soft-go-II/src/context/AuthContext.tsx:85`, during `src/context/AuthContext.test.tsx:134` ("signOut limpa o localStorage…"): the `vi.mock` factory's `signOut: vi.fn()` returns `undefined`, and that test never sets a resolved value.
  2. `[vitest] No "signOut" export is defined on the "../service/auth.service" mock` at `soft-go-II/src/context/AuthContext.tsx:85`, during `src/components/Header.test.tsx:62` ("clicar em 'Sair'…"): the mock at `src/components/Header.test.tsx:9-13` lacks `signOut`.

---

## Fix Plans

### Fix 1: front test suite must exit 0 (Blocker for the gate)

- **Root cause**: two pre-existing test files mock `../service/auth.service` without a promise-returning `signOut`; the new `authService.signOut(token).catch(...)` in `AuthContext.tsx:85` throws inside the click handler.
- **Fix task**:
  - `soft-go-II/src/components/Header.test.tsx:9-13`: add `signOut: vi.fn().mockResolvedValue(undefined)` to the mock factory.
  - `soft-go-II/src/context/AuthContext.test.tsx:63-65` (`beforeEach`): add `vi.mocked(authService.signOut).mockResolvedValue(undefined);` (`vi.clearAllMocks` in `afterEach` keeps implementations, but the factory default is `undefined`).
- **Verify**: `cd soft-go-II && npm test` → 145 passed, no "Unhandled Errors", exit 0.
- **Priority**: Blocker (gate)

### Fix 2: make the "API fails" test detect an uncaught rejection (Minor, surviving mutant F6)

- **Root cause**: `src/context/AuthContext.test.tsx:160-174` only checks storage/navigation/toast; removing `.catch(() => {})` leaves the test green.
- **Fix task**: in that it.each, capture the promise the mock returns (e.g. `const p = Promise.reject(error); vi.mocked(authService.signOut).mockReturnValue(p)`) and register a `process.on("unhandledRejection", spy)` / `window.addEventListener("unhandledrejection", spy)` listener, then `await waitFor(...)` and `expect(spy).not.toHaveBeenCalled()`.
- **Verify**: re-run mutant F6 (remove `.catch`) → the test fails.
- **Priority**: Minor

---

## Requirement Traceability Update

| Requirement | Previous Status | New Status |
| ----------- | --------------- | ---------- |
| REVOKE-01, 02, 05–16 | Done | ✅ Verified (API gate green) |
| REVOKE-03, REVOKE-04 | Done (T5) | ❌ Needs Fix — behaviour verified, but front gate exits 1 |

---

## Summary

**Overall**: ❌ Not Ready

**Spec-anchored check**: 16/16 ACs matched spec outcome, 0 spec-precision gaps
**Sensor**: 15/16 mutations killed (F6 survived, spec-equivalent)
**Gate**: API 153 passed / exit 0; front 145 passed but exit 1 (2 unhandled errors)

**What works**: token versioning end to end (migration, entity, `ver` in tokens, both guards via `verifySession`, `POST /auth/signout` 204 + atomic increment, re-login, other accounts untouched, no `tokenVersion` leak); front clears the session before calling the API and never blocks sign-out.

**Issues found**: front `npm test` exit 1 from incomplete `auth.service` mocks (Fix 1); weak assertion on the swallowed rejection (Fix 2).

**Next steps**: apply Fix 1 (and Fix 2), re-run `cd soft-go-II && npm test`, then re-dispatch the Verifier (iteration 2 of 3).
