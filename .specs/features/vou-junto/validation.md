# Vou Junto com Login Validation

**Date**: 2026-10-08
**Verdict**: PASS
**Iteration**: 3 of 3 (final re-verification after the iteration-2 fix commits)
**Spec**: `.specs/features/vou-junto/spec.md` (JOIN-01..JOIN-28)
**Diff range**: API `soft-go-ii-api` `06b51b4..be7c253` (`master`, 10 commits); front `soft-go-II` `f1de23e..0ffc132` (`main`, 10 commits)
**Fix commits verified in this iteration**: front `0ffc132` (`src/pages/Home.test.tsx`, +4 lines), API `be7c253` (`src/ride-users/ride-users.controller.spec.ts`, +8 lines). `git diff bc4f85c..be7c253` and `git diff ca636c5..0ffc132` touch only those test files. No production code has changed since iteration 1 (`git diff b10541a HEAD -- src` and `git diff 226bf5a HEAD` list only spec/test files).
**Verifier**: independent sub-agent (author ≠ verifier). I re-derived coverage from the spec and re-checked every cited line at HEAD.

Both iteration-2 surviving mutants are now killed by real assertion failures:

- F17 fails at `soft-go-II/src/pages/Home.test.tsx:100`.
- A16 fails at `soft-go-ii-api/src/ride-users/ride-users.controller.spec.ts:132`.

F10 stays killed. All 9 fresh mutants were killed. No gaps remain.

---

## Task Completion

| Task | Status | Notes |
| ---- | ------ | ----- |
| T1–T8 (API) | ✅ Done | `1f48a76`..`b10541a`, plus the test fixes `bc4f85c` and `be7c253` |
| T9–T16 (front) | ✅ Done | `60defbb`..`226bf5a`, plus the test fixes `ca636c5` and `0ffc132` |

---

## Gate Check

| Gate | Command | Result |
| ---- | ------- | ------ |
| API type-check (build) | `npx tsc --noEmit -p tsconfig.build.json` | exit 0 |
| API type-check (all) | `npx tsc --noEmit -p tsconfig.json` | exit 1. The only error is the tolerated pre-existing `test/app.e2e-spec.ts(4,21) TS2307` (`supertest/types`) |
| API lint | `npm run lint` (oxlint) | exit 0. 5 warnings, 0 errors: `no-useless-escape` ×3 in the copied phone regex, and 2 pre-existing unused imports in `transport-type.controller.ts` |
| API tests | `npm test` (vitest) | exit 0: **11 files, 96 passed**, 0 failed, 0 skipped |
| Front lint | `npm run lint` (eslint) | exit 0. 0 errors, 1 warning: `Home.tsx:45` `exhaustive-deps` on `userId`, which is intentional and commented |
| Front build | `npm run build` (`tsc -b && vite build`) | exit 0 |
| Front tests | `npm test` (vitest) | exit 0: **17 files, 109 passed**, 0 failed, 0 skipped |

- **Test count**:
  - API went from 95 to 96. The new test is "telefone vazio é tratado como ausente: 201 sem mexer na conta".
  - Front stayed at 109. The F17 fix added assertions to the existing test "logada com telefone…".
  - No test was removed and no assertion was weakened.

---

## Spec-Anchored Acceptance Criteria

Path prefixes: `api/` = `soft-go-ii-api/src/`, `fe/` = `soft-go-II/src/`. Every line was re-checked at HEAD (`be7c253` / `0ffc132`). Line numbers after the inserted tests have moved: `Home.test.tsx` +4 after line 98, and `ride-users.controller.spec.ts` +8 after line 126.

