# Vou Junto com Login Specification

**PRD**: `prds/refactor-voujunto.md`
**Status**: Approved (2026-10-07)

## Problem Statement

O modal "Quero ir junto!" pede nome e telefone a cada inscrição, mesmo agora que o Soft Go tem login. Por isso não dá para saber se duas inscrições são da mesma pessoa, nem impedir que alguém confirme presença duas vezes. A inscrição precisa usar a usuária logada, e o card precisa mostrar quem vai.

## Goals

- [ ] Uma usuária logada que já tem telefone na conta confirma presença com 1 clique, sem digitar nada.
- [ ] Ninguém se inscreve sem estar logada.
- [ ] A mesma usuária não fica inscrita duas vezes na mesma carona.
- [ ] O card da carona mostra os participantes, cada um ligado à conta certa.

## Out of Scope

| Feature | Reason |
| ------- | ------ |
| Cancelar presença / sair da carona | Não está no PRD |
| Tela de editar perfil (nome, telefone, senha) | Feature futura; o telefone que falta na conta é resolvido pelo modal (JOIN-12/13) |
| Vincular a carona (`rides`) à usuária que a publicou / exigir login para publicar | Não está no PRD; `POST /ride` continua aberto |
| Preencher automaticamente o formulário "Publicar carona" com os dados da conta | Não está no PRD |
| Botão "Vou junto" em caronas de ônibus (vagas ilimitadas) | Comportamento já existente: o card só mostra o botão quando há vagas limitadas. Fica registrado como bug separado |
| Migrar inscrições antigas para usuários | Não há como saber a qual conta cada uma pertence. Elas são apagadas (JOIN-28) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --------------------- | -------------- | --------- | ---------- |
| Nome da tabela | `ride_users` (o PRD diz `rides_people`) | É o nome real no banco e nas entidades | y (fato do código) |
| Origem do telefone | Campo opcional `users.phone`. O cadastro pede como opcional. O modal pede só quando a conta não tem telefone e, se preenchido, grava na conta | Decisão da usuária. Resolve contas antigas sem precisar de tela de perfil | y |
| Onde os participantes aparecem | No card da carona (lista de primeiros nomes) | Decisão da usuária. Atende ao critério de aceite 4 | y |
| Quem vê o WhatsApp dos participantes | Só usuárias logadas. A API inclui `phone` só quando a requisição tem token válido | Decisão da usuária. O `GET /ride` é público | y |
| Deslogada clica "Vou junto" | Vai para `/login`, sem abrir o modal. Depois do login, o fluxo atual já leva para `/` | Comportamento mais simples. A página de login já navega para `/` | n (padrão do agente) |
| Usuária que já confirmou | No lugar do botão "Vou junto", o card mostra "Você vai nesta carona" | Evita o 409 no caminho comum | n (padrão do agente) |
| Mensagem de duplicidade | `409` com `"Você já confirmou presença nesta carona"` | Segue o padrão `ConflictException` da API | n (padrão do agente) |
| Nome exibido do participante | Primeiro nome, vindo da conta (`users.name`) | Menos dado pessoal no endpoint público | n (padrão do agente) |
| Inscrições antigas (sem `user_id`) | A migration apaga essas inscrições e torna `ride_users.user_id` obrigatório (`NOT NULL`). Apagar não tem volta: o `down()` restaura a estrutura, mas não os dados | Decisão da usuária. Com isso, toda inscrição tem uma conta | y |
| Colunas `ride_users.name`, `ride_users.phone` e a constraint `UQ_ride_phone` | São removidas na mesma migration. Nome e telefone passam a vir de `users` | Sem inscrições antigas, essas colunas não têm mais uso. Mantê-las criaria duas fontes para o mesmo dado | n (padrão do agente) |
| Validação do telefone | Front: 11 dígitos, só números, mensagem "Número inválido, informe 11 dígitos (DDD + número)". API: mesma regra BR com DDD do `CreateRideDto`. String vazia = ausente | Reaproveita as regras já usadas no projeto | n (padrão do agente) |
| Token inválido ou expirado no `GET /ride` | A rota responde normalmente, como deslogada (sem telefones). Não responde 401 | O mural continua público | n (padrão do agente) |
| AUTH-23 (inscrição aberta) | Passa a valer só para `GET /ride`, `POST /ride` e `GET /transport-type`. `POST /ride-users` passa a exigir token | É o objetivo deste PRD. Registrado como decisão de projeto | y |

