# Cadastro e Login (Auth) Specification

## Problem Statement

Hoje o Soft Go não tem login: quem publica uma carona ou se inscreve digita nome e telefone toda vez, sem nenhuma validação de identidade. Precisamos de cadastro, login, sessão persistente e logout como base para as próximas funcionalidades, que vão depender de saber quem é a usuária.

## Goals

- [ ] Uma pessoa cria conta (nome, e-mail, senha) e já sai do cadastro logada.
- [ ] Uma pessoa com conta faz login com e-mail + senha e continua logada por até 7 dias sem logar de novo.
- [ ] A API identifica a usuária em requisições autenticadas via `Authorization: Bearer <JWT>` (provado por `GET /auth/me`).
- [ ] Existe um botão "Sair" que encerra a sessão no navegador.
- [ ] Nenhuma senha é armazenada ou devolvida em texto puro.

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
| ------- | ------ |
| Recuperação / redefinição de senha | Fora do PRD; feature futura |
| Verificação de e-mail (envio de link/código) | Fora do PRD; não há infraestrutura de e-mail |
| Login social (Google, Microsoft etc.) | Fora do PRD |
| Proteger `POST /ride`, `POST /ride-users` ou qualquer tela/endpoint existente | Decisão da discussão: esta feature entrega só auth; proteção e pré-preenchimento de nome/telefone ficam para a próxima feature |
| Pré-preencher nome/telefone nos formulários atuais com o usuário logado | Mesmo motivo acima |
| Revogação de token no servidor / logout server-side / blacklist | Decisão da discussão: logout é client-side; registrado como dívida |
| Refresh token / renovação silenciosa | Sessão de duração fixa (7 dias) é suficiente nesta fase |
| Cookie httpOnly | Decisão da discussão: token em localStorage; registrado como dívida |
| Rate limiting / bloqueio por tentativas de login | App interno; registrado como dívida |
| Telefone no cadastro de usuário, edição de perfil, troca de senha, exclusão de conta | Modelo de dados do PRD não inclui; features futuras |

---

## Assumptions & Open Questions

Every ambiguity is resolved or recorded here - nothing is left silently unclear.

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --------------------- | -------------- | --------- | ---------- |
| Escopo de proteção | Nenhuma rota/tela existente passa a exigir login; só `GET /auth/me` é protegido | Escolha da usuária; mantém o contrato atual intacto | y |
| Armazenamento e validade da sessão | JWT em `localStorage`, validade `JWT_EXPIRES_IN` (padrão `7d`) | Escolha da usuária após discutir riscos (XSS, sem revogação); token hoje não protege dado sensível | y |
| Logout | Somente no cliente: apaga token/usuário do `localStorage` | Escolha da usuária; token emitido continua válido até expirar (dívida registrada) | y |
| Pós-cadastro | Signup devolve token + usuário; front salva sessão e vai para `/` | Escolha da usuária | y |
| Regra de senha | 8 a 72 caracteres, sem regra de composição | Escolha da usuária; 8 = mínimo NIST, 72 = limite do bcrypt | y |
| Pontos de entrada na UI | Header: deslogado → "Entrar" / "Criar conta"; logado → "Olá, {primeiro nome}" + "Sair". Rotas `/login` e `/cadastro` | Escolha da usuária | y |
| Normalização de e-mail | `trim` + minúsculas antes de salvar e de buscar; unicidade é case-insensitive na prática | Evita contas duplicadas `Ana@x.com` vs `ana@x.com` | n (padrão do agente) |
| Limites do nome | `trim`; 2 a 120 caracteres | 120 = coluna do PRD; 2 evita nome de uma letra | n (padrão do agente) |
| Limite do e-mail | Formato válido; até 160 caracteres | 160 = coluna do PRD | n (padrão do agente) |
| Mensagem de e-mail duplicado | `409` com `"E-mail já cadastrado"` | PRD pede mensagem clara; segue padrão `ConflictException` da API | n (padrão do agente) |
| Mensagem de login inválido | `401` com `"E-mail ou senha inválidos"`, idêntica para e-mail inexistente e senha errada | PRD exige não revelar qual campo falhou | n (padrão do agente) |
| Enumeração de e-mail | Aceita que o cadastro revela que o e-mail existe (409) | Exigido pelo PRD ("mensagem clara"); por isso não se investe em igualar tempo de resposta no login | n (padrão do agente) |
| Confirmação de senha | Validada só no front; não é enviada à API | `forbidNonWhitelisted` rejeitaria campo extra; a API não precisa dela | n (padrão do agente) |
| Sessão expirada / token inválido no front | Qualquer `401` com sessão ativa limpa a sessão, mostra toast "Sua sessão expirou. Entre novamente." e redireciona para `/login` | Comportamento previsível sem refresh token | n (padrão do agente) |
| Validação da sessão ao abrir o app | Ao carregar com token salvo, o front chama `GET /auth/me`; `401` → limpa sessão | Detecta token expirado/adulterado logo na entrada | n (padrão do agente) |
| `JWT_SECRET` ausente | API não sobe (erro na inicialização) | Evita assinar tokens com segredo vazio/padrão | n (padrão do agente) |
| Local dos artefatos `.specs/` | No repo raiz `soft-go-pd` (`Soft Go/`), que tem `soft-go-ii-api` e `soft-go-II` como submódulos | Feature cruza os dois repos; decisão da usuária de unificar via submódulos | y |

