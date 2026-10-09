# CLAUDE.md — Router do DICEDUC Controle Escolar

> Este arquivo é o **ponto de entrada mínimo** para qualquer agente que for implementar código neste projeto. Leia-o primeiro, sempre.

## Regra de ouro (protocolo de leitura)

Para **implementar uma tarefa**, leia **apenas**:

1. Este arquivo (`CLAUDE.md`).
2. A arquitetura relevante:
   - Backend/API → [.claude/arquitetura-ignite.md](.claude/arquitetura-ignite.md)
   - Frontend/web → [.claude/arquitetura-frontend.md](.claude/arquitetura-frontend.md) **e** [.claude/design-system.md](.claude/design-system.md) (tokens, tipografia, tema, branding por tenant, componentes — leitura obrigatória junto, não opcional)
3. A pasta do épico: `.claude/docs/features/<ID>-<slug>/`. Cada pasta tem um `epic.md` (planejamento: objetivo, histórias, dependências — não é para implementar direto) e, quando já especificado, uma ou mais `spec-*.md` com o nível de detalhe técnico para codar (entidades, use cases, rotas, arquivos). Ex.: `.claude/docs/features/F1-E01-cadastros-sis/epic.md` já contém esse nível de detalhe (épico pequeno, não foi quebrado em specs separadas). Ver status de cada épico em [`.claude/docs/features/README.md`](.claude/docs/features/README.md): só implemente um épico 🔵 especificado — se estiver ⚪ (só planejamento), pare e peça para especificá-lo primeiro.

**NÃO leia `BACKLOG.md`, `ROADMAP.md` ou o documento-fonte de pesquisa para implementar.** Esses arquivos existem para quem está planejando o produto, não para quem está codando — a ficha da tarefa já destila tudo que é necessário (histórias, critérios de aceite, arquivos a tocar, contratos). Se a ficha referenciar algo do backlog, é só um link de rastreabilidade, não leitura obrigatória.

Se a pasta do épico não existir ainda, ou existir só como ⚪ (sem spec), **pare e peça para especificá-la** em vez de improvisar a partir do backlog — assim ela também fica registrada para a próxima vez.

**Documentação viva (as-built) fica nos repositórios, não aqui.** Este hub guarda o que **será** implementado (fichas, specs, percurso). O que **já foi** implementado fica em cada repo: `AGENTS.md` → `.claude/arquitetura.md` (core da arquitetura) e `.claude/dominios/<contexto>.md` (um por domínio: modelo, regras de negócio, decisões com o porquê). Ao concluir uma ficha que altere arquitetura, regras ou decisões de um domínio, atualize esses arquivos no repo tocado — e só o que existe no código.

## O que é este projeto

DICEDUC é um Sistema de Gestão Escolar (SGE) brasileiro. O escopo completo (o "o quê" e o "por quê") está em `.claude/ROADMAP.md`/`.claude/BACKLOG.md`; este arquivo e as fichas cobrem o "como implementar".

## Stack e regra de dependência

- **Backend:** NestJS + Prisma + TypeScript, **Clean Architecture + DDD leve**. Regra de dependência (sempre para dentro):

  ```
  infra  →  domain/application  →  domain/enterprise
                      ↓
                    core
  ```

- **Frontend:** Next.js 16.3.
- **Três repositórios git independentes, lado a lado** dentro da pasta de trabalho `diceduc-controle-escolar/` (que não é repo git, só agrupa): `diceduc-docs` (este — hub de documentação/planejamento, sem código de aplicação, só `.claude/` e este `CLAUDE.md`), `diceduc-api` (backend) e `diceduc-web` (frontend). `diceduc-api`/`diceduc-web` são criados pela ficha `.claude/docs/features/F1-E00-bootstrap/epic.md` **como irmãos de `diceduc-docs`, nunca dentro dele**. Sem ferramenta de monorepo — cada repo resolve suas próprias dependências.

## Mapa de pastas

Layout em disco (workspace):

```
diceduc-controle-escolar/      ← pasta de trabalho (sem git próprio)
  diceduc-docs/                ← repo git: docs e planejamento (este)
  diceduc-api/                 ← repo git: NestJS + Prisma
  diceduc-web/                 ← repo git: Next.js 16.3
```

Caminhos que começam com `diceduc-api/` ou `diceduc-web/` nas fichas são relativos à raiz do workspace (`diceduc-controle-escolar/`) — ou seja, `../diceduc-api/...` visto de dentro do `diceduc-docs`. Nunca crie esses diretórios dentro do `diceduc-docs`.

Repositório `diceduc-api`:

```
AGENTS.md                  entrada da documentação viva (aponta para .claude/arquitetura.md)
CLAUDE.md                  @AGENTS.md
.claude/
  arquitetura.md           core da arquitetura como implementada
  dominios/<contexto>.md   1 por domínio já implementado
src/
  core/                    building blocks compartilhados (Entity, Either, DomainEvents, erros genéricos)
  domain/
    <bounded-context>/
      application/         use cases + ports (repositories/services como abstract class)
      enterprise/          entidades, value objects, eventos de domínio — zero dependência de framework
  infra/
    database/prisma/       PrismaService, repositórios Prisma, mappers
    http/                  controllers, pipes, presenters
    auth/                  JWT strategy, guards, decorators
    env/                   validação Zod de variáveis de ambiente
test/
  repositories/            repositórios in-memory (testes unitários)
  factories/                factories de entidades
prisma/
  schema.prisma
```

