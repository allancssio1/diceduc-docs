# Arquitetura de referência: nest-clean-ignite

> Documento gerado a partir da análise do projeto `nest-clean-ignite` (curso Rocketseat), para servir de referência arquitetural ao `diceduc-controle-escolar`. Stack: NestJS + Prisma + TypeScript, seguindo Clean Architecture com DDD leve.

## 1. Visão geral

O projeto segue **Clean Architecture** com padrões táticos de **DDD**, organizado em duas "bounded contexts" (`forum` e `notification`), cada uma isolada em `domain/<contexto>`. A regra de dependência é sempre para dentro:

```
infra  →  domain/application  →  domain/enterprise
                    ↓
                  core
```

- `enterprise` (regras de negócio da empresa): entidades, value objects, eventos — zero dependência de framework.
- `application` (regras de negócio da aplicação): use cases + interfaces de repositório/serviço (ports).
- `infra`: tudo que é framework/IO — NestJS, Prisma, HTTP, auth, storage. Implementa os ports definidos em `application`.
- `core`: base compartilhada entre todos os contextos (Entity, Either, eventos de domínio, erros genéricos).

## 2. Estrutura de pastas

```
src/
  core/          Building blocks compartilhados (Entity, AggregateRoot, Either, DomainEvents, erros genéricos)
  domain/
    forum/
      application/    use cases, repositories (interfaces/ports), cryptography, storage
      enterprise/     entidades, value objects, eventos de domínio
    notification/
      application/    use cases, repository (interface), subscribers de eventos
      enterprise/     entidade Notification
  infra/
    database/prisma/  PrismaService, repositórios Prisma, mappers
    http/              controllers, pipes, presenters
    auth/              JWT strategy, guards, decorators
    cryptography/       adapters (bcrypt, jwt)
    storage/            adapter (Cloudflare R2 / S3)
    env/                validação e acesso tipado a variáveis de ambiente
test/
  repositories/    repositórios in-memory (para testes unitários)
  factories/       factories de entidades (unit + persistência via Prisma para e2e)
  cryptography/    fakes de hash/encrypter
  storage/         fake uploader
  setup-e2e.ts     isola schema de banco por execução de teste e2e
prisma/
  schema.prisma
  migrations/
```

Cada `domain/<contexto>` é autocontido (`application` + `enterprise`) e plugado em `infra` de forma independente — facilita adicionar novos contextos sem acoplar aos existentes.

## 3. Camada de domínio (`enterprise`)

### Classes base (`src/core/entities/`)

- **`Entity<Props>`**: guarda `protected props: Props` + `UniqueEntityId` privado; expõe `.id` e `.equals()`.
- **`AggregateRoot<Props>`**: estende `Entity`, adiciona lista de `domainEvents` e `addDomainEvent()` (marca o agregado para disparo de eventos).
- **`UniqueEntityId`**: wrapper sobre `randomUUID()` do Node, com `toString()/toValue()/equals()`.
- **`ValueObject<Props>`**: base para VOs com igualdade estrutural.
- **`WatchedList<T>`**: lista genérica que rastreia itens `current/new/removed` — usada para coleções que pertencem a um agregado (ex.: anexos de uma pergunta), permitindo saber o que inserir/remover ao persistir.

### Padrão `Optional` para criação de entidades

```ts
type Optional<T, K extends keyof T> = Pick<Partial<T>, K> & Omit<T, K>
```

Usado no `create()` de cada entidade para tornar opcionais os campos deriváveis/padrão (ex.: `createdAt`, `slug`), mantendo-os obrigatórios internamente:

```ts
static create(props: Optional<QuestionProps, 'createdAt' | 'slug' | 'attachments'>, id?: UniqueEntityId)
```

### Entidades e value objects

