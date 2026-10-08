# Revogar Sessões ao Sair Specification

**PRD**: `prds/revoke-token.md`
**Status**: Done (2026-10-08). Verifier PASS na iteração 2 (`validation.md`)

## Problem Statement

O "Sair" só apaga o token do navegador. A API aceita qualquer token com assinatura válida até expirar (7 dias), então uma cópia do token continua publicando caronas, inscrevendo e vendo telefones depois que a usuária saiu. Agora que o token libera ações e dados pessoais, sair precisa invalidar de verdade os tokens da conta.

## Goals

- [ ] Depois de "Sair", nenhum token emitido antes para a conta é aceito pela API, em nenhum aparelho.
- [ ] Entrar de novo funciona na hora, sem perder nada da conta.
- [ ] Sair de uma conta não afeta outras contas.

## Out of Scope

| Feature | Reason |
| ------- | ------ |
| Sair só deste aparelho / lista de aparelhos conectados | Decisão da usuária: "Sair" encerra todos os aparelhos. Revogar por aparelho exigiria guardar cada sessão no servidor |
| Tela de troca de senha | Não existe. Quando existir, deve incrementar `token_version` |
| Refresh token / renovação silenciosa | Não está no PRD |
| Cookie httpOnly no lugar do `localStorage` | Não está no PRD. A dívida de XSS segue registrada (AD-001) |
| Revogação por inatividade / mudança da validade de 7 dias | Não está no PRD |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --------------------- | -------------- | --------- | ---------- |
| Alcance do "Sair" | Encerra a sessão em todos os aparelhos da conta | Decisão da usuária | y |
| Mecanismo | Coluna `users.token_version` (`integer NOT NULL DEFAULT 0`). O token carrega `ver`. "Sair" incrementa a coluna | Opção 1 escolhida pela usuária | y |
| Rota de saída | `POST /auth/signout`, com `JwtAuthGuard`, responde `204` sem corpo | Mesmo prefixo das rotas de auth. Exige token válido: só a dona da sessão encerra as próprias sessões | n (padrão do agente) |
| Front ao sair | Chama `POST /auth/signout` e, com sucesso ou falha (rede, 401, 500), apaga a sessão local e vai para `/login` | Sair nunca pode travar a usuária logada. Se a chamada falhar, o comportamento é o de hoje | n (padrão do agente) |
| Tokens emitidos antes da feature (sem `ver`) | Recusados: `401` nas rotas com login, anônima no mural | Sem versão não há como validar. Custo: todas entram de novo uma vez (está no PRD) | n (padrão do agente) |
| Consulta ao banco por requisição | Os dois guards leem `users.token_version` a cada requisição com token | É o custo da opção 1. Substitui a parte da AD-001 que dizia que o logout é só no cliente e que o guard não consulta o banco | n (padrão do agente) |
| Conta inexistente com token de assinatura válida | `401` "Não autenticado" nas rotas com login, anônima no mural | Consequência de ler a conta no guard | n (padrão do agente) |
| Login em outro aparelho no mesmo instante do "Sair" | Se o login ler a versão antes do incremento, esse token novo também cai | Corrida rara. Basta entrar de novo | n (padrão do agente) |
| Mensagem do 401 | "Não autenticado", a mesma de hoje | Não revela se o token expirou, foi revogado ou é falso | n (padrão do agente) |

**Open questions:** none. Todas foram resolvidas ou registradas acima.

---

## User Stories

### P1: Sair invalida os tokens da conta ⭐ MVP

**User Story**: Como usuária, quero que "Sair" invalide todos os tokens da minha conta, para ninguém usar minha sessão depois.

**Why P1**: É o PRD inteiro.

**Acceptance Criteria**:

1. WHEN `POST /auth/signout` recebe um token válido THEN the API SHALL incrementar em 1 o `token_version` da conta do token e responder `204` sem corpo. [REVOKE-01]
2. IF `POST /auth/signout` chega sem token, com token inválido ou com token já revogado THEN the API SHALL responder `401` "Não autenticado" e o `token_version` de nenhuma conta SHALL mudar. [REVOKE-02]
3. WHEN a usuária clica em "Sair" THEN the front SHALL chamar `POST /auth/signout` com o token da sessão, apagar a sessão local e navegar para `/login`. [REVOKE-03]
4. IF `POST /auth/signout` falha (erro de rede, `401` ou `500`) THEN the front SHALL apagar a sessão local e navegar para `/login` mesmo assim. [REVOKE-04]

**Independent Test**: Logada em duas abas com cópias do mesmo token, clicar "Sair" numa delas. Na outra, a próxima ação que exige login responde `401` e leva para `/login`. No banco, `users.token_version` subiu 1.

---

### P1: A API só aceita a versão atual ⭐ MVP

**User Story**: Como sistema, quero recusar tokens de versão antiga, para que sair tenha efeito em todos os aparelhos.

**Why P1**: Sem a checagem, incrementar a versão não muda nada.

**Acceptance Criteria**:

1. WHEN o cadastro ou o login emitem um token THEN the token SHALL conter `ver` igual ao `token_version` atual da conta. [REVOKE-05]
2. IF uma rota com `JwtAuthGuard` recebe um token cujo `ver` é diferente do `token_version` atual da conta THEN the API SHALL responder `401` "Não autenticado". [REVOKE-06]
3. IF uma rota com `JwtAuthGuard` recebe um token sem `ver` THEN the API SHALL responder `401` "Não autenticado". [REVOKE-07]
4. IF uma rota com `JwtAuthGuard` recebe um token de assinatura válida cuja conta não existe THEN the API SHALL responder `401` "Não autenticado". [REVOKE-08]
5. IF uma rota com `OptionalJwtAuthGuard` recebe um token revogado, sem `ver` ou de conta inexistente THEN the API SHALL responder como deslogada: `GET /ride` sem a chave `phone` em `owner` e nos `participants`. [REVOKE-09]
6. WHEN um token tem `ver` igual ao `token_version` atual THEN the API SHALL aceitar o token como hoje (`JwtAuthGuard` libera a rota; `OptionalJwtAuthGuard` preenche a usuária). [REVOKE-10]