Repositório `diceduc-web`:

```
AGENTS.md                  entrada da documentação viva (aponta para .claude/arquitetura.md)
CLAUDE.md                  @AGENTS.md
.claude/
  arquitetura.md           core da arquitetura como implementada
  dominios/<contexto>.md   1 por domínio já implementado
...                        ver .claude/arquitetura-frontend.md
```

Nas fichas de feature, caminhos de rota escritos como `app/(portal)/...`/`app/(public)/...` são relativos a `app/[grupo]/[escola]/` — toda navegação do `diceduc-web` fica sob o tenant identificado na URL (ver [.claude/arquitetura-frontend.md §2](.claude/arquitetura-frontend.md#2-estrutura-de-pastas)).

Este repositório (`diceduc-docs`):

```
CLAUDE.md
.claude/
  docs/
    features/
      <ID>-<slug>/             1 pasta por épico
        epic.md                planejamento (objetivo, histórias, dependências)
        spec-*.md               nível de implementação (quando já especificado) — comece por aqui
```

## Bounded contexts do DICEDUC (backend)

Cada contexto vive em `diceduc-api/src/domain/<contexto>/`. Mapeamento épico → contexto (ver `.claude/docs/features/README.md` para a lista completa e status):

| Contexto | Épicos | Conteúdo |
| --- | --- | --- |
| `tenancy` | F1-E0A | Group (tenant) e School — fundação multi-tenant, base de todos os outros contextos |
| `people` | F1-E01 | Alunos, responsáveis, professores, funcionários |
| `academic` | F1-E01, F1-E02, F1-E05 | Séries, turmas, disciplinas, salas, calendário, matrícula, horários |
| `attendance` | F1-E03 | Frequência |
| `assessment` | F1-E04 | Notas, boletins |
| `finance` | F1-E06 | Mensalidades, cobranças, inadimplência |
| `reporting` | F1-E07 | Relatórios administrativos e oficiais |
| `iam` | F1-E10, F1-E09 | Contas de acesso/sessão (F1-E10); perfis de acesso, auditoria, consentimento e direitos do titular LGPD (F1-E09) |
| `communication` | F2-E02 | Comunicação multicanal e notificações |

Novos contextos são adicionados conforme novas fichas forem criadas nas fases seguintes.

## Cheatsheet de convenções do backend

(Resumo — a fonte de verdade é [.claude/arquitetura-ignite.md](.claude/arquitetura-ignite.md), leia-a antes de implementar.)

- **Multi-tenancy é fundacional, não opcional:** todo model de negócio carrega `groupId`; models operacionais também carregam `schoolId`. Toda query passa pelo `GroupContext`/Prisma extension — nunca aceitar `groupId`/`schoolId` vindos do corpo/param da requisição como fonte de autoridade. Uma sessão é sempre de **uma única escola** (sem `allowedSchoolIds`); rota pública sem sessão (login, branding, onboarding) só existe com `@Public()` + `@Throttle`, e onboarding de `Group`/`School` exige `PlatformAdminGuard`. Ver [.claude/arquitetura-ignite.md §11](.claude/arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [F1-E0A](.claude/docs/features/F1-E0A-tenancy/spec-tenancy.md).
- **Dado de aluno é restrito por vínculo, não só por escola:** o `GroupContext` também carrega `allowedStudentIds` (responsável/aluno só enxergam quem tem vínculo — ver [F1-E09](.claude/docs/features/F1-E09-seguranca-lgpd/spec-seguranca-lgpd.md)). Nunca logar senha, token ou CPF — ver [.claude/arquitetura-ignite.md §12](.claude/arquitetura-ignite.md#12-segurança-de-aplicação-baseline).
- Erros de negócio esperados → `Either<Error, Success>` (`left`/`right`), nunca `throw`.
- Repositórios são `abstract class` em `application/repositories/` (não `interface`) — servem de token de DI.
- 1 arquivo por use case em `application/useCases/`, nome kebab-case, classe `XxxUseCase`.
- Validação de entrada HTTP e de env → Zod (nunca `class-validator`).
- Controllers injetam a classe concreta do use case, chamam `.execute()`, tratam `isLeft()/isRight()`.
- Persistência: mapper estático por entidade (`toDomain`/`toPrisma`) em `infra/database/prisma/mappers/`.
- Testes unitários usam repositórios in-memory (`test/repositories/`); e2e usa Prisma real com schema isolado por execução.

## Comandos

> Ficam vazios até `.claude/docs/features/F1-E00-bootstrap/epic.md` ser executado — depois desta ficha, atualize esta seção com os comandos reais (dev, test unit, test e2e, migrate).

## Índice de referência (produto, não implementação)

- [.claude/ROADMAP.md](.claude/ROADMAP.md) — fases, épicos, sequenciamento
- [.claude/BACKLOG.md](.claude/BACKLOG.md) — histórias de usuário e critérios de aceite por épico
- [.claude/arquitetura-ignite.md](.claude/arquitetura-ignite.md) — arquitetura de referência do backend
- [.claude/arquitetura-frontend.md](.claude/arquitetura-frontend.md) — arquitetura de referência do frontend
- [.claude/design-system.md](.claude/design-system.md) — tokens, tipografia, tema e componentes do frontend
- [.claude/docs/features/README.md](.claude/docs/features/README.md) — índice de fichas de tarefa e status
