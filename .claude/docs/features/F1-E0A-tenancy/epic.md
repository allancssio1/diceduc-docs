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
| Rastreabilidade | Não existe no BACKLOG original — decisão de arquitetura tomada durante a especificação da Fase 1: o DICEDUC é multi-tenant desde o primeiro momento, não um retrofit de fase posterior. |

## Objetivo

Estabelecer a fundação multi-tenant do DICEDUC: a entidade `Group` (o tenant — uma secretaria,
município, rede ou escola isolada) e a entidade `School` (escola, sempre vinculada a um `Group`),
mais o mecanismo de isolamento/escopo (`GroupContext`, guard, Prisma extension) que **todos** os
épicos de domínio (F1-E01 em diante) vão usar. Esta ficha roda **antes** de `F1-E01` — nenhum
contexto de domínio deveria ser implementado sem essa base pronta.

## Histórias cobertas

Não vem do BACKLOG (é infraestrutura de arquitetura, não uma história de usuário final), mas
resolve a necessidade de produto de "múltiplas secretarias/municípios podem se cadastrar, cada uma
com várias escolas vinculadas, e cada escola tem seus próprios diretores, professores, alunos etc."

## Escopo / fora de escopo

- **Dentro:** entidades `Group`/`School`; onboarding (registro de group + escolas); `GroupContext`
  (isolamento por `groupId` + escopo por `schoolId`); guard de resolução do contexto; Prisma
  extension de auto-scoping; convenção de unicidade por tenant.
- **Fora:** perfis/papéis em si (`Role`/`UserRole` são de `F1-E09` — esta ficha só define que
  `UserRole` carrega `schoolId?` para decidir o escopo); dashboards consolidados por rede/comparação
  entre escolas (isso é `F2-E07`, que passa a ser só uma camada analítica sobre a base já
  multi-tenant, sem introduzir nenhuma parte do isolamento).

## Dependências e integrações

Depende só de `F1-E00` (bootstrap). É pré-requisito de **todo** épico de domínio da Fase 1 — cada
um deles carrega `groupId`/`schoolId` em suas entidades desde a primeira versão, sem exceção.

## Decisões em aberto / riscos

Nenhuma no momento — já especificado (ver [`spec-tenancy.md`](spec-tenancy.md)).

## Próximo passo

Já especificado em [`spec-tenancy.md`](spec-tenancy.md). Ver status no [índice](../README.md).
