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

### AD-006
- **Decision**: Só a dona exclui a carona (`DELETE /ride/:id`: `403` para outra conta, `404` se não existe). A exclusão apaga as inscrições em cascata (`ON DELETE CASCADE` de `FK_ride_users_ride`) e nunca é bloqueada por haver participantes. Antes de excluir, o front mostra quantas pessoas confirmaram e quem não tem WhatsApp para ser avisada.
- **Reason**: Uma carona que não vai acontecer precisa sumir do mural. Bloquear deixaria as inscritas achando que têm carona, e a dona sem saída, porque ninguém consegue sair de uma carona hoje.
- **Trade-off**: Exclusão definitiva, sem desfazer. Participante sem telefone não recebe aviso ativo: descobre pelo mural. O aviso ativo é o card `prds/notify-ride-deleted.md`.
- **Scope**: `soft-go-ii-api` (`RideService.remove`, `RideController`) e `soft-go-II` (`Card`, `ConfirmDeleteRideModal`, `Home`).
- **Date**: 2026-10-08
- **Status**: active

## Handoff

- **Feature**: delete-rides (`.specs/features/delete-rides/`): spec, context e tasks escritos (Draft). auth, vou-junto, save-rideowner e revoke-token concluídas
- **Phase / Task**: Execute: T1–T2 concluídas (Phase 1 API), próxima T3 (tasks aprovadas em 2026-10-08)
- **Completed**: nada implementado. API: 153 testes; front: 145 testes
- **In-progress** (file:line): none
- **Next step**: Aprovar `tasks.md` e executar T1. Cards futuros criados, nesta ordem: `prds/notify-ride-deleted.md` (cria o canal de aviso) → `prds/leave-ride.md` (reusa o canal para avisar a dona)
- **Blockers**: none
- **Uncommitted files**: `soft-go-ii-api/tsconfig.build.tsbuildinfo` (não commitar); `prds/` na raiz (da usuária)
- **Branch**: raiz `main`; API `master`; front `main`
- **Deploy note**: API e front sobem juntos. Depois do deploy, todos os tokens antigos (sem `ver`) param de valer e todas as usuárias entram de novo uma vez. A migration só adiciona `users.token_version` (não apaga dados). Atenção: a migration `AddOwnerToRides` (save-rideowner) apaga todas as caronas no banco em que ainda não rodou
