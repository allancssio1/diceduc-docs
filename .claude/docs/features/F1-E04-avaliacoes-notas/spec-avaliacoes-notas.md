# F1-E04 — Avaliações, notas e boletins

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E04 — Avaliações, notas e boletins |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | `assessment` |
| Depende de | F1-E02, F1-E05 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E04` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Lançar notas/conceitos com pesos e médias configuráveis por segmento, gerar boletim para responsáveis/alunos e histórico escolar consolidado.

## Histórias cobertas

- `F1-E04-U01` — Coordenação configura fórmula de média por segmento (pesos, recuperação).
- `F1-E04-U02` — Professor lança notas de uma atividade para toda a turma de uma vez.
- `F1-E04-U03` — Responsável visualiza boletim do filho por disciplina/período.
- `F1-E04-U04` — Secretaria gera histórico escolar consolidado do aluno.

## Decisão resolvida nesta spec

O épico deixava em aberto se a fórmula de média seria totalmente livre ou fixa por segmento. Resolução: **configurável, mas não um motor de fórmulas genérico** — `GradeFormula` guarda uma lista de pesos por tipo de avaliação (`weights: { type: string; weight: number }[]`) e uma nota mínima de recuperação (`recoveryMinGrade`); a fórmula de cálculo em si é fixa no código (`média ponderada pelos pesos; se média < recoveryMinGrade, calcula média entre a nota da recuperação e a média original`), evitando um DSL de fórmulas que ninguém pediu.

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md). Específico deste épico:

- `GradeFormula` é configurada **por escola** (`schoolId`), não só por segmento — duas escolas do mesmo group podem ter políticas de recuperação diferentes mesmo no mesmo segmento.
- `Assessment`, `Grade` carregam `groupId`/`schoolId` (herdado da `turmaId`).
- `GenerateTranscriptUseCase` (histórico consolidado) permanece restrito ao `groupId` do aluno — um histórico nunca atravessa groups, mesmo que o aluno tenha estudado em escolas diferentes do mesmo group ao longo dos anos (isso é esperado e suportado, já que todas pertencem ao mesmo `groupId`).

## Modelo de domínio (`enterprise`)

### Contexto `assessment` — `diceduc-api/src/domain/assessment/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `GradeFormula` | `Entity` | `grade-formula.ts` | `segment: Segment` (reusa VO de `academic`, `F1-E01`), `scaleType: GradeScaleType`, `weights: { type: string; weight: number }[]`, `recoveryMinGrade: number` |
| `Assessment` | `AggregateRoot` | `assessment.ts` | `turmaId`, `disciplinaId`, `name`, `type` (ex.: `'PROVA' \| 'TRABALHO'`), `maxScore`, `period` (ex.: `'1_BIMESTRE'`), `status: AssessmentStatus` |
| `Grade` | `Entity` | `grade.ts` | `assessmentId`, `studentId`, `value` (numérico) ou `concept` (string), `status: AssessmentStatus` (herda o status da publicação) |

Value objects — `entities/valueObjects/`:
- `grade-scale.ts` — `GradeScaleType = 'NUMERIC' | 'CONCEPT' | 'BOTH'`.
- `assessment-status.ts` — `AssessmentStatus = 'DRAFT' | 'PUBLISHED'`.

Invariantes:
- `Grade.value` deve respeitar `maxScore` do `Assessment` e o `scaleType` do `GradeFormula` da série (numérico dentro de `[0, maxScore]`, conceito dentro de um conjunto fixo `['A','B','C','D']`).
- Boletim (`F1-E04-U03`) só considera `Grade`s cujo `Assessment.status = 'PUBLISHED'`.

## Use cases (`application`)

