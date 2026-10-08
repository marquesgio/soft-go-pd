# Dona da Carona Specification

**PRD**: `prds/save-rideowner.md`
**Status**: Approved (2026-10-08)

## Problem Statement

Quem publica uma carona digita nome e telefone no formulário, e esses dados ficam soltos em `rides.name` e `rides.phone`, sem vínculo com nenhuma conta. Com login e inscrições já ligadas a `users` (features auth e vou-junto), a carona é o único registro sem dona. Toda carona publicada precisa ficar associada à usuária logada que a criou.

## Goals

- [ ] Publicar uma carona não pede nome. O WhatsApp só é pedido, como campo opcional, quando a conta ainda não tem telefone.
- [ ] Toda carona no banco tem `owner_id` apontando para uma conta em `users`.
- [ ] O card mostra o nome da dona vindo de `users`, e o WhatsApp dela só para quem está logada.
- [ ] Ninguém publica carona sem estar logada.

## Out of Scope

| Feature | Reason |
| ------- | ------ |
| Editar ou excluir a própria carona | Não está no PRD |
| Página "minhas caronas" | Não está no PRD |
| Tela de perfil para editar nome ou telefone | Feature futura. O telefone que falta na conta é resolvido pelo formulário (OWNER-25/26) |
| Atribuir as caronas antigas a alguma conta | Não há como saber quem as publicou. A migration apaga essas caronas (OWNER-21) |
| Mudar `GET /ride/:id` (que filtra por tipo de transporte) | Comportamento atual. Ganha o campo `owner` porque reaproveita `findAllRides`, sem outra mudança |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --------------------- | -------------- | --------- | ---------- |
| Caronas antigas (sem dona) | A migration apaga todas as caronas existentes. As inscrições delas saem juntas pelo `ON DELETE CASCADE` de `ride_users`. `rides.owner_id` passa a ser `NOT NULL`. O `down()` restaura a estrutura, não os dados | Decisão da usuária. O PRD aceita base zerada. Segue o padrão da AD-004 | y |
| Colunas `rides.name` e `rides.phone` | Removidas na mesma migration. Nome e telefone da dona vêm de `users` | Decisão da usuária. Uma fonte só para dados pessoais (AD-004) | y |
| Telefone da dona na resposta | `owner.phone` só vem quando a requisição tem token válido (`users.phone` ou `null`). Sem token, a chave não existe | Decisão da usuária. Mesma regra das participantes (AD-003) | y |
| Nome da dona exibido | Nome completo (`users.name`) | Decisão da usuária. É como o card mostra hoje | y |
| Dona se inscrevendo na própria carona | Não pode. O card mostra "Sua carona" no lugar de "Vou junto", e a API responde `409` com "Você não pode se inscrever na sua própria carona" | Decisão da usuária (comportamento e texto do card). A mensagem do 409 é padrão do agente | y |
| Formato do campo na resposta | `owner: { id, name, phone? }` em cada carona de `GET /ride`. `name` e `phone` deixam de existir no nível da carona | Mesmo formato de `participants`. Remove a duplicidade | n (padrão do agente) |
| Proteção de `POST /ride` | `JwtAuthGuard` na rota. Sem token ou com token inválido: `401` "Não autenticado". Substitui o AUTH-23 nessa rota; `GET /ride` segue com `OptionalJwtAuthGuard` | Critério de aceite 4 do PRD. Mesmo padrão de `POST /ride-users` (AD-003) | y |
| Token válido de conta que não existe mais em `users` | `401` "Não autenticado", sem criar carona | Sem essa checagem, a FK falharia e a API responderia `500` | n (padrão do agente) |
| Deslogada tentando publicar | O botão "Vou para a soft" do mural leva para `/login`. Acessar `/form` direto também redireciona para `/login` | Mesmo comportamento do "Vou junto" deslogada (vou-junto) | n (padrão do agente) |
| Sessão expira enquanto preenche o formulário | Fica com o fluxo atual: o interceptor de `api.ts` limpa a sessão no `401`, e o formulário exibe o toast de erro atual | Comportamento já existente, sem código novo | n (padrão do agente) |
| WhatsApp no formulário | Campo opcional "WhatsApp", exibido só quando a conta logada não tem telefone. Preenchido, a API grava em `users.phone` da dona e o front atualiza a sessão. Validação igual à da vou-junto: front com 11 dígitos numéricos (`optionalPhone`), API com a regra BR com DDD do `CreateRideUserDto`. String vazia = ausente | Pedido da usuária na aprovação. Segue o mesmo fluxo do modal "Vou junto" (JOIN-16/17) | y (comportamento espelhado da vou-junto: padrão do agente) |
| Campos extras no body de `POST /ride` | `name` ou `ownerId` no body: `400` pelo `forbidNonWhitelisted`, sem criar carona. `phone` continua aceito, mas grava na conta, não na carona | A dona sempre vem do token, nunca do body | n (padrão do agente) |

