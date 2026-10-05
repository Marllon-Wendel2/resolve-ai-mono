# Memorial Técnico de Desenvolvimento — Resolve AI

Documento único de decisões técnicas do projeto (frontend, backend e infraestrutura).
Cada decisão segue a mesma ficha: **Contexto → Alternativas → Tradeoff → Decisão → Consequências**.

---

## 1. Contexto

Sistema interno de gestão de solicitações (help desk): usuários autenticados criam
solicitações por categoria (TI, RH, Sales, Finance, Infra) e acompanham o status
(Open, InProgress, Closed). Um dashboard consolida KPIs, distribuição por categoria
e atividade recente.

**Requisitos mínimos assumidos:** autenticação com sessão renovável, CRUD de
solicitações com filtros/paginação, dashboard com leitura agregada, painel
administrável por usuário, ambiente de execução reproduzível com um comando.

## 2. Arquitetura

```
navegador ──► Next.js 16 :3000   (rotas, layouts, middleware de sessão, UI)
                │  axios (NEXT_PUBLIC_API_URL)
                ▼
             NestJS 11 :8082     (auth, users, solitacion, dashboard, seed)
                │  Prisma 7 + driver adapter @prisma/adapter-pg
                ▼
             PostgreSQL 17 :5432 (migrações versionadas, seed no boot)
```

- **Front** faz chamadas diretas do browser para a API (sem BFF/API Routes); CORS
  libera apenas `WHITELIST` (`backend/src/main.ts:9`).
- **Auth**: access token curto (15m) lido do cookie pelo interceptor, refresh token
  (7d) persistido no banco e renovado por `/auth/refresh`.
- **Infra local**: `docker-compose.yml` sobe `db` + migração one-shot + `backend`
  (hot-reload) + `front` (hot-reload); o seed roda no boot do backend
  (`backend/src/seed/seed.service.ts:210`).

---

## 3. Fichas de decisão

### Fundação

#### F1 — Linguagem e framework do backend: NestJS vs Spring vs .NET

- **Contexto:** o autor tem experiência prévia em Node/NestJS e Java/Spring; o projeto
  é pequeno mas precisa de estrutura desde o início.
- **Alternativas:** Java + Spring Boot (tipagem forte, ecossistema maduro); C# + .NET;
  Node + NestJS; Node puro com Express.
- **Tradeoff:** Spring entrega robustez e uma JVM mais previsível em carga, mas exige
  mais setup e verbosidade para um CRUD deste porte. Express é o mais leve, porém
  delega arquitetura (módulos, injeção de dependência, guards) ao desenvolvedor.
  NestJS traz modularidade e DI "de graça", ao custo de um framework maior que pesa
  se o projeto escalar em complexidade.
- **Decisão:** NestJS.
- **Consequências:** organização por módulos desde o início (`app.module.ts`), guards
  e pipes reaproveitáveis; dependência do ciclo de releases do Nest.

#### F2 — Banco e ORM: SQL/Postgres/Prisma vs NoSQL/MySQL/TypeORM

- **Contexto:** a relação entre entidade principal (solicitação) e autor é 1:N,
  relacional e com restrições claras.
- **Alternativas:** MongoDB (NoSQL); MySQL; ORM com decorators (TypeORM/Sequelize).
- **Tradeoff:** NoSQL exigiria modelar a relação manualmente e duplicar dados para
  consultas, além de não oferecer integridade referencial nativa. MySQL atenderia,
  mas Postgres tem melhor aderência ao deploy alvo (Neon) e suporte a tipos/enum mais
  ricos. TypeORM gera mais boilerplate de decorators; Prisma gera tipos TS que
  eliminam uma classe inteira de erros.
- **Decisão:** PostgreSQL + Prisma.
- **Consequências:** enum `Category`/`Status` no schema (`backend/prisma/schema.prisma:46`),
  4 migrações versionadas em `backend/prisma/migrations/`, client gerado versionado
  em `src/generated/prisma`.

#### F3 — DTOs com Zod vs classes + ValidationPipe

- **Contexto:** validar entrada em POST/PATCH e manter tipos sincronizados com a API.
- **Alternativas:** classes com decorators (`class-validator`) e `ValidationPipe` do
  Nest; validação manual dentro do service; Zod.
- **Tradeoff:** decorators + classes resolvem com menos código inicial, mas criam duas
  fontes de verdade (classe e tipo TS) que divergem com o tempo. Validar no service
  mistura camadas. Zod exige um pipe próprio, porém o schema vira simultaneamente
  validação e tipo TypeScript.
