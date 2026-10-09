# Sair da Carona (leave-ride) — Validation Report

## Validation: leave-ride - PASS ✅

**Date**: 2026-10-08
**Spec**: `.specs/features/leave-ride/spec.md` (LEAVE-01..LEAVE-21)
**Diff range**: API `soft-go-ii-api` `541c0ca..432a706` (2 commits: 28faf5f, 432a706) · Front `soft-go-II` `44b4e9b..d3ddb66` (5 commits: 2063e61, 7a06fcb, 7a7c674, 8e2eb19, d3ddb66)
**Iteration**: 1 of 3
**Verifier**: independent sub-agent (author ≠ verifier)

---

## Task Completion

| Task | Status | Notes |
| ---- | ------ | ----- |
| T1 | ✅ Done | `28faf5f` `RideUsersService.remove` (404 ride → 403 → 404 enrollment → delete by enrollment id) |
| T2 | ✅ Done | `432a706` `DELETE /ride-users/:rideId/users/:userId` (`JwtAuthGuard`, two `ParseIntPipe`, `204`, Swagger) + `EXPECTED_GUARDS` entry. Manual local-DB check reported by the author (403 / 204 with spots 0 → 1 / re-join 201 / owner 204 / repeat 404); not re-run by the verifier, per instructions |
| T3 | ✅ Done | `2063e61` `removeRideUser(rideId, userId)` |
| T4 | ✅ Done | `7a06fcb` `ParticipantsList` (collapsible, trash rules) |
| T5 | ✅ Done | `7a7c674` `ConfirmDialog` |
| T6 | ✅ Done | `8e2eb19` `Card` uses `ParticipantsList`; existing participant tests rewritten to open the list first |
| T7 | ✅ Done | `d3ddb66` Home wiring + `handleRemove` error handling |

---

## Spec-Anchored Acceptance Criteria

API paths are relative to `soft-go-ii-api/`, front paths to `soft-go-II/`.

