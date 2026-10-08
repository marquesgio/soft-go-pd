# Dona da Carona Validation

**Date**: 2026-10-08
**Verdict**: PASS
**Iteration**: 1
**Spec**: `.specs/features/save-rideowner/spec.md` (OWNER-01..OWNER-30)
**Diff range**: API `soft-go-ii-api` `be7c253..9589b43` (5 commits: `df81e71`, `83a6d48`, `4db8137`, `4d5ba30`, `9589b43`); front `soft-go-II` `0ffc132..2c873cb` (5 commits: `12e3fdd`, `f694ec7`, `fd04b4c`, `5dbe9d1`, `2c873cb`)
**Verifier**: independent sub-agent (author ≠ verifier). I re-derived coverage from the spec and read every cited line at HEAD.

All 30 ACs have evidence. 26 are covered by automated assertions that target the outcome the spec defines. The 4 migration ACs (OWNER-21..24) were checked by reading the SQL, plus the entity metadata test, plus the author's manual run against a temporary database. All 15 mutants were killed by real assertion failures. One minor spec-precision gap is flagged (OWNER-30): the test is exact, but the spec only says "mensagem em PT-BR". This does not block the verdict.

---

## Task Completion

| Task | Status | Notes |
| ---- | ------ | ----- |
| T1–T5 (API) | ✅ Done | `df81e71`..`9589b43` |
| T6–T10 (front) | ✅ Done | `12e3fdd`..`2c873cb` |

`tasks.md` still says `**Status**: In Progress` at the top (line 13), even though every task is ✅. That is bookkeeping for the orchestrator, not a gap.

---

## Gate Check

| Gate | Command | Result |
| ---- | ------- | ------ |
| API type-check (build) | `npx tsc --noEmit -p tsconfig.build.json` | exit 0 |
| API type-check (all) | `npx tsc --noEmit -p tsconfig.json` | Only error: the tolerated pre-existing `test/app.e2e-spec.ts(4,21) TS2307` |
| API lint | `npm run lint` (oxlint) | 0 errors. Only pre-existing warnings (`no-useless-escape` in phone regexes, unused imports in `transport-type.controller.ts`) |
| API tests | `npm test` | exit 0: **12 files, 121 passed**, 0 failed, 0 skipped |
| Front lint | `npm run lint` | 0 errors, 1 pre-existing warning (`Home.tsx:45`, from commit `226bf5a`, outside this range) |
| Front build | `npm run build` (`tsc -b && vite build`) | exit 0 |
| Front tests | `npm test` | exit 0: **19 files, 136 passed**, 0 failed, 0 skipped |

- **Test count before feature** (vou-junto final validation): API 96, front 109
- **Test count after feature**: API 121 (+25), front 136 (+27). Neither count went down. The changes to existing tests only rename `name` to `owner` in fixtures (`Modal.test.tsx:8`, `Home.test.tsx:32`), plus the guard-map entry `createRide: [JwtAuthGuard]` (`auth.controller.spec.ts`). Nothing was weakened.

---

## Spec-Anchored Acceptance Criteria

API paths are relative to `soft-go-ii-api/`. Front paths are relative to `soft-go-II/`.

### P1: Publicar carona logada

