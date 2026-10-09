# Arquitetura de referência: frontend (Next.js 16.3)

> **Proposta revisável.** Diferente de [arquitetura-ignite.md](arquitetura-ignite.md) (que documenta um projeto de referência existente), este documento é uma convenção proposta para o repositório `diceduc-web` do DICEDUC, pensada para espelhar o mesmo nível de organização do backend (repositório `diceduc-api`). Ajuste livremente antes de considerá-la definitiva.
>
> **Nota de versão:** a API exata do Next.js 16.3 (nomes de APIs, comportamento padrão de cache, etc.) deve ser validada contra a documentação oficial no momento da implementação de cada ficha — não assuma este documento como fonte de verdade sobre a API do framework, só sobre a organização do projeto.

## 1. Visão geral

`diceduc-web` consome a API do `diceduc-api` via um client HTTP tipado. Organização por **feature** (não por tipo de arquivo), espelhando os bounded contexts do backend sempre que fizer sentido para a navegação do usuário (ex.: contexto `academic` → área "Turmas" no app).

```
app        →  camada de rotas (App Router) — fina, delega para features/
features   →  lógica de UI por domínio (componentes, hooks, schemas, api-client)
shared     →  design system, utilitários genéricos, client HTTP base
```

Regra de dependência: `app` depende de `features`; `features` pode depender de `shared`; `shared` não depende de nada de `app`/`features`.

## 2. Estrutura de pastas

Toda navegação fica sob o tenant identificado na própria URL — `app/[grupo]/[escola]/…` — porque
uma sessão sempre pertence a uma escola (ver `arquitetura-ignite.md` §11). As fichas de feature
escrevem os caminhos como `app/(portal)/…`/`app/(public)/…` por brevidade; leia-os como relativos
a `app/[grupo]/[escola]/`:

```
diceduc-web/
  app/
    [grupo]/
      [escola]/
        layout.tsx             busca branding (nome/logo/cor) via GET /public/branding/:grupo/:escola
        (public)/                rotas sem sessão (login, primeiro acesso, esqueci/redefinir senha — atrás de flag)
        (portal)/
          alunos/
          turmas/
          financeiro/
          ...                     uma pasta de rota por área, componente de página fino
    page.tsx                  raiz "/" — página simples "acesse pelo link da sua escola"
    layout.tsx
    globals.css
  features/
    <bounded-context>/         ex.: academic, attendance, assessment, finance
      components/               componentes de UI específicos do contexto
      hooks/                     hooks de dados (data-fetching) do contexto
      api/                       funções tipadas que chamam a API (usam o client de shared/api)
      schemas/                   validação Zod dos formulários/DTOs do contexto
      types.ts
  shared/
    ui/                          design system (botão, input, tabela, dialog...)
    api/
      client.ts                  client HTTP base (fetch tipado, injeta auth token)
    auth/                        sessão, guards de rota, hook useSession
    lib/                         utilitários genéricos (formatação de data, etc.)
  proxy.ts                       proteção de rotas por sessão/perfil
  next.config.ts
```

Cada `features/<contexto>` é autocontido e opcionalmente mapeia 1:1 para um bounded context do backend (`diceduc-api/src/domain/<contexto>`), facilitando encontrar "onde mora a tela que fala com este use case".

## 3. Camada de rotas (`app/`)

