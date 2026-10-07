# Vou Junto com Login Design

**Spec**: `.specs/features/vou-junto/spec.md`
**Context**: `.specs/features/vou-junto/context.md`
**Status**: Approved (2026-10-07)

---

## Architecture Overview

**API.** `ride_users` vira a ligação entre carona e usuária: só guarda `ride_id` e `user_id`, sem nome nem telefone. O telefone vai para `users.phone`.
- `POST /ride-users` usa o `JwtAuthGuard` que já existe, recebe `{ rideId, phone? }` e pega a usuária do token.
- `GET /ride` e `GET /ride/:id` ganham o `OptionalJwtAuthGuard`, que é novo. Ele lê o token se houver e nunca responde 401. Com ele, o service sabe se quem pede está logada e decide se inclui telefones.

**Front.**
- O `Home` decide o que fazer no clique: quem está deslogada vai para `/login`, quem está logada abre o modal.
- O `Modal` vira uma confirmação. Ele só mostra o campo de WhatsApp quando `user.phone` é `null`.
- O `Card` mostra os participantes e troca o botão por "Você vai nesta carona" quando a usuária já está na lista.
- O `AuthContext` ganha `updateUser`, para refletir o telefone salvo na sessão.

```mermaid
graph TD
  subgraph Front [soft-go-II]
    H[pages/Home] -->|deslogada| LG[/login]
    H -->|logada| M[Modal<br/>campo WhatsApp só se user.phone == null]
    M --> RS[ride.service.createRideUser<br/>rideId, phone?]
    H --> C[Card<br/>participantes + 'Você vai nesta carona']
    H --> AC[useAuth: user, updateUser]
    SU[pages/SignUp + WhatsApp opcional] --> AC
  end
  subgraph Back [soft-go-ii-api]
    RC[RideController<br/>GET /ride, GET /ride/:id] -. OptionalJwtAuthGuard .-> RSV[RideService.findAllRides<br/>viewer?]
    RUC[RideUsersController<br/>POST /ride-users 🔒] -. JwtAuthGuard .-> RUS[RideUsersService.join]
    RUS --> UR[UsersRepository<br/>update phone]
    RUS --> RUR[RideUsersRepository]
    AS[AuthService.signUp/toPublicUser + phone]
    RSV --> DB[(rides, ride_users, users)]
    RUR --> DB
  end
  RS -- Bearer --> RUC
  H -- Bearer opcional --> RC
```

### Approaches considered — leitura opcional do token no `GET /ride`

| | Abordagem | Prós | Contras |
|---|---|---|---|
| **Recomendada** | `OptionalJwtAuthGuard` (~20 linhas, mesma lógica do `JwtAuthGuard`), aplicado com `@UseGuards` nas rotas de leitura de `ride`. Token válido preenche `request.user`. Sem token, ou com token inválido, segue sem usuária | Explícito por rota, como pede o AD-001. Reaproveita `JwtService`. Fácil de testar isolado | Mais uma classe de guard |
| Alternativa | O controller lê o header `Authorization` e chama `jwtService.verifyAsync` direto | Sem classe nova | Duplica a lógica de verificar Bearer. Mistura auth com o controller |
| Alternativa | Separar em `GET /ride` (público, sem telefones) e `GET /ride/participants` (protegido) | Semântica de auth "pura" | Duas chamadas no front. Contrato novo para algo que é o mesmo recurso |

### Approaches considered — salvar o telefone do modal

| | Abordagem | Prós | Contras |
|---|---|---|---|
| **Recomendada** | `POST /ride-users` aceita `phone` opcional. O service valida carona, vaga e duplicidade, **depois** grava `users.phone` e **por último** cria a inscrição | Uma chamada. Se o telefone falhar, não há inscrição | Num conflito concorrente (23505) depois do update, o telefone fica salvo sem inscrição. Isso é inofensivo, porque a usuária quis salvar o número |
| Alternativa | Endpoint `PATCH /auth/me` para o telefone, chamado antes do `POST /ride-users` | Separa perfil e inscrição. Reaproveitável numa tela de perfil futura | Duas chamadas. Estado parcial no front se a segunda falhar. Endpoint de perfil está fora do escopo |