| AC | Spec-defined outcome | `file:line` + assertion | Result |
| -- | -------------------- | ----------------------- | ------ |
| JOIN-01 | Logged-in user with a phone: the modal shows the **ride summary** and "Confirmar presença", with no name or phone field. This holds on every join, including the next one | `fe/components/Modal.test.tsx:37-40` `within(dialog).getByText("Dona da Carona")`, `/Porto Alegre/`, `"08:00"`, no `button "Vou junto"`. `:27` no `textbox`. `:46` no `/nome/` label. `fe/pages/Home.test.tsx:89` dialog without a textbox. **`:100` `expect(getSession()?.user.phone).toBe("51999998888")`** after a confirm that sent no phone. **`:102`** the next modal has no textbox. F10, F15 and F17 are killed | ✅ |
| JOIN-02 | `POST /ride-users` with `{ rideId }` + `Authorization: Bearer`, no `name` | `fe/pages/Home.test.tsx:96` `createRideUser` called with `{ rideId: 3 }`; `fe/service/ride.service.test.ts:18,33,39` `api.post("/ride-users", { rideId: 3 })`; `fe/service/api.test.ts:47` `Authorization` = `"Bearer token-abc"` | ✅ |
| JOIN-03 | Valid token + ride with a free spot → inscription with `user_id` from the token, `201` | `api/ride-users/ride-users.controller.spec.ts:86-88` `status 201`, `body toEqual { id:1, rideId:3, userId:7 }`, `rows` persisted; `api/ride-users/ride-users.service.spec.ts:45-46` | ✅ |
| JOIN-04 | Body with `name` → `400` | `ride-users.controller.spec.ts:124-126` `status 400`, `toContain 'property name should not exist'`, `rows []` | ✅ |
| JOIN-05 | `201` → toast "Presença confirmada!", modal closes, list reloads | `fe/pages/Home.test.tsx:94` `Toast("success","Presença confirmada!")`; `:97` `getRides` call count grows; `:98` dialog removed (F21 killed) | ✅ |
| JOIN-06 | Full ride → `409` "Não existem mais vagas disponíveis para este transporte"; front error toast with that message | `api/ride-users/ride-users.service.spec.ts:83-86` `ConflictException`, `message toBe FULL`, no save and no update; `fe/pages/Home.test.tsx:155` `Toast("error", <exact message>)` | ✅ |
| JOIN-07 | No token, invalid token or expired token → `401`, no inscription | `ride-users.controller.spec.ts:104-105` and `:117-118` `status 401`, `rows toEqual []`; `api/auth/auth.controller.spec.ts:346` `create: [JwtAuthGuard]` | ✅ |
| JOIN-08 | Logged out + "Vou junto" → `/login`, no modal | `fe/pages/Home.test.tsx:80-81` | ✅ |
| JOIN-09 | `GET /ride`, `POST /ride`, `GET /transport-type` open without a token | `api/ride/ride.controller.spec.ts:74` (`GET /ride` `200`), `:94`, `:108` (`POST /ride` `201`); `api/auth/auth.controller.spec.ts:338-347` `TransportTypeController: {}`; `:388-393` no `APP_GUARD`; `api/auth/optional-jwt-auth.guard.spec.ts` "sem header: libera como anônima" (A23 killed) | ✅ |
| JOIN-10 | Already joined → `409` "Você já confirmou presença nesta carona", no new row | `ride-users.controller.spec.ts:153-155` `409`, `message toBe(...)`, `rows toHaveLength(1)`; `ride-users.service.spec.ts:68-72` | ✅ |
| JOIN-11 | `UNIQUE` violation → `409` with the same message, never `500` | `ride-users.service.spec.ts:136-137` `23505` → `ConflictException`, `message toBe DUPLICATE`; `:149` other errors rethrown (A22 killed); real DB `23505 UQ_ride_user` (JOIN-28 run) | ✅ |
| JOIN-12 | Logged in and already joined → "Você vai nesta carona" instead of "Vou junto" | `fe/components/Card.test.tsx:66-67`; `fe/pages/Home.test.tsx:224-225` (F23 killed) | ✅ |
| JOIN-13 | Signup shows the optional "WhatsApp (opcional)" field; the API accepts `phone` | `fe/pages/SignUp.test.tsx:142`; `api/auth/auth.controller.spec.ts:129-133` | ✅ |
| JOIN-14 | Valid phone saved; empty or missing → `null` | `auth.controller.spec.ts:133` `toBe('51999998888')`, `:140` empty → `toBeNull()` (A25 killed); `api/auth/auth.service.spec.ts:64,73` | ✅ |
| JOIN-15 | `phone` in `user` for signup, signin and `/auth/me` | `auth.controller.spec.ts:124` (signup), `:251` (signin), `:306` + `:317` (me); `auth.service.spec.ts:180,225` | ✅ |
| JOIN-16 | No phone on the account → modal shows the optional "WhatsApp" field **above** the button | `fe/components/Modal.test.tsx:52-55` `getByLabelText("WhatsApp")`, `compareDocumentPosition(button) & DOCUMENT_POSITION_FOLLOWING` truthy; `fe/pages/Home.test.tsx:105-111` | ✅ |
| JOIN-17 | Phone filled in the modal → saved on the token's account; session updated; the next modal has no field | `ride-users.controller.spec.ts:95` `users.update(7, { phone })`; `ride-users.service.spec.ts:116-117` (update before save); `fe/pages/Home.test.tsx:117-125` payload with phone, `getSession().user.phone` `toBe("51988887777")`, next modal with no textbox; `fe/context/AuthContext.test.tsx:101-103` | ✅ |
| JOIN-18 | Not 11 digits → blocks submit + "Número inválido, informe 11 dígitos (DDD + número)" | `fe/components/Modal.test.tsx:82-84` exact text, `toHaveAccessibleDescription`, `onConfirm` not called; `fe/pages/SignUp.test.tsx:151-155`; `fe/schemas/phone.schema.test.ts:19-20,32` | ✅ |
| JOIN-19 | Phone outside the BR rule → `400` in PT-BR, without creating the account or the inscription | `ride-users.controller.spec.ts:140-145` `400`, exact message, `users.update` not called, `rows toEqual []`; `auth.controller.spec.ts:146-151` `400`, exact message, then `signIn` with the same credentials → `401` | ✅ |
| JOIN-20 | `participants` `{ userId, name }`, first name | `api/ride/ride.service.spec.ts:44,55-58`; `ride.controller.spec.ts:75` | ✅ |
| JOIN-21 | Valid token → each participant has `phone` (string or `null`) | `ride.service.spec.ts:69-72`; `ride.controller.spec.ts:84`, `:106` | ✅ |
| JOIN-22 | No token, or invalid or expired token → `200`, no `phone` key | `ride.service.spec.ts:60` `not.toHaveProperty('phone')`; `ride.controller.spec.ts:74-75`, `:94-95`; `api/auth/optional-jwt-auth.guard.spec.ts:62-63` | ✅ |
| JOIN-23 | Card lists participants' first names | `fe/components/Card.test.tsx:34-36,58`; `fe/pages/Home.test.tsx:213` | ✅ |
| JOIN-24 | Logged in → `https://wa.me/<phone>` link for each participant with a phone | `fe/components/Card.test.tsx:42,48-52`; `fe/pages/Home.test.tsx:203,214` (F24 killed: exact URL) | ✅ |
| JOIN-25 | Spots still subtract each inscription | `ride.service.spec.ts:89-90` `spots toBe(2)`, `spotsRide toBe(4)` (A21 killed); `ride-users.service.spec.ts:83-86`, `:93`, `:104` | ✅ |
| JOIN-26 | `rideId` does not exist → `404` "Corrida não encontrada" | `ride-users.service.spec.ts:55-56` `toBeInstanceOf(NotFoundException)`, `message toBe('Corrida não encontrada')` (A24 killed) | ✅ |
| JOIN-27 | Session expired at confirm → AUTH-26 flow, no success toast | `fe/pages/Home.test.tsx:170-172` `Toast` not called; `fe/service/api.test.ts:93-94`; `fe/context/AuthContext.test.tsx:149-153` | ✅ |
| JOIN-28 | Migration deletes rows with no account and drops `name`, `phone` and `UQ_ride_phone`. `user_id` becomes `NOT NULL` with an FK and `UNIQUE (ride_id,user_id)` | `api/ride-users/entities/ride-user.entity.spec.ts:12-13,24-26,32,42-43`; `api/users/entities/user.entity.spec.ts:19-20`; real run on a throwaway DB in iteration 1 (reused, see below) | ✅ |
| Assumption "String vazia = ausente" on `POST /ride-users` | `phone: ""` is treated as absent: `201`, account untouched | `api/ride-users/ride-users.controller.spec.ts:132-134` `status 201`, `users.update` not called, `rows toEqual [{ id:1, rideId:3, userId:7 }]` (A16 killed). Front: `fe/components/Modal.test.tsx:63` `onConfirm(undefined)` (F22 killed) | ✅ |