| AC | Spec-defined outcome | `file:line` + assertion | Result |
| -- | -------------------- | ----------------------- | ------ |
| LEAVE-01 | passenger's own token → enrollment deleted, `204`, empty body | `src/ride-users/ride-users.controller.spec.ts:196` `expect(res.status).toBe(204)`; `:197` `expect(res.text).toBe('')`; `:198` `expect(enrolled()).toEqual([])`; unit `src/ride-users/ride-users.service.spec.ts:215` `resolves.toBeUndefined()`, `:218` lookup `{ rideId: 3, userId: 7 }`, `:219` `delete` with `10` (the enrollment id) | ✅ PASS |
| LEAVE-02 | owner token → `:userId` enrollment deleted, `204`, empty body | `src/ride-users/ride-users.controller.spec.ts:204` `toBe(204)`; `:205` `enrolled()` `toEqual([])`; unit `src/ride-users/ride-users.service.spec.ts:227-228` lookup `{ rideId: 3, userId: 7 }` + `delete(10)` with requester 5 (owner). Empty body for the owner path follows from the same handler/`@HttpCode(NO_CONTENT)` (`src/ride-users/ride-users.controller.ts:51`), asserted on the passenger path at `:197` | ✅ PASS |
| LEAVE-03 | after deletion `GET /ride` no longer lists the person in `participants`, returns the spot in `transportType.spots`; person can join again / another ride the same day | Deletion: `src/ride-users/ride-users.controller.spec.ts:198,205`. Listing is derived live from `ride_users`: `src/ride/ride.service.spec.ts:100` `spots` `toBe(2)` (4 − 2 enrollments), `:119` full → `0`, `:128` bus stays `null`, `:136` no enrollments → `participants` `toEqual([])` (impl `src/ride/ride.service.ts:93-100`). Re-join: `src/ride-users/ride-users.controller.spec.ts:213` `toBe(201)`, `:214` `[[3, 7]]`. Same-day rule is a live `exists` on `ride_users` (`src/ride/ride.service.spec.ts:277-279`, impl `src/ride/ride.service.ts:110-116`). Front: after leaving, the board reloads and shows "Vou junto" again `soft-go-II/src/pages/Home.test.tsx:427` | ✅ PASS (composed — see observation 1) |
| LEAVE-04 | missing ride → `404` "Carona não encontrada" | `src/ride-users/ride-users.controller.spec.ts:230` `toBe(404)`, `:231` `res.body.message` `toBe('Carona não encontrada')`, `:232` nothing deleted; unit `src/ride-users/ride-users.service.spec.ts:236-238` `NotFoundException`, exact message, `delete` not called | ✅ PASS |
| LEAVE-05 | neither passenger nor owner → `403` "Você só pode sair da carona ou remover passageiras da sua carona"; enrollment kept | `src/ride-users/ride-users.controller.spec.ts:220` `toBe(403)`, `:221-223` exact message, `:224` `enrolled()` `toEqual([[3, 7]])`; unit `src/ride-users/ride-users.service.spec.ts:246-251` `ForbiddenException`, exact message, enrollment not looked up, `delete` not called | ✅ PASS |
| LEAVE-06 | `:userId` not enrolled (incl. owner as target) → `404` "Esta pessoa não está nesta carona" | `src/ride-users/ride-users.controller.spec.ts:238` `toBe(404)`, `:239` exact message, `:240` nothing deleted; unit `src/ride-users/ride-users.service.spec.ts:259-261`; owner as target `:269-272` exact message + lookup `{ rideId: 3, userId: 5 }` + no `delete` | ✅ PASS |
| LEAVE-07 | no `Authorization` / invalid / expired / revoked → `401` "Não autenticado", nothing deleted | `src/ride-users/ride-users.controller.spec.ts:256-258` (no header), `:270-272` (other secret), `:285-287` (expired), `:295-297` (old `ver`): each `toBe(401)` + `message` `toBe('Não autenticado')` + `enrolled()` `[[3, 7]]` | ✅ PASS |
| LEAVE-08 | non-integer `:rideId` or `:userId` → `400`, nothing deleted | `src/ride-users/ride-users.controller.spec.ts:243-250` `it.each(['/ride-users/abc/users/7', '/ride-users/3/users/abc'])` → `toBe(400)` + `enrolled()` `[[3, 7]]` | ✅ PASS |
| LEAVE-09 | `POST /ride-users` keeps `JwtAuthGuard`; no global guard | `src/auth/auth.controller.spec.ts:452` `[RideUsersController, { create: [JwtAuthGuard], remove: [JwtAuthGuard] }]` checked per handler at `:468` (`GUARDS_METADATA` `toEqual`) and `:474` (every handler listed); `:502` `expect(globalGuards).toEqual([])`; HTTP: `src/ride-users/ride-users.controller.spec.ts:117` `POST` without token `toBe(401)` | ✅ PASS |
| LEAVE-10 | with participants: button "Ver passageiros (N)", `aria-expanded="false"`, no names | `src/components/ParticipantsList.test.tsx:34` `toHaveTextContent("Ver passageiros (2)")`, `:35` `aria-expanded` `"false"`, `:36-37` names absent; `src/components/Card.test.tsx:38-40`; page `src/pages/Home.test.tsx:409-410` | ✅ PASS |
| LEAVE-11 | click → first names listed, button "Esconder passageiros" with `aria-expanded="true"`; click again hides | `src/components/ParticipantsList.test.tsx:45-48` names + `"Esconder passageiros"` + `"true"`; `:52-54` hidden again, back to `"Ver passageiros (2)"` / `"false"`; first names: `src/components/Card.test.tsx:47-48` | ✅ PASS |
| LEAVE-12 | no participants → no button, no list | `src/components/ParticipantsList.test.tsx:60` no `button`, `:61` no `list`; `src/components/Card.test.tsx:72` | ✅ PASS |
| LEAVE-13 | open + logged → name with phone links `https://wa.me/<phone>`; logged out → no link | `src/components/ParticipantsList.test.tsx:70-73` `href` `"https://wa.me/51999998888"`, `:74` no link without phone, `:82` `showPhones=false` → no link; `src/components/Card.test.tsx:55,62-66`; page `src/pages/Home.test.tsx:207-210` (logged, `wa.me/51911112222`), `:218-219` (logged out, name without link) | ✅ PASS |
| LEAVE-14 | logged participant: trash only on own name, accessible name "Sair da carona" | `src/components/ParticipantsList.test.tsx:91` exactly one trash, `:93` it is in Bia's `li`, `:97` `onRemove` with `bia`; `src/components/Card.test.tsx:93` no "Remover", `:96` `onRemoveParticipant` with the participant | ✅ PASS |
| LEAVE-15 | owner: trash on every name, "Remover <nome> da carona" | `src/components/ParticipantsList.test.tsx:104` two trashes, `:105` "Remover Ana da carona" in Ana's `li`, `:109` click on "Remover Bia da carona" → `onRemove(bia)`; `src/components/Card.test.tsx:82,85` | ✅ PASS |
| LEAVE-16 | logged out / unrelated account / `showButton={false}` → no trash | `src/components/ParticipantsList.test.tsx:116` (other account), `:123` (logged out), `:126-134` (`canRemove=false` for owner and participant) `toEqual([])`; `src/components/Card.test.tsx:103` `showButton={false}` | ✅ PASS |
| LEAVE-17 | leave → modal "Sair da carona?" + "Cancelar"/"Sair"; owner → "Remover <nome> da carona?" + "Cancelar"/"Remover" | `src/pages/Home.test.tsx:420` heading "Sair da carona?", `:421` button "Sair", `:456` "Cancelar" in the leave dialog; `:438` heading "Remover Bia da carona?", `:440` button "Remover"; component `src/components/ConfirmDialog.test.tsx:26-28` heading, confirm label and "Cancelar" rendered | ✅ PASS |
| LEAVE-18 | "Cancelar", "X" or outside → close, no API call | `src/components/ConfirmDialog.test.tsx:31-41` Cancelar / X / overlay → `onClose` once, `onConfirm` not called; `:49` inside click doesn't close; page `src/pages/Home.test.tsx:458-459` dialog gone + `removeRideUser` `not.toHaveBeenCalled()` | ✅ PASS |
| LEAVE-19 | `DELETE /ride-users/<rideId>/users/<userId>` with token; confirm disabled while sending; on `204` success toast, close, reload | service `src/service/ride.service.test.ts:111-112` `api.delete` once with `"/ride-users/3/users/7"` (token by the existing interceptor); disabled `src/components/ConfirmDialog.test.tsx:71` `toBeDisabled()`, `:74` `onConfirm` once after second click; Home leave `src/pages/Home.test.tsx:423` `removeRideUser(3, 1)`, `:424` `Toast("success", "Você saiu da carona")`, `:425` dialog gone, `:426` reload; owner `:442` `(3, 2)`, `:444` `"Bia foi removida da carona"`, `:446-447` | ✅ PASS |
| LEAVE-20 | `403`/`404` → error toast with API message, close, reload | `src/pages/Home.test.tsx:462-479` `it.each` 403 and 404: `:475` `Toast("error", message)` with the exact API messages, `:476` dialog gone, `:477` reload, `:478` no success toast | ✅ PASS |
| LEAVE-21 | other error → "Não foi possível remover da carona. Tente novamente." and close; `401` → interceptor | `src/pages/Home.test.tsx:491-494` exact generic toast, `:496` dialog gone; `401`: `:510` `expect(Toast).not.toHaveBeenCalled()` | ✅ PASS |

