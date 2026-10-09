# Histórico do Soft Go II

O que foi feito até 2026-10-09, por quê e quais escolhas pesam no futuro. Para montar o ambiente num computador novo, siga o [README](README.md). Para retomar com o Claude Code, abra a pasta raiz e diga "retomar": o estado vivo fica em [`.specs/STATE.md`](.specs/STATE.md).

## Levar para outro notebook

1. **Neste notebook, publique tudo.** Os commits ficam só aqui até o push:
   ```bash
   git push --recurse-submodules=on-demand
   ```
   Esse comando envia a API e o front antes da raiz. Se você der push só na raiz, o clone novo aponta para commits que não existem no GitHub.
2. **No notebook novo**, siga o README: clonar com `--recurse-submodules`, colocar os submódulos na branch (`master` na API, `main` no front), criar os dois `.env` a partir dos `.env.example`, `npm install` e `npm run migrations:run`.
3. **O que não vai pelo git:**
   - Os `.env`, inclusive o `JWT_SECRET`. Com um segredo novo, quem estava logada só precisa entrar de novo.
   - O banco local. O notebook novo começa com o banco vazio: as migrations criam as tabelas e o seed dos tipos de transporte (`1 = Carro`, `2 = Uber`, `3 = Ônibus`). Contas e caronas de teste não vão junto.

## Como o trabalho é organizado