**Status**: ✅ 28/28 ACs and the empty-phone assumption have spec-anchored evidence. No spec-precision gaps.

### JOIN-28: migration evidence (reused from iteration 1)

`git log` shows that the last commit to touch the migrations and the entities is `82c062a` (T2), which is earlier than iteration 1. The migrations involved are `1791413863720-AddColumnPhoneInUsersTable` and `1791413956190-LinkRideUsersToUsers`. `git diff b10541a HEAD -- src` lists only `auth.controller.spec.ts` and `ride-users.controller.spec.ts`. The iteration-1 run on a throwaway DB still applies:

- The run used `db_soft_go_verify`, which was created for it and then dropped. **No iteration touched `db_soft_go`.**
- The legacy rows went from 1 to 0.
- `ride_users` now has the columns `id, ride_id, user_id`, all `NOT NULL`.
- `FK_ride_users_user` points to `users(id)`. `UQ_ride_user UNIQUE (ride_id,user_id)` exists and `UQ_ride_phone` is gone.
- Inserting a duplicate fails with `23505`. A missing `user_id` fails with `23502`, and an unknown user fails with `23503`.
- The schema diff between the entities and the DB is empty, and `down()`/`up()` round-trips.

---

## Discrimination Sensor

**Tier**: expanded (auth + data integrity).