**Status**: ✅ 21/21 ACs covered with spec-anchored assertions; 0 spec-precision gaps.

### Observations (not gaps)

1. **LEAVE-03 has no single automated end-to-end test.** No test runs `DELETE` and then `GET /ride` to see the participant leave and the spot come back. Coverage is composed: (a) the HTTP tests prove the row is deleted, (b) the `findAllRides` unit tests prove `participants` and `spots` are derived live from `ride_users` (`capacity - rideUser.length`), (c) the re-join test proves the person can enroll again. "Join another ride the same day" is not exercised after a leave; the same-day rule (`RideService.create`) queries `ride_users` live, so it follows from (a). The real-database "spots 0 → 1" outcome rests only on the author's manual check. The composition is sound because deletion is the only state change, so this is not counted as a gap.
2. LEAVE-02 empty body is asserted only on the passenger path (`:197`); the owner path shares the same handler and `@HttpCode(NO_CONTENT)`.
3. LEAVE-17 at page level asserts "Cancelar" only in the leave dialog. The owner dialog uses the same `ConfirmDialog`, which always renders "Cancelar" (`ConfirmDialog.test.tsx:28`).
4. LEAVE-21 `500`: the board is not reloaded (`Home.tsx:120-121`). The spec does not ask for a reload in this case.

