# Avisos Context

**Gathered:** 2026-10-08
**Spec:** `.specs/features/notify-ride-deleted/spec.md`
**Status:** Done

---

## Feature Boundary

Avisos dentro do app para três eventos: carona excluída (avisa as participantes), passageira saiu (avisa a dona) e passageira removida (avisa a passageira). Os avisos ficam numa tabela nova e aparecem num sino no Header.

---

## Implementation Decisions

### Canal

- Só aviso no app. Sem e-mail. Decisão da usuária: não depende de serviço externo, alcança toda conta e dá para testar localmente. O custo é que a pessoa só vê o aviso quando abre o app.

### Como ver e limpar

- Sino no Header com o número de não lidos. Clicar abre o painel "Avisos", com a lista do mais novo ao mais antigo, e marca todos como lidos. Decisão da usuária.

### Texto gravado, não montado na leitura

- A carona excluída some do banco. Por isso a frase do aviso é montada e gravada no momento da ação (`message`), com primeiro nome, data, hora e cidade.

### A ação vem antes do aviso

- A ação (excluir, sair, remover) acontece primeiro. O aviso é gravado depois, num `try/catch` que registra o erro no log. Assim uma falha no aviso nunca desfaz nem bloqueia a ação (PRD).
- Na exclusão, as participantes são lidas **antes** do `DELETE`, porque o cascade apaga as inscrições.

### Agent's Discretion

- Tabela `notifications` (`id`, `user_id` FK `users` com `ON DELETE CASCADE`, `type` varchar(30), `message` varchar(300), `created_at` timestamptz default `now()`, `read_at` timestamptz nulo).
- Rotas `GET /notifications` (30 mais recentes + `unread`) e `POST /notifications/read` (`204`).
- Badge "9+" acima de 9; data e hora no painel em `DD/MM HH:mm` (`date-fns`).
- Atualiza ao carregar a página logada e ao abrir o sino; sem tempo real.
- Componente `NotificationBell` no Header.

### Declined / Undiscussed Gray Areas → Assumptions

- Expiração e exclusão de avisos: fora de escopo (registrado na spec).

---

## Specific References

- Módulo novo seguindo `src/ride/` e a skill `/novo-recurso` da API; migration pela skill `/migration`.
- Visual do painel: mesmos tokens do Header (`bg-white`, `text-secondary-text`, `shadow-md`).

---

## Deferred Ideas

- E-mail como segundo canal, se o aviso no app não bastar.
- Atualização em tempo real do sino.