**Method**:
- I created the scratch trees with `git -C <submodule> worktree add --detach <scratch>\verifier-vj3\{api,front} HEAD`. The long scratch path is `C:\Users\giovanna.marques\AppData\Local\Temp\claude\c--Users-giovanna-marques-Documents-soft-go-pd\2e5ceda8-…\scratchpad\verifier-vj3`. `node_modules` came from PowerShell junctions.
- The unmutated scratch trees passed first: API 11 files / 96 tests, front 17 files / 109 tests.
- A Python runner (`scratchpad/vj3_mutate.py`) applied one mutation at a time. It refused any pattern that did not match exactly once (CRLF-aware), ran `npx vitest run <scope>`, and restored the file with `git checkout -- <file>`. After each restore, the scratch `git status --porcelain` was empty.
- A mutant counts as killed only when vitest reports failed tests with exit 1 from assertions. There were no load or compile errors.
- For F17 and A16, I re-ran the mutant to find the failing assertion line:
  - F17: `Home.test.tsx:100:38` (`getSession()?.user.phone`, expected `"51999998888"`).
  - A16: `ride-users.controller.spec.ts:132:24` (status expected `201`, received `400`).

| # | Mutation | File:line | Tests run | Killed? |
| - | -------- | --------- | --------- | ------- |
| **F17 (re-run)** | Session updated even when no phone was typed: `if (phone && user)` → `if (user)` | `fe/pages/Home.tsx:61` | `src/pages` + `src/context` | ✅ **Killed** (1 failed / 34: "logada com telefone…", assertion `Home.test.tsx:100`). It survived in iteration 2 |
| **A16 (re-run)** | `CreateRideUserDto.phone` drops the `'' → undefined` `@Transform` | `api/ride-users/dto/create-ride-user.dto.ts:15` | `src/ride-users` | ✅ **Killed** (1 failed / 23: "telefone vazio é tratado como ausente", assertion `controller.spec.ts:132`). It survived in iteration 2 |
| **F10 (regression)** | Modal drops the ride summary (`<Card {...ride} showButton={false} />` removed) | `fe/components/Modal.tsx:60` | Modal + `src/pages` | ✅ Killed (1 failed / 34: "mostra o resumo da carona…") |
| A21 | `GET /ride` stops subtracting inscriptions from the spots (`spots: spots`) | `api/ride/ride.service.ts:68` | `src/ride` | ✅ Killed (1 failed / 37: "continua descontando as inscrições das vagas") |
| A22 | `23505` mapping keyed on the wrong code (`=== '23503'`) | `api/ride-users/ride-users.service.ts:57` | `src/ride-users` | ✅ Killed (2 failed / 23: the `23505 → 409` test and "relança outros erros do banco") |
| A23 | `OptionalJwtAuthGuard` blocks anonymous requests (no-token path `return false`) | `api/auth/optional-jwt-auth.guard.ts:18` | `src/ride` + `src/auth` | ✅ Killed (4 failed / 94: the guard spec and `GET /ride` "sem token: 200") |
| A24 | Missing ride throws `ConflictException` instead of `NotFoundException` (409, not 404) | `api/ride-users/ride-users.service.ts:31` | `src/ride-users` | ✅ Killed (1 failed / 23: "carona inexistente → 404") |
| A25 | `SignUpDto.phone` drops the `'' → undefined` `@Transform` | `api/auth/dto/sign-up.dto.ts:29` | `src/auth` | ✅ Killed (1 failed / 57: "telefone vazio é salvo como null") |
| F21 | Modal never closes after confirm (`setSelectedRide(null)` removed from `finally`) | `fe/pages/Home.tsx:73` | `src/pages` | ✅ Killed (2 failed / 26) |
| F22 | Modal forwards the empty string (`phone \|\| undefined` → `phone`) | `fe/components/Modal.tsx:27` | `src/components` + `src/pages` | ✅ Killed (1 failed / 51: "com askPhone e campo vazio: confirma sem telefone") |
| F23 | Card shows "Vou junto" to a user who already joined (`!isParticipant` removed) | `fe/components/Card.tsx:140` | `src/components` + `src/pages` | ✅ Killed (2 failed / 51) |
| F24 | Participant WhatsApp link in the wrong format (`wa.me/55${phone}`) | `fe/components/Card.tsx:107` | `src/components` + `src/pages` | ✅ Killed (2 failed / 51) |

**Result**: 12 injected (3 re-runs + 9 fresh), 12 killed, 0 survived. **PASS ✅**

