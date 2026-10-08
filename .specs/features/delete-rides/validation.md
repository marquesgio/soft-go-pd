# Excluir Carona (delete-rides) — Validation Report

## Validation: delete-rides - PASS ✅

**Date**: 2026-10-08
**Spec**: `.specs/features/delete-rides/spec.md` (DEL-01..DEL-16)
**Diff range**: API `soft-go-ii-api` `cb3fc00..541c0ca` (2 commits: 062bb90, 541c0ca) · Front `soft-go-II` `a3e048b..3f74f33` (4 commits: efbab79, feb946a, 429d3f0, 3f74f33)
**Iteration**: 1 of 3
**Verifier**: independent sub-agent (author ≠ verifier)

---

## Task Completion

| Task | Status | Notes |
| ---- | ------ | ----- |
| T1 | ✅ Done | `062bb90` `RideService.remove` + cascade metadata test |
| T2 | ✅ Done | `541c0ca` `DELETE /ride/:id` (guard, `ParseIntPipe`, `204`, Swagger) + guard map. Manual local-DB cascade check reported by the author (`ride_users` 1 → 0); not re-run by the verifier, per instructions |
| T3 | ✅ Done | `efbab79` `deleteRide(id)` |
| T4 | ✅ Done | `feb946a` "Excluir carona" button + `support-04` token |
| T5 | ✅ Done | `429d3f0` `ConfirmDeleteRideModal` |
| T6 | ✅ Done | `3f74f33` Home wiring + error handling |

---

## Spec-Anchored Acceptance Criteria

API paths are relative to `soft-go-ii-api/`, front paths to `soft-go-II/`.

