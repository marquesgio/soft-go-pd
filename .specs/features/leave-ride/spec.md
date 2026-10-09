# Sair da Carona Specification

**PRD**: `prds/leave-ride.md`
**Context**: `.specs/features/leave-ride/context.md`
**Status**: In Progress (2026-10-08)

## Problem Statement

Depois de confirmar presença ("Vou junto"), a passageira não tem como desistir. A vaga fica ocupada à toa. A regra "já confirmou presença em uma carona neste dia" também a impede de entrar em outra carona no mesmo dia. A dona, por sua vez, não consegue tirar da carona quem avisou que não vai. Além disso, a lista de passageiros aparece sempre aberta no card e ocupa espaço no mural.

## Goals

- [ ] A passageira sai da carona pela lista de passageiros do card.
- [ ] A dona remove qualquer passageira da própria carona pela mesma lista.
- [ ] A API garante as duas regras: ninguém remove a inscrição de outra pessoa numa carona que não é sua.
- [ ] A lista de passageiros fica recolhida no card e abre para quem quiser ver.

## Out of Scope

| Feature | Reason |
| ------- | ------ |
| Avisar a dona quando a passageira sai, ou a passageira quando é removida | Decisão da usuária: entra no card `prds/notify-ride-deleted.md`, que passa a cobrir os três eventos |
| Motivo da saída/remoção | Não pedido |
| Impedir a passageira removida de se inscrever de novo | Não pedido. Ela pode se inscrever de novo se houver vaga |
| Mudar `GET /ride-users` e `GET /ride-users/:id` (abertos hoje) | Fora do escopo desta feature |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --------------------- | -------------- | --------- | ---------- |
| Quem pode remover | A própria passageira (sair) e a dona da carona (remover qualquer passageira) | Decisão da usuária | y |
| Aviso para a dona / passageira | Nenhum nesta feature. O card `notify-ride-deleted` cobre carona excluída, passageira que saiu e passageira removida | Decisão da usuária ("agora, aviso depois") | y |
| Lista recolhível | O card mostra o botão "Ver passageiros (N)" fechado por padrão. Aberto, vira "Esconder passageiros" e lista os nomes. Sem participantes, o botão não aparece | Pedido da usuária. Texto e formato: padrão do agente | y (comportamento) / n (texto) |
| Onde fica a ação | Ícone de lixeira à direita do nome, dentro da lista aberta. Passageira logada: só no próprio nome, nome acessível "Sair da carona". Dona: em todos os nomes, nome acessível "Remover <nome> da carona" | Pedido da usuária ("delete para quem estiver nela") | y |
| Confirmação | Modal antes de remover. Saída: "Sair da carona?" e botão "Sair". Remoção pela dona: "Remover <nome> da carona?" e botão "Remover". Os dois têm "Cancelar" | Ação destrutiva, mesmo padrão do excluir carona | n (padrão do agente) |
| Rota | `DELETE /ride-users/:rideId/users/:userId`, protegida por `JwtAuthGuard`, `204` sem corpo. Substitui o `DELETE /ride-users/:rideId` do PRD, que não cobria a remoção pela dona | Uma rota só para os dois casos | n (padrão do agente) |
| Ordem das checagens | `401` → `400` (ids não inteiros) → `404` carona → `403` permissão → `404` inscrição → `204` | Mesmo padrão do `DELETE /ride/:id` | n (padrão do agente) |
| Mensagens | `404` "Carona não encontrada" · `403` "Você só pode sair da carona ou remover passageiras da sua carona" · `404` "Esta pessoa não está nesta carona" · toasts "Você saiu da carona", "<nome> foi removida da carona", "Não foi possível remover da carona. Tente novamente." | PT-BR, exibidas no toast | n (padrão do agente) |
| Dona como alvo | `DELETE` com `:userId` igual à dona → `404` "Esta pessoa não está nesta carona", porque a dona nunca é participante (OWNER-20) | Cai naturalmente na checagem de inscrição | n (padrão do agente) |
| Lixeira no resumo dentro de modais | Com `showButton={false}` (modais de "Vou junto" e de excluir carona), a lista não mostra lixeiras | O resumo é só leitura | n (padrão do agente) |

**Open questions:** none.

---

## User Stories

### P1: API remove inscrição com permissão ⭐ MVP

**User Story**: Como passageira, quero sair de uma carona; como dona, quero remover uma passageira da minha carona; e ninguém mais pode fazer isso.

**Why P1**: É a garantia real. O front só esconde as lixeiras.

**Acceptance Criteria**:

1. WHEN `DELETE /ride-users/:rideId/users/:userId` recebe o token da própria passageira (`userId` do token = `:userId`) THEN the API SHALL apagar a inscrição e responder `204` com corpo vazio. [LEAVE-01]
2. WHEN a mesma rota recebe o token da dona da carona `:rideId` THEN the API SHALL apagar a inscrição de `:userId` e responder `204` com corpo vazio. [LEAVE-02]
3. WHEN uma inscrição é apagada THEN `GET /ride` SHALL deixar de listar a pessoa em `participants` e SHALL devolver a vaga em `transportType.spots`, e a pessoa SHALL poder se inscrever em outra carona no mesmo dia. [LEAVE-03]
4. IF a carona `:rideId` não existe THEN the API SHALL responder `404` com a mensagem "Carona não encontrada". [LEAVE-04]
5. IF o token não é da passageira `:userId` nem da dona da carona THEN the API SHALL responder `403` com a mensagem "Você só pode sair da carona ou remover passageiras da sua carona", e a inscrição SHALL continuar gravada. [LEAVE-05]
6. IF `:userId` não está inscrita na carona (inclusive quando é a dona) THEN the API SHALL responder `404` com a mensagem "Esta pessoa não está nesta carona". [LEAVE-06]
7. IF a requisição chega sem `Authorization`, com token inválido, expirado ou de sessão revogada THEN the API SHALL responder `401` com a mensagem "Não autenticado" e nenhuma inscrição SHALL ser apagada. [LEAVE-07]
8. IF `:rideId` ou `:userId` não é inteiro THEN the API SHALL responder `400` e nenhuma inscrição SHALL ser apagada. [LEAVE-08]
9. WHILE a rota nova exige token, `POST /ride-users` SHALL continuar com `JwtAuthGuard` e nenhum guard global SHALL ser registrado. [LEAVE-09]

**Independent Test**: Com três contas (dona, Ana inscrita, Bia de fora): Bia remove Ana → `403` e Ana continua inscrita; Ana remove a si mesma → `204` e some de `participants`; Ana se inscreve de novo; a dona remove Ana → `204`.

---

### P1: Lista de passageiros recolhível ⭐ MVP

**User Story**: Como pessoa olhando o mural, quero que a lista de passageiros fique escondida até eu pedir para ver.

**Why P1**: Pedido direto da usuária. É onde ficam as lixeiras.

**Acceptance Criteria**:

1. WHILE a carona tem participantes the card SHALL exibir o botão "Ver passageiros (N)", com `aria-expanded="false"`, e SHALL não exibir os nomes. [LEAVE-10]
2. WHEN alguém clica em "Ver passageiros (N)" THEN the card SHALL exibir a lista com o primeiro nome de cada participante e trocar o botão para "Esconder passageiros" com `aria-expanded="true"`; um novo clique SHALL esconder a lista. [LEAVE-11]
3. WHILE a carona não tem participantes the card SHALL não exibir o botão nem a lista. [LEAVE-12]
4. WHILE a lista está aberta e a usuária está logada the nome de cada participante com telefone SHALL ser link `https://wa.me/<telefone>`; deslogada, os nomes SHALL aparecer sem link. [LEAVE-13]

**Independent Test**: No mural, um card com 2 passageiros mostra "Ver passageiros (2)" sem nomes; ao clicar, os nomes aparecem; ao clicar em "Esconder passageiros", somem.

---

### P1: Sair ou remover pela lista ⭐ MVP

**User Story**: Como passageira, quero sair da carona pela lista; como dona, quero remover uma passageira pela lista.

**Why P1**: Núcleo do PRD leave-ride, ampliado pela decisão da usuária.

**Acceptance Criteria**:

1. WHILE a lista está aberta e a usuária logada é participante the front SHALL exibir, só ao lado do próprio nome, um ícone de lixeira com nome acessível "Sair da carona". [LEAVE-14]
2. WHILE a lista está aberta e a usuária logada é a dona the front SHALL exibir, ao lado de cada nome, um ícone de lixeira com nome acessível "Remover <nome> da carona". [LEAVE-15]
3. WHILE a usuária está deslogada, ou não é dona nem participante, ou o card está num modal (`showButton={false}`) the lista SHALL não exibir lixeiras. [LEAVE-16]
4. WHEN a passageira clica em "Sair da carona" THEN the front SHALL abrir um modal com o título "Sair da carona?" e os botões "Cancelar" e "Sair"; WHEN a dona clica em "Remover <nome> da carona" THEN the modal SHALL ter o título "Remover <nome> da carona?" e os botões "Cancelar" e "Remover". [LEAVE-17]
5. WHEN a usuária clica em "Cancelar", no "X" ou fora do modal THEN the front SHALL fechar o modal sem chamar a API. [LEAVE-18]
6. WHEN a usuária confirma THEN the front SHALL enviar `DELETE /ride-users/<rideId>/users/<userId>` com o token, desabilitar o botão de confirmar enquanto envia e, no `204`, exibir o toast de sucesso ("Você saiu da carona" ou "<nome> foi removida da carona"), fechar o modal e recarregar o mural. [LEAVE-19]
7. IF a API responde `403` ou `404` THEN the front SHALL exibir no toast de erro a mensagem da API, fechar o modal e recarregar o mural. [LEAVE-20]
8. IF a API falha com outro erro (exceto `401`) THEN the front SHALL exibir o toast "Não foi possível remover da carona. Tente novamente." e fechar o modal; IF a API responde `401` THEN the front SHALL deixar o interceptor tratar. [LEAVE-21]