### Rewritten participant tests (requirement change, T6)

`8e2eb19` changed 4 Card tests and 2 Home tests so they open the list first. Checked against `44b4e9b`:
- `Card.test.tsx`: the names (`:47-48`), "no link without `showPhones`" (`:55`), and `wa.me` href + "no link for Bia" (`:62-66`) assertions are unchanged; each test only gained `await openParticipants()`. The `getByText("Vão:")` / `queryByText("Vão:")` assertions were replaced because the "Vão:" label no longer exists. The empty-list test now asserts no toggle button (`:72`), which is stricter. The closed-by-default test was added (`:35-40`).
- `Home.test.tsx`: `:207-210` (`wa.me/51911112222`) and `:218-219` (name without link when logged out) are unchanged apart from the added click on "Ver passageiros".
- No test was deleted or skipped. Test count went up: front 174 → 206, API 167 → 185.

---

## Discrimination Sensor

Run in temporary `git worktree`s under the long scratchpad path (`…\scratchpad\wt-leave-api` at `432a706`, `…\scratchpad\wt-leave-front` at `d3ddb66`), with `node_modules` linked by junction. Unmutated baseline in scratch: API 84/84 (`src/ride-users`, `src/auth/auth.controller.spec.ts`), front 143/143 (`src/components`, `src/pages/Home.test.tsx`, `src/service`). A script applied each mutant (exact-match replace), ran the targeted tests, and restored the file. Afterwards `git diff --ignore-cr-at-eol` was empty in both worktrees.