- **Decisão:** Zod + `ZodValidationPipe` (`backend/src/common/pipes/zod-validation.pipe.ts:8`).
- **Consequências:** erro 400 padronizado com `field`/`message`; adição de regra é uma
  linha no schema; contrato do front copia manualmente os campos (ver F12).

#### F4 — Frontend: Next.js vs React puro vs Angular

- **Contexto:** SPA com autenticação, layouts distintos (auth/dashboard) e prazo curto.
- **Alternativas:** Angular (peso e estrutura maiores que o necessário); React puro
  (leve, mas rotas, SSR, layouts e build exigiriam escolher bibliotecas);
  Next.js (React + estrutura pronta).
- **Tradeoff:** Next.js traz convenção e ferramentas integradas, ao custo de uma curva
  de aprendizado de versão (App Router, Server Components) e de reaprender a cada
  upgrade maior. React puro dá controle total, mas transfere todas as decisões para
  você.
- **Decisão:** Next.js (App Router).
- **Consequências:** rotas por pastas, `middleware.ts` e layouts usados de fato; a
  feature "API Routes" citada no material legado **não** foi usada — as chamadas são
  client-side, não há BFF.

### Autenticação e segurança

#### F5 — Access + Refresh token vs sessão única ou blacklist

- **Contexto:** UX não pode exigir login a cada 15 min, mas um JWT longo exposto é
  risco alto.
- **Alternativas:** sessão server-side (stateful); JWT único de longa duração;
  blacklist de tokens revogados no servidor; par access (curto) + refresh (longo,
  persistido).
- **Tradeoff:** sessão dá revogação imediata, porém exige armazenamento de estado e
  escala mal sem store compartilhada. Blacklist mantém JWT stateful na prática
  (leitura por request). O par de tokens mantém o request barato (access stateless) e
  o logout real pelo registro do refresh no banco, ao custo de mais uma tabela, de
  rotina de rotação e da superfície `/auth/refresh`.
- **Decisão:** access 15m + refresh 7d persistido (`RefreshToken` com índice
  `[userId, revoked, expiresAt]`), payload `sub/email/name` (`backend/src/auth/auth.service.ts:140`),
  guard exige `Bearer` e `type === 'access'` (`backend/src/auth/guards/jwt-auth.guard.ts:32`).
- **Consequências:** logout e "logout everywhere" transacionais no banco;
  **dívida**: o front grava os dois tokens em cookie lido por JS, não `HttpOnly`
  (`front/app/lib/api.ts:196`) — um XSS roubaria a sessão (ver backlog B2).

#### F6 — Sessão no Next: middleware vs checagem no cliente

- **Contexto:** proteger `/dashboard` e `/solicitacoes` e redirecionar `/` para login.
- **Alternativas:** checar no cliente (hook/layout); data fetching no servidor com
  verificação de JWT; middleware de rota do Next.
- **Tradeoff:** checar no cliente é trivial, mas pinta a rota privada antes de
  redirecionar. Verificar assinatura no servidor é o mais correto, porém exige ler o
  cookie no servidor em todo acesso. O middleware roda antes do render e custa pouco,
  mas hoje só testa **presença** do cookie (`front/middleware.ts:17`), não validade.
- **Decisão:** `middleware.ts` com lista pública/privada + matcher que exclui assets.
- **Consequências:** redirecionamento barato e centralizado; validade real do token
  continua sendo garantida pela API (401 → refresh). `?redirect=` é gerado
  (`middleware.ts:30`) mas ainda não é lido após o login (backlog B7).

### Frontend

#### F7 — Data fetching: axios + hooks próprios vs React Query/SWR

- **Contexto:** 4 telas precisam de listagem, KPIs, atividade recente e usuário.
- **Alternativas:** biblioteca de cache (React Query/SWR); fetch + hooks manuais;
  Server Components com dados no servidor.
- **Tradeoff:** React Query resolve cache, dedupe, retry, cancelamento e estados
  padronizados por ~13 kB e uma curva de cache keys. Hooks manuais mantêm zero
  dependência e total controle, mas cada preocupação vira código — e é justamente o
  que faltou: não há `AbortController` (o effect chama `fetchData()` sem cancelamento,
  `front/app/hooks/useSolicitacoes.ts:110`),
  resposta atrasada pode sobrescrever filtro mais novo, e `error` é gravado mas nunca
  consumido pela página (`useSolicitacoes.ts:38`).
