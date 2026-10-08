# Excluir Carona Specification

**PRD**: `prds/delete-rides.md`
**Context**: `.specs/features/delete-rides/context.md`
**Status**: In Progress (2026-10-08)

## Problem Statement

Hoje não existe forma de excluir uma carona publicada por engano ou que não vai mais acontecer: ela fica no mural até a data passar. Com a dona gravada em `rides.owner_id` (save-rideowner), dá para liberar a exclusão só para quem publicou. A API precisa garantir essa regra mesmo quando alguém chama o endpoint direto, sem passar pelo front.

## Goals

- [ ] A dona exclui a própria carona pelo card, e ela some do mural de todo mundo.
- [ ] A API só exclui quando o token é da dona: `403` para outras contas, `404` para carona inexistente.
- [ ] Antes de excluir, a dona vê quantas pessoas confirmaram presença e quem não tem WhatsApp para ser avisada.
- [ ] A regra sobre os participantes confirmados fica documentada (cascade, ver Assumptions e `context.md`).

## Out of Scope

| Feature | Reason |
| ------- | ------ |
| Avisar participantes por e-mail ou notificação | Funcionalidade nova. Virou o card `prds/notify-ride-deleted.md` |
| Editar carona | Não está no PRD |
| Desfazer exclusão / exclusão lógica (soft delete) | O PRD pede que a carona suma; não há "lixeira" |
| Participante sair da carona | Não está no PRD. Virou o card `prds/leave-ride.md` (depois de `notify-ride-deleted`) |
| Página "minhas caronas" | Não está no PRD. A dona exclui pelo card no mural |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --------------------- | -------------- | --------- | ---------- |
| Participantes já confirmados | **Cascade**: excluir a carona apaga as inscrições dela (`ride_users`) pelo `ON DELETE CASCADE` que já existe em `FK_ride_users_ride`. A exclusão não é bloqueada por haver participantes | Decisão da usuária. Bloquear deixaria no mural uma carona que não vai acontecer, sem saída para a dona (ninguém consegue sair de uma carona hoje). Com o cascade, a participante fica livre para entrar em outra carona no mesmo dia | y |
| Aviso antes de excluir | Modal de confirmação mostra quantas pessoas confirmaram e o nome das que não têm WhatsApp. A dona avisa por fora quem tem telefone (ela já vê os links no card) | Decisão da usuária. Sem sistema de notificação, a dona é o único canal de aviso | y |
| Participante sem telefone | Não recebe aviso ativo. Descobre ao abrir o mural: a carona sumiu e o selo "Você vai nesta carona" não aparece mais | Limitação aceita pela usuária. O aviso ativo é o card `prds/notify-ride-deleted.md` | y |
| Rota | `DELETE /ride/:id`, no controller existente (`@Controller('ride')`), em vez do `/rides/:id` do PRD | Todas as rotas de carona usam `/ride`. Uma rota no plural só para exclusão quebraria o padrão | n (padrão do agente) |
| Ordem das checagens | Token (`401`) → id inteiro (`400`) → carona existe (`404`) → é a dona (`403`) → exclui (`204`) | `404` antes de `403`: as caronas já são públicas no mural, então revelar que o id existe não vaza nada | n (padrão do agente) |
| Mensagens | `404` "Carona não encontrada" · `403` "Só a dona da carona pode excluí-la" · `401` "Não autenticado" (guard atual) | PT-BR, exibidas no toast do front | n (padrão do agente) |
| Carona com data passada | A API exclui normalmente se for da dona | Não aparece no mural, então o front nunca oferece; não há motivo para a API recusar | n (padrão do agente) |
| Duplo clique / exclusão repetida | O botão de confirmar fica desabilitado enquanto envia. Uma segunda chamada para a mesma carona responde `404` | Não há estado intermediário a proteger: o `DELETE` é uma operação só | n (padrão do agente) |
| Carona excluída por outra aba antes de confirmar | `404` → toast "Carona não encontrada" e o mural recarrega | O mural fica igual ao banco | n (padrão do agente) |

**Open questions:** none.

---

## User Stories

### P1: API exclui só para a dona ⭐ MVP

**User Story**: Como dona de uma carona, quero excluí-la quando ela não vai mais acontecer, e quero que ninguém mais consiga excluí-la.

