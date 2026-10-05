# Resolve AI

Sistema de gestão de solicitações (help desk) com dashboard: usuários cadastram
solicitações por categoria (TI, RH, Sales, Finance, Infra) e acompanham o status
(Open, InProgress, Closed).

## Stack

| Camada | Tecnologia |
| --- | --- |
| Frontend | Next.js 16 (App Router), React 19, Ant Design 6, Tailwind 4, Axios + Zod |
| Backend | NestJS 11, Prisma 7 (driver adapter `pg`), Passport + JWT (access/refresh token) |
| Banco | PostgreSQL 17 |
| Infra local | Docker Compose |

Estrutura do repositório (monorepo com submódulos):

```
resolve-ai-mono/
├── docker-compose.yml   # sobe tudo para desenvolvimento/teste
├── .env.example         # variáveis opcionais do compose
├── backend/             # API NestJS  (submódulo)
└── front/               # App Next.js (submódulo)
```

## Como rodar

Pré-requisito: Docker + Docker Compose.

Se for a primeira vez clonando o repositório, inicialize os submódulos
(`backend/` e `front/` ficam vazios sem isso, e o build falha com
`open Dockerfile: no such file or directory`):

```bash
git submodule update --init --recursive
```

Depois suba tudo:

```bash
docker compose up --build
```

| Serviço | URL | Credenciais |
| --- | --- | --- |
| Frontend | http://localhost:3000 | `admin@admin.com` / `admin123` |
| API | http://localhost:8082 | — |
| Postgres | `localhost:5432` (`resolve` / `resolve` / db `resolve_test`) | — |

O compose já executa as migrações do Prisma (`migrate`) e o seed automático na
subida do backend: 1 usuário administrador e 30 solicitações de teste.

### Comandos úteis

```bash
docker compose logs -f backend
docker compose exec db psql -U resolve -d resolve_test
docker compose down -v
```

Em desenvolvimento os fontes são montados nos containers (hot-reload). Ao
adicionar uma dependência nova em `backend/` ou `front/`, rode
`docker compose up --build` para reconstruir as imagens.

### Variáveis de ambiente

Os valores padrão funcionam sem configuração. Para sobrescrever, copie
`.env.example` para `.env` na raiz. Todas as variáveis da API (incluindo
`DATABASE_URL`, `JWT_SECRET`, `WHITELIST`) estão documentadas em
`backend/.env.example` — em Docker elas são injetadas pelo próprio compose.

## API - Resumo simples

| Método | Rota | Auth |
| --- | --- | --- |
| POST | `/auth/login` · `/auth/refresh` · `/auth/logout` | — |
| POST | `/users` (cadastro) | — |
| GET/PATCH/DELETE | `/users/:id` | JWT |
| POST/GET/PATCH/DELETE | `/solitacion` e `/solitacion/:id` | JWT |
| GET | `/dashboard` (KPIs) | JWT |



> Ainda em correção: a suíte do backend falha por configuração do Jest
> (resolução de `src/...` e imports `.js` → `.ts`).
