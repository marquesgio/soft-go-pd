# Avisos Specification

**PRD**: `prds/notify-ride-deleted.md`
**Context**: `.specs/features/notify-ride-deleted/context.md`
**Status**: Done (2026-10-09). Sem Verifier independente (ver `tasks.md`)

## Problem Statement

Três ações tiram alguém de uma carona sem que a outra parte saiba:
- a dona exclui a carona, e as inscrições saem junto (delete-rides);
- a passageira sai da carona (leave-ride);
- a dona remove uma passageira (leave-ride).

Hoje o único aviso é por fora, pelo WhatsApp, e quem não cadastrou telefone só percebe ao abrir o mural. O sistema precisa avisar a pessoa afetada no momento da ação.

## Goals

- [ ] Cada um dos três eventos gera um aviso para a pessoa certa, com ou sem telefone cadastrado.
- [ ] O aviso identifica a carona e quem fez a ação.
- [ ] A pessoa vê os avisos num sino no topo da página, com o número de não lidos.
- [ ] Uma falha ao gravar o aviso não impede a ação nem gera erro para quem a fez.

## Out of Scope

| Feature | Reason |
| ------- | ------ |
| E-mail | Decisão da usuária: só aviso no app |
| Aviso em tempo real (WebSocket, polling) | A contagem é atualizada ao carregar a página e ao abrir o sino |
| Apagar avisos / limpar a lista | Não pedido |
| Avisos de outros eventos (nova inscrição, carona publicada etc.) | Fora do PRD |
| Expirar avisos antigos | Não pedido. O painel mostra só os 30 mais recentes |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --------------------- | -------------- | --------- | ---------- |
| Canal | Aviso dentro do app, gravado na tabela nova `notifications` | Decisão da usuária. Não depende de serviço externo e alcança toda conta | y |
| Como ver e limpar | Sino no Header com o número de não lidos. Ao clicar, abre o painel "Avisos" com a lista, do mais novo ao mais antigo, e todos passam a contar como lidos | Decisão da usuária | y |
| Quem recebe | Carona excluída → cada participante (não a dona). Passageira saiu → a dona. Passageira removida pela dona → a passageira | PRD | y |
| Texto do aviso | Frase pronta gravada no momento da ação, porque a carona excluída deixa de existir: "Carla excluiu a carona de 15/10 às 08:00 saindo de Porto Alegre." · "Ana saiu da sua carona de 15/10 às 08:00 saindo de Porto Alegre." · "Carla removeu você da carona de 15/10 às 08:00 saindo de Porto Alegre." Nome = primeiro nome; data `DD/MM`; hora `HH:mm` | O aviso identifica a carona, a pessoa e a ação, sem depender de dados que somem | n (padrão do agente) |
| Falha ao gravar o aviso | A ação acontece primeiro. O aviso é gravado depois, e um erro ao gravar é registrado no log da API e engolido; a resposta da ação não muda | Critério de aceite 3 do PRD | y (PRD) / n (forma) |
| Rotas | `GET /notifications` → `{ unread, items }` com os 30 mais recentes da conta do token; `POST /notifications/read` → `204` e marca todos como lidos. As duas exigem `JwtAuthGuard` | Mesmo padrão de auth das outras rotas privadas | n (padrão do agente) |
| Formato do item | `{ id, type, message, createdAt, read }`. `type` ∈ `ride_deleted`, `passenger_left`, `passenger_removed` | O front só exibe `message`; `type` fica para usos futuros | n (padrão do agente) |
| Badge | Número de não lidos sobre o sino; acima de 9 mostra "9+"; zero não mostra badge | Padrão comum | n (padrão do agente) |
| Quando atualizar | Ao carregar a página logada e ao abrir o painel | Sem tempo real (Out of Scope) | n (padrão do agente) |
| Conta apagada | `notifications.user_id` com `ON DELETE CASCADE` | Não há exclusão de conta hoje; evita lixo se surgir | n (padrão do agente) |

**Open questions:** none.

---

## User Stories

### P1: Avisos gravados nos três eventos ⭐ MVP

**User Story**: Como pessoa afetada por uma mudança numa carona, quero que o sistema registre um aviso para mim.

**Why P1**: Núcleo do PRD.

