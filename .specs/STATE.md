# STATE

## Decisions

### AD-001
- **Decision**: Autenticação por JWT Bearer emitido com `@nestjs/jwt` (sem Passport) e verificado por `JwtAuthGuard` aplicado por rota (`@UseGuards`), nunca global; no front o token fica em `localStorage` e o logout é só no cliente.
- **Reason**: Menos dependências e código explícito para um único fluxo e-mail/senha; rotas existentes continuam abertas; usuária aceitou os riscos de XSS/sem revogação nesta fase.
- **Trade-off**: Sem revogação de token e exposição a XSS; login social/SSO exigirá adotar Passport ou nova estratégia. Revisitar (cookie httpOnly e/ou revogação) quando rotas sensíveis passarem a exigir auth.
- **Scope**: `soft-go-ii-api` (todas as rotas protegidas futuras) e `soft-go-II` (`service/session.ts`, `service/api.ts`, `AuthProvider`).
- **Date**: 2026-10-07
- **Status**: active (logout e consulta ao banco substituídos pela AD-005)

### AD-002
- **Decision**: O front (`soft-go-II`) passa a ter testes com vitest + jsdom + Testing Library (`npm test`), co-localizados como `*.test.ts(x)` ao lado do código.
- **Reason**: Gate automatizado por tarefa; antes o front só tinha lint/build.
- **Trade-off**: Novas devDependencies e manutenção de testes de componente.
- **Scope**: `soft-go-II`.
- **Date**: 2026-10-07
- **Status**: active

### AD-003
- **Decision**: Rotas públicas que personalizam a resposta para quem está logada usam `OptionalJwtAuthGuard`, aplicado por rota. Token válido preenche `request.user`; ausente ou inválido segue como anônima, sem 401. Toda rota que grava dados ligados a uma usuária exige `JwtAuthGuard`: `POST /ride-users` (vou-junto) e `POST /ride` (save-rideowner, a dona vem do token). Isso substitui o AUTH-23 nessas rotas. `GET /ride` e `GET /transport-type` seguem sem exigir token.
- **Reason**: O mural continua público, e dados pessoais (telefones de participantes) só aparecem para quem está logada. Inscrição precisa de dona para impedir duplicidade (feature vou-junto).
- **Trade-off**: Um token adulterado não é sinalizado nessas rotas: só o `/auth/me` do front derruba a sessão. É mais uma classe de guard para manter.
- **Scope**: `soft-go-ii-api` (`RideController` leituras e `createRide`, `RideUsersController.create`, futuras rotas públicas personalizadas).
- **Updated**: 2026-10-08 (save-rideowner: `POST /ride` passa a exigir token)
- **Date**: 2026-10-07
- **Status**: active

### AD-004
- **Decision**: `ride_users` só liga carona e usuária (`ride_id`, `user_id`, `UNIQUE (ride_id, user_id)`). Nome e telefone de participante vêm de `users` (`users.phone` opcional). Inscrições sem conta não existem.
- **Reason**: Uma fonte só para dados pessoais. Decisão da usuária de apagar as inscrições antigas sem conta.
- **Trade-off**: Perda irreversível das inscrições antigas. Toda inscrição exige conta. Exibir participante exige join com `users`.
- **Scope**: `soft-go-ii-api` (`ride_users`, `users`, `RideService`, `RideUsersService`) e `soft-go-II` (tipos `Participant`, `User.phone`).
- **Date**: 2026-10-07
- **Status**: active

### AD-005
- **Decision**: Toda conta tem `users.token_version`. O JWT carrega `ver`, e os dois guards (via `verifySession`) só aceitam o token se `ver` for igual à versão atual da conta. `POST /auth/signout` incrementa a versão: "Sair" encerra a sessão em todos os aparelhos. Substitui, na AD-001, "logout é só no cliente" e "o guard não consulta o banco".
- **Reason**: O token passou a liberar ações e dados pessoais (vou-junto, save-rideowner); uma cópia do token não pode continuar valendo depois de sair. Opção mais simples que revoga de verdade.
- **Trade-off**: Uma consulta a `users` por requisição com token. Não existe "sair só deste aparelho". Tokens emitidos antes da feature (sem `ver`) deixam de valer. O risco de XSS do `localStorage` continua (AD-001).
- **Scope**: `soft-go-ii-api` (`auth/`, todas as rotas com `JwtAuthGuard`/`OptionalJwtAuthGuard`) e `soft-go-II` (`AuthContext.signOut`). Futura troca de senha deve incrementar `token_version`.
- **Date**: 2026-10-08
- **Status**: active

## Handoff

- **Feature**: revoke-token (`.specs/features/revoke-token/`) — **concluída**. auth, vou-junto e save-rideowner também estão concluídas
- **Phase / Task**: Execute concluído (T1–T5); Verifier PASS na iteração 2 (`validation.md`: 16/16 ACs, sensor 17/17)
- **Completed**: T1–T5 + correção de mocks de teste (iteração 1). API: 153 testes; front: 145 testes. Migration `AddTokenVersionInUsersTable` aplicada no banco local (só adiciona coluna)
- **In-progress** (file:line): none
- **Next step**: Nenhum. API (`cb3fc00`), front (`a3e048b`) e raiz enviados ao GitHub em 2026-10-08
- **Blockers**: none
- **Uncommitted files**: `soft-go-ii-api/tsconfig.build.tsbuildinfo` (não commitar); `prds/` na raiz (da usuária)
- **Branch**: raiz `main`; API `master`; front `main`
- **Deploy note**: API e front sobem juntos. Depois do deploy, todos os tokens antigos (sem `ver`) param de valer e todas as usuárias entram de novo uma vez. A migration só adiciona `users.token_version` (não apaga dados). Atenção: a migration `AddOwnerToRides` (save-rideowner) apaga todas as caronas no banco em que ainda não rodou
