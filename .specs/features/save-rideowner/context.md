# Dona da Carona Context

**Gathered:** 2026-10-08
**Spec:** `.specs/features/save-rideowner/spec.md`
**Status:** Ready for design

---

## Feature Boundary

Toda carona publicada fica ligada à usuária logada que a criou (`rides.owner_id`). O formulário deixa de pedir nome e telefone, `POST /ride` passa a exigir login, e o card mostra a dona com dados vindos de `users`.

---

## Implementation Decisions

### Caronas antigas e migration

- A migration apaga todas as caronas existentes, e as inscrições saem junto pelo cascade. Depois cria `owner_id NOT NULL` com FK para `users`.
- As colunas `rides.name` e `rides.phone` são removidas.

### Exibição da dona

- O card mostra o nome completo da dona (`users.name`).
- O WhatsApp da dona só aparece para quem está logada (mesma regra da AD-003).

### WhatsApp no formulário

- Fica como campo opcional, decidido pela usuária na aprovação. Funciona como no modal "Vou junto": aparece só quando a conta não tem telefone, e o número preenchido é gravado em `users.phone` e atualizado na sessão.

### Própria carona

- A dona não se inscreve na própria carona. No lugar de "Vou junto", o card mostra "Sua carona". A API responde `409` se ela tentar mesmo assim.

### Agent's Discretion

- Formato `owner: { id, name, phone? }` na resposta de `GET /ride`.
- Mensagem do `409`: "Você não pode se inscrever na sua própria carona".
- Deslogada: "Vou para a soft" e acesso direto a `/form` levam para `/login`.
- Token válido de conta inexistente: `401`.

### Declined / Undiscussed Gray Areas → Assumptions

- Sessão expirada no meio do formulário: fica com o fluxo atual do interceptor. Registrado nas Assumptions da spec.

---

## Specific References

- Seguir o padrão da feature vou-junto: migration que apaga os dados sem conta (`LinkRideUsersToUsers`), telefone condicionado ao login (`toParticipant` em `ride.service.ts`) e o estado do card no lugar do botão ("Você vai nesta carona").

---

## Deferred Ideas

None. A discussão ficou dentro do escopo da feature.