---

## Code Reuse Analysis

### Existing Components to Leverage

| Component | Location | How to Use |
| --------- | -------- | ---------- |
| `JwtAuthGuard` | `soft-go-ii-api/src/auth/jwt-auth.guard.ts` | Protege `POST /ride-users`. O `OptionalJwtAuthGuard` copia a extração do Bearer |
| `AuthModule` (exporta `JwtModule` + `JwtAuthGuard`) | `soft-go-ii-api/src/auth/auth.module.ts` | `RideModule` e `RideUsersModule` importam para usar os guards |
| `UsersModule` / `UsersRepository` | `soft-go-ii-api/src/users/` | `RideUsersService` lê o usuário e atualiza `phone` |
| Validação de telefone BR | `soft-go-ii-api/src/ride/dto/create-ride.dto.ts:47-54` | Mesmos decorators (`Transform` vazio→`undefined`, `IsPhoneNumber('BR')`, `Matches`) em `SignUpDto` e `CreateRideUserDto` |
| Padrão 23505 → 409 | `soft-go-ii-api/src/auth/auth.service.ts` | Mesma captura para `UQ_ride_user` |
| Testes HTTP com módulo real + fakes | `soft-go-ii-api/src/auth/auth.controller.spec.ts` | Mesmo formato (`overrideModule`, `setupApp`) para `ride-users` e `ride` |
| `Toast`, `InputForm` (`error`), zod + RHF | `soft-go-II/src/components`, `pages/SignUp.tsx` | Modal e campo WhatsApp do cadastro |
| `useAuth`, `session.ts`, interceptors | `soft-go-II/src/context`, `src/service` | Token vai sozinho. 401 no confirmar segue o fluxo de sessão expirada (AUTH-26) |
| `firstName` | `soft-go-II/src/utils/firstName.ts` | Não é usado na API. A API já devolve o primeiro nome |
| `httpError` (teste) | `soft-go-II/src/test/http-error.ts` | Testes do Home e do Modal |

### Integration Points

| System | Integration Method |
| ------ | ------------------ |
| Banco | 2 migrations: `AddColumnPhoneInUsersTable` e `LinkRideUsersToUsers` (apaga inscrições, troca `name`/`phone` por `user_id`) |
| `GET /ride` | Mesmo contrato. Cada carona ganha `participants`. Resposta sem `rideUser` (como hoje) |
| `POST /ride-users` | **Contrato quebrado de propósito**: body `{ rideId, phone? }` em vez de `{ rideId, name, phone? }`, e exige Bearer |
| Auth | `user` nas respostas de auth ganha `phone`. `SignUpDto` aceita `phone` |
| Teste AUTH-23 | Passa a esperar guard só em `RideUsersController.create` e o `OptionalJwtAuthGuard` nas leituras de `RideController` (mudança registrada no AD-003) |

---

## Components

### API

#### Migration `AddColumnPhoneInUsersTable`
- `up`: `ALTER TABLE "users" ADD "phone" character varying(15)`. `down`: `DROP COLUMN "phone"`.

#### Migration `LinkRideUsersToUsers`
- `up`, em ordem:
  1. `DELETE FROM "ride_users"`. Nenhuma linha tem conta, porque a coluna ainda nem existe (JOIN-28).
  2. `DROP CONSTRAINT "UQ_ride_phone"`, `DROP COLUMN "name"`, `DROP COLUMN "phone"`.
  3. `ADD "user_id" integer NOT NULL`.
  4. `ADD CONSTRAINT "FK_ride_users_user" FOREIGN KEY ("user_id") REFERENCES "users"("id")`.
  5. `ADD CONSTRAINT "UQ_ride_user" UNIQUE ("ride_id", "user_id")`.
- `down`: `DELETE FROM "ride_users"`, remove a UQ, a FK e a coluna, e recria `name VARCHAR(100) NOT NULL`, `phone VARCHAR(15)` e `UQ_ride_phone`. Só a estrutura volta, os dados não.