**Open questions:** none. Todas foram resolvidas ou registradas acima.

---

## User Stories

### P1: Publicar carona logada ⭐ MVP

**User Story**: Como usuária logada, quero que a carona que publico fique registrada no meu nome automaticamente, sem digitar meus dados de novo.

**Why P1**: É o núcleo do PRD (critérios de aceite 1 e 2).

**Acceptance Criteria**:

1. WHEN uma usuária logada cuja conta tem telefone abre `/form` THEN the front SHALL exibir o formulário sem os campos "Seu Nome" e "WhatsApp". [OWNER-01]
2. WHEN a usuária logada envia o formulário válido THEN the front SHALL enviar `POST /ride` com o header `Authorization: Bearer <token>` e um body sem as chaves `name` e `ownerId`. [OWNER-02]
3. WHEN `POST /ride` recebe token válido e body válido THEN the API SHALL criar a carona com `owner_id` igual ao id do token e responder `201`. [OWNER-03]
4. IF o body de `POST /ride` contém `name` ou `ownerId` THEN the API SHALL responder `400` e nenhuma carona SHALL ser gravada. [OWNER-04]
5. WHEN `POST /ride` responde `201` THEN the front SHALL exibir o toast de sucesso e navegar para `/`. [OWNER-05]

6. WHILE a conta logada não tem telefone the front SHALL exibir em `/form` o campo opcional "WhatsApp", sem o campo "Seu Nome". [OWNER-25]
7. WHEN a usuária publica com o campo "WhatsApp" preenchido THEN the front SHALL enviar `phone` no body, the API SHALL gravar o telefone em `users.phone` da conta do token, e the front SHALL atualizar a sessão com o telefone, de modo que o próximo `/form` não mostre o campo. [OWNER-26]
8. WHEN a usuária publica com o campo "WhatsApp" vazio THEN the front SHALL enviar o body sem a chave `phone`, e a sessão SHALL manter o telefone que já tinha (`null`). [OWNER-27]
9. IF `POST /ride` recebe `phone: ""` THEN the API SHALL criar a carona, responder `201` e manter `users.phone` como estava. [OWNER-28]
10. IF o campo "WhatsApp" preenchido não tem exatamente 11 dígitos numéricos THEN the front SHALL bloquear o envio e exibir "Número inválido, informe 11 dígitos (DDD + número)" abaixo do campo. [OWNER-29]
11. IF `POST /ride` recebe `phone` fora da regra BR com DDD THEN the API SHALL responder `400` com mensagem em PT-BR, nenhuma carona SHALL ser gravada e `users.phone` SHALL continuar como estava. [OWNER-30]

**Independent Test**: Logada, abrir `/form`, preencher data, hora, cidade e transporte e publicar. O formulário não tem nome nem WhatsApp, o mural mostra a carona com o nome da conta, e no banco `rides.owner_id` é o id da conta.

---

### P1: Só quem está logada publica ⭐ MVP

**User Story**: Como sistema, quero que só usuárias autenticadas publiquem caronas, para que toda carona tenha uma dona.

**Why P1**: Critério de aceite 4 do PRD.

**Acceptance Criteria**:

1. IF `POST /ride` chega sem header `Authorization` THEN the API SHALL responder `401` com a mensagem "Não autenticado" e nenhuma carona SHALL ser gravada. [OWNER-06]
2. IF `POST /ride` chega com token inválido ou expirado THEN the API SHALL responder `401` com a mensagem "Não autenticado" e nenhuma carona SHALL ser gravada. [OWNER-07]
3. IF `POST /ride` chega com token válido cujo id não existe em `users` THEN the API SHALL responder `401` com a mensagem "Não autenticado" e nenhuma carona SHALL ser gravada. [OWNER-08]
4. WHEN uma usuária deslogada clica em "Vou para a soft" no mural THEN the front SHALL navegar para `/login`. [OWNER-09]
5. WHEN uma usuária deslogada acessa `/form` THEN the front SHALL redirecionar para `/login` sem exibir o formulário. [OWNER-10]