**Open questions:** none. Todas foram resolvidas ou registradas acima.

---

## User Stories

### P1: Confirmar presença logada ⭐ MVP

**User Story**: Como usuária logada, quero confirmar presença numa carona com um clique, sem digitar meus dados de novo, para confirmar mais rápido.

**Why P1**: É o núcleo do PRD.

**Acceptance Criteria**:

1. WHEN uma usuária logada cujo telefone está salvo na conta clica em "Vou junto" THEN the front SHALL abrir o modal com o resumo da carona e o botão "Confirmar presença", sem campos de nome ou telefone. [JOIN-01]
2. WHEN a usuária confirma no modal THEN the front SHALL enviar `POST /ride-users` com `{ rideId }` e o header `Authorization: Bearer <token>`, sem `name`. [JOIN-02]
3. WHEN `POST /ride-users` recebe token válido e `rideId` de uma carona com vaga THEN the API SHALL criar a inscrição com `user_id` = id do token e responder `201`. [JOIN-03]
4. IF o body de `POST /ride-users` contém `name` THEN the API SHALL responder `400`. [JOIN-04]
5. WHEN a confirmação retorna `201` THEN the front SHALL exibir o toast "Presença confirmada!", fechar o modal e recarregar a lista de caronas. [JOIN-05]
6. IF a carona está lotada THEN the API SHALL responder `409` com a mensagem atual ("Não existem mais vagas disponíveis para este transporte"), e the front SHALL exibir essa mensagem em um toast de erro. [JOIN-06]

**Independent Test**: Logada com uma conta que tem telefone, clicar "Vou junto", depois "Confirmar presença". Aparece o toast "Presença confirmada!", o card mostra "Você vai nesta carona" e, no banco, `ride_users.user_id` é o id da conta.

---

### P1: Só quem está logada confirma ⭐ MVP

**User Story**: Como sistema, quero que só usuárias autenticadas confirmem presença, para que cada inscrição pertença a uma conta.

**Why P1**: Critério de aceite 2 do PRD.

**Acceptance Criteria**:

1. IF `POST /ride-users` é chamado sem token, com token inválido ou com token expirado THEN the API SHALL responder `401` sem criar inscrição. [JOIN-07]
2. WHILE a usuária está deslogada, WHEN ela clica em "Vou junto" THEN the front SHALL navegar para `/login` sem abrir o modal. [JOIN-08]
3. The API SHALL continuar aceitando `GET /ride`, `POST /ride` e `GET /transport-type` sem token. [JOIN-09]

**Independent Test**: Deslogada, clicar "Vou junto" leva para `/login`. Um `curl -X POST /ride-users` sem token devolve `401`.

---

### P1: Sem presença duplicada ⭐ MVP

**User Story**: Como dona de uma carona, quero que cada colega apareça uma vez só, para saber quantas pessoas vão de verdade.

**Why P1**: Critério de aceite 3 do PRD.

**Acceptance Criteria**:

1. IF a usuária do token já tem inscrição na carona THEN the API SHALL responder `409` com `message: "Você já confirmou presença nesta carona"` sem criar outra. [JOIN-10]
2. IF duas confirmações simultâneas da mesma usuária na mesma carona violam a constraint `UNIQUE (ride_id, user_id)` THEN the API SHALL responder `409` com a mesma mensagem (nunca `500`). [JOIN-11]
3. WHILE a usuária logada está inscrita numa carona the card SHALL exibir "Você vai nesta carona" no lugar do botão "Vou junto". [JOIN-12]

**Independent Test**: Confirmar presença e chamar `POST /ride-users` de novo com o mesmo token. A resposta é `409` e o banco continua com uma inscrição só.