**Acceptance Criteria**:

1. WHEN a dona exclui uma carona com participantes THEN the API SHALL gravar um aviso `ride_deleted` para cada participante, com a mensagem "<primeiro nome da dona> excluiu a carona de <DD/MM> às <HH:mm> saindo de <cidade>.", e nenhum para a dona. [NOTIFY-01]
2. WHEN a dona exclui uma carona sem participantes THEN the API SHALL não gravar aviso. [NOTIFY-02]
3. WHEN a passageira sai da carona (`DELETE /ride-users/:rideId/users/:userId` com o próprio token) THEN the API SHALL gravar para a dona um aviso `passenger_left` com a mensagem "<primeiro nome da passageira> saiu da sua carona de <DD/MM> às <HH:mm> saindo de <cidade>.". [NOTIFY-03]
4. WHEN a dona remove uma passageira THEN the API SHALL gravar para a passageira um aviso `passenger_removed` com a mensagem "<primeiro nome da dona> removeu você da carona de <DD/MM> às <HH:mm> saindo de <cidade>.", e nenhum para a dona. [NOTIFY-04]
5. IF gravar o aviso falha THEN the ação SHALL continuar concluída, com a mesma resposta (`204`), e o erro SHALL ir para o log da API. [NOTIFY-05]
6. IF a ação é recusada (`401`, `403`, `404`) THEN the API SHALL não gravar aviso. [NOTIFY-06]

**Independent Test**: Dona exclui uma carona com Ana inscrita; `GET /notifications` com o token de Ana traz "Carla excluiu a carona de 15/10 às 08:00 saindo de Porto Alegre." com `read: false`.

---

### P1: Ler avisos pela API ⭐ MVP

**User Story**: Como usuária logada, quero buscar meus avisos e marcá-los como lidos.

**Why P1**: É o que o sino consome.

**Acceptance Criteria**:

1. WHEN `GET /notifications` recebe um token válido THEN the API SHALL responder `200` com `{ unread, items }`: `items` são os 30 avisos mais recentes da conta do token, do mais novo ao mais antigo, cada um `{ id, type, message, createdAt, read }`, e `unread` é o total de não lidos da conta. [NOTIFY-07]
2. WHILE a conta não tem avisos `GET /notifications` SHALL responder `{ unread: 0, items: [] }`. [NOTIFY-08]
3. WHEN `POST /notifications/read` recebe um token válido THEN the API SHALL marcar como lidos todos os avisos da conta do token, sem mexer nos de outras contas, e responder `204`. [NOTIFY-09]
4. IF as rotas de avisos recebem requisição sem token, com token inválido, expirado ou de sessão revogada THEN the API SHALL responder `401` "Não autenticado". [NOTIFY-10]
5. WHILE as rotas de avisos exigem token, nenhum guard global SHALL ser registrado. [NOTIFY-11]

**Independent Test**: Com dois avisos não lidos, `GET` traz `unread: 2`; depois de `POST /notifications/read`, `GET` traz `unread: 0` e `read: true` nos dois.

---

### P1: Sino no topo ⭐ MVP

**User Story**: Como usuária logada, quero ver no topo da página quantos avisos não li e abrir a lista.

**Why P1**: É como a pessoa recebe o aviso.

**Acceptance Criteria**:

1. WHILE a usuária está logada the Header SHALL exibir um botão de sino com nome acessível "Avisos"; deslogada, SHALL não exibir. [NOTIFY-12]
2. WHILE há avisos não lidos the sino SHALL exibir o número (ou "9+" acima de 9) e o nome acessível "Avisos (<N> não lidos)"; sem não lidos, SHALL não exibir número. [NOTIFY-13]
3. WHEN a usuária clica no sino THEN the front SHALL buscar os avisos, abrir o painel "Avisos" com as mensagens do mais novo ao mais antigo, cada uma com data e hora (`DD/MM HH:mm`), e, se havia não lidos, SHALL chamar `POST /notifications/read` e zerar o número. [NOTIFY-14]
4. WHILE não há avisos the painel SHALL exibir "Nenhum aviso por enquanto.". [NOTIFY-15]
5. WHEN a usuária clica de novo no sino THEN the painel SHALL fechar. [NOTIFY-16]
6. IF buscar os avisos falha THEN the sino SHALL continuar sem número e o painel SHALL exibir "Não foi possível carregar os avisos." [NOTIFY-17]