| AC | Spec-defined outcome | `file:line` + assertion | Result |
| -- | -------------------- | ----------------------- | ------ |
| OWNER-01 | User with a phone: no "Seu Nome" field, no "WhatsApp" field | `src/pages/Form.test.tsx:75-76` - `expect(screen.queryByLabelText("Seu Nome *")).not.toBeInTheDocument()`; `expect(screen.queryByLabelText(/WhatsApp/)).not.toBeInTheDocument()` | ✅ PASS |
| OWNER-02 | `POST /ride` sends `Authorization: Bearer <token>`; body has no `name` and no `ownerId` | Body: `src/service/ride.service.test.ts:77-79` - `expect(body).not.toHaveProperty("name")`, `.not.toHaveProperty("ownerId")`, `expect(body).toEqual(ride)`; `src/pages/Form.test.tsx:104` - `expect(payload).not.toHaveProperty("name")`. Header: `src/service/api.test.ts:78` - `expect(adapter.mock.calls[0][0].headers.Authorization).toBe("Bearer token-abc")`. That test checks the interceptor registered on the shared `api` instance, which `createRide` uses (`src/service/ride.service.ts:19`) | ✅ PASS |
| OWNER-03 | `201`, with `owner_id` = the token's id | `src/ride/ride.controller.spec.ts:140-142` - `expect(res.status).toBe(201)`; `expect(savedRides).toHaveLength(1)`; `expect(savedRides[0]).toMatchObject({ ownerId: 1, ... })` (token `sub: 1`, line 84); `src/ride/ride.service.spec.ts:195-200` | ✅ PASS |
| OWNER-04 | `name` or `ownerId` in the body returns `400`, and no ride is saved | `src/ride/ride.controller.spec.ts:145-153` (`it.each` over `name`, `ownerId`) - `expect(res.status).toBe(400)`; `toContain(\`property ${field} should not exist\`)`; `expect(savedRides).toEqual([])` | ✅ PASS |
| OWNER-05 | `201` shows the success toast and navigates to `/` | `src/pages/Form.test.tsx:96,105` - `await screen.findByText("página inicial")`; `expect(Toast).toHaveBeenCalledWith("success")` | ✅ PASS |
| OWNER-25 | Account without a phone: optional "WhatsApp" field shown, no "Seu Nome" | `src/pages/Form.test.tsx:83-84` - `expect(screen.getByLabelText("WhatsApp")).toBeInTheDocument()`; `expect(screen.queryByLabelText("Seu Nome *")).not.toBeInTheDocument()` | ✅ PASS |
| OWNER-26 | Front sends `phone`; API saves it to `users.phone` of the token's account; front updates the session | Front: `src/pages/Form.test.tsx:117-120` - `toMatchObject({ phone: "51988887777" })`; `expect(getSession()?.user.phone).toBe("51988887777")`. API: `src/ride/ride.controller.spec.ts:204-206` - `expect(users.update).toHaveBeenCalledWith(1, { phone: '51999998888' })`; `expect(savedRides[0]).not.toHaveProperty('phone')`. "The next `/form` does not show the field" follows from the updated session plus OWNER-01 (`Form.test.tsx:76`) | ✅ PASS |
| OWNER-27 | Empty WhatsApp: body has no `phone` key; session keeps `null` | `src/pages/Form.test.tsx:131-132` - `.not.toHaveProperty("phone")`; `expect(getSession()?.user.phone).toBeNull()`; `src/service/ride.service.test.ts:70` | ✅ PASS |
| OWNER-28 | `phone: ""` returns `201`, the ride is created, and `users.phone` is unchanged | `src/ride/ride.controller.spec.ts:212-214` - `expect(res.status).toBe(201)`; `expect(savedRides).toHaveLength(1)`; `expect(users.update).not.toHaveBeenCalled()` | ✅ PASS |
| OWNER-29 | Not exactly 11 digits: submit blocked, with "Número inválido, informe 11 dígitos (DDD + número)" | `src/pages/Form.test.tsx:143-144` - `expect(await screen.findByText(PHONE_ERROR))` (`PHONE_ERROR` is the exact string, line 27); `expect(rideService.createRide).not.toHaveBeenCalled()` | ✅ PASS |
| OWNER-30 | Phone outside the BR/DDD rule: `400` with a "mensagem em PT-BR"; no ride saved; `users.phone` unchanged | `src/ride/ride.controller.spec.ts:220-223` - `expect(res.status).toBe(400)`; `toContain('Numero de telefone inválido!')`; `expect(savedRides).toEqual([])`; `expect(users.update).not.toHaveBeenCalled()` | ✅ PASS · ⚠️ Spec-precision gap: the spec does not name the exact message. The test asserts the exact `CreateRideDto` message, so coverage is fine |

### P1: Só quem está logada publica