- Entidades em `src/domain/forum/enterprise/entities/` (`Question`, `Answer`, `Comment`, `Attachment`, `Student`, `Instructor`, etc.). `Question` e `Answer` são `AggregateRoot`; comentários/anexos são `Entity` simples.
- Construtor privado + getters/setters que mutam `this.props` e chamam um `touch()` privado para atualizar `updatedAt`.
- Value objects em `entities/valueObjects/` (ex.: `Slug` — gera slug a partir do título).
- **Idioma dos nomes: sempre inglês.** Entidade, model, campo, enum, rota, nome de arquivo e comentário — tudo em inglês, mesmo quando o termo é pedagógico brasileiro sem tradução 1:1 natural (ex.: `Turma`→`ClassGroup`, `Serie`→`GradeLevel`). **Decisão (2026-10-12): revertida a decisão anterior de manter `Serie`/`Turma`/`Disciplina`/`Sala`/`Horario` em português — foram todos renomeados** (histórico, SQL das migrations incluído, já que não havia banco implantado em produção). Só prosa de documentação, texto de UI (título, label, mensagem exibida ao usuário) e dado de teste (nome de aluno/escola etc.) continuam em português. Glossário de referência:

  | Português | Inglês |
  |---|---|
  | `Turma` | `ClassGroup` |
  | `Serie` | `GradeLevel` |
  | `Disciplina` | `Subject` |
  | `Sala` | `Room` |
  | `Horario` | `ScheduleSlot` |
  | `Turno` (`MANHA/TARDE/NOITE/INTEGRAL`) | `Shift` (`MORNING/AFTERNOON/EVENING/FULL_TIME`) |

  Siglas e nomes próprios brasileiros sem equivalente (CPF, RG, INEP, LGPD, BNCC, Pix) continuam como estão — não são o tipo de termo pedagógico genérico a que a decisão anterior se referia.

### Eventos de domínio

- Barramento estático `DomainEvents` (`src/core/events/domain-events.ts`): mapa de handlers por nome da classe do evento + lista de agregados marcados para disparo.
- Eventos concretos em `src/domain/forum/enterprise/events/` (ex.: `AnswerCreatedEvent`), disparados dentro de setters da entidade via `this.addDomainEvent(...)`.
- Repositórios chamam `DomainEvents.dispatchEventsForAggregate(id)` após persistir (`create`/`save`) — só aí os handlers são efetivamente executados.
- **Subscribers** (padrão Observer) em `src/domain/notification/application/subscribers/` — o contexto `notification` escuta eventos do contexto `forum` (ex.: `OnAnswerCreated`) e reage chamando o próprio use case de notificação. Não há broker externo, é tudo em processo.

## 4. Camada de aplicação (`application`)

- Um arquivo por use case em `application/useCases/`, nome kebab-case, classe `XxxUseCase` com `@Injectable()`.
- Injeta apenas **interfaces (ports)** de repositório/serviço no construtor — nunca implementações concretas.
- Retorno via monad **`Either<Error, Success>`** (`src/core/Either.ts`, classes `Left`/`Right` com `isLeft()/isRight()`), evitando lançar exceções para erros esperados de negócio:

```ts
type XxxUseCaseResponse = Either<ResourceNotFoundError | NotAllowedError, { question: Question }>

async execute(...): Promise<XxxUseCaseResponse> {
  if (!question) return left(new ResourceNotFoundError())
  return right({ question })
}
```

- Erros específicos de use case em `useCases/errors/` implementando `UseCaseError` (`{ message: string }`); erros genéricos reutilizáveis (`ResourceNotFoundError`, `NotAllowedError`) em `src/core/errors/errors/`.
- Interfaces de repositório são **abstract classes** (não `interface` do TS), para poderem ser usadas como token de injeção de dependência do Nest:

```ts
export abstract class QuestionsRepository {
  abstract create(question: Question): Promise<Question>
  abstract save(question: Question): Promise<void>
  abstract findBySlug(slug: string): Promise<Question | null>
  abstract findManyRecents(params: PaginationParams): Promise<Question[]>
}
```

- Ports adicionais nesse padrão: `cryptography/{hash-generator,hash-comparator,encrypter}.ts`, `storage/uploader.ts`.

