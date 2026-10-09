# Soft Go II

Mural de caronas para ir até a Soft: alguém publica uma carona (data, hora, cidade, tipo de transporte, vagas) e colegas se inscrevem nela.

Este repositório reúne os dois projetos como **submódulos git** e guarda os planos de features em `.specs/`.

| Pasta | O que é | Repositório |
|---|---|---|
| `soft-go-ii-api/` | API REST (NestJS + TypeORM + PostgreSQL) | [marquesgio/soft-go-ii-api](https://github.com/marquesgio/soft-go-ii-api) |
| `soft-go-II/` | Frontend (React + Vite + TypeScript + Tailwind) | [marquesgio/soft-go-II](https://github.com/marquesgio/soft-go-II) |
| `.specs/` | Specs, designs, tarefas e estado das features (skill `tlc-spec-driven`) | este repo |
| `.claude/skills/` | Skill `tlc-spec-driven` usada para planejar e executar as features | este repo |

O que já foi feito, por quê e as escolhas que pesam no futuro estão em [`HISTORICO.md`](HISTORICO.md). Os cards de produto de cada feature ficam em [`prds/`](prds/).

## Começando em um computador novo

### 1. Clonar com os submódulos

```bash
git clone --recurse-submodules https://github.com/marquesgio/soft-go-pd.git "Soft Go"
cd "Soft Go"
```

Se já clonou sem a flag: `git submodule update --init`.

### 2. Colocar os submódulos em uma branch

Os submódulos vêm em *detached HEAD* (parados num commit). Antes de commitar neles, entre na branch:

```bash
git -C soft-go-ii-api checkout master
git -C soft-go-II checkout main
```

### 3. Recriar o que não é versionado

`.env` não vai para o git. Crie:

- `soft-go-ii-api/.env` (modelo em `soft-go-ii-api/.env.example`):
  ```
  DB_HOST=localhost
  DB_PORT=5432
  DB_USERNAME=
  DB_PASSWORD=
  DB_NAME=
  JWT_SECRET=          # a partir da feature de auth
  JWT_EXPIRES_IN=7d    # a partir da feature de auth
  ```
- `soft-go-II/.env`:
  ```
  VITE_BASE_URL=http://localhost:3000
  ```

Pré-requisitos: Node.js, um PostgreSQL local com o banco de `DB_NAME` criado e Python 3 (usado pelos scripts da skill; `py` no Windows, `python3` no Mac/Linux).

### 4. Instalar e preparar o banco

```bash
cd soft-go-ii-api && npm install && npm run migrations:run
cd ../soft-go-II && npm install
```

### 5. Rodar

```bash
# API — http://localhost:3000 (Swagger em /docs)
cd soft-go-ii-api && npm run start:dev

# Front — http://localhost:5173
cd soft-go-II && npm run dev
```

## Continuar o trabalho com o Claude Code

Abra o Claude Code **na pasta raiz** (`Soft Go/`), não dentro de um dos projetos: é aqui que ficam o `CLAUDE.md`, o `.specs/` e a skill.

- **Retomar uma feature:** diga `retomar auth` (ou o nome da feature). O Claude lê `.specs/STATE.md`, confere com o git e propõe o próximo passo.
- **Parar no meio de uma tarefa:** diga `pausar trabalho`. O snapshot de retomada é gravado em `.specs/STATE.md`; depois commite e dê push (veja abaixo).

Features planejadas ficam em `.specs/features/<feature>/` (`spec.md`, `context.md`, `design.md`, `tasks.md`).

## Fluxo de commits com submódulos

1. Commite o código **dentro do submódulo** (`soft-go-ii-api` ou `soft-go-II`), no repositório dele.
2. Na raiz, commite o ponteiro atualizado do submódulo junto com as mudanças em `.specs/`:
   ```bash
   git add soft-go-ii-api .specs
   git commit -m "docs: ..."
   ```
3. Publique os submódulos primeiro e a raiz por último:
   ```bash
   git -C soft-go-ii-api push origin master
   git -C soft-go-II push origin main
   git push origin main
   ```
   O atalho `git push --recurse-submodules=on-demand` não funciona aqui: ele tenta enviar a branch `main` também na API, que usa `master`.
   Não dê push só na raiz: ela passaria a apontar para commits que não existem no GitHub e o clone em outra máquina quebra.

Para atualizar em outra máquina:

```bash
git pull --recurse-submodules
```

Se um submódulo voltar para *detached HEAD*, repita o passo 2.

## Convenções

Detalhes de cada projeto em [`CLAUDE.md`](CLAUDE.md) e [`soft-go-ii-api/CLAUDE.md`](soft-go-ii-api/CLAUDE.md). Em resumo: textos em PT-BR, commits em Conventional Commits (`feat: ...`, `fix: ...`) e nunca commitar `dist/`, `.env` ou `tsconfig.build.tsbuildinfo`.