**Independent Test**: Gerar um token, incrementar `token_version` da conta no banco e chamar `GET /auth/me` (`401`) e `GET /ride` (sem telefones). Um token novo, emitido depois do incremento, responde `200` nas duas.

---

### P1: Entrar de novo e outras contas ⭐ MVP

**User Story**: Como usuária, quero entrar de novo depois de sair, ou trocar de conta, sem perder nada.

**Why P1**: É o critério de aceite 4 e 5 do PRD, e a preocupação da usuária na aprovação.

**Acceptance Criteria**:

1. WHEN a usuária entra de novo depois de sair THEN the API SHALL emitir um token com o `ver` novo, aceito pelas rotas com login. [REVOKE-11]
2. WHEN a conta A sai THEN the tokens da conta B SHALL continuar aceitos. [REVOKE-12]
3. The signout SHALL alterar só `token_version` da conta: nome, e-mail, telefone, hash de senha, caronas e inscrições SHALL ficar iguais. [REVOKE-13]

**Independent Test**: Entrar na conta A e depois na B. Sair da B. Entrar na A de novo e publicar uma carona (`201`). O telefone e as caronas da B continuam iguais no banco.

---

### P1: Migration da versão ⭐ MVP

**User Story**: Como sistema, quero a coluna de versão com valor inicial, sem afetar as contas existentes.

**Why P1**: Base de todas as outras histórias.

**Acceptance Criteria**:

1. WHEN a migration roda `up()` THEN the table `users` SHALL ganhar `token_version integer NOT NULL DEFAULT 0`, com as contas existentes em `0` e nenhum outro dado alterado. [REVOKE-14]
2. WHEN a migration roda `down()` THEN the table `users` SHALL ficar sem a coluna `token_version`, sem apagar contas. [REVOKE-15]
3. The `GET /auth/me`, signup e signin SHALL continuar sem expor `token_version` nas respostas. [REVOKE-16]

**Independent Test**: Rodar a migration com contas existentes: todas ficam com `token_version = 0` e o resto igual. Revert remove só a coluna.

---

## Edge Cases

- IF a usuária clica "Sair" duas vezes seguidas (duas abas) THEN the segunda chamada SHALL receber `401` (token já revogado) e o front SHALL encerrar a sessão local mesmo assim (REVOKE-02, REVOKE-04).
- WHEN o token revogado chega a uma rota com login a partir do front THEN o interceptor atual SHALL tratar como sessão expirada (toast "Sua sessão expirou. Entre novamente." e `/login`), sem código novo.

---

## Implicit-Requirement Sweep

| Dimension | Result |
| --------- | ------ |
| Input validation & bounds | N/A because `POST /auth/signout` não tem body |
| Failure / partial-failure states | REVOKE-04 (falha da chamada não impede sair) |
| Idempotency / retry / duplicate handling | REVOKE-02 (segunda saída com o mesmo token = `401`, sem incremento extra) |
| Auth boundaries & rate limits | REVOKE-01/02/06–10. Rate limit: N/A because nenhuma rota do projeto tem limite e o PRD não pede |
| Concurrency / ordering | Corrida login × sair registrada nas Assumptions. Incremento atômico no banco (`token_version + 1`) |
| Data lifecycle / expiry | REVOKE-07 (tokens antigos sem `ver`), REVOKE-14/15 (migration) |
| Observability | N/A because o projeto não tem logging estruturado e o PRD não pede |
| External-dependency failure | REVOKE-04 |
| State-transition integrity | REVOKE-11/12/13 |

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| -------------- | ----- | ----- | ------ |
| REVOKE-01 | P1: Sair invalida os tokens da conta | Execute | Verified (T4) |
| REVOKE-02 | P1: Sair invalida os tokens da conta | Execute | Verified (T4) |
| REVOKE-03 | P1: Sair invalida os tokens da conta | Execute | Verified (T5) |
| REVOKE-04 | P1: Sair invalida os tokens da conta | Execute | Verified (T5) |
| REVOKE-05 | P1: A API só aceita a versão atual | Execute | Verified (T2) |
| REVOKE-06 | P1: A API só aceita a versão atual | Execute | Verified (T4) |
| REVOKE-07 | P1: A API só aceita a versão atual | Execute | Verified (T3) |
| REVOKE-08 | P1: A API só aceita a versão atual | Execute | Verified (T3) |
| REVOKE-09 | P1: A API só aceita a versão atual | Execute | Verified (T4) |
| REVOKE-10 | P1: A API só aceita a versão atual | Execute | Verified (T3) |
| REVOKE-11 | P1: Entrar de novo e outras contas | Execute | Verified (T4) |
| REVOKE-12 | P1: Entrar de novo e outras contas | Execute | Verified (T4) |
| REVOKE-13 | P1: Entrar de novo e outras contas | Execute | Verified (T4) |
| REVOKE-14 | P1: Migration da versão | Execute | Verified (T1) |
| REVOKE-15 | P1: Migration da versão | Execute | Verified (T1) |
| REVOKE-16 | P1: Migration da versão | Execute | Verified (T2) |

**Coverage:** 16 total, 16 mapped to tasks, 0 unmapped

---

## Success Criteria

- [ ] Um token copiado antes do "Sair" recebe `401` em `POST /ride` logo depois.
- [ ] Entrar de novo depois de sair leva o mesmo tempo de hoje e nada da conta muda.