### `diceduc-api/src/domain/assessment/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `ConfigureGradeFormulaUseCase` | `configure-grade-formula.ts` | `segment`, `scaleType`, `weights`, `recoveryMinGrade` | `Either<InvalidWeightsError, { gradeFormula: GradeFormula }>` | `invalid-weights-error.ts` (soma dos pesos deve ser 100) |
| `CreateAssessmentUseCase` | `create-assessment.ts` | `turmaId`, `disciplinaId`, `name`, `type`, `maxScore`, `period` | `Either<ResourceNotFoundError, { assessment: Assessment }>` | — |
| `LaunchGradesUseCase` | `launch-grades.ts` | `assessmentId`, `grades: { studentId, value }[]`, `teacherId?` | `Either<ResourceNotFoundError \| InvalidGradeValueError \| NotAllowedError, { grades: Grade[] }>` | `invalid-grade-value-error.ts` |
| `PublishAssessmentUseCase` | `publish-assessment.ts` | `assessmentId` | `Either<ResourceNotFoundError, void>` | — |
| `CalculateFinalAverageUseCase` | `calculate-final-average.ts` | `studentId`, `turmaId`, `disciplinaId`, `period` | `Either<ResourceNotFoundError, { average: number; recoveryApplied: boolean }>` | — |
| `GenerateReportCardUseCase` | `generate-report-card.ts` | `studentId`, `period?` | `Either<ResourceNotFoundError, { entries: { disciplina: string; grades: Grade[]; average: number }[] }>` | — |
| `GenerateTranscriptUseCase` | `generate-transcript.ts` | `studentId` | `Either<ResourceNotFoundError, { years: { schoolYear: number; result: FinalResult; averages: Record<string, number> }[] }>` | — |

Ports (`application/repositories/`): `GradeFormulasRepository`, `AssessmentsRepository`, `GradesRepository`.

Regras de negócio principais:
- `LaunchGradesUseCase`: salva sempre com `Assessment.status = 'DRAFT'` até `PublishAssessmentUseCase` ser chamado; permite reexecução (upsert por `assessmentId + studentId`) para lançamentos incrementais/rascunho. Quando `teacherId` é informado (o controller só o passa se o ator autenticado tiver papel `professor`), exige `HorariosRepository.existsActiveFor(teacherId, turmaId, disciplinaId, date)` (`turmaId`/`disciplinaId` do `Assessment`, `date` = hoje; de `F1-E05`) — senão `NotAllowedError`. Mesma lógica de `RecordAttendanceUseCase` (`F1-E03`): impede lançar nota fora da própria disciplina/turma.
- `PublishAssessmentUseCase`: flip irreversível de `DRAFT → PUBLISHED` (sem endpoint de despublicar nesta ficha).
- `CalculateFinalAverageUseCase`: aplica `GradeFormula` do segmento da série da turma; se `média < recoveryMinGrade`, busca `Grade` do tipo `'RECUPERACAO'` (mesmo mecanismo de `Assessment.type`) e recalcula conforme a regra fixa descrita acima.
- `GenerateTranscriptUseCase`: consolida `EnrollmentsRepository` (de `F1-E02`) + médias por ano letivo; usado para documento de transferência.

## Persistência (`infra/database/prisma`)

- `diceduc-api/prisma/schema.prisma`: novos models `GradeFormula`, `Assessment`, `Grade`; enums `GradeScaleType`, `AssessmentStatus`.
- Repositórios: `prisma-grade-formulas-repository.ts`, `prisma-assessments-repository.ts`, `prisma-grades-repository.ts` em `infra/database/prisma/repositories/`.
- Mappers: `grade-formula-mapper.ts`, `assessment-mapper.ts`, `grade-mapper.ts` em `infra/database/prisma/mappers/`.

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Presenter |
| --- | --- | --- | --- |
| `POST /grade-formulas` | `configure-grade-formula.controller.ts` | dados de `ConfigureGradeFormulaUseCase` | `grade-formula-presenter.ts` |
| `POST /assessments` | `create-assessment.controller.ts` | dados de `CreateAssessmentUseCase` | `assessment-presenter.ts` |
| `POST /assessments/:assessmentId/grades` | `launch-grades.controller.ts` | `{ grades: [{ studentId, value }] }` | `grade-presenter.ts` (lista) |
| `PATCH /assessments/:assessmentId/publish` | `publish-assessment.controller.ts` | — (param) | — (204) |
| `GET /students/:studentId/report-card` | `get-report-card.controller.ts` | query `{ period? }` | `report-card-presenter.ts` |
| `GET /students/:studentId/transcript` | `get-transcript.controller.ts` | — (param) | `transcript-presenter.ts` (PDF, ver nota abaixo) |

Geração de PDF (boletim/histórico): usar a lib de PDF definida no bootstrap (`F1-E00`) — se ainda não decidida, `pdf-lib` é a opção padrão (leve, sem dependência de headless browser). Reaproveitar a mesma lib em `F1-E07`.

Perfis exigidos: `ConfigureGradeFormulaUseCase`/`CreateAssessmentUseCase`/`PublishAssessmentUseCase` → `coordenação`; `LaunchGradesUseCase` → `professor` da turma/disciplina; `GenerateReportCardUseCase`/`GenerateTranscriptUseCase` → `responsável`/`aluno`/`secretaria` (guard fino vem de `F1-E09`; por ora `JwtAuthGuard` padrão).

