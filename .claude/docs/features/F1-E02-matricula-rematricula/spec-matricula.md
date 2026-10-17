# F1-E02 — Matrícula, rematrícula e turmas

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E02 — Matrícula, rematrícula e turmas |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | `academic` |
| Depende de | F1-E01 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E02` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Matricular e rematricular alunos em turmas respeitando limite de vagas, e calcular a situação final (aprovado/reprovado/recuperação) ao fim do ano letivo com base em regras de promoção configuráveis por série.

## Histórias cobertas

- `F1-E02-U01` — Matricular aluno em turma respeitando limite de vagas (com pré-matrícula/reserva).
- `F1-E02-U02` — Processar rematrícula reaproveitando dados existentes e sinalizando pendência financeira.
- `F1-E02-U03` — Configurar regras de promoção/reprovação por série e calcular situação final do aluno.

## Decisões resolvidas nesta spec

- **Bloqueio por inadimplência na rematrícula:** implementado como configuração única do sistema (`RenewalPolicy`, singleton, sem dimensão por escola/unidade — isso é assunto de `F2-E07`), com um campo booleano `blockOnDebt`. Quando `true`, `RenewEnrollmentUseCase` retorna erro se houver pendência; quando `false`, apenas sinaliza (`hasFinancialPendency: true` na resposta) sem bloquear.
- **Cálculo de situação final antes de F1-E03/F1-E04 existirem:** define-se aqui o port `AttendanceStatsProvider` e `GradesStatsProvider` (`abstract class`, em `academic/application/services/`). Nesta ficha, cadastrar uma implementação stub (`StubAttendanceStatsProvider`, `StubGradesStatsProvider`) que retorna sempre "dados insuficientes" (`{ available: false }`), fazendo `CalculateFinalResultUseCase` responder com `finalResult = 'PENDING'` até que `F1-E03`/`F1-E04` implementem os providers reais (substituindo o binding no módulo, sem alterar o use case).
- **Vagas de turma:** o port `ClassGroupsRepository.hasEnrolledStudents` e `countEnrolledStudents`, deixados como stub em `F1-E01`, são implementados de verdade aqui, sobre a tabela `Enrollment`.

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md). Específico deste épico:

- `Enrollment`, `PromotionRule` carregam `groupId`; `Enrollment` também carrega `schoolId` (herdado da `ClassGroup` de destino).
- `RenewalPolicy`, descrita antes como "singleton" (1 linha), passa a ser **1 linha por group** (`groupId` como chave), não mais um singleton global do sistema — cada group configura sua própria política de bloqueio por dívida.
- `EnrollStudentUseCase`/`RenewEnrollmentUseCase` validam que `classGroupId` pertence ao mesmo `groupId`/`schoolId` do `studentId` antes de matricular.

## Modelo de domínio (`enterprise`)

### Contexto `academic` — `diceduc-api/src/domain/academic/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `Enrollment` | `AggregateRoot` | `enrollment.ts` | `studentId`, `classGroupId`, `schoolYearId`, `status: EnrollmentStatus`, `finalResult: FinalResult`, `enrolledAt`, `updatedAt` |
| `PromotionRule` | `Entity` | `promotion-rule.ts` | `gradeLevelId`, `minGrade`, `minAttendancePercentage` |

Value objects — `entities/valueObjects/`:
- `enrollment-status.ts` — `EnrollmentStatus = 'PRE_ENROLLED' | 'CONFIRMED' | 'CANCELLED' | 'TRANSFERRED'`.
- `final-result.ts` — `FinalResult = 'PENDING' | 'APPROVED' | 'FAILED' | 'RECOVERY'`.

Invariantes:
- `Enrollment` nasce com `status = 'PRE_ENROLLED'` se `preEnroll: true` na entrada, senão `'CONFIRMED'`.
- Uma `ClassGroup` não aceita nova `Enrollment` com `status` ativo (`PRE_ENROLLED`/`CONFIRMED`) se `countEnrolledStudents(classGroupId) >= classGroup.capacity`.
- `finalResult` só pode ser calculado (`!= 'PENDING'`) quando `AttendanceStatsProvider`/`GradesStatsProvider` retornam `available: true`.

## Use cases (`application`)

