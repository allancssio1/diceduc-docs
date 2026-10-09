# F1-E0A — Multi-tenancy (Group e School)

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E0A — Multi-tenancy (Group e School) |
| Fase | F1 (pré-requisito de todas as demais fichas de domínio) |
| Prioridade | Must |
| Estimativa | M |
| Bounded context(s) | `tenancy` |
| Depende de | F1-E00 (bootstrap) |
| Rastreabilidade | Decisão de arquitetura — ver [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) para o modelo completo (esta ficha implementa esse modelo). |

## Objetivo

Implementar a fundação multi-tenant: entidades `Group` (tenant) e `School`, o onboarding de ambas,
e o mecanismo de isolamento (`groupId`) + escopo de permissão (`schoolId`) que todo épico de
domínio subsequente (F1-E01 em diante) vai consumir. **Leia primeiro
[`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero)**
— esta spec implementa o modelo descrito lá, não repete a explicação conceitual.

## Modelo de domínio (`enterprise`)

### Contexto `tenancy` — `diceduc-api/src/domain/tenancy/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `Group` | `AggregateRoot` | `group.ts` | `name`, `slug: Slug` (único global), `type: GroupType`, `document?` (CNPJ), `status: GroupStatus`, `dpoName?`, `dpoEmail?` (encarregado LGPD do controlador, exibido em `F1-E09`), `logoUrl?`, `primaryColor?`, `createdAt`, `updatedAt` |
| `School` | `Entity` | `school.ts` | `groupId`, `name`, `slug: Slug` (único por group), `inepCode?`, `address: Address` (reusa VO de `people`, `F1-E01` — se `F1-E0A` for implementado antes, declarar o VO aqui e `F1-E01` reusa de `tenancy`; a ordem de quem declara não importa, o importante é não duplicar), `status: SchoolStatus`, `logoUrl?`, `primaryColor?` (sobrescrevem o do `Group` quando presentes), `createdAt`, `updatedAt` |

Value objects — `entities/valueObjects/`:
- `slug.ts` — `Slug` (kebab-case, `^[a-z0-9]+(-[a-z0-9]+)*$`, validado na criação — usado tanto por `Group` quanto por `School`).
- `group-type.ts` — `GroupType = 'SECRETARIA' | 'MUNICIPIO' | 'REDE' | 'ESCOLA'` (rótulo informativo, não muda nenhum comportamento de isolamento).
- `group-status.ts` — `GroupStatus = 'ACTIVE' | 'SUSPENDED'`.
- `school-status.ts` — `SchoolStatus = 'ACTIVE' | 'INACTIVE'`.

Invariantes:
- `Group` e `School` **não** carregam `groupId`/`schoolId` como as demais entidades do sistema — são elas a raiz da hierarquia de tenancy.
- `School.groupId` deve referenciar um `Group` existente com `status = 'ACTIVE'`.
- Um `Group` com `status = 'SUSPENDED'` bloqueia login de qualquer usuário vinculado a ele (verificado no `GroupScopeGuard`, não nesta entidade).
- `logoUrl` é uma URL externa informada no onboarding (`ponytail:` sem upload próprio; entra quando houver um port de storage no projeto).
- `primaryColor` é **opcional com fallback**, não "a cor do tenant": quando ausente em `School` e em `Group`, o frontend usa a cor padrão do design system — o backend nunca inventa uma cor. O valor é um hex simples (`string?`, sem VO próprio); toda a derivação para tokens CSS (contraste, variantes claro/escuro) é responsabilidade do `diceduc-web`, não desta ficha — ver [design-system.md §5](../../../design-system.md#5-branding-por-tenant).

## Use cases (`application`)

### `diceduc-api/src/domain/tenancy/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `RegisterGroupUseCase` | `register-group.ts` | `name`, `slug`, `type`, `document?`, `dpoName?`, `dpoEmail?` | `Either<InvalidDocumentError \| SlugAlreadyInUseError, { group: Group }>` | `invalid-document-error.ts`, `slug-already-in-use-error.ts` |
| `RegisterSchoolUseCase` | `register-school.ts` | `groupId`, `name`, `slug`, `inepCode?`, `address` | `Either<ResourceNotFoundError \| SlugAlreadyInUseError, { school: School }>` | reusa `slug-already-in-use-error.ts` |
| `ListGroupSchoolsUseCase` | `list-group-schools.ts` | `groupId` | `Either<ResourceNotFoundError, { schools: School[] }>` | — |
| `GetTenantBrandingUseCase` | `get-tenant-branding.ts` | `groupSlug`, `schoolSlug` | `Either<ResourceNotFoundError, { groupName, schoolName, logoUrl?, primaryColor? }>` | — (grupo `SUSPENDED`, escola `INACTIVE` ou slug inexistente → `ResourceNotFoundError`, sem distinguir o motivo) |

Ports (`application/repositories/`): `GroupsRepository` (inclui `findBySlug`), `SchoolsRepository` (inclui `findBySlug(groupId, slug)`).

Regras de negócio principais:
- `RegisterGroupUseCase`/`RegisterSchoolUseCase` rodam **fora** do `GroupContext` (onboarding —
  ver seção HTTP): não há `groupId` de sessão ainda quando um `Group` está sendo criado pela
  primeira vez.
- `RegisterSchoolUseCase`: valida que `groupId` existe e está `ACTIVE` antes de criar a escola.
- Quem pode chamar estes use cases é resolvido no controller: `PlatformAdminGuard` (ver seção
  HTTP) — só a equipe DICEDUC cria groups/escolas no MVP, não há auto-cadastro público.
- `GetTenantBrandingUseCase`: roda dentro de um `GroupContext` **preliminar** (só `groupId`,
  resolvido a partir do `groupSlug`) — o suficiente para localizar `School` por `slug` sem expor
  dado de outro group.

## Infra transversal de escopo (`core`/`infra`)

Esta é a parte mais importante da ficha — o contrato que **todos** os outros contextos vão usar.

### `GroupContext` — `diceduc-api/src/core/tenancy/group-context.ts`

```ts
export type GroupContext = {
  groupId: string
  schoolId: string
  allowedStudentIds: string[] | null // null = sem restrição; [] = nenhum aluno — preenchido em F1-E09
}
```

Implementado com `AsyncLocalStorage<GroupContext>`, expondo `run(context, callback)` (popula) e
`get(): GroupContext` (lê o contexto corrente; lança erro de programação — não de negócio — se
chamado fora de uma requisição autenticada, já que isso indica um bug de wiring, não um caso de erro esperado).
**Uma sessão sempre pertence a uma única escola** (decisão de produto): não existe lista de
escolas permitidas, só a `schoolId` corrente — ver `arquitetura-ignite.md` §11.

### `GroupScopeGuard` — `diceduc-api/src/infra/auth/group-scope.guard.ts`

- Roda **depois** do `JwtAuthGuard` já existente (`F1-E00`).
- Lê `groupId`/`schoolId` do payload do access token (`@CurrentUser()` — `F1-E10` é quem emite
  esse token com os dois campos).
- Resolve `allowedStudentIds` (port `AllowedStudentsProvider`, de `F1-E09` — nesta ficha, se
  `F1-E09` ainda não estiver implementado, usar um provider stub `AllowAllStudentsProvider` que
  sempre retorna `allowedStudentIds: null`, substituído pela implementação real quando `F1-E09`
  existir, análogo ao padrão de stub já usado em `F1-E02`).
- Popula `GroupContext.run({ groupId, schoolId, allowedStudentIds }, () => next())`.
- Se `Group.status = 'SUSPENDED'`, retorna `403` antes de prosseguir.

### `PlatformAdminGuard` — `diceduc-api/src/infra/auth/platform-admin.guard.ts`

- Guard próprio para as rotas de onboarding (criação de `Group`/`School`) — só a equipe DICEDUC cria
  clientes no MVP, sem auto-cadastro público.
- Compara o header `x-platform-key` com a env `PLATFORM_ADMIN_KEY` (schema Zod, string, mínimo 32
  caracteres) usando `crypto.timingSafeEqual` (evita timing attack numa comparação de segredo).
- Combinado com `@Public()` (roda antes/fora do `JwtAuthGuard`/`GroupScopeGuard`) e com
  `@Throttle()` mais restritivo que o padrão (ver [arquitetura-ignite.md §12](../../../arquitetura-ignite.md#12-segurança-de-aplicação-baseline)).

### Prisma Client Extension — `diceduc-api/src/infra/database/prisma/extensions/tenant-scope.extension.ts`

- Aplicada ao `PrismaService` (`$extends`) para todo model que tiver `groupId` no schema.
- **Leitura** (`findMany`, `findFirst`, `findUnique`→`findFirst` quando precisa filtrar, `count`,
  etc.): mescla `where: { groupId: GroupContext.get().groupId }`; se o model tiver `schoolId`,
  mescla também `where: { schoolId: GroupContext.get().schoolId }` (valor único, não uma lista). Se
  `allowedStudentIds !== null`: para o model `Student`, filtra por `id`; para qualquer outro model
  com coluna `studentId`, filtra por ela.
- **Escrita** (`create`, `createMany`): mescla `data: { groupId: GroupContext.get().groupId }` e,
  quando o model tem a coluna, `data: { schoolId: GroupContext.get().schoolId }` — os dois sempre
  do contexto, nunca de um DTO de entrada. Como só existe uma escola por sessão, não há mais
  necessidade de um helper de validação contra uma lista de escolas permitidas (ver nota abaixo).
- Models `Group`/`School` ficam **fora** desta extension (são a raiz, não têm `groupId`).

**Nota para quem já viu uma versão anterior desta spec:** o helper `assertSchoolInScope` e o erro
`SchoolNotInScopeError` **não existem mais** — eram necessários quando uma sessão podia enxergar
várias escolas (`allowedSchoolIds`); com uma escola por sessão, a extension já resolve sozinha, e
os use cases de escrita dos épicos seguintes **não recebem `schoolId` como parâmetro de entrada**.

## Persistência (`infra/database/prisma`)

- `diceduc-api/prisma/schema.prisma`: novos models `Group`, `School` (sem `groupId`/`schoolId`
  próprios); enums `GroupType`, `GroupStatus`, `SchoolStatus`. A partir desta ficha, **todo novo
  model de negócio criado em qualquer épico seguinte inclui `groupId String` indexado** (e
  `schoolId String` indexado, se operacional) — convenção registrada em `arquitetura-ignite.md §11`.
- Repositórios: `prisma-groups-repository.ts`, `prisma-schools-repository.ts` em `infra/database/prisma/repositories/`.
- Extension: `infra/database/prisma/extensions/tenant-scope.extension.ts` (ver seção acima).
- Mappers: `group-mapper.ts`, `school-mapper.ts`.

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Presenter | Contexto |
| --- | --- | --- | --- | --- |
| `POST /groups` | `register-group.controller.ts` | dados de `RegisterGroupUseCase` | `group-presenter.ts` | `@Public()` + `PlatformAdminGuard` + `@Throttle` — fora do `GroupScopeGuard` |
| `POST /groups/:groupId/schools` | `register-school.controller.ts` | dados de `RegisterSchoolUseCase` (exceto `groupId`) | `school-presenter.ts` | idem — onboarding |
| `GET /schools` | `list-group-schools.controller.ts` | — | `school-presenter.ts` (lista) | dentro do `GroupScopeGuard` normal (usa `groupId` da sessão, não de param) |
| `GET /public/branding/:groupSlug/:schoolSlug` | `get-tenant-branding.controller.ts` | — (params) | JSON simples `{ groupName, schoolName, logoUrl?, primaryColor? }` | `@Public()` + `@Throttle` — usado pela tela de login antes de qualquer autenticação |

O onboarding (`POST /groups`, `POST /groups/:groupId/schools`) roda sem `GroupScopeGuard` porque
ainda não existe usuário/sessão vinculada ao `Group` que está sendo criado — em vez disso, exige a
chave de administrador da plataforma (`PlatformAdminGuard`). Só a equipe DICEDUC cria groups/escolas
no MVP; a criação do primeiro usuário admin de cada group é `ProvisionGroupAdmin`, em `F1-E10`.
Exemplo de uso via `curl` documentado no README do `diceduc-api` (não há tela de onboarding no
`diceduc-web` — ver Frontend).

## Frontend (`diceduc-web`)

Sem telas de onboarding — a criação de `Group`/`School` é feita pela equipe DICEDUC direto na API
(ver seção HTTP). O `diceduc-web` só consome o que já existe:

- `app/[grupo]/[escola]/(portal)/admin/escolas/page.tsx` — lista de escolas do group logado (`GET /schools`), tela administrativa (não há mais seletor de escola — ver `arquitetura-frontend.md` §10, uma sessão é sempre de uma escola).
- `features/tenancy/api/tenancy.ts` (`listGroupSchools`, `getTenantBranding`), `features/tenancy/schemas/tenancy.ts`.
- `shared/auth/useSession.ts`: passa a expor `groupId` e `schoolId` (ver `arquitetura-frontend.md §10`) — não mais `allowedSchoolIds`.

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/tenancy/enterprise/entities/group.ts`
- [ ] `diceduc-api/src/domain/tenancy/enterprise/entities/school.ts`
- [ ] `diceduc-api/src/domain/tenancy/enterprise/entities/valueObjects/slug.ts`
- [ ] `diceduc-api/src/domain/tenancy/enterprise/entities/valueObjects/group-type.ts`
- [ ] `diceduc-api/src/domain/tenancy/enterprise/entities/valueObjects/group-status.ts`
- [ ] `diceduc-api/src/domain/tenancy/enterprise/entities/valueObjects/school-status.ts`
- [ ] `diceduc-api/src/domain/tenancy/application/useCases/register-group.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/tenancy/application/useCases/register-school.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/tenancy/application/useCases/list-group-schools.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/tenancy/application/useCases/get-tenant-branding.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/tenancy/application/repositories/groups-repository.ts`
- [ ] `diceduc-api/src/domain/tenancy/application/repositories/schools-repository.ts`
- [ ] `diceduc-api/src/core/tenancy/group-context.ts`
- [ ] `diceduc-api/src/infra/auth/group-scope.guard.ts`
- [ ] `diceduc-api/src/infra/auth/platform-admin.guard.ts`
- [ ] `diceduc-api/src/infra/auth/allow-all-students-provider.ts` (stub, até `F1-E09` existir)
- [ ] `diceduc-api/prisma/schema.prisma` (editar — models `Group`/`School` + convenção `groupId`/`schoolId` para todo model futuro)
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-groups-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-schools-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/extensions/tenant-scope.extension.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/group-mapper.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/school-mapper.ts`
- [ ] `diceduc-api/src/infra/http/controllers/register-group.controller.ts`
- [ ] `diceduc-api/src/infra/http/controllers/register-school.controller.ts`
- [ ] `diceduc-api/src/infra/http/controllers/list-group-schools.controller.ts`
- [ ] `diceduc-api/src/infra/http/controllers/get-tenant-branding.controller.ts`
- [ ] `diceduc-api/src/infra/http/presenters/group-presenter.ts`, `school-presenter.ts`
- [ ] `diceduc-web/app/[grupo]/[escola]/(portal)/admin/escolas/page.tsx`
- [ ] `diceduc-web/features/tenancy/api/tenancy.ts`
- [ ] `diceduc-web/features/tenancy/schemas/tenancy.ts`
- [ ] `diceduc-web/shared/auth/useSession.ts` (editar)

## Testes

- Repositórios in-memory: `in-memory-groups-repository.ts`, `in-memory-schools-repository.ts`
  (e, a partir desta ficha, **todo** repositório in-memory de qualquer contexto passa a filtrar
  por `groupId`/`schoolId`/`allowedStudentIds` manualmente, replicando a extension — documentar
  isso no arquivo de cada repositório in-memory subsequente).
- Factories: `make-group.ts`, `make-school.ts`.
- Unit specs: `RegisterSchoolUseCase` rejeita `groupId` inexistente ou `SUSPENDED`; `RegisterGroupUseCase`/`RegisterSchoolUseCase` rejeitam `slug` duplicado; `GetTenantBrandingUseCase` retorna `ResourceNotFoundError` para slug inexistente, grupo `SUSPENDED` ou escola `INACTIVE`; `GetTenantBrandingUseCase` sem `primaryColor` em `School` nem em `Group` devolve o campo ausente (não uma cor default) — quem decide o fallback visual é o `diceduc-web`, não o use case.
- Teste de integração dedicado da extension: criar duas entidades de teste em `Group`s diferentes
  via Prisma real, provar que uma query sem filtro explícito (usando o client já estendido) só
  retorna a do `Group`/escola do contexto corrente — este teste é a prova de que o isolamento
  funciona antes de qualquer contexto de domínio ser implementado sobre ele. Cobrir também o
  filtro por `allowedStudentIds`.
- E2E: um `.e2e-spec.ts` por controller listado na seção HTTP, incluindo `POST /groups` sem o
  header `x-platform-key` → `403`, e `GET /public/branding/...` com slug válido → `200` sem
  nenhum campo sensível.

## Definition of Done

- [ ] `Group`/`School` implementados com use cases de onboarding testados.
- [ ] `GroupContext`, `GroupScopeGuard`, `PlatformAdminGuard` e a Prisma extension implementados e
      cobertos pelo teste de integração de isolamento descrito em Testes.
- [ ] Migração Prisma aplicada sem erro.
- [ ] Onboarding de group + escolas (via API, com `PlatformAdminGuard`) e branding público
      funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E0A` para 🟢 quando implementado — **antes** de qualquer épico de domínio (F1-E01 em diante) ser iniciado.
