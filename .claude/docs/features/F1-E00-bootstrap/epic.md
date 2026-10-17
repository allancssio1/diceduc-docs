# F1-E00 — Bootstrap dos repositórios (diceduc-api + diceduc-web)

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E00 — Bootstrap dos repositórios (diceduc-api + diceduc-web) |
| Fase | F1 (pré-requisito de todas as demais fichas) |
| Prioridade | Must |
| Estimativa | M |
| Bounded context(s) | — (infraestrutura de projeto, não domínio de negócio) |
| Depende de | — |
| Rastreabilidade | Não existe no BACKLOG — é um *enabler* técnico necessário antes de qualquer épico de produto poder ser implementado. |

## Objetivo

Criar **dois repositórios git independentes** — `diceduc-api` (backend) e `diceduc-web` (frontend) — com toda a fundação técnica descrita em [arquitetura-ignite.md](../../../arquitetura-ignite.md) (backend) e [arquitetura-frontend.md](../../../arquitetura-frontend.md) (frontend), **sem nenhuma regra de negócio ainda**. Ao final desta ficha, `F1-E01` (e as demais) só precisam adicionar código de domínio sobre uma base já funcional em cada repositório.

Este repositório (`diceduc-docs`) não recebe código nesta ficha — ele continua sendo só o hub de documentação/planejamento.

## Escopo

### Repositórios

- [ ] Criar os 2 repositórios git `diceduc-api` e `diceduc-web` **como pastas irmãs de `diceduc-docs`, dentro de `diceduc-controle-escolar/`** (`git init` em cada uma), cada um independente (histórico, versionamento e deploy próprios).
- [ ] Em cada repositório: `package.json` próprio com scripts (`dev`, `build`, `test`, `lint`), lint/format próprio (ESLint + Prettier) e `.gitignore` cobrindo `node_modules`, `dist`/`.next`, `.env`.
- [ ] Não há ferramenta de workspace/monorepo (pnpm workspaces, turborepo etc.) — cada repositório é standalone e resolve suas próprias dependências.
- [ ] Padrão de documentação as-built em cada repositório (`diceduc-api` e `diceduc-web`):
  - `AGENTS.md` — ponto de entrada: aponta para `.claude/arquitetura.md` e define a regra de manutenção (atualizar ao implementar algo que toque arquitetura ou um domínio).
  - `CLAUDE.md` — uma única linha: `@AGENTS.md` (o Claude Code lê `CLAUDE.md`, não `AGENTS.md`).
  - `.claude/arquitetura.md` — core da arquitetura **como está implementada** após este bootstrap: camadas, convenções efetivas, comandos reais e seção "Domínios" (índice contexto → arquivo).
  - `.claude/dominios/.gitkeep` — pasta vazia; cada ficha de domínio cria o seu `dominios/<contexto>.md` ao ser implementada.
  - Documentar só o que existe no código. O que será feito fica em `diceduc-docs` (fichas e specs), não nos repositórios.

### `diceduc-api` (NestJS + Prisma)

Estrutura conforme arquitetura-ignite.md §2:

- [ ] Projeto NestJS inicializado (`@nestjs/core`, `@nestjs/common`, `@nestjs/platform-express`, `@nestjs/config`).
- [ ] `src/core/`:
  - [ ] `entities/entity.ts` — `Entity<Props>`
  - [ ] `entities/aggregate-root.ts` — `AggregateRoot<Props>`
  - [ ] `entities/unique-entity-id.ts` — `UniqueEntityId`
  - [ ] `entities/value-object.ts` — `ValueObject<Props>`
  - [ ] `entities/watched-list.ts` — `WatchedList<T>`
  - [ ] `either.ts` — `Either<L, R>`, `left()`, `right()`, `isLeft()`, `isRight()`
  - [ ] `events/domain-events.ts` — barramento estático de eventos de domínio
  - [ ] `errors/errors/resource-not-found-error.ts`, `not-allowed-error.ts` (implementam `UseCaseError`)