**Open questions:** none - all resolved or logged above (required before the spec is confirmed).

---

## User Stories

### P1: Cadastro (Sign Up) ⭐ MVP

**User Story**: Como usuária do Soft Go, quero criar uma conta com nome, e-mail e senha, para que o sistema passe a me identificar.

**Why P1**: Sem conta não existe login nem sessão.

**Acceptance Criteria**:

1. WHEN `POST /auth/signup` recebe `{ name, email, password }` válidos THEN the API SHALL criar o usuário e responder `201` com `{ accessToken, user: { id, name, email } }`. [AUTH-01]
2. The API SHALL gravar em `users.password_hash` apenas o hash bcrypt da senha, e o valor gravado SHALL ser diferente da senha enviada e validar com `bcrypt.compare(senha, hash) === true`. [AUTH-02]
3. The API SHALL never include `password` nem `passwordHash`/`password_hash` em nenhuma resposta HTTP. [AUTH-03]
4. IF o e-mail enviado (após `trim` + minúsculas) já existe THEN the API SHALL responder `409` com `message: "E-mail já cadastrado"` e não criar outro registro. [AUTH-04]
5. IF dois cadastros simultâneos com o mesmo e-mail violam a constraint `UNIQUE` do banco THEN the API SHALL responder `409` com `"E-mail já cadastrado"` (nunca `500`). [AUTH-05]
6. IF `password` tem menos de 8 ou mais de 72 caracteres THEN the API SHALL responder `400` com mensagem em PT-BR ("A senha deve ter entre 8 e 72 caracteres"). [AUTH-06]
7. IF `email` não tem formato válido ou passa de 160 caracteres THEN the API SHALL responder `400` com mensagem em PT-BR. [AUTH-07]
8. IF `name` (após `trim`) tem menos de 2 ou mais de 120 caracteres THEN the API SHALL responder `400` com mensagem em PT-BR. [AUTH-08]
9. WHEN um usuário é criado THEN the API SHALL persistir o e-mail com `trim` e em minúsculas. [AUTH-09]
10. IF a senha e a confirmação de senha diferem THEN the signup form SHALL bloquear o envio e exibir "As senhas não coincidem" abaixo do campo de confirmação. [AUTH-10]
11. IF algum campo do formulário de cadastro viola as regras de AUTH-06/07/08 THEN the signup form SHALL bloquear o envio e exibir a mensagem PT-BR abaixo do campo. [AUTH-11]
12. WHEN o cadastro retorna `201` THEN the front SHALL salvar a sessão, exibir o toast de sucesso "Conta criada com sucesso" e navegar para `/`. [AUTH-12]
13. IF o cadastro retorna `409` THEN the front SHALL exibir toast de erro com "E-mail já cadastrado" e permanecer em `/cadastro`. [AUTH-13]

**Independent Test**: Em `/cadastro`, preencher nome/e-mail/senha/confirmação → cai em `/` logada com "Olá, {nome}" no header; no banco, `password_hash` começa com `$2` e não é a senha. Repetir com o mesmo e-mail em maiúsculas → toast "E-mail já cadastrado".

---

### P1: Login (Sign In) ⭐ MVP

**User Story**: Como usuária com conta, quero entrar com e-mail e senha, para que o sistema saiba quem eu sou.

**Why P1**: É o caminho principal para obter uma sessão.