**Why P1**: Critérios de aceite 2 e 3 do PRD. É a garantia real; o front só esconde o botão.

**Acceptance Criteria**:

1. WHEN `DELETE /ride/:id` recebe o token válido da dona da carona THEN the API SHALL apagar a carona e responder `204` com corpo vazio. [DEL-01]
2. WHEN uma carona é excluída THEN the API SHALL apagar junto todas as inscrições dela em `ride_users`, e `GET /ride` SHALL deixar de listar a carona. [DEL-02]
3. IF a carona `:id` não existe THEN the API SHALL responder `404` com a mensagem "Carona não encontrada". [DEL-03]
4. IF o token é válido mas a conta não é a dona da carona THEN the API SHALL responder `403` com a mensagem "Só a dona da carona pode excluí-la", e a carona e suas inscrições SHALL continuar gravadas. [DEL-04]
5. IF `DELETE /ride/:id` chega sem `Authorization`, com token inválido, expirado ou de sessão revogada THEN the API SHALL responder `401` com a mensagem "Não autenticado" e nenhuma carona SHALL ser apagada. [DEL-05]
6. IF `:id` não é um número inteiro THEN the API SHALL responder `400` e nenhuma carona SHALL ser apagada. [DEL-06]
7. WHILE `DELETE /ride/:id` exige token, `GET /ride` and `GET /transport-type` SHALL continuar respondendo sem token, e nenhum guard global SHALL ser registrado. [DEL-07]

**Independent Test**: Com dois tokens (dona e outra conta), `curl -X DELETE /ride/<id>` com o token da outra conta responde `403` e a carona continua no `GET /ride`; com o token da dona responde `204`, a carona some do `GET /ride` e `SELECT count(*) FROM ride_users WHERE ride_id = <id>` retorna 0.

---

### P1: Dona exclui pelo card ⭐ MVP

**User Story**: Como dona, quero um botão de excluir no card da minha carona, com uma confirmação que me lembre quem já confirmou presença.

**Why P1**: Critério de aceite 1 do PRD e a decisão sobre participantes.

**Acceptance Criteria**:

1. WHEN o card mostra uma carona cuja dona é a usuária logada THEN the front SHALL exibir o botão "Excluir carona" junto do selo "Sua carona". [DEL-08]
2. WHILE a usuária está deslogada ou não é a dona da carona the front SHALL não exibir o botão "Excluir carona" no card. [DEL-09]
3. WHEN a dona clica em "Excluir carona" THEN the front SHALL abrir um modal com o título "Excluir carona?", o resumo da carona (dona, cidade, hora, transporte) e os botões "Cancelar" e "Excluir". [DEL-10]
4. WHILE a carona tem participantes the modal SHALL exibir "1 pessoa confirmou presença e será removida da carona." ou "N pessoas confirmaram presença e serão removidas da carona.", e WHILE não tem, SHALL exibir "Ninguém confirmou presença ainda.". [DEL-11]
5. WHILE algum participante não tem WhatsApp the modal SHALL listar os nomes dessas pessoas em "Sem WhatsApp para avisar: <nomes separados por vírgula>", e WHILE todos têm, SHALL não exibir essa linha. [DEL-12]
6. WHEN a dona clica em "Cancelar", no "X" ou fora do modal THEN the front SHALL fechar o modal sem chamar a API. [DEL-13]
7. WHEN a dona clica em "Excluir" THEN the front SHALL enviar `DELETE /ride/<id>` com o token, desabilitar o botão enquanto envia, e, no `204`, exibir o toast de sucesso "Carona excluída", fechar o modal e recarregar o mural sem a carona. [DEL-14]
8. IF a API responde `403` ou `404` THEN the front SHALL exibir no toast de erro a mensagem da API, fechar o modal e recarregar o mural. [DEL-15]
9. IF a API falha com outro erro (exceto `401`) THEN the front SHALL exibir o toast "Não foi possível excluir a carona. Tente novamente." e manter a carona no mural. IF a API responde `401` THEN the front SHALL deixar o interceptor tratar (toast de sessão + `/login`). [DEL-16]

**Independent Test**: Logada como dona, a carona mostra "Excluir carona"; o modal informa "2 pessoas confirmaram presença…" e "Sem WhatsApp para avisar: Bia"; "Excluir" mostra "Carona excluída" e a carona some. Logada como outra conta, o mesmo card não tem o botão.