## 5. Camada de infraestrutura — Prisma

- **`PrismaService`**: estende `PrismaClient`, implementa `OnModuleInit`/`OnModuleDestroy` (`$connect`/`$disconnect`).
- **Repositórios Prisma** (`infra/database/prisma/repositories/`): implementam os ports do domínio, injetam `PrismaService`, convertem entrada/saída via mapper.
- **Mappers** (`infra/database/prisma/mappers/`): classe estática por entidade com `toDomain(raw): Entity` e `toPrisma(entity): Prisma.CreateInput`, convertendo `string` ↔ `UniqueEntityId` e primitivos ↔ value objects.
- Coleções (ex.: anexos) usam o diff do `WatchedList` (`getNewItems()`/`getRemovedItems()`) dentro do repositório para gerar `createMany`/`deleteMany` corretos.
- **`DatabaseModule`**: registra `PrismaService` + cada repositório via `{ provide: AbstractRepository, useClass: PrismaRepository }`, exportando os tokens abstratos para outros módulos consumirem sem conhecer Prisma.

## 6. Camada HTTP

- Controllers em `infra/http/controllers/`, um por use case, nomeados pelo verbo/rota (ex.: `create-question.controller.ts`).
- **Validação com Zod** (sem `class-validator`/`class-transformer`): schema `z.object({...})` no escopo do módulo, aplicado via `ZodValidationPipe` custom (`infra/http/pipes/zod-validation-pipe.ts`), erros formatados com `zod-validation-error`.
- Controller injeta a **classe concreta** do use case (não uma interface — use cases são providers diretos), chama `.execute()`, verifica `result.isLeft()` e lança `HttpException` apropriada.
- **Presenters** (`infra/http/presenters/`): `toHTTP(entity)` estático, converte entidade de domínio em JSON de resposta.
- **Autenticação**: access token JWT RS256 (15 min) + refresh token opaco rotativo (7 dias) via Passport (`infra/auth/`) — detalhe completo de emissão/renovação em [F1-E10](docs/features/F1-E10-identidade-acesso/spec-identidade-acesso.md):
  - `JwtStrategy` valida o access token com schema Zod, chave pública lida do env (base64). Payload: `{ sub, groupId, schoolId }` (ver §11 — uma sessão vale para uma escola).
  - `JwtAuthGuard` registrado globalmente (`APP_GUARD`) — toda rota exige autenticação por padrão, exceto as marcadas com decorator `@Public()`.
  - `@CurrentUser()` decorator extrai o payload do usuário autenticado (`{ sub, groupId, schoolId }`).

## 7. Organização de módulos NestJS

- `AppModule`: composition root — importa `ConfigModule.forRoot({ validate })`, `AuthModule`, `HttpModule`, `EnvModule`. Não declara controllers/providers próprios.
- `HttpModule`: importa `DatabaseModule`, `CryptographyModule`, `StorageModule`; declara todos os controllers e todos os use cases como providers.
- `DatabaseModule`, `CryptographyModule`, `StorageModule`: seguem o padrão `{ provide: AbstractPort, useClass: ConcreteAdapter }`, exportando os tokens abstratos.
- `EnvModule`/`EnvService`: wrapper tipado sobre `ConfigService` do `@nestjs/config`.

## 8. Testes

- **Unitários** (`*.spec.ts` ao lado do use case): usam **repositórios in-memory** (`test/repositories/`) que implementam os mesmos ports do domínio, guardando entidades em array e replicando paginação/filtros/diff de anexos.
- **Factories** (`test/factories/make-*.ts`): função `makeX(override?, id?)` com dados padrão via `@faker-js/faker`; versão `@Injectable() XFactory` com `makePrismaX()` persiste no banco real via `PrismaService` — usada em testes e2e.
- **Fakes de ports**: `test/cryptography/fake-hasher.ts`, `fake-encrypter.ts`, `test/storage/faker-uploader.ts`.
- **E2E** (`*.controller.e2e-spec.ts` ao lado do controller): `Test.createTestingModule({ imports: [AppModule, DatabaseModule] })` + `supertest`. `test/setup-e2e.ts` cria um schema Postgres único por execução (`randomUUID()`), roda `prisma migrate deploy` e derruba o schema no `afterAll`.
- Dois configs do Vitest: `vitest.config.ts` (unitário) e `vitest.config.e2e.ts` (e2e, com `setupFiles`).