**Independent Test**: Deslogada, clicar em "Vou para a soft" leva para `/login`. Abrir `/form` pela URL também leva para `/login`. `curl -X POST /ride` sem token responde `401`, e a tabela `rides` não muda.

---

### P1: Mural mostra a dona ⭐ MVP

**User Story**: Como colega olhando o mural, quero ver quem publicou cada carona, com o nome vindo da conta dela.

**Why P1**: Critério de aceite 3 do PRD.

**Acceptance Criteria**:

1. The API SHALL incluir em cada carona de `GET /ride` o objeto `owner` com `id` igual a `rides.owner_id` e `name` igual a `users.name` completo. [OWNER-11]
2. WHEN `GET /ride` recebe token válido THEN the API SHALL incluir em cada `owner` a chave `phone` com o valor de `users.phone` (ou `null` quando a conta não tem telefone). [OWNER-12]
3. IF `GET /ride` chega sem token ou com token inválido THEN the API SHALL omitir a chave `phone` de cada `owner`. [OWNER-13]
4. The API SHALL omitir as chaves `name` e `phone` do nível da carona em `GET /ride`. [OWNER-14]
5. The front SHALL exibir no card o `owner.name` e, no círculo do avatar, a inicial calculada a partir de `owner.name`. [OWNER-15]
6. WHILE a usuária está logada e `owner.phone` está preenchido, the front SHALL exibir no card o botão "WhatsApp" com link para `https://wa.me/<owner.phone>`. [OWNER-16]
7. WHILE a usuária está deslogada, the front SHALL não exibir o botão "WhatsApp" no card. [OWNER-17]
8. IF `owner.phone` é `null` THEN the front SHALL não exibir o botão "WhatsApp" no card. [OWNER-18]

**Independent Test**: Publicar uma carona com uma conta que tem telefone. Deslogada, o card mostra o nome completo da conta e não mostra WhatsApp. Logada com outra conta, o botão "WhatsApp" aparece com o telefone da dona. A resposta de `GET /ride` não tem `name` nem `phone` na raiz da carona.

---

### P1: Migration da dona ⭐ MVP

**User Story**: Como sistema, quero que o banco garanta que toda carona tem dona, para não haver caronas soltas.

**Why P1**: É o modelo de dados do PRD. Sem a migration, nada acima funciona.

**Acceptance Criteria**:

1. WHEN a migration roda `up()` THEN the database SHALL ficar sem nenhuma das caronas existentes antes dela, e sem as inscrições (`ride_users`) dessas caronas. [OWNER-21]
2. WHEN a migration roda `up()` THEN the table `rides` SHALL ter a coluna `owner_id integer NOT NULL` com a FK `FK_rides_owner` para `users(id)`. [OWNER-22]
3. WHEN a migration roda `up()` THEN the table `rides` SHALL ficar sem as colunas `name` e `phone`. [OWNER-23]
4. WHEN a migration roda `down()` THEN the table `rides` SHALL voltar a ter `name varchar(100) NOT NULL` e `phone varchar(15)` e ficar sem `owner_id` e sem `FK_rides_owner`. [OWNER-24]

**Independent Test**: Com caronas e inscrições na base local, rodar a migration. `rides` e `ride_users` ficam vazias, `\d rides` mostra `owner_id NOT NULL` com a FK e sem `name` e `phone`. Rodar `migration:revert` volta a estrutura anterior.

---

### P2: Dona não se inscreve na própria carona

**User Story**: Como dona de uma carona, não quero ver "Vou junto" na minha própria carona, porque não faz sentido ocupar uma vaga dela.

**Why P2**: Consequência de ter dona. O PRD não pede, mas sem isso a dona pode ocupar uma vaga da própria carona.

**Acceptance Criteria**:

1. WHEN uma usuária logada vê o card de uma carona cujo `owner.id` é o id dela THEN the front SHALL exibir "Sua carona" no lugar do botão "Vou junto". [OWNER-19]
2. IF `POST /ride-users` recebe um `rideId` cuja dona é a usuária do token THEN the API SHALL responder `409` com a mensagem "Você não pode se inscrever na sua própria carona" e nenhuma inscrição SHALL ser gravada. [OWNER-20]

**Independent Test**: Logada, publicar uma carona com vagas. No mural, o card dela mostra "Sua carona" e não "Vou junto". Um `POST /ride-users` com o token dela e esse `rideId` responde `409`, e `ride_users` não muda.

---

## Edge Cases