---

## Edge Cases

- IF a carona é do tipo ônibus (vagas ilimitadas) THEN the front SHALL oferecer a exclusão normalmente para a dona (DEL-08).
- IF a dona clica duas vezes em "Excluir" THEN the front SHALL enviar uma única requisição, porque o botão fica desabilitado enquanto envia (DEL-14).
- IF a carona já foi excluída em outra aba THEN a API SHALL responder `404` e the front SHALL recarregar o mural (DEL-03, DEL-15).
- IF a requisição de outra conta chega com um body qualquer THEN the API SHALL responder `403` sem apagar nada; o `DELETE` não lê body (DEL-04).

---

## Implicit-Requirement Sweep

| Dimension | Result |
| --------- | ------ |
| Input validation & bounds | DEL-06 (`:id` não inteiro → `400`) |
| Failure / partial-failure states | DEL-02: a carona e as inscrições saem num único `DELETE` (cascade no banco), sem estado parcial. DEL-15/16 no front |
| Idempotency / retry / duplicate handling | Repetir o `DELETE` responde `404` (Assumptions). Front evita duplo envio (DEL-14) |
| Auth boundaries & rate limits | DEL-04/05 (posse e token), DEL-07 (rotas abertas continuam abertas, sem guard global). Rate limit: N/A because nenhuma rota do projeto tem limite e o PRD não pede |
| Concurrency / ordering | Inscrição chegando durante a exclusão: N/A because o `INSERT` em `ride_users` falha pela FK depois que a carona some, e a API já trata carona inexistente como `404` |
| Data lifecycle / expiry | DEL-02 (inscrições apagadas junto). Exclusão é definitiva, sem soft delete (Out of Scope) |
| Observability | N/A because o projeto não tem logging ou métricas estruturadas e o PRD não pede |
| External-dependency failure | N/A because não há chamada externa nova; aviso por e-mail ficou para `prds/notify-ride-deleted.md` |
| State-transition integrity | Carona: publicada → excluída (terminal). DEL-04 impede a transição por quem não é dona |

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| -------------- | ----- | ----- | ------ |
| DEL-01 | P1: API exclui só para a dona | Execute | Implemented (T1, T2) |
| DEL-02 | P1: API exclui só para a dona | Execute | Implemented (T1, T2) |
| DEL-03 | P1: API exclui só para a dona | Execute | Implemented (T1, T2) |
| DEL-04 | P1: API exclui só para a dona | Execute | Implemented (T1, T2) |
| DEL-05 | P1: API exclui só para a dona | Execute | Implemented (T2) |
| DEL-06 | P1: API exclui só para a dona | Execute | Implemented (T2) |
| DEL-07 | P1: API exclui só para a dona | Execute | Implemented (T2) |
| DEL-08 | P1: Dona exclui pelo card | Tasks | Mapped (T4, T6) |
| DEL-09 | P1: Dona exclui pelo card | Execute | Implemented (T4) |
| DEL-10 | P1: Dona exclui pelo card | Tasks | Mapped (T5, T6) |
| DEL-11 | P1: Dona exclui pelo card | Tasks | Mapped (T5) |
| DEL-12 | P1: Dona exclui pelo card | Tasks | Mapped (T5) |
| DEL-13 | P1: Dona exclui pelo card | Tasks | Mapped (T5, T6) |
| DEL-14 | P1: Dona exclui pelo card | Tasks | Mapped (T3, T5, T6) |
| DEL-15 | P1: Dona exclui pelo card | Tasks | Mapped (T6) |
| DEL-16 | P1: Dona exclui pelo card | Tasks | Mapped (T6) |

**Coverage:** 16 total, 16 mapped to tasks, 0 unmapped

---

## Success Criteria

- [ ] Só a dona vê o botão "Excluir carona", e só o token dela consegue `204` no `DELETE /ride/:id`.
- [ ] Chamar a API direto com o token de outra conta responde `403` e não apaga nada.
- [ ] Depois da exclusão, `SELECT count(*) FROM ride_users WHERE ride_id = <id>` retorna 0.
- [ ] A regra dos participantes (cascade + aviso no modal) está registrada nesta spec, no `context.md` e na AD-006.