| # | File:line | Mutation | Killed? (fails) |
| - | --------- | -------- | --------------- |
| A1 | `soft-go-ii-api/src/ride-users/ride-users.service.ts:79` | drop the owner branch of the permission check | ✅ 3 |
| A2 | `soft-go-ii-api/src/ride-users/ride-users.service.ts:79` | allow any requester (`false`) | ✅ 2 |
| A3 | `soft-go-ii-api/src/ride-users/ride-users.service.ts:79` | drop the passenger (self) branch | ✅ 4 |
| A4 | `soft-go-ii-api/src/ride-users/ride-users.service.ts:79-87` | swap 403 and enrollment-404 order | ✅ 1 |
| A5 | `soft-go-ii-api/src/ride-users/ride-users.service.ts:89` | delete the wrong row (`rideId` instead of enrollment id) | ✅ 5 |
| A6 | `soft-go-ii-api/src/ride-users/ride-users.controller.ts:58` | remove `ParseIntPipe` on `:userId` | ✅ 1 |
| A7 | `soft-go-ii-api/src/ride-users/ride-users.controller.ts:57` | remove `ParseIntPipe` on `:rideId` | ✅ 1 |
| A8 | `soft-go-ii-api/src/ride-users/ride-users.controller.ts:51` | remove `@HttpCode(204)` (answers 200) | ✅ 2 |
| A9 | `soft-go-ii-api/src/ride-users/ride-users.service.ts:86` | wrong not-enrolled 404 message | ✅ 3 |
| A10 | `soft-go-ii-api/src/ride-users/ride-users.service.ts:76` | missing ride answers 403 instead of 404 | ✅ 2 |
| A11 | `soft-go-ii-api/src/ride-users/ride-users.controller.ts:61` | ignore token identity (requester = `:userId`) | ✅ 1 |
| A12 | `soft-go-ii-api/src/ride-users/ride-users.controller.ts:48-49` | remove `JwtAuthGuard` from the route | ✅ 11 |
| F1 | `soft-go-II/src/components/ParticipantsList.tsx:23` | list starts open | ✅ 22 |
| F2 | `soft-go-II/src/components/ParticipantsList.tsx:31` | "Sair da carona" trash on every name for any logged account | ✅ 9 |
| F3 | `soft-go-II/src/components/ParticipantsList.tsx:30` | owner label without the name | ✅ 3 |
| F4 | `soft-go-II/src/components/ParticipantsList.tsx:29` | ignore `canRemove` (trash in modal summary) | ✅ 3 |
| F5 | `soft-go-II/src/pages/Home.tsx:111` | success toast swapped (leave vs remove) | ✅ 2 |
| F6 | `soft-go-II/src/pages/Home.tsx:118` | no reload after `403`/`404` | ✅ 2 |
| F7 | `soft-go-II/src/components/ConfirmDialog.tsx:57` | confirm button not disabled while sending | ✅ 1 |
| F8 | `soft-go-II/src/service/ride.service.ts:45` | URL params swapped | ✅ 1 |
| F9 | `soft-go-II/src/pages/Home.tsx:111` | no reload after `204` | ✅ 2 |
| F10 | `soft-go-II/src/components/ParticipantsList.tsx:39` | `aria-expanded` stuck at `false` | ✅ 1 |
| F11 | `soft-go-II/src/pages/Home.tsx:198` | leave dialog shows the owner title | ✅ 1 |
| F12 | `soft-go-II/src/components/ParticipantsList.tsx:25` | toggle shown with zero participants | ✅ 2 |
| F13 | `soft-go-II/src/components/ParticipantsList.tsx:56` | `wa.me` link shown when logged out | ✅ 3 |
| F14 | `soft-go-II/src/pages/Home.tsx:118` | `403`/`404` toast ignores the API message | ✅ 2 |
| F15 | `soft-go-II/src/pages/Home.tsx:115` | Home shows a toast on `401` (remove flow) | ✅ 1 |
| F16 | `soft-go-II/src/pages/Home.tsx:123` | modal not closed after the result | ✅ 6 |
| F17 | `soft-go-II/src/components/ParticipantsList.tsx:43` | wrong count in "Ver passageiros (N)" | ✅ 4 |
| F18 | `soft-go-II/src/pages/Home.tsx:199` | confirm labels swapped ("Sair"/"Remover") | ✅ 6 |

**Sensor depth**: P0 (authorization + data deletion), 30 manual behavior-level mutations covering every new branch.
**Sensor outcome**: 30/30 killed, 0 survived.
**Isolation**: junctions removed with `rmdir` before `git worktree remove --force` + `git worktree prune`; the real `node_modules` are intact. `git status --porcelain` of root (` M soft-go-ii-api`, `?? prds/`), API (` M tsconfig.build.tsbuildinfo`) and front (clean) was identical before and after (diffed). `git stash` was not used.

---

## Code Quality