**Independent Test**: Logada como Ana com 1 aviso não lido, o sino mostra "1"; ao clicar, o painel mostra a mensagem e o número some.

---

## Edge Cases

- IF a carona excluída tinha 3 participantes THEN the API SHALL gravar 3 avisos, um por participante (NOTIFY-01).
- IF a dona remove a si mesma ou alguém não inscrito THEN a API responde `404` e SHALL não gravar aviso (NOTIFY-06).
- IF `POST /notifications/read` é chamado sem avisos não lidos THEN the API SHALL responder `204` sem erro (NOTIFY-09).
- IF a conta tem mais de 30 avisos THEN `items` SHALL trazer só os 30 mais recentes, e `unread` SHALL contar todos os não lidos (NOTIFY-07).

---

## Implicit-Requirement Sweep

| Dimension | Result |
| --------- | ------ |
| Input validation & bounds | Rotas sem body. Limite de 30 itens (NOTIFY-07) |
| Failure / partial-failure states | NOTIFY-05 (aviso falha, ação não), NOTIFY-17 (front) |
| Idempotency / retry / duplicate handling | `POST /notifications/read` é idempotente (NOTIFY-09) |
| Auth boundaries & rate limits | NOTIFY-09/10/11: cada conta só lê e marca os próprios avisos. Rate limit: N/A because nenhuma rota do projeto tem limite |
| Concurrency / ordering | N/A because cada evento grava avisos novos sem atualizar linhas existentes; marcar lido é um único `UPDATE` |
| Data lifecycle / expiry | Sem expiração (Out of Scope). `ON DELETE CASCADE` em `user_id` |
| Observability | NOTIFY-05: falha ao gravar aviso vai para o log da API (`Logger` do Nest) |
| External-dependency failure | N/A because não há serviço externo (sem e-mail) |
| State-transition integrity | Aviso: não lido → lido (NOTIFY-09/14). Não volta a não lido |

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| -------------- | ----- | ----- | ------ |
| NOTIFY-01 | P1: Avisos gravados nos três eventos | Execute | Implemented (T2, T4) |
| NOTIFY-02 | P1: Avisos gravados nos três eventos | Execute | Implemented (T2, T4) |
| NOTIFY-03 | P1: Avisos gravados nos três eventos | Execute | Implemented (T2, T5) |
| NOTIFY-04 | P1: Avisos gravados nos três eventos | Execute | Implemented (T2, T5) |
| NOTIFY-05 | P1: Avisos gravados nos três eventos | Execute | Implemented (T2, T4, T5) |
| NOTIFY-06 | P1: Avisos gravados nos três eventos | Execute | Implemented (T4, T5) |
| NOTIFY-07 | P1: Ler avisos pela API | Execute | Implemented (T2, T3) |
| NOTIFY-08 | P1: Ler avisos pela API | Execute | Implemented (T3) |
| NOTIFY-09 | P1: Ler avisos pela API | Execute | Implemented (T2, T3) |
| NOTIFY-10 | P1: Ler avisos pela API | Execute | Implemented (T3) |
| NOTIFY-11 | P1: Ler avisos pela API | Execute | Implemented (T3) |
| NOTIFY-12 | P1: Sino no topo | Execute | Implemented (T8) |
| NOTIFY-13 | P1: Sino no topo | Execute | Implemented (T7) |
| NOTIFY-14 | P1: Sino no topo | Execute | Implemented (T6, T7) |
| NOTIFY-15 | P1: Sino no topo | Execute | Implemented (T7) |
| NOTIFY-16 | P1: Sino no topo | Execute | Implemented (T7) |
| NOTIFY-17 | P1: Sino no topo | Execute | Implemented (T7) |

**Coverage:** 17 total, 17 mapped to tasks, 0 unmapped

---

## Success Criteria

- [ ] Excluir carona, sair e remover passageira geram aviso para a pessoa afetada, inclusive sem telefone.
- [ ] O sino mostra os não lidos e zera ao abrir.
- [ ] Derrubar a gravação de avisos não quebra nenhuma das três ações.