**Isolation**:
- I removed both junctions with `cmd //c rmdir`. The real `node_modules` are intact: `soft-go-ii-api/node_modules/@nestjs` and `soft-go-II/node_modules/react` are still present.
- I ran `git worktree remove --force` on both trees, then `git worktree prune`.
- `git worktree list` shows only the main trees: API `be7c253 [master]`, front `0ffc132 [main]`, root `68f9d05 [main]`.
- `git status --porcelain` matches the baseline I recorded before the sensor, exactly:
  - root: ` M soft-go-ii-api`, `?? .specs/features/vou-junto/validation.md`, `?? prds/`
  - API: ` M tsconfig.build.tsbuildinfo`
  - front: clean

---

## Code Quality

| Check | Status |
| ----- | ------ |
| Fix commits are test-only and scoped to the reported gaps | ✅ `Home.test.tsx` +4 lines (F17); `ride-users.controller.spec.ts` +8 lines (A16) |
| New assertions target the spec outcome, not just presence | ✅ Exact session phone value, then no textbox on the next modal; `201` + `users.update` not called + exact `rows` |
| Spec-anchored outcome check | ✅ |
| Per-layer coverage (domain 1:1; HTTP happy + edge + error) | ✅ |
| Every test maps to an AC, edge case or Done-when | ✅ (the new tests map to JOIN-01 follow-up join and the "String vazia = ausente" assumption) |
| Documented guidelines (`CLAUDE.md`, `soft-go-ii-api/CLAUDE.md`) | ✅ |

---

## Edge Cases

- [x] JOIN-26: ride not found → `404` (`ride-users.service.spec.ts:55-56`)
- [x] JOIN-27: expired session at confirm → no success toast (`Home.test.tsx:170-172`)
- [x] JOIN-28: migration (throwaway DB in iteration 1; unchanged since)
- [x] Phone not updated when the join fails as a duplicate or because the ride is full (`ride-users.service.spec.ts:72,86`)
- [x] User with a phone confirms without typing one → session phone kept, and the next modal has no field (`Home.test.tsx:100-102`)
- [x] `POST /ride-users` with `phone: ""` → treated as absent (`ride-users.controller.spec.ts:132-134`)

---

## Requirement Traceability Update

| Requirement | New Status |
| ----------- | ---------- |
| JOIN-01..JOIN-28 | ✅ Verified |

---

## Iteration history

| Iteration | Range (API / front) | Gate | Sensor | Verdict | Gaps |
| --------- | ------------------- | ---- | ------ | ------- | ---- |
| 1 | `06b51b4..b10541a` / `f1de23e..226bf5a` | API 95, front 108 | 26 injected per the iteration-1 report (A1–A12, F1–F13 recorded in the scratch results), 25 killed; F10 survived | FAIL | F10 / JOIN-01 ride summary missing from the modal (Major); JOIN-16 field position (minor); JOIN-19 "no record created" (minor) |
| 2 | `06b51b4..bc4f85c` / `f1de23e..ca636c5` | API 95, front 109 | 16 injected, 14 killed; F17 and A16 survived | FAIL | F17 / JOIN-01 follow-up join loses the session phone (Major); A16 / empty phone on `POST /ride-users` (Minor). All iteration-1 gaps closed (F10, F14, A13, A14 killed) |
| 3 | `06b51b4..be7c253` / `f1de23e..0ffc132` | API 96, front 109 | 12 injected (F17, A16, F10 re-runs + A21–A25, F21–F24), 12 killed | **PASS** | None. Fix commits `0ffc132` and `be7c253` are test-only and kill F17 and A16 |

---

## Summary

**Overall**: ✅ Ready.

**Spec-anchored check**: 28/28 ACs matched the spec outcome. The "String vazia = ausente" assumption is now covered. No spec-precision gaps.
**Sensor**: 12/12 killed. F17 and A16 are now killed, and F10 stays killed.
**Gate**: API 96 passed, front 109 passed, 0 failed. tsc, lint and build are green. The only tolerated error is `test/app.e2e-spec.ts(4,21) TS2307`.

**What works**:
- Logged-in, one-click join that keeps the session phone.
- `401` without a token.
- `409` for a duplicate join, including the `23505` race.
- The account phone is collected by the modal and saved.
- Participants appear on the card, with phones only for logged-in viewers.
- The migration.

**Next steps**: none for the feature. Optionally run the interactive UAT from the spec's "Independent Test" lines.

---

## validate_state output

```
$ py .claude/skills/tlc-spec-driven/scripts/validate_state.py vou-junto
validate_state: 0 error(s) across [vou-junto]
EXIT=0
```