| AC | Spec-defined outcome | `file:line` + assertion | Result |
| -- | -------------------- | ----------------------- | ------ |
| OWNER-06 | No `Authorization` header: `401` "Não autenticado", no ride saved | `src/ride/ride.controller.spec.ts:159-161` - `toBe(401)`; `expect(res.body.message).toBe('Não autenticado')`; `expect(savedRides).toEqual([])` | ✅ PASS |
| OWNER-07 | Invalid or expired token: `401` "Não autenticado", no ride saved | `src/ride/ride.controller.spec.ts:172-174` (signed with a different secret) and `:186-188` (expired), with the same three assertions | ✅ PASS |
| OWNER-08 | Valid token for an account that no longer exists: `401` "Não autenticado", no ride saved | `src/ride/ride.controller.spec.ts:196-198` (`sub: 99`) - `toBe(401)`; `toBe('Não autenticado')`; `toEqual([])`; `src/ride/ride.service.spec.ts:210-213` (`UnauthorizedException`, no `save`, no `update`) | ✅ PASS |
| OWNER-09 | Logged out, clicking "Vou para a soft" goes to `/login` | `src/pages/Home.test.tsx:238` - `expect(screen.getByText("página de login")).toBeInTheDocument()`. The logged-in contrast case is at `:247` | ✅ PASS |
| OWNER-10 | Logged out, opening `/form` redirects to `/login` without rendering the form | `src/components/RequireAuth.test.tsx:33-34` - `getByRole("heading", { name: "Entrar" })`; `expect(screen.queryByText("formulário de carona")).not.toBeInTheDocument()`. Uses the real `AppRoutes` | ✅ PASS |

### P1: Mural mostra a dona

| AC | Spec-defined outcome | `file:line` + assertion | Result |
| -- | -------------------- | ----------------------- | ------ |
| OWNER-11 | `owner.id` = `rides.owner_id`; `owner.name` = full `users.name` | `src/ride/ride.service.spec.ts:122` - `expect(ride.owner).toEqual({ id: 5, name: 'Carla Dona Silva' })`; `src/ride/ride.controller.spec.ts:97` (HTTP, `ownerId: 2` gives `owner.id: 2`); relation loaded: `ride.service.spec.ts:49-53` | ✅ PASS |
| OWNER-12 | Valid token: `owner.phone` = `users.phone`, or `null` | `src/ride/ride.controller.spec.ts:109` - `toEqual({ id: 2, name: 'Carla Dona', phone: '51988887777' })`; `src/ride/ride.service.spec.ts:147` - `toEqual({ ..., phone: null })` | ✅ PASS |
| OWNER-13 | No token, or an invalid token: no `phone` key on `owner` | `src/ride/ride.controller.spec.ts:97,119` - `toEqual({ id: 2, name: 'Carla Dona' })` (exact equality excludes the key), for both `/ride` and `/ride/1`; `src/ride/ride.service.spec.ts:130` - `not.toHaveProperty('phone')` | ✅ PASS |
| OWNER-14 | No `name` or `phone` at the ride level | `src/ride/ride.service.spec.ts:155-157` - `expect(ride).not.toHaveProperty('name')`, `'phone'`, `'ownerId'`; `src/ride/entities/ride.entity.spec.ts:35-36` - the columns no longer exist | ✅ PASS |
| OWNER-15 | The card shows `owner.name`, and the avatar initial comes from `owner.name` | `src/components/Card.test.tsx:90-91` - `getByText("Carla Dona Silva")`; `getByText("CD")` | ✅ PASS |
| OWNER-16 | Logged in and `owner.phone` set: "WhatsApp" button linking to `https://wa.me/<owner.phone>` | `src/components/Card.test.tsx:97` - `expect(whatsapp()).toHaveAttribute("href", "https://wa.me/51988887777")`. Home passes `showPhones={!!user}` (`Home.tsx:124`), and that wiring is checked at `Home.test.tsx:206` | ✅ PASS |
| OWNER-17 | Logged out: no WhatsApp button, even if a phone is present | `src/components/Card.test.tsx:103` - `expect(whatsapp()).not.toBeInTheDocument()` | ✅ PASS |
| OWNER-18 | `owner.phone` is `null`: no WhatsApp button | `src/components/Card.test.tsx:109` - `expect(whatsapp()).not.toBeInTheDocument()` | ✅ PASS |