#### `User` entity (alteração)
- `phone: string | null`. Coluna `varchar(15)`, `nullable: true`. Ela é carregada por padrão (só `passwordHash` tem `select: false`).

#### `RideUser` entity (alteração)
- Remove `name` e `phone`. Adiciona `userId: number` (`name: 'user_id'`) e `@ManyToOne('User') @JoinColumn({ name: 'user_id', foreignKeyConstraintName: 'FK_ride_users_user' }) user`.
- `@Unique('UQ_ride_user', ['rideId', 'userId'])` substitui `UQ_ride_phone`.
- O `migration:generate --dryrun` precisa continuar sem diferenças (lição L-002: cobrir as opções da coluna).

#### `OptionalJwtAuthGuard` — `src/auth/optional-jwt-auth.guard.ts`
- `canActivate` sempre retorna `true`.
- Com Bearer válido, preenche `request.user = { id, email }`. Sem token, ou com token malformado, assinado com outro segredo ou expirado, deixa `request.user` indefinido.
- É exportado pelo `AuthModule`.

#### `SignUpDto` / `AuthService` (alteração)
- `SignUpDto.phone?` usa os decorators de telefone do `CreateRideDto`.
- `signUp` salva `phone ?? null`.
- `PublicUser` e `toPublicUser` ganham `phone`, que entra em signup, signin e me (JOIN-13/14/15).

#### `CreateRideUserDto` (reescrito)
- `rideId` (`@IsInt`) e `phone?` (decorators BR). Sem `name`, então um `name` no body gera 400 por `forbidNonWhitelisted` (JOIN-04).

#### `RideUsersService.join(userId, dto)` (substitui `create`)
1. Busca a carona com `transportType`. Se não existir, `NotFoundException('Corrida não encontrada')` (JOIN-26).
2. `existsBy({ rideId, userId })`. Se já existir, `ConflictException('Você já confirmou presença nesta carona')` (JOIN-10).
3. Conta as inscrições contra `transportType.spots`. Se estiver lotada, a `ConflictException` de vagas atual (JOIN-06).
4. Busca a usuária. Se `dto.phone` vier, `usersRepository.update(userId, { phone })` (JOIN-17).
5. `save({ rideId, userId })`. O erro `23505` vira o mesmo 409 de duplicidade (JOIN-11).
6. Retorna `{ id, rideId, userId }`.
- `findAll`/`findOne` continuam, agora sem nome e telefone.

#### `RideUsersController` (alteração)
- `POST` com `@UseGuards(JwtAuthGuard)` e `@ApiBearerAuth()`. Chama `join(request.user.id, dto)`. `@ApiBody` atualizado: `{ rideId, phone? }`.

#### `RideService.findAllRides(filters, viewer?)` (alteração)
- Carrega `relations: { transportType: true, rideUser: { user: true } }`.
- Para cada inscrição, monta `{ userId, name: primeiro nome de user.name }`. Se `viewer` existir, inclui também `phone: user.phone` (JOIN-20/21/22).
- O cálculo de vagas não muda (JOIN-25).
- `findOneRide` repassa o `viewer`.

#### `RideController` (alteração)
- `GET /` e `GET /:id` com `@UseGuards(OptionalJwtAuthGuard)`. Passam `request.user` como `viewer`.

### Front

#### `types/index.ts`
- `User.phone: string | null`. `SignUpPayload.phone?: string`.
- `Participant { userId: number; name: string; phone?: string | null }`. `Ride.participants: Participant[]`.
- `JoinRidePayload { rideId: number; phone?: string }` substitui `RideUser`.

#### `service/session.ts`
- `isSession` **não** exige `phone`. Sessões salvas antes desta feature continuam válidas, com `phone` ausente tratado como `null`.

#### `context/AuthContext.tsx`
- `updateUser(user)` grava em `saveSession({ ...session, user })` e no estado. O `getMe()` do mount também passa a usar `updateUser`, para refletir o telefone novo.

