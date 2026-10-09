# Sair da Carona Context

**Gathered:** 2026-10-08
**Spec:** `.specs/features/leave-ride/spec.md`
**Status:** Done

---

## Feature Boundary

A passageira sai de uma carona, e a dona remove passageiras da própria carona, as duas pela lista de passageiros do card. A lista passa a ficar recolhida. A API (`DELETE /ride-users/:rideId/users/:userId`) só aceita o pedido da própria passageira ou da dona.

---

## Implementation Decisions

### Quem remove

- A própria passageira (sair) e a dona (remover qualquer passageira da carona dela). Decisão da usuária, que amplia o PRD `leave-ride`.
- Uma rota só para os dois casos. A permissão é "token = `:userId`" ou "token = dona da carona".

### Lista de passageiros

- Recolhida por padrão no card, com o botão "Ver passageiros (N)" / "Esconder passageiros". Pedido da usuária.
- A lixeira fica dentro da lista, à direita do nome: no próprio nome para a passageira, em todos os nomes para a dona. Pedido da usuária.

### Aviso

- Nesta feature ninguém é avisado. A usuária decidiu fazer o leave-ride antes do card de aviso. O `prds/notify-ride-deleted.md` passa a cobrir três eventos: carona excluída (avisa participantes), passageira saiu (avisa a dona) e passageira removida (avisa a passageira).

### Agent's Discretion

- Textos: "Ver passageiros (N)", "Esconder passageiros", "Sair da carona?", "Remover <nome> da carona?", botões "Sair"/"Remover", toasts "Você saiu da carona", "<nome> foi removida da carona", "Não foi possível remover da carona. Tente novamente."
- Mensagens da API: `404` "Carona não encontrada", `403` "Você só pode sair da carona ou remover passageiras da sua carona", `404` "Esta pessoa não está nesta carona".
- Confirmação num componente genérico novo (`ConfirmDialog`: título, texto do botão, `onConfirm`). O `ConfirmDeleteRideModal` fica como está.
- A lista vira um componente próprio (`ParticipantsList`), usado pelo `Card`.
- A lixeira usa o token `support-04`, como a de excluir carona.

### Declined / Undiscussed Gray Areas → Assumptions

- Passageira removida pode se inscrever de novo: sim, se houver vaga (registrado no Out of Scope da spec).

---

## Specific References

- Lixeira e modal: seguir o ícone de excluir carona em `components/Card.tsx` (T7 de delete-rides) e o `ConfirmDeleteRideModal`.
- Tratamento de erro: seguir `handleDelete` em `pages/Home.tsx`.
- Rota protegida com `204`: seguir `RideController.removeRide`.

---

## Deferred Ideas

- Avisos de saída e remoção: card `prds/notify-ride-deleted.md`.