- **Decisão:** axios + hooks próprios com interceptores.
- **Consequências:** camada de API centralizada e tipada (`front/app/lib/api.ts`);
  estados de erro/loading tratados caso a caso (backlog B4/B5); migração posterior
  para React Query é incremental, hook a hook.

#### F8 — Formulários: React Hook Form + Zod vs Ant Design Form vs controlado

- **Contexto:** login, cadastro e modal de criação/edição com validação e erros por
  campo.
- **Alternativas:** estado controlado (`useState` por campo); `Form` do Ant Design;
  React Hook Form + Zod resolver.
- **Tradeoff:** controlado é óbvio e sem dependência, mas duplica lógica de touched/
  errors/submit em 3 telas. AntD Form resolve isso, mas amarra o visual ao design
  system do antd. RHF + Zod padroniza `register`/`errors`/`isSubmitting` e reaproveita
  o mesmo schema do backend, ao custo de mais uma dependência e da curva de API.
- **Decisão:** React Hook Form + Zod (`@hookform/resolvers`, schemas em
  `front/app/lib/validations/auth.ts` com `satisfies z.ZodType`).
- **Consequências:** erros por campo acessíveis e consistentes; schema inline do modal
  de solicitação ficou fora do padrão (backlog B8).

#### F9 — Design system: Ant Design **+** Tailwind vs Tailwind puro vs adotar antd

- **Contexto:** o projeto começou com antd e migrou visual para Tailwind dark.
- **Alternativas:** manter os dois; adotar antd de verdade (theme/ConfigProvider);
  remover antd e ficar em Tailwind.
- **Tradeoff:** manter os dois é o caminho de menor esforço imediato, mas paga
  cssinjs e o peso de `@ant-design/*` por **dois** componentes — `Button.tsx:4` e
  `Checkbox.tsx:4` são os únicos imports de antd no app. O `Button` descarta as props
  do antd (`Omit`, `Button.tsx:7`) e briga por especificidade com `!important`
  (`Button.tsx:40-46`); o `Checkbox` é usado uma vez e ignora o tema dark. Adotar antd
  de verdade dá componentes ricos (Table/Form/Select) imediatamente, mas exige
  renunciar ao visual Tailwind já construído.
- **Decisão:** **pendente** — estado atual é a mistura; recomendação registrada:
  remover o antd (revertível) e reescrever os dois wrappers em Tailwind puro.
- **Consequências:** enquanto não decidido, há custo de bundle e de consistência
  visual (backlog B6).

#### F10 — Componentes próprios vs lib de acessibilidade (Radix/shadcn)

- **Contexto:** tabela, modal, filtros, paginação, badges.
- **Alternativas:** lib de primitivos acessíveis; componentes próprios com Tailwind;
  componentes do antd.
- **Tradeoff:** lib entrega foc trap, roving tabindex e ARIA corretos sem estudo, ao
  custo de dependência e de defaults visuais. Próprios são 100% do nosso visual e
  zero peso, mas cada detalhe de a11y é responsabilidade nossa — a qual foi assumida:
  `<dialog>` nativo no modal (foco grátis), `aria-invalid`/`aria-describedby`/`role="alert"`
  nos inputs, `aria-current` na paginação.
- **Decisão:** componentes próprios + primitivos nativos (`<dialog>`, `<table>`).
- **Consequências:** base semântica boa (`DataTable.tsx`, `Input.tsx`, `Pagination.tsx`),
  mas os filtros ficaram com `<label>` sem `htmlFor` (backlog B9) — a11y precisa de
  revisão contínua, não de decisão única.

#### F11 — Estado: local + props vs Redux/Zustand

- **Contexto:** filtros, paginação, modal e dados de lista vivem em uma página; o
  resto é global mínimo (tema, sessão via cookie).
- **Alternativas:** Redux/Zustand; Context global; estado local + prop drilling.
- **Tradeoff:** store global dá depuração centralizada e evita prop drilling, mas
  riqueza visual imediata, mas introduz boilerplate e mentalidade de store para um
  app com um único domínio de
  tela. Estado local mantém fluxo previsível (`useState`/`useCallback`) e o estado
  vive perto de quem usa.
- **Decisão:** estado local por página + hooks de dados; sessão fora do React (cookie).
- **Consequências:** baixo acoplamento; a página de solicitações concentra 425 linhas
  com toolbar, filtros, tabela e modal (backlog B3) — o custo aparece em tamanho, não
  em estado espalhado.