## 9. Principais dependências

| Categoria | Bibliotecas |
|---|---|
| Framework | `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express`, `@nestjs/config` |
| Auth | `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt` (RS256) |
| Persistência | `@prisma/client`, `prisma` (Postgres) |
| Validação | `zod`, `zod-validation-error` |
| Hash | `bcryptjs` |
| Storage | `@aws-sdk/client-s3`, `multer` (Cloudflare R2, API compatível com S3) |
| Segurança de borda | `helmet`, `@nestjs/throttler` (ver §12) |
| E-mail | `nodemailer` (ver [F1-E10](docs/features/F1-E10-identidade-acesso/spec-identidade-acesso.md) — módulo usado pela recuperação de senha, atrás de flag) |
| Testes | `vitest`, `@nestjs/testing`, `supertest`, `@faker-js/faker` |

## 10. Configuração de ambiente

- Schema Zod em `infra/env/env.ts` valida `DATABASE_URL`, `JWT_PRIVATE_KEY`/`JWT_PUBLIC_KEY` (base64, decodificadas com `Buffer.from(key, 'base64')`), `PORT` e variáveis do R2/S3.
- Validação plugada no `ConfigModule.forRoot({ validate: (env) => envSchema.parse(env), isGlobal: true })` — a aplicação falha ao subir se o `.env` estiver inválido.
- `EnvService` expõe acesso tipado (`.get('CHAVE')`) em vez de `process.env` direto.

---

## 11. Multi-tenancy (fundacional desde o dia zero)

O DICEDUC é **multi-tenant desde o início** — não um retrofit de fase posterior. O tenant é o
**`Group`** (grupo): uma organização que pode ser uma secretaria, um município, uma rede ou uma
escola isolada (o campo `type` é só um rótulo informativo, não muda o comportamento). Um `Group`
tem uma ou mais `School` (escolas). Bounded context `tenancy` — ver
[`F1-E0A`](docs/features/F1-E0A-tenancy/spec-tenancy.md) para o detalhe de implementação.

### Três dimensões, três papéis diferentes

- **`groupId` — isolamento (boundary de segurança).** Todo model de negócio carrega `groupId`.
  Nenhum usuário de um `Group` pode ler ou escrever dado de outro `Group`, sem exceção.
- **`schoolId` — a escola da sessão atual.** **Uma sessão sempre pertence a exatamente uma
  escola** (decisão de produto: quem atua em várias escolas do mesmo group — professor, diretor de
  rede — faz login separado em cada uma; não existe visão "todas as escolas de uma vez" no MVP —
  isso é a `F2-E07`, Multiunidade). Nas tabelas operacionais (`Student`, `Teacher`, `ClassGroup`,
  `Invoice`, etc.), `schoolId` é sempre o da sessão corrente, nunca uma lista. Um `UserRole` sem
  `schoolId` (*group-wide*, ex.: mantenedora/admin do Group) significa que o usuário **pode logar
  em qualquer escola do Group**, não que ele vê todas de uma vez — modelado em `F1-E09`.
- **`allowedStudentIds` — restrição por aluno (IDOR).** Para papéis `GUARDIAN`/`STUDENT`, a
  sessão também carrega quais alunos aquele usuário pode enxergar — sem isso, nada impede um
  responsável de trocar o `studentId` na URL e ler o boletim de outro aluno da mesma escola.
  Resolvido em `F1-E09`.

### `GroupContext` (`AsyncLocalStorage`)

Contexto de request populado a partir do usuário autenticado, nunca a partir do corpo/query da
requisição:

```ts
type GroupContext = {
  groupId: string
  schoolId: string
  allowedStudentIds: string[] | null // null = sem restrição (equipe da escola); [] = nenhum aluno
}
```

- `GroupScopeGuard` (`infra/auth/`) roda **depois** do `JwtAuthGuard`: lê `groupId`/`schoolId` do
  payload do access token, resolve `allowedStudentIds` (via `UserRolesRepository`/vínculo
  aluno↔responsável, `F1-E09`) e popula o ALS para a duração da requisição.
- Rotas públicas que ainda não têm sessão (branding do tenant, login, refresh, onboarding) rodam
  **fora** deste guard, algumas delas dentro de um `GroupContext` **preliminar** só com `groupId`
  (resolvido a partir do slug da URL, ver `Group.slug`/`School.slug` abaixo) — o suficiente para
  localizar o usuário/grupo sem expor nada de outro tenant.
- Onboarding de `Group`/`School` (criação da primeira escola de um cliente) roda atrás de
  `PlatformAdminGuard` (chave de administrador da plataforma, `F1-E0A`) — não é mais um "guard de
  signup" a definir depois, é a mesma trava usada desde o início.

### Enforcement via Prisma Client Extension

Uma única `$extends` (`infra/database/prisma/extensions/tenant-scope.extension.ts`) intercepta
toda query dos models de negócio:

- **Leitura:** injeta `where: { groupId }`; se o model tem `schoolId`, injeta também
  `where: { schoolId }` (o da sessão, sempre um valor único — não uma lista). Se
  `allowedStudentIds !== null`: filtra `Student` por `id` e qualquer outro model por `studentId`.
- **Escrita (`create`/`createMany`):** injeta `data: { groupId }` e, quando o model tem a coluna,
  `data: { schoolId }` automaticamente — os dois vêm sempre do `GroupContext`, nunca do
  corpo/param da requisição. Isso elimina a necessidade de um helper tipo "assert escola no
  escopo": como só existe uma escola por sessão, não há lista contra a qual validar.

Isso garante que nenhum repositório "esqueça" de filtrar — o isolamento e o escopo são uma
propriedade da camada de infra, não uma disciplina que cada repositório precisa lembrar de aplicar.

### Personalização por tenant (slugs)

`Group.slug` (único global) e `School.slug` (único por group) identificam o tenant na própria URL
do `diceduc-web`: `app/[group]/[school]/…` (ver `arquitetura-frontend.md` §2). Uma rota pública
(`GET /public/branding/:groupSlug/:schoolSlug`, `F1-E0A`) devolve nome/logo/cor para a tela de
login montar a identidade visual antes de qualquer autenticação.

### Convenções derivadas

- **Prisma schema:** todo model de negócio tem `groupId String` (+ índice); models operacionais
  também têm `schoolId String` (+ índice). `Group` e `School` (em `tenancy`) não têm essas colunas
  (são elas próprias a raiz da hierarquia).
- **Domínio (`enterprise`):** entidades recebem `groupId` (e `schoolId`, quando aplicável) como
  props normais, setadas a partir do `GroupContext` no `execute()` do use case — nunca hardcoded,
  nunca aceitas cruas de um DTO de entrada não confiável. Use cases de escrita **não** recebem
  `schoolId` como parâmetro de entrada (era assim antes de `F1-E10`; o `GroupContext` já resolve).
- **Unicidade:** índices únicos que hoje seriam globais (CPF, `RoleName`) passam a ser compostos
  com `groupId` (único **por group**, não globalmente). Índices que fazem sentido por escola (ex.:
  `SchoolYear.year`) são compostos com `schoolId`. `User.email` (`F1-E10`) é único **por group** —
  um mesmo e-mail pode ter contas independentes em groups diferentes (ex.: um professor que
  leciona em duas secretarias distintas). `Group.slug` é a única unicidade **global** do sistema.
- **Testes:** repositórios in-memory replicam o filtro de `groupId`/`schoolId`/`allowedStudentIds`
  manualmente (sem a Prisma extension) para os unit specs continuarem provando isolamento/escopo
  sem banco.