### `diceduc-api/src/domain/academic/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `EnrollStudentUseCase` | `enroll-student.ts` | `studentId`, `classGroupId`, `schoolYearId`, `preEnroll?: boolean` | `Either<ResourceNotFoundError \| ClassGroupFullError, { enrollment: Enrollment }>` | `class-group-full-error.ts` |
| `RenewEnrollmentUseCase` | `renew-enrollment.ts` | `previousEnrollmentId`, `targetClassGroupId` | `Either<ResourceNotFoundError \| ClassGroupFullError \| StudentHasDebtError, { enrollment: Enrollment; hasOutstandingBalance: boolean }>` | `student-has-debt-error.ts` (só lançado se `RenewalPolicy.blockOnDebt = true`) |
| `ConfigurePromotionRuleUseCase` | `configure-promotion-rule.ts` | `gradeLevelId`, `minGrade`, `minAttendancePercentage` | `Either<ResourceNotFoundError, { promotionRule: PromotionRule }>` | — |
| `CalculateFinalResultUseCase` | `calculate-final-result.ts` | `enrollmentId` | `Either<ResourceNotFoundError, { finalResult: FinalResult }>` | — |
| `ConfigureRenewalPolicyUseCase` | `configure-renewal-policy.ts` | `blockOnDebt: boolean` | `Either<never, { renewalPolicy: RenewalPolicy }>` | — |

Ports (`application/repositories/`): `EnrollmentsRepository`, `PromotionRulesRepository`, `RenewalPolicyRepository` (singleton — `get()`/`save()`).
Ports de serviço (`application/services/`): `FinanceStatusProvider` (`abstract hasOpenDebt(studentId): Promise<boolean>` — implementado futuramente por `F1-E06`; nesta ficha, cadastrar `StubFinanceStatusProvider` retornando sempre `false`), `AttendanceStatsProvider`, `GradesStatsProvider` (assinaturas descritas na seção "Decisões resolvidas").

Regras de negócio principais:
- `EnrollStudentUseCase`: verifica capacidade da turma via `ClassGroupsRepository.countEnrolledStudents`; bloqueia se lotada; permite `preEnroll` para reservar vaga sem confirmar.
- `RenewEnrollmentUseCase`: busca `Student`/`Guardian`s já cadastrados (não recadastra); usa `FinanceStatusProvider.hasOpenDebt`; sugere `targetClassGroupId` da série seguinte se `finalResult = 'APPROVED'` na matrícula anterior, ou da mesma série se `'FAILED'` (validação de que a sugestão bate cabe à camada de apresentação/frontend, o use case apenas aceita o `targetClassGroupId` informado).
- `CalculateFinalResultUseCase`: busca `PromotionRule` da série da turma; consulta os dois providers; se ambos `available: true`, aplica `grade >= minGrade && attendance >= minAttendancePercentage` → `APPROVED`, caso contrário `FAILED` ou `RECOVERY` (critério de recuperação fica a cargo de `F1-E04`, aqui basta o binary aprovado/reprovado); resultado é sempre revisável (`UpdateFinalResultUseCase` não existe nesta ficha — ajuste manual é feito reexecutando `ConfigurePromotionRuleUseCase` + recálculo, ou diretamente via `EnrollmentsRepository.save` chamado por um endpoint de ajuste simples, ver HTTP).

## Persistência (`infra/database/prisma`)

- `diceduc-api/prisma/schema.prisma`: novos models `Enrollment`, `PromotionRule`, `RenewalPolicy` (tabela de 1 linha); enums `EnrollmentStatus`, `FinalResult`.
- Repositórios: `prisma-enrollments-repository.ts`, `prisma-promotion-rules-repository.ts`, `prisma-renewal-policy-repository.ts` em `infra/database/prisma/repositories/`. `prisma-class-groups-repository.ts` (já existente, de `F1-E01`) ganha a implementação real de `hasEnrolledStudents`/`countEnrolledStudents` sobre `Enrollment`.
- Adapters de serviço: `infra/database/prisma/services/stub-finance-status-provider.ts`, `stub-attendance-stats-provider.ts`, `stub-grades-stats-provider.ts` (implementações provisórias descritas acima).
- Mappers: `enrollment-mapper.ts`, `promotion-rule-mapper.ts` em `infra/database/prisma/mappers/`.

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Presenter |
| --- | --- | --- | --- |
| `POST /enrollments` | `enroll-student.controller.ts` | dados de `EnrollStudentUseCase` | `enrollment-presenter.ts` |
| `POST /enrollments/:enrollmentId/renew` | `renew-enrollment.controller.ts` | `{ targetClassGroupId }` | `enrollment-presenter.ts` |
| `GET /students/:studentId/enrollments` | `get-student-enrollments.controller.ts` | — (param) | `enrollment-presenter.ts` (lista) |
| `POST /promotion-rules` | `configure-promotion-rule.controller.ts` | dados de `ConfigurePromotionRuleUseCase` | `promotion-rule-presenter.ts` |
| `POST /enrollments/:enrollmentId/calculate-final-result` | `calculate-final-result.controller.ts` | — (param) | `enrollment-presenter.ts` |
| `PATCH /enrollments/:enrollmentId/final-result` | `adjust-final-result.controller.ts` | `{ finalResult }` (ajuste manual pela coordenação) | `enrollment-presenter.ts` |
| `PUT /renewal-policy` | `configure-renewal-policy.controller.ts` | `{ blockOnDebt }` | `renewal-policy-presenter.ts` |