**Acceptance Criteria**:

1. WHEN `POST /auth/signin` recebe `{ email, password }` de um usuário existente com a senha correta THEN the API SHALL responder `200` com `{ accessToken, user: { id, name, email } }`. [AUTH-14]
2. WHEN o e-mail do login difere do cadastrado só por maiúsculas/minúsculas ou espaços nas pontas THEN the API SHALL autenticar normalmente. [AUTH-15]
3. IF o e-mail não existe ou a senha está errada THEN the API SHALL responder `401` com o mesmo corpo nos dois casos, `message: "E-mail ou senha inválidos"`. [AUTH-16]
4. WHEN o login retorna `200` THEN the front SHALL salvar a sessão e navegar para `/`. [AUTH-17]
5. IF o login retorna `401` THEN the front SHALL exibir toast de erro "E-mail ou senha inválidos" e permanecer em `/login`. [AUTH-18]

**Independent Test**: Com conta criada, `/login` com credenciais certas → `/` logada. Com e-mail inexistente e com senha errada → a mesma resposta `401` e o mesmo toast.

---

### P1: Sessão persistente ⭐ MVP

**User Story**: Como usuária logada, quero continuar logada ao recarregar ou voltar ao app, para não precisar entrar a cada ação.

**Why P1**: Critério de aceite do PRD ("token/sessão identifica a usuária nas próximas requisições").

**Acceptance Criteria**:

1. The API SHALL emitir `accessToken` como JWT HS256 assinado com `JWT_SECRET`, contendo `sub` = id do usuário e `email`, com expiração `JWT_EXPIRES_IN` (padrão `7d`). [AUTH-19]
2. WHEN `GET /auth/me` recebe `Authorization: Bearer <token válido>` THEN the API SHALL responder `200` com `{ id, name, email }` do usuário do token. [AUTH-20]
3. IF `GET /auth/me` é chamado sem token, com token de assinatura inválida, malformado ou expirado THEN the API SHALL responder `401`. [AUTH-21]
4. IF `JWT_SECRET` não está definido THEN the API SHALL falhar na inicialização com erro explícito. [AUTH-22]
5. The API SHALL continuar aceitando `GET/POST /ride`, `POST /ride-users` e `GET /transport-type` sem token, com o mesmo contrato atual. [AUTH-23]
6. WHILE existe sessão salva no `localStorage` the front SHALL enviar `Authorization: Bearer <token>` em toda requisição à API. [AUTH-24]
7. WHEN o app carrega com sessão salva THEN the front SHALL chamar `GET /auth/me` e, com `200`, exibir a usuária como logada sem pedir login. [AUTH-25]
8. IF uma requisição retorna `401` WHILE existe sessão salva THEN the front SHALL limpar a sessão, exibir o toast "Sua sessão expirou. Entre novamente." e navegar para `/login`. [AUTH-26]
9. WHILE a usuária está logada the front SHALL redirecionar `/login` e `/cadastro` para `/`. [AUTH-27]
10. WHILE a usuária está deslogada the header SHALL exibir os links "Entrar" (`/login`) e "Criar conta" (`/cadastro`). [AUTH-28]
11. WHILE a usuária está logada the header SHALL exibir "Olá, {primeiro nome}" e o botão "Sair". [AUTH-29]

**Independent Test**: Logar, recarregar a página → continua "Olá, {nome}"; DevTools mostra `Authorization: Bearer` nas chamadas. Adulterar o token no `localStorage` e recarregar → volta deslogada em `/login` com o toast de sessão expirada.

---

### P1: Logout ⭐ MVP

**User Story**: Como usuária logada, quero sair da conta, para que ninguém use minha sessão naquele navegador.

**Why P1**: Critério de aceite do PRD ("existe uma forma de deslogar").

**Acceptance Criteria**:

1. WHEN a usuária clica em "Sair" THEN the front SHALL remover token e usuário do `localStorage`, exibir o header deslogado e navegar para `/login`. [AUTH-30]
2. WHEN a sessão foi encerrada (logout ou expiração) THEN the front SHALL enviar as requisições seguintes sem o header `Authorization`. [AUTH-31]

**Independent Test**: Logada, clicar "Sair" → `/login`, header com "Entrar"/"Criar conta", `localStorage` sem a chave da sessão, recarregar continua deslogada.

---

## Edge Cases