---

### P1: Telefone na conta ⭐ MVP

**User Story**: Como usuária, quero informar meu WhatsApp uma vez só, para que colegas possam me chamar sem que eu digite o número a cada carona.

**Why P1**: Decisão em aberto do PRD, resolvida com `users.phone`.

**Acceptance Criteria**:

1. The signup form SHALL exibir o campo opcional "WhatsApp (opcional)", e `POST /auth/signup` SHALL aceitar `phone` opcional. [JOIN-13]
2. WHEN o cadastro é enviado com telefone válido THEN the API SHALL salvar `users.phone`. Vazio ou ausente SHALL ser salvo como `null`. [JOIN-14]
3. The API SHALL incluir `phone` (string ou `null`) no objeto `user` das respostas de signup, signin e `GET /auth/me`. [JOIN-15]
4. WHILE a conta logada não tem telefone the modal SHALL exibir o campo opcional "WhatsApp" acima do botão "Confirmar presença". [JOIN-16]
5. WHEN a usuária confirma presença com o campo "WhatsApp" preenchido THEN the API SHALL salvar o telefone em `users.phone` da conta do token, e the front SHALL atualizar a sessão. Assim o modal seguinte não mostra o campo. [JOIN-17]
6. IF o telefone (no cadastro ou no modal) não tem 11 dígitos numéricos THEN the front SHALL bloquear o envio e exibir "Número inválido, informe 11 dígitos (DDD + número)" abaixo do campo. [JOIN-18]
7. IF a API recebe `phone` fora da regra BR com DDD (a mesma do `CreateRideDto`) THEN the API SHALL responder `400` com mensagem em PT-BR, sem criar a conta ou a inscrição. [JOIN-19]

**Independent Test**: Cadastrar com WhatsApp e verificar que o `GET /auth/me` traz o `phone`. Entrar com uma conta sem telefone, abrir o modal e ver o campo. Confirmar com número e abrir o modal de outra carona: o campo não aparece mais.

---

### P1: Participantes no card ⭐ MVP

**User Story**: Como colega, quero ver no card quem vai na carona, para saber com quem vou.

**Why P1**: Critério de aceite 4 do PRD.

**Acceptance Criteria**:

1. The API SHALL incluir em cada carona do `GET /ride` a lista `participants`, com `{ userId, name }` por inscrição. `userId` é o id da conta e `name` é só o primeiro nome, vindo de `users.name`. [JOIN-20]
2. WHEN `GET /ride` recebe token válido THEN each participant SHALL incluir `phone` (string ou `null`). [JOIN-21]
3. IF `GET /ride` é chamado sem token, ou com token inválido ou expirado THEN the API SHALL responder `200` sem a chave `phone` em nenhum participante. [JOIN-22]
4. The card SHALL listar os primeiros nomes dos participantes da carona. [JOIN-23]
5. WHILE a usuária está logada the card SHALL exibir, para cada participante com telefone, um link de WhatsApp `https://wa.me/<phone>`. [JOIN-24]
6. The API SHALL descontar das vagas da carona cada inscrição existente, mantendo o cálculo atual de vagas disponíveis. [JOIN-25]

**Independent Test**: Deslogada, o card mostra "Vão: Ana, Giovanna" sem links. Logada, os nomes com telefone viram links de WhatsApp. O `GET /ride` sem token não tem nenhuma chave `phone` em `participants`.

---

## Edge Cases

- IF `rideId` não existe THEN the API SHALL responder `404` com a mensagem atual ("Corrida não encontrada"). [JOIN-26]
- IF a sessão expira e a usuária clica em "Confirmar presença" THEN the front SHALL seguir o fluxo de sessão expirada (AUTH-26: limpa a sessão, toast, `/login`) e SHALL NOT exibir o toast de sucesso. [JOIN-27]
- WHEN a migration da feature roda THEN it SHALL apagar as linhas de `ride_users` sem conta vinculada, remover `name`, `phone` e `UQ_ride_phone`, e deixar `user_id` como `NOT NULL` com FK para `users(id)` e `UNIQUE (ride_id, user_id)`. [JOIN-28]