- [ ] `src/infra/env/env.ts` — schema Zod (`DATABASE_URL`, `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY`, `PORT`), `env.module.ts`, `env.service.ts` (wrapper tipado sobre `ConfigService`).
- [ ] `src/infra/database/prisma/prisma.service.ts` — estende `PrismaClient`, `OnModuleInit`/`OnModuleDestroy`.
- [ ] `src/infra/database/database.module.ts` — registra `PrismaService` (sem repositórios ainda — cada ficha de domínio adiciona os seus).
- [ ] `src/infra/auth/`:
  - [ ] `jwt.strategy.ts` — valida payload com Zod, chave pública via env (base64). **Payload inclui `groupId` desde esta ficha** (claim obrigatório no schema Zod) — é a base que `F1-E0A` usa para popular o `GroupContext`; `jwt.strategy.ts` só valida o formato, não resolve escopo de escola (isso é responsabilidade do `GroupScopeGuard`, implementado em `F1-E0A` depois que `Group`/`School` existirem).
  - [ ] `jwt-auth.guard.ts` — registrado globalmente via `APP_GUARD`.
  - [ ] `public.decorator.ts` — `@Public()`.
  - [ ] `current-user.decorator.ts` — `@CurrentUser()`.
  - [ ] `auth.module.ts`.
