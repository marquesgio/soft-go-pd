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

- **Feature**: auth (`.specs/features/auth/`) — **concluída**
- **Phase / Task**: Execute concluído (T1–T20); Verifier PASS na iteração 2 (`validation.md`, 34/34 ACs, sensor 15/15)
- **Completed**: T1–T20 + correções de teste do Verifier (API `06b51b4`, front `f1de23e`). API: 46 testes; front: 63 testes
- **In-progress** (file:line): none
- **Next step**: Nenhum na feature. Pendente da usuária: `git push --recurse-submodules=on-demand` quando quiser publicar (nada foi enviado ao GitHub). Próxima feature sugerida: proteger `POST /ride`/`POST /ride-users` e pré-preencher nome/telefone com o usuário logado.
- **Blockers**: none
- **Uncommitted files**: `soft-go-ii-api/tsconfig.build.tsbuildinfo` (não commitar); `prds/` na raiz (da usuária, fora do fluxo)
- **Branch**: raiz `main`; API `master`; front `main`
- **Known gap (não bloqueante)**: um `app.useGlobalGuards(...)` direto em `main.ts` não seria detectado por teste (só via `setupApp`/módulos).