### P1: Migration da dona (checked by inspection; no automated test, as agreed for this feature)

| AC | Spec-defined outcome | Evidence | Result |
| -- | -------------------- | -------- | ------ |
| OWNER-21 | `up()` empties `rides` and their `ride_users` | `src/migrations/1791478056118-AddOwnerToRides.ts:8` - `DELETE FROM "rides"`, together with `src/migrations/1790080215751-CreateTable.ts:37` - `FOREIGN KEY ("ride_id") REFERENCES "rides"("id") ON DELETE CASCADE`. The author also did a manual run on a temporary DB (seeded an old row, ran the migration, both tables were empty) | ✅ PASS (inspection) |
| OWNER-22 | `owner_id integer NOT NULL` with FK `FK_rides_owner` to `users(id)` | `AddOwnerToRides.ts:11-12` - `ADD "owner_id" integer NOT NULL`; `ADD CONSTRAINT "FK_rides_owner" FOREIGN KEY ("owner_id") REFERENCES "users"("id")`. Entity: `src/ride/entities/ride.entity.spec.ts:12-13,24-27` (`nullable` is `false`, `foreignKeyConstraintName` is `'FK_rides_owner'`). `migration:generate` dry run showed no diff (author) | ✅ PASS (inspection + entity test) |
| OWNER-23 | No `name` or `phone` columns | `AddOwnerToRides.ts:9-10` - `DROP COLUMN "name"`, `DROP COLUMN "phone"`; `ride.entity.spec.ts:35-36` | ✅ PASS (inspection + entity test) |
| OWNER-24 | `down()`: `name varchar(100) NOT NULL` and `phone varchar(15)` come back; no `owner_id`; no FK | `AddOwnerToRides.ts:17-21` - `DELETE`, then `DROP CONSTRAINT "FK_rides_owner"`, `DROP COLUMN "owner_id"`, `ADD "phone" character varying(15)`, `ADD "name" character varying(100) NOT NULL`. The `DELETE` comes first, so adding `NOT NULL` succeeds | ✅ PASS (inspection) |

### P2: Dona não se inscreve na própria carona

| AC | Spec-defined outcome | `file:line` + assertion | Result |
| -- | -------------------- | ----------------------- | ------ |
| OWNER-19 | The owner's own card shows "Sua carona" instead of "Vou junto" (even when the ride is full) | `src/components/Card.test.tsx:117-118` - `getByText("Sua carona")`; `queryByRole("button", { name: "Vou junto" })).not.toBeInTheDocument()`; full ride: `:124`; another user: `:130-131`; integrated in Home: `src/pages/Home.test.tsx:257-258` | ✅ PASS |
| OWNER-20 | `409` "Você não pode se inscrever na sua própria carona", no signup saved | `src/ride-users/ride-users.controller.spec.ts:154-156` (token `sub: 7`, ride 4 `ownerId: 7`) - `toBe(409)`; `expect(res.body.message).toBe('Você não pode se inscrever na sua própria carona')`; `expect(rideUsers.rows).toEqual([])`; `src/ride-users/ride-users.service.spec.ts:57-60` (no `save`, no phone `update`) | ✅ PASS |

**Status**: ✅ All 30 ACs covered. ⚠️ 1 minor spec-precision gap flagged (OWNER-30).

Payload/conjunction rule: the request bodies are checked on their values, not just on whether a call happened (`ride.service.test.ts:79` `toEqual(ride)`, `Form.test.tsx:98-103`). The persisted values are checked too (`savedRides[0]` `toMatchObject({ ownerId: 1 })`, `users.update` `toHaveBeenCalledWith(1, { phone })`).

Persisted-state rule: every rejection AC (OWNER-04/06/07/08/20/30) asserts that nothing was saved (`savedRides` / `rideUsers.rows` `toEqual([])`), and, where relevant, that `users.update` was not called.

---

## Edge Cases