## Frontend (`diceduc-web`)

- `app/(portal)/notas/formulas/page.tsx` — configuração de `GradeFormula` por segmento (`F1-E04-U01`).
- `app/(portal)/notas/turmas/[turmaId]/lancamento/page.tsx` — lançamento em lote (`F1-E04-U02`).
- `app/(portal)/notas/[studentId]/boletim/page.tsx` — boletim consumido pelo portal do responsável/aluno (tela compartilhada com `F1-E08`).
- `features/assessment/components/GradeFormulaForm.tsx`, `LaunchGradesTable.tsx`, `ReportCard.tsx`.
- `features/assessment/api/assessments.ts` (`configureGradeFormula`, `createAssessment`, `launchGrades`, `publishAssessment`, `getReportCard`, `getTranscript`), `features/assessment/schemas/assessment.ts`.

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/assessment/enterprise/entities/grade-formula.ts`
- [ ] `diceduc-api/src/domain/assessment/enterprise/entities/assessment.ts`
- [ ] `diceduc-api/src/domain/assessment/enterprise/entities/grade.ts`
- [ ] `diceduc-api/src/domain/assessment/enterprise/entities/valueObjects/grade-scale.ts`
- [ ] `diceduc-api/src/domain/assessment/enterprise/entities/valueObjects/assessment-status.ts`
- [ ] `diceduc-api/src/domain/assessment/application/useCases/configure-grade-formula.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/assessment/application/useCases/create-assessment.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/assessment/application/useCases/launch-grades.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/assessment/application/useCases/publish-assessment.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/assessment/application/useCases/calculate-final-average.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/assessment/application/useCases/generate-report-card.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/assessment/application/useCases/generate-transcript.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/assessment/application/repositories/grade-formulas-repository.ts`
- [ ] `diceduc-api/src/domain/assessment/application/repositories/assessments-repository.ts`
- [ ] `diceduc-api/src/domain/assessment/application/repositories/grades-repository.ts`
- [ ] `diceduc-api/prisma/schema.prisma` (editar)
- [ ] `diceduc-api/src/infra/database/prisma/repositories/*.ts` (3 repositórios, ver seção Persistência)
- [ ] `diceduc-api/src/infra/database/prisma/mappers/*.ts` (3 mappers, ver seção Persistência)
- [ ] `diceduc-api/src/infra/http/controllers/*.ts` (6 controllers, ver seção HTTP)
- [ ] `diceduc-api/src/infra/http/presenters/*.ts` (4 presenters, ver seção HTTP)
- [ ] `diceduc-web/app/(portal)/notas/formulas/page.tsx`
- [ ] `diceduc-web/app/(portal)/notas/turmas/[turmaId]/lancamento/page.tsx`
- [ ] `diceduc-web/app/(portal)/notas/[studentId]/boletim/page.tsx`
- [ ] `diceduc-web/features/assessment/components/*.tsx`
- [ ] `diceduc-web/features/assessment/api/assessments.ts`
- [ ] `diceduc-web/features/assessment/schemas/assessment.ts`

## Testes

- Repositórios in-memory: `in-memory-grade-formulas-repository.ts`, `in-memory-assessments-repository.ts`, `in-memory-grades-repository.ts`.
- Factories: `make-grade-formula.ts`, `make-assessment.ts`, `make-grade.ts`.
- Unit specs: validação de pesos somando 100 (`ConfigureGradeFormulaUseCase`), validação de nota fora da escala (`LaunchGradesUseCase`), professor sem horário vigente naquela turma/disciplina → `NotAllowedError` (`LaunchGradesUseCase`, usando o `in-memory-horarios-repository.ts` de `F1-E05`), boletim não expõe rascunho (`GenerateReportCardUseCase`), cálculo de recuperação (`CalculateFinalAverageUseCase`).
- E2E: um `.e2e-spec.ts` por controller listado na seção HTTP.

## Definition of Done

- [ ] Critérios de aceite de `F1-E04-U01` a `F1-E04-U04` no BACKLOG satisfeitos.
- [ ] Todos os use cases e controllers implementados com testes unitários e e2e passando.
- [ ] Migração Prisma aplicada sem erro.
- [ ] Configuração de fórmula, lançamento em lote, boletim (com PDF) e histórico (com PDF) funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E04` para 🟢 quando implementado.
