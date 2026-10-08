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

### AD-003
- **Decision**: Rotas públicas que personalizam a resposta para quem está logada usam `OptionalJwtAuthGuard`, aplicado por rota. Token válido preenche `request.user`; ausente ou inválido segue como anônima, sem 401. `POST /ride-users` passa a exigir `JwtAuthGuard`, o que substitui o AUTH-23 nessa rota (`GET /ride`, `POST /ride` e `GET /transport-type` seguem sem exigir token).
- **Reason**: O mural continua público, e dados pessoais (telefones de participantes) só aparecem para quem está logada. Inscrição precisa de dona para impedir duplicidade (feature vou-junto).
- **Trade-off**: Um token adulterado não é sinalizado nessas rotas: só o `/auth/me` do front derruba a sessão. É mais uma classe de guard para manter.
- **Scope**: `soft-go-ii-api` (`RideController` leituras, `RideUsersController.create`, futuras rotas públicas personalizadas).
- **Date**: 2026-10-07
- **Status**: active

### AD-004
- **Decision**: `ride_users` só liga carona e usuária (`ride_id`, `user_id`, `UNIQUE (ride_id, user_id)`). Nome e telefone de participante vêm de `users` (`users.phone` opcional). Inscrições sem conta não existem.
- **Reason**: Uma fonte só para dados pessoais. Decisão da usuária de apagar as inscrições antigas sem conta.
- **Trade-off**: Perda irreversível das inscrições antigas. Toda inscrição exige conta. Exibir participante exige join com `users`.
- **Scope**: `soft-go-ii-api` (`ride_users`, `users`, `RideService`, `RideUsersService`) e `soft-go-II` (tipos `Participant`, `User.phone`).
- **Date**: 2026-10-07
- **Status**: active

## Handoff

- **Feature**: vou-junto (`.specs/features/vou-junto/`). A feature auth está concluída
- **Phase / Task**: Phase 3 concluída (T9–T12); próxima: Phase 4 / T13
- **Completed**: T1–T12 (API 95 testes; front 81 testes) (migrations aplicadas no banco local: `users.phone` e `LinkRideUsersToUsers`, inscrições antigas apagadas com o "sim" da usuária)
- **In-progress** (file:line): none
- **Next step**: T13 (WhatsApp opcional no cadastro). Depois da T16, rodar o Verifier. Execução inline, pausando ao fim de cada fase; commits direto em `master`/`main`.
- **Blockers**: none
- **Uncommitted files**: `soft-go-ii-api/tsconfig.build.tsbuildinfo` (não commitar); `prds/` na raiz (da usuária)
- **Branch**: raiz `main`; API `master`; front `main`
- **Gate notes**: API build gate = `tsc -p tsconfig.build.json` + `tsc -p tsconfig.json` (tolera só `test/app.e2e-spec.ts(4,21) TS2307`) + lint + `npm test`. Front build gate = `npm run lint && npm run build && npm test`.