- [x] Owner without a phone who leaves WhatsApp empty: the ride is published and no WhatsApp button is shown. Covered by `Form.test.tsx:131-132` and `Card.test.tsx:109`
- [x] The owner's own ride is full: the card still shows "Sua carona". Covered by `Card.test.tsx:121-125`
- [x] `phone: ""` (old front format): the API returns `201`. Covered by `ride.controller.spec.ts:209-215`

---

## Discrimination Sensor

**Sensor depth**: P0-style (auth and data integrity): 15 manual behavior-level mutations. Each ran in a temporary `git worktree` in the session scratchpad, with `node_modules` linked through a junction. The junctions were removed before `git worktree remove --force` and `git worktree prune`.

| # | File:line | Mutation | Killed? | Killing test(s) |
| - | --------- | -------- | ------- | --------------- |
| A1 | API `src/ride/ride.service.ts:93` | Dropped the owner-existence check (`if (false)`) | ✅ Killed | `ride.controller.spec.ts` "token de conta inexistente: 401"; `ride.service.spec.ts` "conta inexistente: 401", "salva a carona com ownerId" |
| A2 | API `src/ride/ride.service.ts:37` | `owner.phone` always set (leaks it without a viewer) | ✅ Killed | 6 tests: `ride.service.spec.ts` "sem viewer: owner sem a chave phone", "owner com id e nome completo"; `ride.controller.spec.ts` GET "sem token" / "token inválido" ×2 paths |
| A3 | API `src/ride/ride.controller.ts:25` | Removed `@UseGuards(JwtAuthGuard)` from `POST /ride` | ✅ Killed | 8 tests: every `POST /ride` 401 case, the 201 case, and the `auth.controller.spec.ts` guard map |
| A4 | API `src/ride-users/ride-users.service.ts:34` | Inverted the own-ride check (`===` to `!==`) | ✅ Killed | 16 tests, including "dona da carona → 409" and "outra usuária na carona da dona → inscrição criada" |
| A5 | API `src/ride/ride.service.ts:120` | Removed `usersRepository.update(ownerId, { phone })` | ✅ Killed | `ride.service.spec.ts` "com phone: grava o telefone na conta"; `ride.controller.spec.ts` "com telefone válido: 201" |
| A6 | API `src/ride/ride.service.ts:114` | Dropped `ownerId` from `rideRepository.create` | ✅ Killed | `ride.service.spec.ts` "salva a carona com ownerId"; `ride.controller.spec.ts` "com token válido: 201 ... ownerId do token" |
| A7 | API `src/ride/ride.service.ts:108` | Spread `createRideDto` instead of `rideData` (phone saved on the ride) | ✅ Killed | `ride.service.spec.ts:222`; `ride.controller.spec.ts:206` |
| F1 | Front `src/components/Card.tsx:119` | WhatsApp shown without `showPhones` | ✅ Killed | `Card.test.tsx` "deslogada: sem botão WhatsApp, mesmo se o telefone vier" |
| F2 | Front `src/pages/Form.tsx:230` | WhatsApp field always rendered | ✅ Killed | `Form.test.tsx` "conta com telefone: sem 'Seu Nome' e sem 'WhatsApp'" |
| F3 | Front `src/pages/Home.tsx:103` | "Vou para a soft" link always `/form` | ✅ Killed | `Home.test.tsx` "deslogada: 'Vou para a soft' leva para /login" |
| F4 | Front `src/components/RequireAuth.tsx:8` | Guard never redirects | ✅ Killed | `RequireAuth.test.tsx` "deslogada em /form é redirecionada para /login" |
| F5 | Front `src/pages/Form.tsx:96` | Removed `updateUser` after a publish with a phone | ✅ Killed | `Form.test.tsx` "com WhatsApp: envia phone e a sessão passa a ter o telefone" |
| F6 | Front `src/components/Card.tsx:37` | `isOwner = false` | ✅ Killed | `Card.test.tsx` "dona logada: 'Sua carona'" ×2; `Home.test.tsx` "logada vendo a própria carona" |
| F7 | Front `src/service/ride.service.ts:27` | `phone` always sent (including empty/undefined) | ✅ Killed | `ride.service.test.ts` "sem phone quando não há telefone", "telefone vazio não é enviado" |
| F8 | Front `src/components/Card.tsx:122` | WhatsApp href uses `owner.name` | ✅ Killed | `Card.test.tsx` "logada e dona com telefone: botão WhatsApp para wa.me/<telefone da dona>" |