- IF a dona não tem telefone na conta e deixa o "WhatsApp" vazio THEN the front SHALL publicar a carona normalmente, e o card SHALL não exibir WhatsApp (OWNER-18, OWNER-27).
- IF a carona da própria usuária está lotada THEN the front SHALL exibir "Sua carona" (OWNER-19), não o estado de carona lotada.
- IF o body de `POST /ride` traz `phone: ""` (formato antigo do front) THEN the API SHALL tratar como ausente e responder `201` (OWNER-28).

---

## Implicit-Requirement Sweep

| Dimension | Result |
| --------- | ------ |
| Input validation & bounds | OWNER-04 (campos que o DTO não declara), OWNER-29/30 (telefone). As outras validações do `CreateRideDto` não mudam |
| Failure / partial-failure states | OWNER-08 (conta inexistente gera `401`, não `500`). Sessão expirada segue o fluxo atual (Assumptions) |
| Idempotency / retry / duplicate handling | N/A because publicar duas caronas iguais já é permitido hoje e o PRD não muda isso |
| Auth boundaries & rate limits | OWNER-06/07/08/09/10, OWNER-12/13/17. Rate limit: N/A because nenhuma rota do projeto tem limite e o PRD não pede |
| Concurrency / ordering | N/A because a dona é gravada junto da carona, num único `INSERT` |
| Data lifecycle / expiry | OWNER-21 (apagar caronas antigas), OWNER-24 (`down()` sem dados) |
| Observability | N/A because o projeto não tem logging ou métricas estruturadas e o PRD não pede |
| External-dependency failure | N/A because não há chamada externa nova |
| State-transition integrity | OWNER-20 (dona não vira participante da própria carona) |

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| -------------- | ----- | ----- | ------ |
| OWNER-01 | P1: Publicar carona logada | Design | Pending |
| OWNER-02 | P1: Publicar carona logada | Design | Pending |
| OWNER-03 | P1: Publicar carona logada | Execute | Done (T4) |
| OWNER-04 | P1: Publicar carona logada | Execute | Done (T4) |
| OWNER-05 | P1: Publicar carona logada | Design | Pending |
| OWNER-06 | P1: Só quem está logada publica | Execute | Done (T4) |
| OWNER-07 | P1: Só quem está logada publica | Execute | Done (T4) |
| OWNER-08 | P1: Só quem está logada publica | Execute | Done (T4) |
| OWNER-09 | P1: Só quem está logada publica | Design | Pending |
| OWNER-10 | P1: Só quem está logada publica | Design | Pending |
| OWNER-11 | P1: Mural mostra a dona | Execute | Done (T3) |
| OWNER-12 | P1: Mural mostra a dona | Execute | Done (T4) |
| OWNER-13 | P1: Mural mostra a dona | Execute | Done (T4) |
| OWNER-14 | P1: Mural mostra a dona | Execute | Done (T3) |
| OWNER-15 | P1: Mural mostra a dona | Design | Pending |
| OWNER-16 | P1: Mural mostra a dona | Design | Pending |
| OWNER-17 | P1: Mural mostra a dona | Design | Pending |
| OWNER-18 | P1: Mural mostra a dona | Design | Pending |
| OWNER-19 | P2: Dona não se inscreve na própria carona | Design | Pending |
| OWNER-20 | P2: Dona não se inscreve na própria carona | Design | Pending |
| OWNER-21 | P1: Migration da dona | Execute | Done (T1) |
| OWNER-22 | P1: Migration da dona | Execute | Done (T1) |
| OWNER-23 | P1: Migration da dona | Execute | Done (T1) |
| OWNER-24 | P1: Migration da dona | Execute | Done (T1) |
| OWNER-25 | P1: Publicar carona logada | Design | Pending |
| OWNER-26 | P1: Publicar carona logada | Execute | Implementing (T2) |
| OWNER-27 | P1: Publicar carona logada | Design | Pending |
| OWNER-28 | P1: Publicar carona logada | Execute | Done (T4) |
| OWNER-29 | P1: Publicar carona logada | Design | Pending |
| OWNER-30 | P1: Publicar carona logada | Execute | Done (T4) |

**Coverage:** 30 total, 0 mapped to tasks, 30 unmapped ⚠️

---

## Success Criteria

- [ ] Publicar uma carona logada leva só os campos de data, hora, local, transporte, vagas e observação, mais o WhatsApp opcional quando a conta não tem telefone.
- [ ] `SELECT count(*) FROM rides WHERE owner_id IS NULL` retorna 0, e a coluna não aceita `NULL`.
- [ ] Nenhuma requisição sem token cria carona.
- [ ] Deslogada, `GET /ride` não expõe telefone de ninguém, nem da dona nem das participantes.
