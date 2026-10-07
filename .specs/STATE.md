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
- **Phase / Task**: Phase 1 concluída (T1–T4); próxima: Phase 2 / T5
- **Completed**: T1, T2, T3, T4
- **In-progress** (file:line): none
- **Next step**: Iniciar T5 (DTOs `SignUpDto`/`SignInDto` em `soft-go-ii-api/src/auth/dto/`). Execução inline, pausando ao fim de cada fase; commits direto em `master`/`main`.
- **Blockers**: none. Antes de subir a API com o `AuthModule` (T9), definir `JWT_SECRET` no `soft-go-ii-api/.env`.
- **Uncommitted files**: `soft-go-ii-api/tsconfig.build.tsbuildinfo` (não commitar); `prds/` na raiz (da usuária, fora do fluxo)
- **Branch**: raiz `main`; API `master`; front `main`
- **Gate notes**: build gate da API usa `tsc -p tsconfig.build.json` + `tsc -p tsconfig.json` tolerando só o erro antigo `test/app.e2e-spec.ts(4,21) TS2307` + lint + `npm test`.
