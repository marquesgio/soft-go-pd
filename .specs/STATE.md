# STATE

## Decisions

### AD-001
- **Decision**: Autenticação por JWT Bearer emitido com `@nestjs/jwt` (sem Passport) e verificado por `JwtAuthGuard` aplicado por rota (`@UseGuards`), nunca global; no front o token fica em `localStorage` e o logout é só no cliente.
- **Reason**: Menos dependências e código explícito para um único fluxo e-mail/senha; rotas existentes continuam abertas; usuária aceitou os riscos de XSS/sem revogação nesta fase.
- **Trade-off**: Sem revogação de token e exposição a XSS; login social/SSO exigirá adotar Passport ou nova estratégia. Revisitar (cookie httpOnly e/ou revogação) quando rotas sensíveis passarem a exigir auth.
- **Scope**: `soft-go-ii-api` (todas as rotas protegidas futuras) e `soft-go-II` (`service/session.ts`, `service/api.ts`, `AuthProvider`).
- **Date**: 2026-10-07
- **Status**: active

### AD-002
- **Decision**: O front (`soft-go-II`) passa a ter testes com vitest + jsdom + Testing Library (`npm test`), co-localizados como `*.test.ts(x)` ao lado do código.
- **Reason**: Gate automatizado por tarefa; antes o front só tinha lint/build.
- **Trade-off**: Novas devDependencies e manutenção de testes de componente.
- **Scope**: `soft-go-II`.
- **Date**: 2026-10-07
- **Status**: active

## Handoff

- **Feature**: auth (`.specs/features/auth/`)
- **Phase / Task**: Phase 3 concluída (T10–T14); próxima: Phase 4 / T15
- **Completed**: T1–T14 (API: 44 testes; front: 33 testes)
- **In-progress** (file:line): none
- **Next step**: Iniciar T15 (`AuthProvider`/`useAuth` em `soft-go-II/src/context/AuthContext.tsx` + `App.tsx`). Execução inline, pausando ao fim de cada fase; commits direto em `master`/`main`. Depois de T20, rodar o Verifier.
- **Blockers**: none
- **Uncommitted files**: `soft-go-ii-api/tsconfig.build.tsbuildinfo` (não commitar); `prds/` na raiz (da usuária, fora do fluxo)
- **Branch**: raiz `main`; API `master`; front `main`
- **Gate notes**: API build gate = `tsc -p tsconfig.build.json` + `tsc -p tsconfig.json` (tolera só `test/app.e2e-spec.ts(4,21) TS2307`) + lint + `npm test`. Front build gate = `npm run lint && npm run build && npm test`.