- **Sessão:** o access token carrega `groupId` e `schoolId`; `allowedStudentIds` é resolvido via
  banco a cada requisição (`GroupScopeGuard`), nunca embutido no token, para revogação de acesso
  ser imediata. Detalhe completo de emissão/renovação de sessão em `F1-E10`.

## 12. Segurança de aplicação (baseline)

Baseline aplicado desde `F1-E00` (bootstrap) e válido para todos os épicos seguintes — não é uma
ficha própria, é uma convenção transversal, como o multi-tenancy do §11.

- **Headers e superfície de borda:** `helmet()` aplicado no `main.ts`. Sem `app.enableCors()` — o
  `diceduc-web` nunca é chamado direto do browser (padrão BFF, ver `arquitetura-frontend.md` §6), só
  o servidor do Next chama a API, então CORS de browser não se aplica.
- **Rate limiting:** `@nestjs/throttler` global — 100 req/min por IP nas rotas autenticadas, 5/min
  nas rotas públicas sensíveis (`POST /sessions`, `/sessions/refresh`, `/password-resets`).
  `ponytail:` armazenamento em memória do throttler; trocar por um storage compartilhado (Redis)
  se a API rodar em mais de uma instância.
- **`trust proxy`** ligado no Nest — necessário para o throttler e para logs enxergarem o IP real
  atrás de um proxy/load balancer.
- **Erros:** nunca repassar a mensagem de exceção interna ou do Prisma na resposta HTTP — o filtro
  de exceção padrão do Nest já esconde o stack em produção; não criar um filtro global próprio só
  para isso.
- **Logs:** nunca logar senha, token (access/refresh), CPF ou o corpo bruto da requisição em rotas
  de autenticação/dados pessoais.
- **Senha:** hash com `bcryptjs`, custo 12. Política de tamanho (8–72 caracteres), sem exigir
  regra de composição (maiúscula/número/símbolo) — critério do NIST 800-63B, que reduz atrito sem
  perder segurança real.
- **Segredos:** só via variável de ambiente (chave JWT, `PLATFORM_ADMIN_KEY`, credenciais de
  banco/SMTP), nunca hardcoded ou versionado. `ponytail:` rotação de chave JWT ainda é manual (gerar
  par novo, atualizar env, reiniciar); migrar para múltiplas chaves ativas com `kid` no header do
  JWT se for preciso rotacionar sem derrubar sessões em andamento.
- **Dependências:** `pnpm audit --prod --audit-level=high` rodado no CI (ver `F1-E00`) — falha o
  build em vulnerabilidade `high`/`critical` conhecida em dependência de produção.
- **Dados em trânsito e em repouso (produção):** `DATABASE_URL` com `sslmode=require` (TLS na
  conexão com o Postgres); criptografia em repouso do disco/backup confirmada como ativa no
  provedor gerenciado escolhido (GCP/AWS/DigitalOcean) — não é criptografia de coluna (isso
  quebraria os índices únicos por `groupId`, como o de CPF). Backup com restauração testada
  **fica fora do MVP**, registrado como pendência da Fase 2 em `ROADMAP.md`.

## O que aproveitar no `diceduc-controle-escolar`

- Separar `domain` (regras de negócio puras) de `infra` (Nest/Prisma/HTTP) desde o início evita acoplamento e facilita testes unitários rápidos (sem banco).
- Usar `Either` para erros de negócio esperados (aluno já matriculado, turma lotada, etc.) e reservar exceções para erros realmente excepcionais.
- Repositórios como abstract class = um único padrão de DI, fácil trocar implementação (ex.: in-memory nos testes, Prisma em produção).
- Zod para validação de entrada HTTP e de env evita duplicar regras de validação com DTOs de classe.
- Eventos de domínio + subscribers são úteis se houver múltiplos módulos que precisam reagir a ações (ex.: matrícula concluída → gerar boleto, enviar notificação) sem acoplar os módulos diretamente.