- IF o body de signup/signin tem campo não declarado (ex.: `confirmPassword`) THEN the API SHALL responder `400` (comportamento existente de `forbidNonWhitelisted`). [AUTH-32]
- IF o `localStorage` contém JSON de sessão corrompido THEN the front SHALL tratar como deslogada e remover a chave, sem quebrar a renderização. [AUTH-33]
- WHEN o signin recebe `401` sem sessão salva THEN the front SHALL exibir apenas "E-mail ou senha inválidos" (sem o toast de sessão expirada nem redirecionamento). [AUTH-34]

---

## Implicit-Requirement Dimensions

| Dimension | Resolution |
| --------- | ---------- |
| Input validation & bounds | AUTH-06, 07, 08, 10, 11, 32 |
| Failure / partial-failure states | AUTH-13, 18, 26, 33; erro inesperado no front → toast genérico de erro existente |
| Idempotency / retry / duplicate handling | AUTH-04, 05 (e-mail único, dedup por constraint) |
| Auth boundaries & rate limits | AUTH-20, 21, 23; rate limit N/A because app interno e fora do PRD (registrado em Out of Scope) |
| Concurrency / ordering | AUTH-05 (corrida de cadastro resolvida pela constraint UNIQUE) |
| Data lifecycle / expiry | AUTH-19 (expiração 7d), AUTH-26, 30; exclusão de conta N/A because fora do escopo |
| Observability | N/A because a API não tem logging estruturado hoje; seguir o logger padrão do Nest, sem logar senha nem token |
| External-dependency failure | N/A because não há dependência externa nova (só Postgres, já existente) |
| State-transition integrity | AUTH-27, 28, 29, 30, 31 (deslogada ↔ logada) |

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| -------------- | ----- | ----- | ------ |
| AUTH-01 | P1: Cadastro | Design | Pending |
| AUTH-02 | P1: Cadastro | Design | Pending |
| AUTH-03 | P1: Cadastro | Design | Pending |
| AUTH-04 | P1: Cadastro | Design | Pending |
| AUTH-05 | P1: Cadastro | Design | Pending |
| AUTH-06 | P1: Cadastro | Design | Pending |
| AUTH-07 | P1: Cadastro | Design | Pending |
| AUTH-08 | P1: Cadastro | Design | Pending |
| AUTH-09 | P1: Cadastro | Design | Pending |
| AUTH-10 | P1: Cadastro | Design | Pending |
| AUTH-11 | P1: Cadastro | Design | Pending |
| AUTH-12 | P1: Cadastro | Design | Pending |
| AUTH-13 | P1: Cadastro | Design | Pending |
| AUTH-14 | P1: Login | Design | Pending |
| AUTH-15 | P1: Login | Design | Pending |
| AUTH-16 | P1: Login | Design | Pending |
| AUTH-17 | P1: Login | Design | Pending |
| AUTH-18 | P1: Login | Design | Pending |
| AUTH-19 | P1: Sessão | Design | Pending |
| AUTH-20 | P1: Sessão | Design | Pending |
| AUTH-21 | P1: Sessão | Design | Pending |
| AUTH-22 | P1: Sessão | Design | Pending |
| AUTH-23 | P1: Sessão | Design | Pending |
| AUTH-24 | P1: Sessão | Design | Pending |
| AUTH-25 | P1: Sessão | Design | Pending |
| AUTH-26 | P1: Sessão | Design | Pending |
| AUTH-27 | P1: Sessão | Design | Pending |
| AUTH-28 | P1: Sessão | Design | Pending |
| AUTH-29 | P1: Sessão | Design | Pending |
| AUTH-30 | P1: Logout | Design | Pending |
| AUTH-31 | P1: Logout | Design | Pending |
| AUTH-32 | Edge | Design | Pending |
| AUTH-33 | Edge | Design | Pending |
| AUTH-34 | Edge | Design | Pending |

**Coverage:** 34 total, 0 mapped to tasks, 34 unmapped ⚠️ (mapeamento acontece na fase Tasks)

---

## Success Criteria

- [ ] Os 5 critérios de aceite do PRD são demonstráveis no app rodando localmente (cadastro duplicado, hash, erro genérico, token nas requisições, logout).
- [ ] Nenhuma linha da tabela `users` contém senha em texto puro; nenhuma resposta da API expõe hash.
- [ ] Fluxos atuais (mural, publicar carona, inscrição) continuam funcionando sem login.