- [ ] `src/infra/http/pipes/zod-validation-pipe.ts`.
- [ ] `src/infra/http/http.module.ts` — importa `DatabaseModule`; sem controllers ainda.
- [ ] `src/app.module.ts` — composition root: `ConfigModule.forRoot({ validate, isGlobal: true })`, `AuthModule`, `HttpModule`, `EnvModule`, `ThrottlerModule.forRoot(...)` + `{ provide: APP_GUARD, useClass: ThrottlerGuard }`.
- [ ] `src/main.ts` — `app.use(helmet())`, `app.set('trust proxy', 1)` (ver [arquitetura-ignite.md §12](../../../arquitetura-ignite.md#12-segurança-de-aplicação-baseline)). Sem `app.enableCors()` — a API só é chamada pelo servidor do `diceduc-web` (padrão BFF, ver `arquitetura-frontend.md` §6).
- [ ] `prisma/schema.prisma` — apenas datasource/generator configurados, sem models de negócio.
- [ ] `test/setup-e2e.ts` — cria schema Postgres isolado **por arquivo** de teste e2e (`randomUUID()`), roda `prisma migrate deploy`, derruba o schema no `afterAll` daquele arquivo.
- [ ] `vitest.config.ts` (unitário) e `vitest.config.e2e.ts` (e2e, com `setupFiles`).
- [ ] `.env.example` com todas as variáveis exigidas pelo schema de env.
- [ ] `docker-compose.yml` — Postgres local (dev **e** testes); só isso — não há Dockerfile da aplicação nesta ficha, a API roda com `pnpm dev`/`pnpm test` direto no host. Dockerfile de deploy entra no épico de deploy, ainda não especificado.
- [ ] `package.json` — script `audit`: `pnpm audit --prod --audit-level=high`.
- [ ] `.github/workflows/ci.yml` — `push` (qualquer branch): lint, typecheck, testes unitários. `pull_request` para `main`: os mesmos passos + testes e2e (Postgres como `services:` do job, mesma imagem do `docker-compose.yml`) + `pnpm audit`.
- [ ] `README.md` do repositório com instruções de setup local (`pnpm install`, subir Postgres via `docker compose up`, `pnpm prisma migrate dev`, `pnpm dev`) e uma nota sobre configurar a proteção da branch `main` no GitHub (merge só com os checks do CI verdes — configuração feita direto no GitHub, não neste repositório).

### `diceduc-web` (Next.js 16.3)

Estrutura conforme [arquitetura-frontend.md](../../../arquitetura-frontend.md) §2:

- [ ] Projeto Next.js 16.3 inicializado (App Router).
- [ ] Pastas base: `app/(public)/`, `app/(portal)/`, `features/`, `shared/ui/`, `shared/api/client.ts`, `shared/auth/`, `shared/lib/`.
- [ ] `shadcn init` — gera `components.json` apontando `shared/ui/` como diretório de componentes (ver [design-system.md §1](../../../design-system.md#1-stack)).
- [ ] `app/globals.css` — tokens do design system (`:root`/`.dark`, ver [design-system.md §2](../../../design-system.md#2-tokens-globalscss)) coladas exatamente como especificado, mais a config do Tailwind v4 (`@theme`/import, a validar contra a doc oficial).
- [ ] `app/layout.tsx` — `next/font/google` com Public Sans (`--font-heading`) e Roboto (`--font-sans`, ver [design-system.md §3](../../../design-system.md#3-tipografia)); `ThemeProvider` (`next-themes`, `attribute="class"`, `defaultTheme="system"`, `enableSystem`); `<html suppressHydrationWarning>` (ver [design-system.md §4](../../../design-system.md#4-modo-escuro)).
- [ ] `shared/ui/ThemeToggle.tsx` — alterna claro/escuro (ver [design-system.md §4](../../../design-system.md#4-modo-escuro)).
- [ ] `shared/api/client.ts` — client HTTP tipado apontando para a API (usa variável de ambiente para a base URL do `diceduc-api`, já que são repositórios/deploys separados), marcado `server-only` (ver `arquitetura-frontend.md` §4).
- [ ] `proxy.ts` — placeholder de proteção de rota (sem lógica de sessão/perfil ainda, só estrutura — a lógica real de sessão/renovação entra em `F1-E10`).
- [ ] `app/[group]/[school]/` — estrutura de pastas do tenant na URL (sem lógica de branding ainda, isso entra em `F1-E0A`); página pública mínima (`app/[group]/[school]/(public)/login/page.tsx`, sem shell) e página protegida mínima (`app/[group]/[school]/(portal)/page.tsx`, com o shell mínimo — sidebar + header com `ThemeToggle`, ver [design-system.md §6](../../../design-system.md#6-shell-do-portal), usando o componente `sidebar` do shadcn) para provar que o roteamento, o proxy, os tokens, a fonte e o tema funcionam juntos. `app/page.tsx` (raiz) com um texto simples de placeholder.
- [ ] `next.config.ts` — headers de segurança básicos (ver [arquitetura-ignite.md §12](../../../arquitetura-ignite.md#12-segurança-de-aplicação-baseline) e `arquitetura-frontend.md` §6; a API exata de configuração deve ser validada contra a doc do Next 16.3 na hora de implementar).
- [ ] `package.json` — script `audit`: `pnpm audit --prod --audit-level=high`.
- [ ] `.github/workflows/ci.yml` — `push`: lint, testes unitários. `pull_request` para `main`: os mesmos passos + `next build` + `pnpm audit`. (Playwright entra no CI deste workflow a partir de `F1-E08`, quando existem os primeiros testes e2e de UI.)
- [ ] `.env.example` com a URL da API (`diceduc-api`, rodando localmente em outra porta/processo).
- [ ] `README.md` do repositório com instruções de setup local (`pnpm install`, `pnpm dev`, variável de ambiente apontando para o `diceduc-api` local já rodando) e a mesma nota sobre proteção da branch `main`.

### Multi-tenancy

Esta ficha **não** implementa `Group`/`School` nem o mecanismo de escopo (`GroupContext`,
`GroupScopeGuard`, Prisma extension) — isso é o núcleo de
[`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md), que roda logo em seguida e ainda depende de models
Prisma que não existem neste bootstrap. A única responsabilidade de tenancy aqui é preparar o
terreno: o payload do JWT (`jwt.strategy.ts`, acima) já nasce com o campo `groupId`, para que
`F1-E0A` não precise voltar a mexer no schema de autenticação.

### Atualizar documentação (neste hub)

- [ ] Atualizar a seção "Comandos" de [`CLAUDE.md`](../../../../CLAUDE.md) com os comandos reais de cada repositório (dev, test unit, test e2e, migrate, build) assim que definidos.

## Arquivos a criar/editar (checklist)

```
diceduc-api/package.json
diceduc-api/AGENTS.md
diceduc-api/CLAUDE.md
diceduc-api/.claude/arquitetura.md
diceduc-api/.claude/dominios/.gitkeep
diceduc-api/src/core/**
diceduc-api/src/infra/env/**
diceduc-api/src/infra/database/prisma/prisma.service.ts
diceduc-api/src/infra/database/database.module.ts
diceduc-api/src/infra/auth/**
diceduc-api/src/infra/http/pipes/zod-validation-pipe.ts
diceduc-api/src/infra/http/http.module.ts
diceduc-api/src/main.ts
diceduc-api/src/app.module.ts
diceduc-api/prisma/schema.prisma
diceduc-api/test/setup-e2e.ts
diceduc-api/vitest.config.ts
diceduc-api/vitest.config.e2e.ts
diceduc-api/.env.example
diceduc-api/docker-compose.yml
diceduc-api/.github/workflows/ci.yml
diceduc-api/README.md

diceduc-web/package.json
diceduc-web/AGENTS.md
diceduc-web/CLAUDE.md
diceduc-web/.claude/arquitetura.md
diceduc-web/.claude/dominios/.gitkeep
diceduc-web/components.json
diceduc-web/app/globals.css
diceduc-web/app/layout.tsx
diceduc-web/app/page.tsx
diceduc-web/app/[group]/[school]/(public)/login/page.tsx
diceduc-web/app/[group]/[school]/(portal)/page.tsx
diceduc-web/shared/ui/ThemeToggle.tsx
diceduc-web/shared/api/client.ts
diceduc-web/proxy.ts
diceduc-web/next.config.ts
diceduc-web/.env.example
diceduc-web/.github/workflows/ci.yml
diceduc-web/README.md
```

## Testes

- [ ] `diceduc-api`: um spec trivial (ex.: `either.spec.ts`) só para validar que o harness Vitest unitário roda.
- [ ] `diceduc-api`: um e2e trivial (ex.: health check endpoint público) para validar que `setup-e2e.ts` sobe o schema isolado corretamente **e** que a resposta traz o header `x-content-type-options: nosniff` (prova de que o `helmet()` está ativo).
- [ ] `diceduc-web`: build (`next build`) sem erros.

## Definition of Done

- [ ] `pnpm install` funciona de forma independente em cada repositório (`diceduc-api` e `diceduc-web`).
- [ ] `diceduc-api` sobe localmente (`pnpm dev` dentro do repositório) e conecta ao Postgres do `docker-compose.yml`.
- [ ] `diceduc-web` sobe localmente (`pnpm dev` dentro do repositório, apontando via env para o `diceduc-api` local) e a página protegida redireciona para login sem sessão.
- [ ] A página protegida renderiza com os tokens do design system, a fonte (Public Sans/Roboto) e o shell mínimo (sidebar + header); alternar claro/escuro no `ThemeToggle` funciona.
- [ ] `prisma migrate dev` aplica a migração inicial (vazia) sem erro.
- [ ] Suítes de teste (unit e e2e) do `diceduc-api` rodam e passam (mesmo que triviais).
- [ ] `CLAUDE.md` (neste hub) atualizado com os comandos reais de cada repositório.
- [ ] Cada repositório tem `AGENTS.md` → `.claude/arquitetura.md` descrevendo o que foi implementado, `CLAUDE.md` com `@AGENTS.md` e `.claude/dominios/` vazia.
- [ ] Nenhum código de domínio de negócio incluído nesta ficha — isso é responsabilidade de `F1-E01` em diante.