---

## Implicit-Requirement Dimensions

| Dimension | Resolution |
| --------- | ---------- |
| Input validation & bounds | JOIN-04, 18, 19 |
| Failure / partial-failure states | JOIN-06, 26, 27. Se salvar o telefone na conta falhar, a inscrição não é criada (a ordem fica para o design) |
| Idempotency / retry / duplicate handling | JOIN-10, 11, 12 |
| Auth boundaries & rate limits | JOIN-07, 08, 09, 21, 22. Rate limit N/A, porque o app é interno (já registrado como dívida em auth) |
| Concurrency / ordering | JOIN-11 (constraint única resolve a corrida); lotação concorrente segue a checagem atual (dívida existente) |
| Data lifecycle / expiry | JOIN-28: as inscrições antigas são apagadas uma única vez na migration. Fora isso, N/A, porque cancelar presença está fora do escopo |
| Observability | N/A: segue o logger padrão do Nest, sem logar telefone nem token |
| External-dependency failure | N/A: nenhuma dependência externa nova |
| State-transition integrity | JOIN-08, 12, 17 (deslogada → logada → inscrita; sem telefone → com telefone) |

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| -------------- | ----- | ----- | ------ |
| JOIN-01 | P1: Confirmar presença logada | Tasks | Pending |
| JOIN-02 | P1: Confirmar presença logada | Tasks | Pending |
| JOIN-03 | P1: Confirmar presença logada | Execute | Done (T6) |
| JOIN-04 | P1: Confirmar presença logada | Execute | Done (T6) |
| JOIN-05 | P1: Confirmar presença logada | Tasks | Pending |
| JOIN-06 | P1: Confirmar presença logada | Tasks | Pending |
| JOIN-07 | P1: Só logada confirma | Execute | Done (T6) |
| JOIN-08 | P1: Só logada confirma | Tasks | Pending |
| JOIN-09 | P1: Só logada confirma | Execute | Done (T8) |
| JOIN-10 | P1: Sem duplicidade | Tasks | Pending |
| JOIN-11 | P1: Sem duplicidade | Execute | Done (T5) |
| JOIN-12 | P1: Sem duplicidade | Tasks | Pending |
| JOIN-13 | P1: Telefone na conta | Execute | Done (T13) |
| JOIN-14 | P1: Telefone na conta | Execute | Done (T3) |
| JOIN-15 | P1: Telefone na conta | Execute | Done (T9) |
| JOIN-16 | P1: Telefone na conta | Tasks | Pending |
| JOIN-17 | P1: Telefone na conta | Tasks | Pending |
| JOIN-18 | P1: Telefone na conta | Execute | Done (T15) |
| JOIN-19 | P1: Telefone na conta | Execute | Done (T6) |
| JOIN-20 | P1: Participantes no card | Execute | Done (T7) |
| JOIN-21 | P1: Participantes no card | Execute | Done (T8) |
| JOIN-22 | P1: Participantes no card | Execute | Done (T8) |
| JOIN-23 | P1: Participantes no card | Execute | Done (T14) |
| JOIN-24 | P1: Participantes no card | Tasks | Pending |
| JOIN-25 | P1: Participantes no card | Execute | Done (T7) |
| JOIN-26 | Edge | Execute | Done (T5) |
| JOIN-27 | Edge | Tasks | Pending |
| JOIN-28 | Edge | Execute | Done (T2) |

**Coverage:** 28 total, 28 mapped to tasks (ver `tasks.md` → Requirement Coverage), 0 unmapped

---

## Success Criteria

- [ ] Uma usuária com telefone na conta confirma presença com 2 cliques ("Vou junto", "Confirmar presença") e sem digitar nada.
- [ ] Zero inscrições duplicadas `(ride_id, user_id)` no banco, garantido pela constraint.
- [ ] Nenhum telefone de participante aparece em resposta de `GET /ride` sem token.
