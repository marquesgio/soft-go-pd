# Vou Junto com Login Context

**Gathered:** 2026-10-07
**Spec:** `.specs/features/vou-junto/spec.md`
**Status:** Ready for design

---

## Feature Boundary

A inscrição em carona ("Quero ir junto!") passa a usar a usuária logada. Ela exige login, impede presença duplicada por conta e mostra os participantes no card. O telefone de contato passa a ficar na conta (`users.phone`). Publicar carona continua igual e aberto.

---

## Implementation Decisions

### A. Telefone

- Coluna nova e opcional `users.phone`.
- O cadastro (`/cadastro`) ganha o campo "WhatsApp (opcional)".
- O modal pede o WhatsApp **só** quando a conta não tem telefone. Se preenchido, o número é salvo na conta e vale para as próximas vezes. Isso resolve as contas antigas sem precisar de tela de perfil.
- O formulário "Publicar carona" não muda. O WhatsApp dele continua sendo o contato da carona (`rides.phone`).
- `ride_users` deixa de guardar nome e telefone: as colunas `name`, `phone` e a constraint `UQ_ride_phone` são removidas. A fonte desses dados é a conta.

### A2. Inscrições antigas

- Decisão da usuária: as inscrições antigas não continuam valendo. A migration apaga as linhas sem `user_id` e torna `user_id` `NOT NULL`. Apagar não tem volta.

### B. Listagem de participantes

- Os participantes aparecem no card da carona, com o primeiro nome de cada um.
- O `GET /ride` devolve `participants: [{ userId, name, phone? }]`.

### C. Privacidade

- O WhatsApp dos participantes só aparece para usuárias logadas.
- A API só inclui `phone` quando a requisição ao `GET /ride` traz token válido. Sem token, ou com token inválido, a rota continua `200` e responde como para quem está deslogada.

### D. Autenticação da inscrição

- `POST /ride-users` exige token. Isso substitui parte do AUTH-23; registrar em `STATE.md`.
- No front, quem está deslogada e clica "Vou junto" vai para `/login`.

### Agent's Discretion

- Texto e layout da lista de participantes no card e do aviso "Você vai nesta carona".
- Como o `GET /ride` lê o token de forma opcional (guard opcional ou leitura no controller).
- Ordem de gravação telefone/inscrição (em transação ou com checagem prévia).

### Declined / Undiscussed Gray Areas → Assumptions

Registrados na tabela Assumptions do spec:
- redirecionar quem está deslogada para `/login`;
- "Você vai nesta carona" para quem já confirmou;
- texto do 409 de duplicidade;
- telefone validado com as regras atuais;
- token inválido no `GET /ride` é tratado como deslogada.

---

## Specific References

- O modal atual (`soft-go-II/src/components/Modal.tsx`) mantém o resumo da carona (`Card` com `showButton={false}`) e o botão "Confirmar presença".
- O link de WhatsApp segue o padrão do card: `https://wa.me/<phone>`.

---

## Deferred Ideas

- Cancelar presença.
- Tela de editar perfil (nome e telefone).
- Vincular a carona à usuária que a publicou e exigir login para publicar.
- Botão "Vou junto" em caronas de ônibus. Bug que já existia: o botão só aparece com vagas limitadas.