- App Router, Server Components por padrão; `"use client"` só onde há interatividade/estado local.
- Componentes de página são **finos**: buscam dados (via `features/<contexto>/api` ou hooks) e compõem componentes de `features/<contexto>/components`. Nada de lógica de negócio dentro de `app/`.
- Agrupamento de rotas por área de acesso: `(public)` (sem sessão) e `(portal)` (autenticado, com `proxy.ts` validando/renovando a sessão e `getSession()` resolvendo o papel).
- `app/[grupo]/[escola]/layout.tsx` busca o branding do tenant (`GET /public/branding/:grupo/:escola`, com `cache()` do React) e aplica nome/logo/cor: `School.primaryColor` → `Group.primaryColor` → cor padrão do design system, sobrescrevendo só os tokens de marca (`--primary`, `--primary-foreground`, `--ring`, `--sidebar-primary`, `--sidebar-ring`) — detalhe completo da cascata e da derivação em [design-system.md §5](design-system.md#5-branding-por-tenant). Slug de grupo ou escola inexistente → `notFound()`.
- Rotas dinâmicas (`[id]`) e paralelas/interceptadas apenas quando o caso de uso exigir (ex.: modal de detalhe de aluno sobre a lista).

## 4. Camada de dados (`features/<contexto>/api` + `shared/api`)

- `shared/api/client.ts`: client HTTP único (`fetch` tipado), marcado `server-only` — só roda no servidor do Next (padrão BFF, ver §6). Injeta o access token como `Bearer` a partir do cookie da sessão, trata erros de forma consistente (mapeia erro HTTP → tipo de erro de domínio conhecido pelo frontend). Mutações disparadas por Client Components passam por Server Actions, que chamam este client — o browser nunca importa `client.ts` diretamente.
- `features/<contexto>/api/`: uma função por operação (`getStudent`, `createStudent`, `listTurmas`...), tipada com os DTOs de resposta da API. Não expõe `fetch` cru para os componentes.
- Data-fetching em Server Components sempre que possível (menos JS no client); Client Components usam hooks de `features/<contexto>/hooks/` para mutações e dados que dependem de interação.
- Sem duplicar validação: os schemas Zod usados nos formulários (`features/<contexto>/schemas/`) devem espelhar os schemas de entrada do backend descritos na ficha da tarefa correspondente.

## 5. Formulários e validação

- React Hook Form + Zod resolver, usando os schemas de `features/<contexto>/schemas/`.
- Mensagens de erro client-side e as regras de negócio retornadas pela API (`Either` mapeado para erro HTTP) convergem para o mesmo componente de exibição de erro do design system (`shared/ui`).

## 6. Autenticação e autorização

Padrão **BFF** (backend-for-frontend): o browser nunca chama o `diceduc-api` diretamente, só o
servidor do `diceduc-web` (Server Components, Server Actions, Route Handlers). Isso elimina CORS
(a API não precisa aceitar origem de browser) e reduz a superfície de CSRF a Server Actions +
checagem de `Origin`. Detalhe completo de emissão/renovação de sessão em
[F1-E10](docs/features/F1-E10-identidade-acesso/spec-identidade-acesso.md).

- **Cookies:** `diceduc_access` (15 min) e `diceduc_refresh` (7 dias), ambos httpOnly, Secure,
  `SameSite=Lax`, com `Path=/{grupoSlug}` — permite ficar logado em grupos diferentes ao mesmo
  tempo no mesmo navegador (ex.: professor que leciona em duas secretarias).
- **`proxy.ts`:** extrai grupo e escola do pathname. Sem cookie → redireciona para o login daquela
  escola. Access expirado (ou a menos de 1 min de expirar) → chama `POST /sessions/refresh` e grava
  os cookies novos antes de servir a página (cookie só pode ser escrito no proxy, numa Server
  Action ou num Route Handler — nunca durante a renderização). Refresh inválido/expirado → apaga
  os cookies e vai para o login. **Se o `schoolId` da sessão não bate com a escola da URL**, apaga
  os cookies (com `POST /sessions/logout` best-effort) e redireciona para o login daquela escola —
  mudar de escola sempre exige novo login (ver `arquitetura-ignite.md` §11).
- **`getSession()`** (`shared/auth/session.ts`, `server-only`): chama `GET /me`, envolvido em
  `cache()` do React — uma única chamada por requisição, mesmo sendo lido no layout e em várias
  páginas/componentes da mesma árvore (necessário porque um layout não é re-renderizado em
  navegação entre páginas irmãs, então a checagem de papel precisa acontecer em cada página
  também, não só no layout). O cache não sobrevive entre requisições, então uma conta desativada
  ou um papel removido refletem já na próxima navegação. A autoridade de fato continua sendo a
  API — `getSession()` só evita telas/ações visíveis para quem não tem o papel, nunca é a única
  proteção do dado.
- **CSRF:** coberto pelas proteções nativas de Server Actions do Next, mais checagem de `Origin`
  em Route Handlers que fazem mutação.
- Controle de visibilidade por perfil (secretaria, coordenação, professor, responsável, aluno) feito tanto no `proxy.ts`/layout (rotas inteiras, via `getSession()`) quanto em componentes (`shared/auth` expõe hook/guard para esconder ações específicas).
- Headers de segurança e CSP configurados em `next.config.ts` (ver docs oficiais do Next 16.3 para a API exata).

## 7. Design system (`shared/ui`)

- Componentes de UI puros, sem chamada de API nem lógica de negócio — recebem dados via props.
- Base para os componentes específicos de `features/<contexto>/components`, que combinam componentes de `shared/ui` com dados/hooks do contexto.
- Tokens, tipografia, tema claro/escuro, branding por tenant, shell do portal e inventário de componentes: ver [design-system.md](design-system.md) — leitura obrigatória junto com este documento para qualquer tarefa de frontend (ver `CLAUDE.md`).

## 8. Testes

- Componentes/hooks: Vitest + Testing Library, mesma stack de teste do backend por consistência de tooling entre os repositórios.
- Mocks de API: funções de `features/<contexto>/api` são mockadas nos testes de componente — nunca mocka-se `fetch` diretamente.
- E2E de fluxo crítico (login, matrícula, lançamento de nota): Playwright, rodando contra `diceduc-api` local.

## 9. Principais dependências (proposta)

| Categoria | Bibliotecas |
| --- | --- |
| Framework | `next` (16.3), `react`, `react-dom` |
| Formulários | `react-hook-form`, `@hookform/resolvers`, `zod` |
| Estado servidor/cache | recursos nativos do App Router (`fetch` cache, Server Actions) — evitar lib extra de data-fetching a menos que a necessidade apareça |
| UI | `tailwindcss` (v4), `shadcn/ui` (componentes copiados para `shared/ui/`, não pacote), `next-themes` (tema claro/escuro), `next/font` (Public Sans + Roboto) — ver [design-system.md](design-system.md) |
| Testes | `vitest`, `@testing-library/react`, `playwright` |
| Segredos de build | `PASSWORD_RESET_ENABLED` (flag da recuperação de senha por e-mail, default `false` — ver `F1-E10`) |

## 10. Multi-tenancy

O usuário autenticado pertence a um `Group` (tenant) e, dentro dele, sua sessão está sempre presa
a uma única `School` — ver [arquitetura-ignite.md §11](arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero)
para o modelo completo. Implicações no frontend:

- **Tenant na URL:** o grupo e a escola vêm do path (`app/[grupo]/[escola]/…`), não de subdomínio
  — um subdomínio por tenant (`<group>.diceduc.app`) segue como evolução possível, não faz parte do
  MVP.
- `shared/auth`: `useSession()` expõe `groupId`, `schoolId` e `role` resolvidos do backend (via
  `getSession()`) — nunca inferidos/setados no cliente. Não existe mais `allowedSchoolIds`: como a
  sessão é de uma única escola, não há lista para expor.
- **Sem seletor de escola:** trocar de escola sempre passa por um novo login (ver §6), então não
  existe componente de seletor no MVP. A tela `(portal)/minha-conta` lista, como links, as outras
  escolas do group a que o usuário tem acesso, avisando que abrir um link exige logar de novo.
- `shared/api/client.ts` nunca envia `groupId`/`schoolId` como parâmetro de autoridade — o backend
  resolve os dois a partir da sessão; qualquer `schoolId` que a UI mande em filtros de tela é só
  redundante com o da sessão, revalidado no backend.

## 11. O que aproveitar do padrão do backend

- Organização por domínio (`features/<contexto>` espelhando `domain/<contexto>`) mantém a navegação mental consistente entre os dois repositórios (`diceduc-api` e `diceduc-web`).
- Reuso de Zod entre frontend e backend reduz duplicação de regras de validação (mesmo que os schemas não sejam literalmente compartilhados via pacote comum, a *fonte* de cada regra é a ficha da tarefa).
- Erros de negócio (`Either` no backend) devem ter um mapeamento único e prático no client HTTP, para que toda tela trate erro de negócio de forma consistente em vez de cada componente reinventar tratamento de erro.