Perfis exigidos: `secretaria`/`coordenação` (guard fino vem de `F1-E09`; por ora `JwtAuthGuard` padrão).

## Frontend (`diceduc-web`)

- `app/(portal)/enrollments/new/page.tsx` — formulário de matrícula/pré-matrícula (`F1-E02-U01`).
- `app/(portal)/enrollments/[studentId]/renewal/page.tsx` — fluxo de rematrícula com aviso de pendência financeira (`F1-E02-U02`).
- `app/(portal)/enrollments/promotion-rules/page.tsx` — CRUD de `PromotionRule` por série (`F1-E02-U03`).
- `features/academic/components/EnrollmentForm.tsx`, `RenewalForm.tsx`, `PromotionRuleForm.tsx`, `FinalResultBadge.tsx`.
- `features/academic/api/enrollments.ts` (`enrollStudent`, `renewEnrollment`, `getStudentEnrollments`, `calculateFinalResult`), `features/academic/schemas/enrollment.ts`.

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/academic/enterprise/entities/enrollment.ts`
- [ ] `diceduc-api/src/domain/academic/enterprise/entities/promotion-rule.ts`
- [ ] `diceduc-api/src/domain/academic/enterprise/entities/valueObjects/enrollment-status.ts`
- [ ] `diceduc-api/src/domain/academic/enterprise/entities/valueObjects/final-result.ts`
- [ ] `diceduc-api/src/domain/academic/application/useCases/enroll-student.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/renew-enrollment.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/configure-promotion-rule.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/calculate-final-result.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/configure-renewal-policy.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/repositories/enrollments-repository.ts`
- [ ] `diceduc-api/src/domain/academic/application/repositories/promotion-rules-repository.ts`
- [ ] `diceduc-api/src/domain/academic/application/repositories/renewal-policy-repository.ts`
- [ ] `diceduc-api/src/domain/academic/application/services/finance-status-provider.ts`
- [ ] `diceduc-api/src/domain/academic/application/services/attendance-stats-provider.ts`
- [ ] `diceduc-api/src/domain/academic/application/services/grades-stats-provider.ts`
- [ ] `diceduc-api/prisma/schema.prisma` (editar)
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-enrollments-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-promotion-rules-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-renewal-policy-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-class-groups-repository.ts` (editar — implementar `hasEnrolledStudents`/`countEnrolledStudents`)
- [ ] `diceduc-api/src/infra/database/prisma/services/stub-finance-status-provider.ts`
- [ ] `diceduc-api/src/infra/database/prisma/services/stub-attendance-stats-provider.ts`
- [ ] `diceduc-api/src/infra/database/prisma/services/stub-grades-stats-provider.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/enrollment-mapper.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/promotion-rule-mapper.ts`
- [ ] `diceduc-api/src/infra/http/controllers/*.ts` (7 controllers, ver seção HTTP)
- [ ] `diceduc-api/src/infra/http/presenters/enrollment-presenter.ts`, `promotion-rule-presenter.ts`, `renewal-policy-presenter.ts`
- [ ] `diceduc-web/app/(portal)/enrollments/new/page.tsx`
- [ ] `diceduc-web/app/(portal)/enrollments/[studentId]/renewal/page.tsx`
- [ ] `diceduc-web/app/(portal)/enrollments/promotion-rules/page.tsx`
- [ ] `diceduc-web/features/academic/components/EnrollmentForm.tsx`, `RenewalForm.tsx`, `PromotionRuleForm.tsx`, `FinalResultBadge.tsx`
- [ ] `diceduc-web/features/academic/api/enrollments.ts`
- [ ] `diceduc-web/features/academic/schemas/enrollment.ts`

## Testes

- Repositórios in-memory: `in-memory-enrollments-repository.ts`, `in-memory-promotion-rules-repository.ts`, `in-memory-renewal-policy-repository.ts`.
- Factories: `make-enrollment.ts`, `make-promotion-rule.ts`.
- Unit specs cobrindo: bloqueio por lotação (`EnrollStudentUseCase`), reaproveitamento de dados + sinalização/bloqueio por dívida (`RenewEnrollmentUseCase`, testado com `blockOnDebt` `true` e `false`), cálculo binário aprovado/reprovado com providers stub (`CalculateFinalResultUseCase` deve retornar `PENDING` quando `available: false`).
- E2E: um `.e2e-spec.ts` por controller listado na seção HTTP.

## Definition of Done

- [ ] Critérios de aceite de `F1-E02-U01` a `F1-E02-U03` no BACKLOG satisfeitos.
- [ ] Todos os use cases e controllers implementados com testes unitários e e2e passando.
- [ ] Migração Prisma aplicada sem erro.
- [ ] Fluxo de matrícula, rematrícula com aviso de pendência e configuração de regra de promoção funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E02` para 🟢 quando implementado.
