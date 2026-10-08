# Excluir Carona Context

**Gathered:** 2026-10-08
**Spec:** `.specs/features/delete-rides/spec.md`
**Status:** Done

---

## Feature Boundary

A dona de uma carona pode excluí-la pelo card no mural. A API (`DELETE /ride/:id`) só aceita o pedido quando o token é da dona: `403` para outras contas, `404` para carona inexistente, `401` sem login. A carona some do mural de todo mundo.

---

## Implementation Decisions

### Participantes já confirmados (a decisão principal do PRD)

- **Cascade.** Excluir a carona apaga as inscrições dela. Não usamos código novo: o banco já tem `ON DELETE CASCADE` em `ride_users.ride_id` (`FK_ride_users_ride`, migration `CreateTable` e entidade `RideUser`). A exclusão nunca é bloqueada por haver participantes.
- **Por que não bloquear:** o PRD fala de uma carona que "não vai mais acontecer". Se a exclusão fosse bloqueada, ela continuaria no mural e as pessoas inscritas achariam que têm carona. A dona também ficaria sem saída, porque hoje ninguém consegue sair de uma carona. Bloquear não avisa ninguém, só esconde o problema.
- **Efeito colateral bom:** a regra "já confirmou presença em uma carona neste dia" (`RideService.create`, `RideUsersService.join`) deixa de prender a participante, que pode entrar em outra carona no mesmo dia.

### Aviso antes de excluir

- Um modal de confirmação ("Excluir carona?") mostra o resumo da carona, quantas pessoas confirmaram presença e o nome de quem não tem WhatsApp ("Sem WhatsApp para avisar: Ana, Bia").
- Assim, a dona pode avisar pelo WhatsApp quem tem telefone (os links já aparecem no card para quem está logada) antes de confirmar.
- Sem participantes: "Ninguém confirmou presença ainda."

### Participante sem telefone

- Hoje não há aviso ativo para ela. Ela descobre pelo mural: a carona some e o selo "Você vai nesta carona" não aparece mais.
- A usuária aceitou essa limitação nesta feature e pediu o card de aviso ativo: `prds/notify-ride-deleted.md`.

### Agent's Discretion

- Rota `DELETE /ride/:id` (singular, como o resto do controller), não `/rides/:id` do PRD.
- Mensagens: `404` "Carona não encontrada"; `403` "Só a dona da carona pode excluí-la"; toast de sucesso "Carona excluída"; erro genérico "Não foi possível excluir a carona. Tente novamente."
- Ordem das checagens na API: `401` → `400` → `404` → `403` → `204`.
- Botão "Excluir carona" ao lado do selo "Sua carona", com ícone `Trash2` (lucide). A cor destrutiva usa o token novo `support-04` (`#dc2626`), criado no `@theme` para erro a pedido da usuária.
- O modal de confirmação é um componente novo (`ConfirmDeleteRideModal`), separado do `Modal` de inscrição, com o mesmo visual (overlay, "X", `Card` com `showButton={false}`).
- `403`/`404` no front: toast com a mensagem da API e recarga do mural.

### Declined / Undiscussed Gray Areas → Assumptions

- Carona com data passada, duplo clique e exclusão em outra aba: padrões do agente, registrados nas Assumptions da spec.

---

## Specific References

- Visual e comportamento do modal: seguir `components/Modal.tsx` (overlay `bg-primary/60`, `role="dialog"`, fechar no "X" e fora do modal).
- Tratamento de erro no Home: seguir `handleConfirm` em `pages/Home.tsx` (`401` fica com o interceptor; `404`/`409` → toast com a mensagem da API).
- Rota protegida com status sem corpo: seguir `POST /auth/signout` (`@HttpCode(HttpStatus.NO_CONTENT)`).

---

## Deferred Ideas

- **Avisar participantes quando a carona for excluída** (e-mail ou notificação no app). A usuária pediu o card agora: `prds/notify-ride-deleted.md`. Fica fora desta feature.
- **Passageira sair da carona.** Card `prds/leave-ride.md`, que vem depois do de aviso: a dona é avisada pelo mesmo canal quando alguém sai. Não muda a decisão do cascade: carona cancelada precisa sumir do mural de qualquer forma.