| AC | Spec-defined outcome | `file:line` + assertion | Result |
| -- | -------------------- | ----------------------- | ------ |
| DEL-01 | owner token → ride deleted, `204`, empty body | `src/ride/ride.controller.spec.ts:293` `expect(res.status).toBe(204)`; `:294` `expect(res.text).toBe('')`; `:295` `expect(storedRides).toEqual([])`; unit `src/ride/ride.service.spec.ts:321` `resolves.toBeUndefined()`, `:324` `delete` called with `3` | ✅ PASS |
| DEL-02 | `ride_users` rows removed with the ride; `GET /ride` no longer lists it | cascade: `src/ride-users/entities/ride-user.entity.spec.ts:57` `foreignKeyConstraintName` `toBe('FK_ride_users_ride')`, `:58` `relation?.options.onDelete` `toBe('CASCADE')`; participants don't block: `src/ride/ride.service.spec.ts:332` `delete` called with `3` for a ride with 2 participants; listing: `src/ride/ride.controller.spec.ts:299` `expect(list.body).toEqual([])`. Real-DB cascade: author's manual check (count 1 → 0) | ✅ PASS (metadata + HTTP + author's DB check) |
| DEL-03 | missing ride → `404` "Carona não encontrada" | `src/ride/ride.controller.spec.ts:316` `toBe(404)`, `:317` `res.body.message` `toBe('Carona não encontrada')`, `:318` nothing deleted; unit `src/ride/ride.service.spec.ts:340-342` `NotFoundException`, exact message, `delete` not called | ✅ PASS |
| DEL-04 | other account → `403` "Só a dona da carona pode excluí-la"; ride and its joins stay | `src/ride/ride.controller.spec.ts:305` `toBe(403)`, `:306` exact message, `:307` `storedRides` ids `[3]` (the stored ride object carries its `rideUser`), `:310` `GET /ride` still lists `[3]`; unit `src/ride/ride.service.spec.ts:350-352` `ForbiddenException`, exact message, `delete` not called | ✅ PASS |
| DEL-05 | no `Authorization` / invalid / expired / revoked → `401` "Não autenticado", nothing deleted | `src/ride/ride.controller.spec.ts:331-333` (no header), `:345-347` (other secret), `:360-362` (expired), `:371-373` (old `ver`): each `toBe(401)` + `message` `toBe('Não autenticado')` + `storedRides` ids `[3]` | ✅ PASS |
| DEL-06 | non-integer `:id` → `400`, nothing deleted | `src/ride/ride.controller.spec.ts:324` `toBe(400)`; `:325` `storedRides` ids `[3]` | ✅ PASS |
| DEL-07 | `GET /ride` and `GET /transport-type` still open; no global guard | `src/ride/ride.controller.spec.ts:379` `GET /ride` without token `toBe(200)`; `src/auth/auth.controller.spec.ts:453` `[TransportTypeController, {}]` (no guard on any handler, asserted by the it.each), `:449` `removeRide: [JwtAuthGuard]`; `:502` `expect(globalGuards).toEqual([])` | ✅ PASS (`/transport-type` structural via guard metadata) |
| DEL-08 | owner sees "Excluir carona" next to "Sua carona" | `src/components/Card.test.tsx:167` `getByText("Sua carona")` + `:168-170` click on `button "Excluir carona"` → `onDeleteRide` `toHaveBeenCalledTimes(1)`; bus `:180`, full ride `:190` `toBeInTheDocument()`; page level `src/pages/Home.test.tsx:283` `findByRole("button", { name: "Excluir carona" })` for the logged owner | ✅ PASS |
| DEL-09 | logged out or non-owner → no button | `src/components/Card.test.tsx:196` (other user) and `:202` (logged out) `not.toBeInTheDocument()`; `:208` `showButton={false}`; `src/pages/Home.test.tsx:301` other account `not.toBeInTheDocument()` | ✅ PASS |
| DEL-10 | modal: title "Excluir carona?", owner, city, hour, transport, "Cancelar" and "Excluir" | `src/components/ConfirmDeleteRideModal.test.tsx:40` heading `"Excluir carona?"`; `:41` `"Carla Dona Silva"`; `:42` `/Porto Alegre/`; `:43` `"08:00"`; `:44` `"Carro"`; `:45-46` buttons `Cancelar`/`Excluir`; opened from Home `src/pages/Home.test.tsx:291` | ✅ PASS |
| DEL-11 | 1 → "1 pessoa confirmou presença e será removida da carona."; N → "N pessoas confirmaram presença e serão removidas da carona."; 0 → "Ninguém confirmou presença ainda." | `src/components/ConfirmDeleteRideModal.test.tsx:61` exact singular; `:70` exact plural (3); `:53` exact zero text | ✅ PASS |
| DEL-12 | "Sem WhatsApp para avisar: <names, comma-separated>" only for participants without phone; hidden when all have phone | `src/components/ConfirmDeleteRideModal.test.tsx:77` `getByText("Sem WhatsApp para avisar: Bia, Caio")` (Ana with phone excluded); `:83` all with phone → `not.toBeInTheDocument()`; `:54` none → hidden | ✅ PASS |
| DEL-13 | "Cancelar", "X" or outside → close without API call | `src/components/ConfirmDeleteRideModal.test.tsx:93-94` (Cancelar), `:102-103` (X), `:111-112` (overlay): `onClose` once, `onConfirm` not called; `:120` inside click doesn't close; page level `src/pages/Home.test.tsx:309-310` dialog gone + `deleteRide` `not.toHaveBeenCalled()` | ✅ PASS |
| DEL-14 | `DELETE /ride/<id>` with token; button disabled while sending; on `204` toast "Carona excluída", modal closes, board reloads without the ride | service `src/service/ride.service.test.ts:91-92` `api.delete` once with `"/ride/7"` (token added by the existing interceptor, `src/service/api.test.ts:42`); disabled: `src/components/ConfirmDeleteRideModal.test.tsx:142` `toBeDisabled()`, `:145` `onConfirm` once after second click; Home `src/pages/Home.test.tsx:318` `deleteRide` with `3`, `:319` `Toast("success", "Carona excluída")`, `:320` dialog gone, `:321` `getRides` called again, `:323` ride no longer shown | ✅ PASS |
| DEL-15 | `403`/`404` → error toast with the API message, close, reload | `src/pages/Home.test.tsx:336` `Toast("error", "Só a dona da carona pode excluí-la")`, `:338` dialog gone, `:339` reload; `:350` `Toast("error", "Carona não encontrada")`, `:352` dialog gone, `:353` reload | ✅ PASS |
| DEL-16 | other error → toast "Não foi possível excluir a carona. Tente novamente.", ride kept; `401` → left to the interceptor | `src/pages/Home.test.tsx:363-366` exact generic toast, `:369` "Sua carona" still on the board; `401`: `:380` `expect(Toast).not.toHaveBeenCalled()` (interceptor behaviour anchored by existing `src/service/api.test.ts:85`) | ✅ PASS |

**Status**: ✅ 16/16 ACs covered with spec-anchored assertions; 0 spec-precision gaps.

Minor observations (not gaps):
- DEL-02 is proved by entity metadata + fake repository; the real `ON DELETE CASCADE` in Postgres rests on the author's manual DB check (verifier did not touch the DB, per instructions).
- DEL-07 `GET /transport-type` is asserted structurally (no guard metadata), not with an HTTP call.
- DEL-16: on a `500` the modal also closes (`src/pages/Home.test.tsx:368`). The spec is silent on the modal for this case; `tasks.md` T6 specifies "fecha", so it matches the plan.

---

## Discrimination Sensor

Run in temporary `git worktree`s (`scratchpad/wt-api` at `541c0ca`, `scratchpad/wt-front` at `3f74f33`) with junctioned `node_modules`. Baseline in scratch: API 124/124 (`src/ride`, `src/ride-users`, `auth.controller.spec.ts`), front 110/110 (`src/components`, `Home.test.tsx`, `src/service`). Each mutant applied by a script, targeted tests run, file restored; final `git diff --ignore-cr-at-eol` of both worktrees empty.

| # | File:line | Mutation | Killed? (fails) |
| - | --------- | -------- | --------------- |
| A1 | `soft-go-ii-api/src/ride/ride.service.ts:155` | drop the owner check (anyone deletes) | ✅ 2 |
| A2 | `soft-go-ii-api/src/ride/ride.service.ts:152` | missing ride throws `403` instead of `404` | ✅ 2 |
| A3 | `soft-go-ii-api/src/ride/ride.service.ts:156` | wrong `403` message | ✅ 2 |
| A4 | `soft-go-ii-api/src/ride/ride.controller.ts:96` | remove `@HttpCode(204)` (DELETE answers 200) | ✅ 1 |
| A5 | `soft-go-ii-api/src/ride/ride.controller.ts:94` | `JwtAuthGuard` → `OptionalJwtAuthGuard` | ✅ 5 (401 tests + guard map) |
| A6 | `soft-go-ii-api/src/ride/ride.controller.ts:102` | remove `ParseIntPipe` | ✅ 1 |
| A7 | `soft-go-ii-api/src/ride/ride.service.ts:160` | remove `rideRepository.delete(id)` | ✅ 3 |
| A8 | `soft-go-ii-api/src/ride-users/entities/ride-user.entity.ts:18` | `onDelete: 'CASCADE'` → `'RESTRICT'` | ✅ 1 |
| A9 | `soft-go-ii-api/src/ride/ride.controller.ts:105` | ignore token identity (fixed userId = owner) | ✅ 1 |
| F1 | `soft-go-II/src/components/Card.tsx:139` | show the button to non-owners | ✅ 3 |
| F2 | `soft-go-II/src/components/Card.tsx:139` | ignore `showButton` (button inside the modal summary) | ✅ 1 |
| F3 | `soft-go-II/src/components/ConfirmDeleteRideModal.tsx:15` | drop singular branch ("1 pessoas confirmaram…") | ✅ 1 |
| F4 | `soft-go-II/src/components/ConfirmDeleteRideModal.tsx:25` | list participants WITH phone under "Sem WhatsApp" | ✅ 2 |
| F5 | `soft-go-II/src/components/ConfirmDeleteRideModal.tsx:81` | confirm button not disabled while sending | ✅ 1 |
| F6 | `soft-go-II/src/pages/Home.tsx:93` | no reload after `403`/`404` | ✅ 2 |
| F7 | `soft-go-II/src/pages/Home.tsx:92` | `403`/`404` toast ignores the API message | ✅ 2 |
| F8 | `soft-go-II/src/pages/Home.tsx:86` | no reload after `204` | ✅ 1 |
| F9 | `soft-go-II/src/pages/Home.tsx:90` | Home shows a toast on `401` | ✅ 1 |
| F10 | `soft-go-II/src/service/ride.service.ts:41` | URL `/rides/:id` | ✅ 1 |
| F11 | `soft-go-II/src/pages/Home.tsx:97-99` | modal not closed after the result | ✅ 5 |

**Sensor depth**: P0 (authorization + data deletion) — 20 manual behavior-level mutations covering every new branch.
**Sensor outcome**: 20/20 killed, 0 survived.
**Isolation**: junctions removed with `rmdir` before `git worktree remove --force` + `git worktree prune` (real `node_modules` intact). `git status --porcelain` of root (` M soft-go-ii-api`, `?? prds/`), API (` M tsconfig.build.tsbuildinfo`) and front (clean) identical before and after (diffed).

---

## Code Quality

| Principle | Status |
| --------- | ------ |
| Minimum code | ✅ (`remove` 14 lines, no code deleting `ride_users`; cascade left to the DB as decided) |
| Surgical changes | ✅ (only the files listed in tasks; one new token in `@theme`) |
| No scope creep | ✅ |
| Matches patterns | ✅ (ESM `.js` imports, `@HttpCode(NO_CONTENT)` like signout, Swagger `@ApiResponse`, `Toast` helper, Tailwind tokens, PT-BR texts) |
| Spec-anchored outcome check | ✅ |
| Per-layer Coverage Expectation | ✅ (service all branches; HTTP happy + 400/401×4/403/404 + persisted state + open GET + no global guard; UI each modal element + singular/plural) |
| Every test maps to a requirement | ✅ |
| Documented guidelines followed (`CLAUDE.md`, `soft-go-ii-api/CLAUDE.md`, L-001/002/004/005/006) | ✅ |

---

## Edge Cases

- [x] Bus ride (unlimited spots) → owner gets the button: `soft-go-II/src/components/Card.test.tsx:180`.
- [x] Double click on "Excluir" → single request: `soft-go-II/src/components/ConfirmDeleteRideModal.test.tsx:142-145` (disabled + `onConfirm` once).
- [x] Ride already deleted in another tab → API `404` (`soft-go-ii-api/src/ride/ride.controller.spec.ts:316-317`), front toast + reload (`soft-go-II/src/pages/Home.test.tsx:350-353`).
- [x] Other account sends a body → `403`, nothing deleted: handler declares no `@Body` (`soft-go-ii-api/src/ride/ride.controller.ts:101-104`), so `forbidNonWhitelisted` has nothing to validate; `403` path asserted at `soft-go-ii-api/src/ride/ride.controller.spec.ts:305-307`. Not exercised with a body (structural, not a gap).

---

## Gate Check (re-run by verifier)

| Gate | Command | Outcome |
| ---- | ------- | ------- |
| Build (API) | `npx tsc --noEmit -p tsconfig.build.json` | exit 0 |
| Build (API) | `npm run lint` | exit 0 (pre-existing warnings only, none in feature files) |
| Build (API) | `npm test` | **167 passed**, 0 failed, 12 files, exit 0 |
| Build (Front) | `npm run lint` | exit 0 (1 pre-existing warning `Home.tsx:48` `useCallback` deps, line untouched by the diff) |
| Build (Front) | `npm run build` | exit 0 |
| Build (Front) | `npm test` | **173 passed**, 0 failed, 20 files, exit 0 |

- **Test count before feature**: API 153 · front 145
- **Test count after feature**: API 167 (+14) · front 173 (+28)
- **Skipped tests**: none
- **Failures**: none

---

## Fix Plans

None.

---

## Requirement Traceability Update

| Requirement | Previous Status | New Status |
| ----------- | --------------- | ---------- |
| DEL-01..DEL-16 | Implemented | ✅ Verified |

---

## Summary

**Overall**: ✅ Ready

**Spec-anchored check**: 16/16 ACs matched spec outcome, 0 spec-precision gaps
**Sensor**: 20/20 mutations killed
**Gate**: API 167 passed / exit 0; front 173 passed / exit 0

**What works**: `DELETE /ride/:id` with the 401 → 400 → 404 → 403 → 204 order and exact PT-BR messages; joins removed by the existing FK cascade; open routes still open and no global guard; owner-only "Excluir carona" button; confirmation modal with exact singular/plural/zero texts and the "Sem WhatsApp" list; disabled button while sending; Home handles 204/403/404/other/401 as specified.

**Issues found**: none.

**Next steps**: orchestrator updates `spec.md` traceability (DEL-01..16 → Verified) and commits it with the root bump.