#### `schemas/phone.schema.ts`
- `optionalPhone`: string vazia, ou exatamente 11 dígitos com a mensagem "Número inválido, informe 11 dígitos (DDD + número)" (JOIN-18). É usado no `signUpSchema` (`phone`) e no schema do modal.

#### `pages/SignUp.tsx`
- Campo "WhatsApp (opcional)". O payload só inclui `phone` quando ele está preenchido. O `auth.service.signUp` repassa `phone` se existir.

#### `components/Modal.tsx` (reescrito)
- Props: `ride`, `onClose`, `onConfirm(phone?: string)`, `askPhone: boolean`.
- RHF + zod. O campo "WhatsApp" aparece só com `askPhone` (JOIN-01/16). Botão "Confirmar presença".

#### `pages/Home.tsx` (alteração)
- `handleJoin(ride)`: sem `user`, `navigate('/login')` (JOIN-08). Com `user`, abre o modal.
- `handleConfirm(phone?)`:
  1. `createRideUser({ rideId, ...(phone && { phone }) })`.
  2. Com sucesso: toast "Presença confirmada!" e, se veio `phone`, `updateUser({ ...user, phone })`. Depois `loadRides()` (JOIN-05/17).
  3. Num 409 ou 404, toast com `response.data.message` (JOIN-06/10). Num 401, nada além do fluxo do interceptor (JOIN-27). Em outros erros, toast genérico.
  4. Fecha o modal em todos os casos.
- `loadRides` depende de `user?.id`, para recarregar ao entrar e ao sair e mostrar ou esconder telefones.
- O schema zod antigo (nome e telefone) sai do Home.

#### `components/Card.tsx` (alteração)
- Props novas: `participants`, `currentUserId?: number`, `showPhones: boolean`.
- Mostra "Vão: Ana, Bia". Com `showPhones`, quem tem `phone` vira link `https://wa.me/<phone>` (JOIN-23/24).
- Se `participants` tem `currentUserId`, mostra "Você vai nesta carona" no lugar do botão (JOIN-12).

#### `service/ride.service.ts`
- `createRideUser(payload: JoinRidePayload)`.

---

## Data Models

```sql
-- AddColumnPhoneInUsersTable
ALTER TABLE "users" ADD "phone" character varying(15);

-- LinkRideUsersToUsers
DELETE FROM "ride_users";
ALTER TABLE "ride_users" DROP CONSTRAINT "UQ_ride_phone";
ALTER TABLE "ride_users" DROP COLUMN "name";
ALTER TABLE "ride_users" DROP COLUMN "phone";
ALTER TABLE "ride_users" ADD "user_id" integer NOT NULL;
ALTER TABLE "ride_users" ADD CONSTRAINT "FK_ride_users_user" FOREIGN KEY ("user_id") REFERENCES "users"("id");
ALTER TABLE "ride_users" ADD CONSTRAINT "UQ_ride_user" UNIQUE ("ride_id", "user_id");
```

```typescript
// GET /ride (cada carona)
interface RideResponse { /* campos atuais */ participants: Participant[] }
interface Participant { userId: number; name: string /* primeiro nome */; phone?: string | null /* só com token válido */ }

// POST /ride-users
interface JoinRideBody { rideId: number; phone?: string }
interface JoinRideResponse { id: number; rideId: number; userId: number }

// PublicUser (auth)
interface PublicUser { id: number; name: string; email: string; phone: string | null }
```

**Relationships**: `ride_users.ride_id → rides.id` (CASCADE, já existe); `ride_users.user_id → users.id` (novo).

---

## Error Handling Strategy

| Error Scenario | Handling | User Impact |
| -------------- | -------- | ----------- |
| Confirmar sem token, ou com token inválido ou expirado | `JwtAuthGuard` → 401 | Interceptor: sessão expirada, toast e `/login` |
| Já inscrita | Pré-checagem ou 23505 → 409 "Você já confirmou presença nesta carona" | Toast com a mensagem (o card normalmente já esconde o botão) |
| Carona lotada | 409 atual | Toast com a mensagem da API |
| Carona não existe | 404 "Corrida não encontrada" | Toast com a mensagem |
| Telefone inválido | Front bloqueia (zod); a API responde 400 se escapar | Mensagem abaixo do campo |
| `name` no body | 400 (`forbidNonWhitelisted`) | Não ocorre pelo front |
| Token inválido no `GET /ride` | Optional guard ignora | Lista sem telefones |