- **Três repositórios.** A raiz [`soft-go-pd`](https://github.com/marquesgio/soft-go-pd) guarda os planos e aponta para os dois projetos como submódulos: [`soft-go-ii-api`](https://github.com/marquesgio/soft-go-ii-api) (NestJS + TypeORM + PostgreSQL) e [`soft-go-II`](https://github.com/marquesgio/soft-go-II) (React + Vite + Tailwind).
- **`prds/`**: os cards de produto, um por feature. São a entrada de cada trabalho.
- **`.specs/features/<feature>/`**: o plano e a prova de cada feature, feitos com a skill `tlc-spec-driven`:
  - `spec.md`: requisitos com IDs;
  - `context.md`: decisões tomadas com a usuária;
  - `tasks.md`: tarefas atômicas;
  - `validation.md`: relatório do Verifier.
- **Ciclo de cada feature:** especificar, decidir os pontos em aberto, quebrar em tarefas, aprovar. Depois, cada tarefa tem testes escritos a partir da spec e um commit próprio no submódulo, seguido de um commit na raiz. No fim, um agente independente (o Verifier) confere cada requisito e injeta bugs de propósito para ver se os testes os pegam.
- **`.specs/STATE.md`**: decisões de arquitetura (AD-001 a AD-006) e o snapshot de onde o trabalho parou.

## Features entregues

Todas estão concluídas e passaram no Verifier. Os números de testes são os do fim de cada feature.

| Feature | O que entrega | Por quê |
|---|---|---|
| **auth** | Cadastro (nome, e-mail, senha), login com e-mail e senha, sessão de 7 dias, `GET /auth/me`, botão "Sair". Senha com hash, nunca devolvida | Antes não havia login: quem publicava ou se inscrevia digitava nome e telefone toda vez. Todas as features seguintes dependem de saber quem é a usuária |
| **vou-junto** | "Vou junto" com um clique para quem está logada; telefone pedido só se a conta não tiver; sem inscrição duplicada; o card mostra quem vai | O modal pedia nome e telefone a cada inscrição, e não havia como evitar duplicidade |
| **save-rideowner** | Toda carona tem dona (`rides.owner_id`); o formulário não pede mais nome; o card mostra a dona; só quem está logada publica; a dona não se inscreve na própria carona | A carona era o único registro sem vínculo com uma conta |
| **revoke-token** | "Sair" invalida todos os tokens da conta, em todos os aparelhos | O "Sair" só apagava o token do navegador, e uma cópia dele continuava valendo por 7 dias |
| **delete-rides** | A dona exclui a própria carona pela lixeira ao lado do tipo de transporte, com um modal que mostra quantas pessoas confirmaram e quem não tem WhatsApp | Não havia como tirar do mural uma carona publicada por engano ou cancelada |
| **leave-ride** | A lista de passageiros fica recolhida ("Ver passageiros (N)"). Aberta, a passageira sai da carona pela lixeira no próprio nome, e a dona remove qualquer passageira | A passageira não conseguia desistir e a vaga ficava presa. A dona também não conseguia tirar quem não ia |
| **notify-ride-deleted** | Avisos dentro do app: carona excluída avisa as participantes; passageira que sai avisa a dona; passageira removida é avisada. Sino no topo com o número de não lidos; abrir marca tudo como lido | Essas três ações aconteciam em silêncio, e quem não tinha telefone só percebia ao abrir o mural |

**Ajustes avulsos (fora das features):**
- **Filtros de cidade e data:** ficam lado a lado e dividem a largura da tela.
- **Busca por cidade:** ignora maiúsculas, minúsculas e acentos, e acha a cidade por parte do nome ("sao" encontra "São Leopoldo" e "São Paulo").

**Testes:** API com 239 e front com 220, todos passando. O front não tinha testes antes da feature auth.

## Escolhas que impactam o futuro

### Autenticação e sessão
- **JWT Bearer sem Passport, com guards aplicados rota a rota, nunca globais** (AD-001). O token fica no `localStorage` do front. Isso deixa o código simples, mas expõe o token a XSS. Login social ou SSO vai exigir outra estratégia.
- **Rotas de leitura continuam públicas** (AD-003). `GET /ride` e `GET /transport-type` não exigem login. Com login, a resposta inclui os telefones. Toda rota que grava dados exige login.
- **"Sair" encerra a sessão em todos os aparelhos** (AD-005). Cada conta tem `users.token_version`, o token carrega essa versão, e sair incrementa o número. Não existe "sair só deste aparelho", e cada requisição com token faz uma consulta a `users`. Uma futura troca de senha também deve incrementar `token_version`.

### Dados
- **Dados pessoais só em `users`** (AD-004). Inscrições (`ride_users`) e caronas (`rides.owner_id`) apontam para contas, não guardam mais nome nem telefone. Não existe inscrição sem conta.
- **Duas migrations apagaram dados de propósito**, porque os dados antigos não tinham conta ligada:
  - `LinkRideUsersToUsers` apagou as inscrições antigas;
  - `AddOwnerToRides` apagou as caronas antigas.

  Num banco que ainda não rodou essas migrations (por exemplo, produção), **todas as caronas e inscrições existentes somem**. Avalie antes de rodar `migrations:run` num banco com dados reais.
- **Excluir carona apaga as inscrições junto** (AD-006), pelo `ON DELETE CASCADE` que o banco já tinha. A exclusão nunca é bloqueada por haver participantes, porque uma carona cancelada precisa sumir do mural. É definitiva: não há lixeira para desfazer.

### Regras de negócio
- **Quem vai de carona não publica carona no mesmo dia.** Quem confirmou presença numa carona recebe `409` ao tentar publicar outra na mesma data. Sair da carona libera a publicação. A regra não vale para inscrições: dá para se inscrever em mais de uma carona no mesmo dia. As specs de delete-rides e leave-ride sugerem o contrário ao falar em "entrar em outra carona no mesmo dia"; o que vale é o código.
- **Quem pode remover uma inscrição:** a própria passageira (sair) e a dona da carona (remover). As duas usam a mesma rota, `DELETE /ride-users/:rideId/users/:userId`. Outras contas recebem `403`.
- **Avisos só dentro do app, sem e-mail** (notify-ride-deleted). A pessoa só vê o aviso quando abre o app; não há tempo real (o sino atualiza ao carregar a página e ao abrir). A frase do aviso é gravada pronta no momento da ação, porque a carona excluída some do banco. A ação sempre vem antes do aviso: se gravar o aviso falhar, a ação continua valendo e o erro só vai para o log. Os avisos não expiram; o painel mostra os 30 mais recentes. E-mail fica como ideia futura.
- **A busca por cidade é feita na API, em memória**, depois de buscar as caronas no banco. Para o volume de um mural isso não pesa. Se um dia houver milhares de caronas, vale passar para o banco (extensão `unaccent` do PostgreSQL + `ILIKE`).

### Front
- **Testes com vitest + Testing Library** (AD-002), ao lado do código (`*.test.tsx`).
- **Os ids dos tipos de transporte estão fixos no front** (`1 = Carro`, `2 = Uber`, `3 = Ônibus`). Se o seed mudar, atualize `pages/Form.tsx` e `components/InputTransportForm.tsx`.
- **Token de cor `support-04` (`#dc2626`)** para ações destrutivas (lixeiras, botões de excluir). O texto de erro dos formulários ainda usa `support-03`.

## Pendências

- **Nenhum card aberto.** Todos os PRDs em `prds/` foram entregues.
- **notify-ride-deleted sem Verifier independente:** foi entregue com gates e checagem ponta a ponta, mas o agente que confere requisitos e injeta bugs não rodou (pedido de rapidez). Rode-o quando puder: "validar notify-ride-deleted".
- **Migration nova `CreateTableNotifications`:** só cria a tabela `notifications`. Rode `npm run migrations:run` no notebook novo e em qualquer banco antes de subir a API.
- **Teste que falta (apontado pelo Verifier do leave-ride):** nenhum teste automático exclui uma inscrição e depois confere no `GET /ride` que a vaga voltou. Isso foi confirmado só manualmente.
- **Fora de escopo, mas notado:** `GET /ride-users` e `GET /ride-users/:id` estão abertos sem login. A tabela de contrato no `CLAUDE.md` ainda descreve `createRideUser` com `name`, que não existe mais.
- **Banco local deste notebook:** ficaram 8 contas de teste das checagens manuais (`*.delete.<timestamp>@teste.com`, `*.leave.<timestamp>@teste.com` e `*.notify.<timestamp>@teste.com`). Elas não vão para o notebook novo.