**Result**: 15/15 killed - PASS ✅

Note: in the first front sensor pass, every test file failed to load ("no tests") because vitest was launched from the 8.3 short path (`GIOVAN~1.MAR`). That pass was thrown out. The front mutants were re-run from the long path, and every kill listed above is a real assertion failure. The unmutated front worktree passed (`Card.test.tsx` 14/14) before the re-run.

**Isolation**: `git status --porcelain` was identical before and after:
- root: ` M soft-go-ii-api` (caused only by the submodule's pre-existing dirty file) and `?? prds/`
- API: ` M tsconfig.build.tsbuildinfo`
- front: clean
- `git worktree list` in both submodules shows only the main worktree.

---

## Code Quality

| Principle | Status |
| --------- | ------ |
| Minimum code | ✅ `toOwner` mirrors the existing `toParticipant`. `RequireAuth` is 13 lines and mirrors `GuestOnly` |
| Surgical changes | ✅ The diff touches only the ride/ride-users/front files the tasks name. `Form.tsx` also wires `errors` into existing fields (L-003, part of T9 done-when) |
| No scope creep | ✅ |
| Matches patterns | ✅ ESM `.js` imports in the API; relative imports without extension in the front; Tailwind tokens; `Toast` helper; PT-BR text |
| Spec-anchored outcome check | ✅ (1 spec-precision gap flagged: OWNER-30) |
| Per-layer coverage (domain 1:1; routes happy + edge + error) | ✅ `POST /ride`: 201, 400×2, 401×4, phone valid/empty/invalid. `GET /ride` and `/ride/:id`: no token / valid / invalid. `POST /ride-users`: own-ride 409 |
| Every test maps to a requirement | ✅ The extra required-field validation tests in `Form.test.tsx:165-193` map to T9 done-when (L-003) |
| Documented guidelines followed | ✅ `CLAUDE.md` (root), `soft-go-ii-api/CLAUDE.md` |

Non-blocking observation: `RideService.create` saves the ride and then calls `usersRepository.update` without a transaction (`ride.service.ts:117-121`). If the phone update failed after the save, the ride would exist and the request would return 500. The DTO validates the phone beforehand, so this is unlikely, and the spec does not ask for atomicity. It is not a gap.

---

## Fix Plans

None required. Optional, for the spec author: in future specs, give the exact text of API validation messages (see OWNER-30).

---

## Requirement Traceability Update

| Requirement | Previous Status | New Status |
| ----------- | --------------- | ---------- |
| OWNER-01..OWNER-20, OWNER-25..OWNER-30 | Done (T2–T10) | ✅ Verified |
| OWNER-21..OWNER-24 | Done (T1) | ✅ Verified (inspection + manual DB run by the author) |

---

## Summary

**Overall**: ✅ Ready

**Spec-anchored check**: 30/30 ACs matched the spec outcome. 1 minor spec-precision gap (OWNER-30, message not named in the spec; the test is exact anyway)
**Sensor**: 15/15 mutations killed
**Gate**: API 121 passed; front 136 passed; front build and type-checks green

**What works**: owner taken from the token on `POST /ride` (guarded; 401 for a missing, invalid, expired, or orphaned token, with no write); `owner { id, name, phone? }` in `GET /ride`, with the phone only for logged-in viewers; the owner cannot join her own ride (409 in the API, "Sua carona" in the front); the form without a name field, with an optional WhatsApp field that is saved to the account and the session; `/form` and "Vou para a soft" gated behind login; the migration drops old rides and enforces `owner_id NOT NULL` + `FK_rides_owner`.

**Issues found**: none blocking.

**Next steps**: The orchestrator updates the traceability statuses in `spec.md` and the status line in `tasks.md`, then commits.
