# Auth Context

**Gathered:** 2026-10-07
**Spec:** `.specs/features/auth/spec.md`
**Status:** Ready for design

---

## Feature Boundary

Cadastro (nome, e-mail, senha), login (e-mail + senha), sessão JWT persistente no navegador e logout, em API (`soft-go-ii-api`) e front (`soft-go-II`). Nenhuma rota ou tela existente passa a exigir login.

---

## Implementation Decisions

### A. Escopo de proteção

- Só auth. Mural, publicar carona e inscrição continuam abertos e com o contrato atual.
- A API ganha `GET /auth/me` protegido, que prova que o token identifica a usuária.
- Proteger rotas existentes e pré-preencher nome/telefone ficam para a próxima feature.

### B. Sessão e logout

- JWT em `localStorage`, validade padrão 7 dias, configurável por `JWT_EXPIRES_IN`.
- Logout apenas no cliente (apaga a sessão local).
- A usuária discutiu e aceitou os riscos: roubo via XSS e ausência de revogação. Registrar como dívida: migrar para cookie httpOnly e/ou revogação quando rotas sensíveis passarem a exigir auth.

### C. Pós-cadastro

- Signup devolve token + usuário; o front salva a sessão e vai para `/` já logada.

### D. Validação

- Senha: 8 a 72 caracteres, sem regra de composição.

### UI

- Header: deslogada → "Entrar" e "Criar conta"; logada → "Olá, {nome}" + botão "Sair".
- Páginas novas `/login` e `/cadastro`; logada, essas rotas redirecionam para `/`.

### Agent's Discretion

- Normalização de e-mail, limites de nome/e-mail, textos exatos das mensagens, tratamento de `401` global no front, validação da sessão com `/auth/me` ao abrir o app (todos registrados como premissas no spec).

### Declined / Undiscussed Gray Areas → Assumptions

- Nenhuma área recusada. Detalhes de menor impacto foram registrados como premissas (padrão do agente) no spec.

---

## Specific References

- Layout das páginas novas segue o padrão visual de `pages/Form.tsx` (tokens Tailwind do `index.css`, `InputForm`, `Button`).

---

## Deferred Ideas

- Proteger `POST /ride` e `POST /ride-users` e preencher nome/telefone a partir do usuário logado.
- Cookie httpOnly e/ou revogação de token no servidor.
- Rate limiting de login.
