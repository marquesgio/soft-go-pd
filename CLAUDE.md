# Soft Go II

Mural de caronas para ir até a Soft: alguém publica uma carona (data, hora, cidade de saída, tipo de transporte, vagas) e colegas se inscrevem nela. Esta pasta é o repo raiz [`soft-go-pd`](https://github.com/marquesgio/soft-go-pd), que guarda os planos em `.specs/` e tem os dois projetos como **submódulos git** (cada um com seu próprio repositório e commits):

| Pasta | O que é | Stack |
|---|---|---|
| `soft-go-II/` | Frontend | React 19 + Vite 8 + TypeScript + Tailwind 4 |
| `soft-go-ii-api/` | API REST | NestJS 12 + TypeORM + PostgreSQL |

A API tem seu próprio `soft-go-ii-api/CLAUDE.md` com convenções detalhadas (ESM com `.js` nos imports, repositórios customizados, migrations, Swagger) e skills em `soft-go-ii-api/.claude/skills/` (`/migration`, `/novo-recurso`). Leia esse arquivo antes de mexer no backend.

## Rodando localmente

```bash
# API (porta 3000, Swagger em http://localhost:3000/docs)
cd soft-go-ii-api && npm run start:dev

# Frontend (Vite, porta 5173)
cd soft-go-II && npm run dev
```

- Front: `.env` com `VITE_BASE_URL` apontando para a API (ex.: `http://localhost:3000`). Usado em `src/service/api.ts` (axios).
- API: `.env` com `DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`, `DB_NAME`. CORS já liberado (`app.enableCors()`).
- Front não tem testes; valide com `npm run lint` e `npm run build` (`tsc -b` faz o type-check). API: `npm test`, `npm run lint` (oxlint).

## Contrato entre front e API

Toda chamada HTTP do front fica em `soft-go-II/src/service/ride.service.ts`; os tipos espelham as entidades da API em `soft-go-II/src/types/index.ts`. Ao mudar um DTO/entidade na API, atualize os dois arquivos do front.

| Front | Endpoint | Observação |
|---|---|---|
| `getRides(id, date, city)` | `GET /ride?id=&date=&city=` | **`id` aqui é o id do tipo de transporte**, não da carona. Não retorna datas passadas. |
| `createRide(payload)` | `POST /ride` | `transportType` é enviado como **número** (id), não objeto. `hour` em `HH:mm`, `date` ISO `YYYY-MM-DD`. |
| `getAllRidesTypes()` | `GET /transport-type` | `spots: null` = ilimitado (ônibus). |
| `createRideUser(payload)` | `POST /ride-users` | `{ rideId, name, phone? }`. Carona lotada → `409`; mesmo telefone na mesma carona → conflito. |
| `deleteRide(id)` | `DELETE /ride/:id` | Exige token. `204` só para a dona; outra conta → `403`, inexistente → `404`. As inscrições saem junto (cascade, AD-006). |

- A API usa `ValidationPipe` com `forbidNonWhitelisted`: qualquer campo extra no body vira `400`. Não envie campos que o DTO não declara.
- Telefone: o front valida 11 dígitos só números (zod); a API valida telefone BR com DDD. String vazia é tratada como ausente nos dois lados.
- Os tipos de transporte vêm de seed (`SeedTransportType`): `1 = Carro`, `2 = Uber`, `3 = Ônibus`. O formulário (`pages/Form.tsx`) e `components/InputTransportForm.tsx` têm esses ids **hardcoded** (ícones e esconder o campo "Vagas" para ônibus); se o seed mudar, atualize-os.

## Frontend (`soft-go-II`)

- Rotas (`src/Routes.tsx`): `/` → `pages/Home.tsx` (lista, filtros por cidade/data/transporte, modal de inscrição); `/form` → `pages/Form.tsx` (publicar carona).
- Formulários: `react-hook-form` + `zod` (`zodResolver`). Mensagens de validação em PT-BR.
- Feedback ao usuário: use o helper `Toast("success" | "error", mensagem?)` de `components/Toast.tsx` (react-hot-toast; o `<Toaster>` está em `App.tsx`). `react-toastify` está instalado mas não é usado.
- Estilo: só classes Tailwind. Cores e fonte são tokens definidos em `@theme` no `src/index.css` (`bg-default`, `text-secondary-text`, `bg-surface-secondary`, `border-light`, ...). Use esses tokens em vez de cores arbitrárias.
- Ícones: `lucide-react`. Datas: `date-fns`.
- Imports relativos sem extensão (diferente da API, que exige `.js`).

## Convenções comuns

- Textos de interface, mensagens de erro e validação em **português (PT-BR)**.
- Commits em Conventional Commits curtos: `feat: ...`, `fix: ...`. Cada projeto commita no próprio repositório (submódulo); depois, commite no repo raiz o bump do ponteiro do submódulo junto com as mudanças em `.specs/`.
- Clonar tudo: `git clone --recurse-submodules https://github.com/marquesgio/soft-go-pd.git`.
- Não commitar `dist/`, `.env` nem `tsconfig.build.tsbuildinfo`.