#### F12 — Contrato com a API: tipos manuais vs geração OpenAPI/codegen

- **Contexto:** o front precisa espelhar enums, shapes e headers do backend.
- **Alternativas:** tipos escritos à mão (`front/app/types/index.ts`); gerar tipos a
  partir de OpenAPI; compartilhar um pacote de tipos no monorepo.
- **Tradeoff:** tipos manuais não custam nada de infra, mas não têm compilador que
  reclame quando o backend muda — e as divergências provadas estão no código: o front
  envia categoria `TH` (`SolicitacaoModal.tsx:38`, `solicitacoes/page.tsx:271,298`)
  enquanto o enum do backend é `RH` (`schema.prisma:46`), criar/editar com `TH`
  retorna 400; o front lê `x-total-count` (`api.ts:172`) que o backend nunca envia;
  `useUser` lê `payload.userName` (`useUser.ts:36`) que não é assinado no JWT. Codegen
  elimina a classe de bug, ao custo de mais um passo de build e de versionamento.
- **Decisão:** tipos manuais (hoje).
- **Consequências:** contrato precisa ser checado manualmente em cada entrega;
  backlog B1 corrige as divergências pontuais; codegen fica como evolução.

#### F13 — `"use client"` em tudo vs Server Components

- **Contexto:** o App Router permite buscar dados e renderizar no servidor.
- **Alternativas:** tudo client (estado imediato, simples); páginas server-side com
  Suspense; híbrido (server shell + ilhas client).
- **Tradeoff:** client-only é o de menor fricção — sem serialização, com interatividade
  imediata — mas depende de JS para o primeiro conteúdo e impede prefetch/cache no
  servidor. Server Components reduzem bundle e melhoram LCP, ao custo de pensar
  fronteiras servidor/cliente e de serializar dados.
- **Decisão:** client-first; pages de auth já são server-rendered, as de dados não
  (`"use client"` em 26 de 31 componentes `.tsx`).
- **Consequências:** primeiro paint do dashboard depende de duas chamadas do browser;
  migração incremental é possível por rota (backlog B10).

#### F14 — Paginação: offset + "load more" vs cursor vs server-driven

- **Contexto:** lista com filtros e volume crescente.
- **Alternativas:** páginas numeradas (offset); cursor; carregamento incremental
  ("load more"); virtualização.
- **Tradeoff:** offset + números dá visão de "onde estou", mas custa queries caras em
  offset alto. Cursor é estável para dados em movimento, porém não pula para uma
  página arbitrária. "Load more" é o de melhor UX para varredura linear e implementação
  simples — desde que o total seja confiável.
- **Decisão:** offset + "load more" com `limit/offset`, dedupe por id.
- **Consequências:** como o backend não expõe total (F12), `hasMore` virou heurística
  (`length >= pageSize`) e o contador da tela é `0`; backlog B1 resolve o contrato.

### Infraestrutura e qualidade

#### F15 — Ambiente: compose dev com hot-reload vs build de produção

- **Contexto:** entregar ambiente de teste reproduzível (API, front e banco) com um
  comando.
- **Alternativas:** compose de produção (build multi-stage, sem hot-reload); compose
  dev com volumes; apenas scripts locais.
- **Tradeoff:** produção é mais fiel ao deploy, mas troca de código exige rebuild e
  atrasa o ciclo de teste. Dev com bind mount + volumes de `node_modules` dá alteração
  em milissegundos, ao custo de divergência entre imagem e host (trocar dependência
  exige `--build`) e de atenção a mounts (apagar `node_modules` do host quebra o
  container em execução).
- **Decisão:** compose dev (`db` → `migrate` one-shot com `prisma migrate deploy` →
  `backend` → `front`), com healthchecks e seed automático no boot.
- **Consequências:** `docker compose up --build` entrega o sistema funcional com dados
  de teste; `down -v` reseta o banco. Correções incorporadas nesse esforço:
  `prisma7.config.ts` → `prisma.config.ts` (o CLI do Prisma 7 só auto-descobre o nome
  padrão), `dotenv` movido para `dependencies` (é importado em runtime) e os dois
  `package-lock.json` regenerados (faltavam peers `@emnapi/*`, quebrando `npm ci`).

#### F16 — Estratégia de testes: backend unit vs front sem testes