---

## Risks & Concerns

| Concern | Location (file:line) | Impact | Mitigation |
| ------- | -------------------- | ------ | ---------- |
| Migration apaga dados de verdade (35 inscrições no banco local) | `LinkRideUsersToUsers` (nova) | Perda irreversível | Decisão da usuária. A tarefa da migration pede confirmação antes de `migrations:run` no banco local, e o teste completo roda antes num banco temporário |
| Front e API em produção desalinhados: o front antigo envia `name` para a API nova e recebe 400 | `soft-go-II/src/pages/Home.tsx:56-61` | Inscrição quebra entre deploys | Subir API e front juntos. Sem deploy hoje (só local) |
| Checagem de vaga não é atômica (contar e depois inserir) | `soft-go-ii-api/src/ride-users/ride-users.service.ts:28-36` | Corrida rara pode passar do limite | Dívida que já existia. Fora do escopo |
| A vaga usa `transportType.spots` e ignora `rides.spots_ride` | `ride-users.service.ts:32`, `ride.service.ts:40-45` | Inconsistência que já existia entre o card e a regra | Fora do escopo; registrar como bug separado |
| `GET /ride/:id` filtra por **tipo de transporte**, não por carona, e reusa `findAllRides` | `soft-go-ii-api/src/ride/ride.controller.ts:63-66` | Também devolve participantes e precisaria esconder telefones | Aplicar o mesmo `OptionalJwtAuthGuard` e `viewer` |
| `Home` usa `FormData` e um schema local, e o Modal tem `id=""` nos inputs (label sem vínculo) | `soft-go-II/src/pages/Home.tsx:18-30`, `components/Modal.tsx:58-71` | Acessibilidade e testes difíceis | Modal reescrito com RHF, `id` reais e `InputForm` com `error` |
| Toast de erro do Home fala em "cadastrar a corrida" | `soft-go-II/src/pages/Home.tsx:84` | Mensagem errada | Mensagens novas e explícitas |
| Sessões antigas sem `phone` no `localStorage` | `soft-go-II/src/service/session.ts` | Sessão tratada como inválida se `isSession` exigir `phone` | `isSession` não exige `phone`, e `getMe` no mount atualiza |
| Telefones de participantes ficam visíveis a qualquer usuária logada | `RideService` | Exposição dentro da empresa | Decisão da usuária (C. Privacidade) |
| Teste AUTH-23 vai falhar | `soft-go-ii-api/src/auth/auth.controller.spec.ts` (bloco "Rotas existentes continuam abertas") | Mudança intencional | Atualizado na mesma tarefa do guard, com referência ao AD-003. O teste "Nenhum guard global" continua |

---

## Tech Decisions

| Decision | Choice | Rationale |
| -------- | ------ | --------- |
| Leitura opcional do token | `OptionalJwtAuthGuard` por rota | Explícito, como pede o AD-001. Token inválido é tratado como deslogada, então o mural nunca quebra |
| Fonte do nome e do telefone do participante | `users`, via relação. `ride_users` só liga carona e usuária | Uma fonte só. Atende à decisão de apagar as inscrições antigas |
| Telefone do modal | No mesmo `POST /ride-users` | Uma chamada. O telefone é gravado antes da inscrição |
| Primeiro nome | Calculado na API | Endpoint público expõe menos dado |
| Duas migrations | `users.phone` separado de `ride_users` | Cada uma revertível isoladamente. Nomes descritivos (skill `migration`) |

> Decisões de projeto: **AD-003** (rotas públicas que personalizam a resposta usam `OptionalJwtAuthGuard`; token inválido = anônima; `POST /ride-users` passa a exigir login, o que substitui o AUTH-23 nessa rota) e **AD-004** (`ride_users` só liga carona e usuária; os dados pessoais ficam em `users`).