**Independent Test**: Logada como Ana (inscrita), abrir a lista mostra a lixeira só no nome "Ana"; "Sair" mostra "Você saiu da carona" e o card volta a mostrar "Vou junto". Logada como dona, a lista mostra lixeira em todos os nomes; "Remover" em Bia mostra "Bia foi removida da carona".

---

## Edge Cases

- IF a passageira sai de uma carona lotada THEN `GET /ride` SHALL devolver a vaga e o card SHALL voltar a mostrar "Vou junto" para outras contas (LEAVE-03).
- IF a carona é de ônibus (vagas ilimitadas) THEN sair e remover SHALL funcionar igual, e `transportType.spots` SHALL continuar `null` (LEAVE-03).
- IF a inscrição já foi removida em outra aba THEN a API SHALL responder `404` "Esta pessoa não está nesta carona" e o front SHALL recarregar o mural (LEAVE-06, LEAVE-20).
- IF a dona tenta remover a si mesma pela API THEN a API SHALL responder `404` "Esta pessoa não está nesta carona" (LEAVE-06).

---

## Implicit-Requirement Sweep

| Dimension | Result |
| --------- | ------ |
| Input validation & bounds | LEAVE-08 |
| Failure / partial-failure states | Um `DELETE` em `ride_users`, sem estado parcial. LEAVE-20/21 no front |
| Idempotency / retry / duplicate handling | Repetir responde `404` (LEAVE-06). Front evita duplo envio (LEAVE-19) |
| Auth boundaries & rate limits | LEAVE-05/07/09, LEAVE-14/15/16. Rate limit: N/A because nenhuma rota do projeto tem limite e o PRD não pede |
| Concurrency / ordering | Inscrição e saída simultâneas da mesma pessoa: N/A because a `UNIQUE (ride_id, user_id)` e o `DELETE` por id deixam o banco consistente |
| Data lifecycle / expiry | A inscrição é apagada, sem histórico (Out of Scope) |
| Observability | N/A because o projeto não tem logging estruturado e o PRD não pede |
| External-dependency failure | N/A because o aviso ficou para `notify-ride-deleted` |
| State-transition integrity | Inscrita → fora da carona. LEAVE-03 libera a vaga e a regra do mesmo dia |

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| -------------- | ----- | ----- | ------ |
| LEAVE-01 | P1: API remove inscrição com permissão | Tasks | Mapped (T1, T2) |
| LEAVE-02 | P1: API remove inscrição com permissão | Tasks | Mapped (T1, T2) |
| LEAVE-03 | P1: API remove inscrição com permissão | Tasks | Mapped (T2) |
| LEAVE-04 | P1: API remove inscrição com permissão | Tasks | Mapped (T1, T2) |
| LEAVE-05 | P1: API remove inscrição com permissão | Tasks | Mapped (T1, T2) |
| LEAVE-06 | P1: API remove inscrição com permissão | Tasks | Mapped (T1, T2) |
| LEAVE-07 | P1: API remove inscrição com permissão | Tasks | Mapped (T2) |
| LEAVE-08 | P1: API remove inscrição com permissão | Tasks | Mapped (T2) |
| LEAVE-09 | P1: API remove inscrição com permissão | Tasks | Mapped (T2) |
| LEAVE-10 | P1: Lista de passageiros recolhível | Tasks | Mapped (T4, T7) |
| LEAVE-11 | P1: Lista de passageiros recolhível | Tasks | Mapped (T4) |
| LEAVE-12 | P1: Lista de passageiros recolhível | Tasks | Mapped (T4) |
| LEAVE-13 | P1: Lista de passageiros recolhível | Tasks | Mapped (T4, T7) |
| LEAVE-14 | P1: Sair ou remover pela lista | Tasks | Mapped (T4, T6) |
| LEAVE-15 | P1: Sair ou remover pela lista | Tasks | Mapped (T4, T6) |
| LEAVE-16 | P1: Sair ou remover pela lista | Tasks | Mapped (T4, T6) |
| LEAVE-17 | P1: Sair ou remover pela lista | Tasks | Mapped (T5, T7) |
| LEAVE-18 | P1: Sair ou remover pela lista | Tasks | Mapped (T5, T7) |
| LEAVE-19 | P1: Sair ou remover pela lista | Tasks | Mapped (T3, T5, T7) |
| LEAVE-20 | P1: Sair ou remover pela lista | Tasks | Mapped (T7) |
| LEAVE-21 | P1: Sair ou remover pela lista | Tasks | Mapped (T7) |

**Coverage:** 21 total, 21 mapped to tasks, 0 unmapped

---

## Success Criteria

- [ ] A passageira sai da carona pela lista, e a vaga volta no mural.
- [ ] A dona remove passageiras da própria carona; outras contas recebem `403` pela API.
- [ ] A lista de passageiros começa fechada em todos os cards.