- **Contexto:** garantir regressão em auth, CRUD e cálculo de KPIs.
- **Alternativas:** só testes manuais; unit com Jest em ambos; unit + e2e (Playwright).
- **Tradeoff:** testes dão confiança para refatorar, mas custam setup e manutenção —
  em app de UI o maior retorno costuma vir de e2e de fluxo crítico, não de unit de
  componente. O backend já tem 9 suítes de spec, mas a config do Jest está incompleta
  (`backend/package.json:67` — `rootDir: "src"` sem resolução de `src/...` nem de
  imports `.js` → `.ts`), então a suíte **não roda** hoje. O front tem zero testes e
  nenhum script de type-check.
- **Decisão:** unit no backend (estrutura criada), e2e/visual pendente; front sem
  testes por ora.
- **Consequências:** backlog B11 — corrigir a config do Jest é o menor custo por maior
  ganho; type-check no CI do front é o segundo.

---

## 4. Estado atual

**Sólido (mantém-se):**

- Interceptor de 401 com *single-flight refresh* e fila de requisições concorrentes
  (`front/app/lib/api.ts:6`–`:105`) — evita corrida de refresh.
- Acessibilidade real em várias peças: `aria-invalid`/`aria-describedby`/`role="alert"`
  nos inputs, `<nav aria-label>` + `aria-current` na paginação, `<dialog>` nativo no
  modal, `role="img"` no gráfico.
- Design tokens centralizados no `@theme` do Tailwind 4, `:focus-visible` global e
  `prefers-reduced-motion` respeitado (`front/app/globals.css`).
- Tabela genérica `<DataTable<T>>` com skeleton, empty state próprio e markup
  semântico.
- Backend modular com guards por controller, validação Zod em todas as rotas de
  escrita, seed idempotente e migrações versionadas.

**Débito (não corrigido nesta entrega):**

- Bug de contrato: categoria `TH` no front vs `RH` no backend (criar/editar → 400).
- `total` sempre `0` e `hasMore` heurístico: backend não envia `x-total-count` nem
  expõe headers via CORS.
- Erro da lista nunca exibido: primeira falha mostra "nenhuma solicitação".
- Tokens em cookie acessível por JS (XSS rouba access + refresh).
- Mistura Ant Design + Tailwind com `!important` e dois componentes de cada sistema.
- Página de solicitações com 425 linhas, `columns` recriado a cada render e
  `key={rowIndex}` (`DataTable.tsx:166`).
- Estado `showFilters` nunca lido (botão "Filtros" não faz nada,
  `SolicitacoesFilters.tsx:51`).
- Fonte Inter carregada mas não usada (`globals.css:21` fixa a string literal) e
  import duplicado em `app/(auth)/layout.tsx`.
- Suíte de testes do backend não roda; front sem testes e sem `typecheck`.

## 5. Backlog priorizado (ganha × custa)

| # | Item | Ganha | Custa |
|---|---|---|---|
| B1 | Corrigir contrato: `TH`→`RH`, total via `{items, total}` ou header exposto | elimina 400 e contador zerado | mudança coordenada front+back |
| B2 | Refresh token em cookie `HttpOnly` (access efêmero) | XSS deixa de roubar a sessão | exige backend/CORS + leitura no middleware |
| B3 | Extrair `DetailsModal`/colunas da página de 425 linhas | manutenção e teste locais | refactor mecânico |
| B4 | Consumir `error` da lista com ação de retry | "sem dados" ≠ "quebrou" | copy + estado novo na UI |
| B5 | `AbortController`/id de requisição nos hooks | mata race ao digitar filtro | ~10 linhas por hook |
| B6 | Remover Ant Design (ou adotá-lo de verdade) | um sistema de design, bundle menor | reimplementar os 2 wrappers |
| B7 | Ler `?redirect=` após login | pós-login volta à rota pedida | pequena lógica no form |
| B8 | Mover schema do modal para `lib/validations` | single source of truth | movimentação de código |
| B9 | Passada de a11y nos filtros (`htmlFor`, nomes, `aria-label`) | usável por teclado/leitor | teste manual com NVDA/VoiceOver |
| B10 | `loading.tsx`/`error.tsx` + Server Components no dashboard | primeiro conteúdo sem JS, falha por seção | refator de fronteira servidor/cliente |
| B11 | Corrigir config do Jest + `typecheck` no CI | testes e tipos voltam a proteger | setup pequeno de infra |

## 6. Documentação legada

- `backend/docs/decisões.md` — fichas 1 a 3 e 5 tiveram origem aqui (versão original).
- `front/docs/MEMORIAL TÉCNICO DE DESENVOLVIMENTO.md` — origem da ficha F4.
