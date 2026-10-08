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

## Handoff

- **Feature**: save-rideowner (`.specs/features/save-rideowner/`) — **concluída**. As features auth e vou-junto também estão concluídas
- **Phase / Task**: Execute concluído (T1–T10); Verifier PASS na iteração 1 (`validation.md`: 30/30 ACs, sensor 15/15)
- **Completed**: T1–T10. API: 121 testes; front: 136 testes. Migration `AddOwnerToRides` aplicada no banco local (caronas e inscrições antigas apagadas com consentimento)
- **In-progress** (file:line): none
- **Next step**: Nenhum na feature. Pendente da usuária: decidir quando publicar (`git push` da API, do front e depois da raiz; nada foi enviado ao GitHub)
- **Blockers**: none
- **Uncommitted files**: `soft-go-ii-api/tsconfig.build.tsbuildinfo` (não commitar); `prds/` na raiz (da usuária)
- **Branch**: raiz `main`; API `master`; front `main`
- **Deploy note**: `POST /ride` mudou de contrato (exige Bearer; sem `name`; `phone` grava na conta) e `GET /ride` troca `name`/`phone` por `owner`. A migration apaga todas as caronas. API e front precisam subir juntos