| Principle | Status |
| --------- | ------ |
| Minimum code | ✅ (`remove` is 19 lines; `ParticipantsList` and `ConfirmDialog` carry only the props the design names) |
| Surgical changes | ✅ (only the files listed in `tasks.md`; `ConfirmDeleteRideModal` untouched as decided) |
| No scope creep | ✅ (no notifications; `GET /ride-users` unchanged) |
| Matches patterns | ✅ (ESM `.js` imports, `@HttpCode(NO_CONTENT)` + `@ApiResponse` like `removeRide`, `ParseIntPipe`, `Toast` helper, `support-04` token, PT-BR texts, `handleRemove` mirrors `handleDelete`) |
| Spec-anchored outcome check | ✅ |
| Per-layer Coverage Expectation | ✅ (service all branches; HTTP happy ×2 + 400×2 / 401×4 / 403 / 404×2 + persisted state + guard map + no global guard; UI each list element and each trash visibility rule) |
| Every test maps to a requirement | ✅ |
| Documented guidelines followed (`CLAUDE.md`, `soft-go-ii-api/CLAUDE.md`, L-001/004/005/006) | ✅ |

---

## Edge Cases

- [x] Passenger leaves a full ride → spot returned: derived (`soft-go-ii-api/src/ride/ride.service.spec.ts:119` full → `0`, `:100` count-based), row deleted (`soft-go-ii-api/src/ride-users/ride-users.controller.spec.ts:198`), "Vou junto" back after reload (`soft-go-II/src/pages/Home.test.tsx:427`). Real DB: author's manual check (0 → 1).
- [x] Bus ride (unlimited) → `spots` stays `null`: `soft-go-ii-api/src/ride/ride.service.spec.ts:128`; `remove` does not read `transportType` (`ride-users.service.ts:73`).
- [x] Enrollment already removed in another tab → API `404` "Esta pessoa não está nesta carona" (`soft-go-ii-api/src/ride-users/ride-users.controller.spec.ts:238-239`), front toast + reload (`soft-go-II/src/pages/Home.test.tsx:464,475-477`).
- [x] Owner removes herself via API → `404` "Esta pessoa não está nesta carona" (`soft-go-ii-api/src/ride-users/ride-users.service.spec.ts:264-272`).

---

## Gate Check (re-run by verifier)

| Gate | Command | Outcome |
| ---- | ------- | ------- |
| Build (API) | `npx tsc --noEmit -p tsconfig.build.json` | exit 0 |
| Build (API) | `npm run lint` | exit 0 (pre-existing warnings only, none in feature files) |
| Build (API) | `npm test` | **185 passed**, 0 failed, 12 files, exit 0 |
| Build (Front) | `npm run lint` | exit 0 (1 pre-existing warning `Home.tsx:52` `useCallback` deps, the same line as the old `:48`, shifted by the new imports and state) |
| Build (Front) | `npm run build` | exit 0 |
| Build (Front) | `npm test` | **206 passed**, 0 failed, exit 0 |

- **Test count before feature**: API 167 · front 174
- **Test count after feature**: API 185 (+18) · front 206 (+32)
- **Skipped tests**: none
- **Failures**: none

---

## Fix Plans

None.

---

## Requirement Traceability Update

| Requirement | Previous Status | New Status |
| ----------- | --------------- | ---------- |
| LEAVE-01..LEAVE-21 | Implemented | ✅ Verified |

---

## Summary

**Overall**: ✅ Ready

**Spec-anchored check**: 21/21 ACs matched spec outcome, 0 spec-precision gaps
**Sensor**: 30/30 mutations killed
**Gate**: API 185 passed / exit 0; front 206 passed / exit 0

**What works**: `DELETE /ride-users/:rideId/users/:userId` with the 401 → 400 → 404 ride → 403 → 404 enrollment → 204 order and exact PT-BR messages; only the passenger or the owner can remove, and the row deleted is the enrollment itself; guard map and "no global guard" hold. The front list starts collapsed with the correct count and `aria-expanded`; the trash appears only on the user's own name, or on every name for the owner, and never in modal summaries. The confirm dialog uses the right title and button per case, disables the button while sending, and handles 204/403/404/other/401 as specified.

**Issues found**: none. LEAVE-03 is covered by composition plus the author's manual DB check (observation 1).

**Next steps**: orchestrator updates `spec.md` traceability (LEAVE-01..21 → Verified) and commits it with the root bump.
