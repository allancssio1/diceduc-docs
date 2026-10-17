# F1-E03 — Gestão de frequência

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E03 — Gestão de frequência |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | M |
| Bounded context(s) | `attendance` |
| Depende de | F1-E02, F1-E05 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E03` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Registrar presença/falta por turma e aula, calcular automaticamente o percentual de frequência do aluno e permitir justificar faltas com documento/atestado.

## Histórias cobertas

- `F1-E03-U01` — Professor faz a chamada de uma turma por aula.
- `F1-E03-U02` — Coordenação visualiza alunos com frequência abaixo do limite mínimo.
- `F1-E03-U03` — Secretaria justifica uma falta com base em atestado/documento.

## Decisão resolvida nesta spec

O épico sinalizava risco de `F1-E05` (horários) não existir ainda. Resolução: **não modelar uma entidade `ClassSession` própria**. `AttendanceRecord` guarda diretamente `classGroupId + subjectId + date`, sem depender de um `ScheduleSlot` específico — "uma aula" é o par `(classGroup, subject, date)`, suficiente para os critérios de aceite. Se `F1-E05` já existir, o formulário de chamada pode pré-selecionar `subjectId` a partir da grade do dia, mas isso é só UX (frontend), não uma dependência de dado no backend.

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md). Específico deste épico:

- `AttendanceRecord` carrega `groupId` e `schoolId` (herdado da `classGroupId`).
- `GetAttendanceBelowThresholdUseCase` (consulta por turma) já fica automaticamente restrito ao escopo do usuário via `GroupContext` — nenhuma lógica adicional de filtro necessária no use case além do que a Prisma extension de `F1-E0A` já aplica.

## Modelo de domínio (`enterprise`)

### Contexto `attendance` — `diceduc-api/src/domain/attendance/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `AttendanceRecord` | `Entity` | `attendance-record.ts` | `studentId`, `classGroupId`, `subjectId`, `date`, `status: AttendanceStatus`, `justification?: string`, `justifiedBy?: string`, `justifiedAt?: Date`, `recordedBy: string`, `createdAt`, `updatedAt` |

Value objects — `entities/valueObjects/`:
- `attendance-status.ts` — `AttendanceStatus = 'PRESENT' | 'ABSENT' | 'JUSTIFIED'`.

Invariantes:
- `date` deve estar dentro do intervalo do `SchoolYear` vigente e não pode coincidir com um `CalendarEvent` do tipo `HOLIDAY`/`RECESS` (validação cruzando com `SchoolYearsRepository` de `F1-E01`).
- Transição para `'JUSTIFIED'` só é permitida a partir de `'ABSENT'`, e sempre grava `justifiedBy`/`justifiedAt`.

## Use cases (`application`)

### `diceduc-api/src/domain/attendance/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `RecordAttendanceUseCase` | `record-attendance.ts` | `classGroupId`, `subjectId`, `date`, `records: { studentId, status }[]`, `recordedBy`, `teacherId?` | `Either<InvalidCalendarDateError \| NotAllowedError, { records: AttendanceRecord[] }>` | `invalid-calendar-date-error.ts` |
| `GetAttendanceBelowThresholdUseCase` | `get-attendance-below-threshold.ts` | `classGroupId`, `thresholdPercentage` | `Either<ResourceNotFoundError, { students: { studentId: string; percentage: number }[] }>` | — |
| `JustifyAbsenceUseCase` | `justify-absence.ts` | `attendanceRecordId`, `justification`, `justifiedBy` | `Either<ResourceNotFoundError \| AbsenceAlreadyJustifiedError, { record: AttendanceRecord }>` | `absence-already-justified-error.ts` |

Ports (`application/repositories/`): `AttendanceRecordsRepository` — inclui `findByClassGroupAndDate(classGroupId, subjectId, date)`, `calculatePercentage(studentId, classGroupId): Promise<number>` (usado tanto pelo use case de limiar quanto futuramente pelo `AttendanceStatsProvider` de `F1-E02`).

Regras de negócio principais:
- `RecordAttendanceUseCase`: grava em lote um registro por aluno da turma para a mesma `(classGroupId, subjectId, date)`; se já existir chamada para essa combinação, sobrescreve os registros existentes (idempotente); valida a data contra o calendário letivo antes de salvar qualquer registro. Quando `teacherId` é informado (o controller só o passa se o ator autenticado tiver papel `professor`, não `secretaria`/`coordenação`), exige `ScheduleSlotsRepository.existsActiveFor(teacherId, classGroupId, subjectId, date)` (de `F1-E05`) antes de gravar — senão `NotAllowedError`. Evita que um professor registre chamada de uma turma/disciplina onde não leciona, mesmo tendo o papel correto.
- `GetAttendanceBelowThresholdUseCase`: calcula `calculatePercentage` por aluno ativo da turma e retorna só os abaixo de `thresholdPercentage`.
- `JustifyAbsenceUseCase`: só aceita registros com `status = 'ABSENT'`; grava auditoria mínima (`justifiedBy`, `justifiedAt`) — trilha de auditoria completa/consultável fica em `F1-E09`, aqui só os campos na própria entidade.

## Persistência (`infra/database/prisma`)

- `diceduc-api/prisma/schema.prisma`: novo model `AttendanceRecord` (índice único em `studentId + classGroupId + subjectId + date`); enum `AttendanceStatus`.
- Repositório: `infra/database/prisma/repositories/prisma-attendance-records-repository.ts`.
- Mapper: `infra/database/prisma/mappers/attendance-record-mapper.ts`.

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Presenter |
| --- | --- | --- | --- |
| `POST /class-groups/:classGroupId/attendance` | `record-attendance.controller.ts` | `{ subjectId, date, records: [{ studentId, status }] }` | `attendance-record-presenter.ts` (lista) |
| `GET /class-groups/:classGroupId/attendance/below-threshold` | `get-attendance-below-threshold.controller.ts` | query `{ threshold }` | JSON simples `{ studentId, percentage }[]` |
| `PATCH /attendance/:attendanceRecordId/justify` | `justify-absence.controller.ts` | `{ justification }` | `attendance-record-presenter.ts` |

Perfis exigidos: `RecordAttendanceUseCase` → `professor` da turma; `GetAttendanceBelowThresholdUseCase` → `coordenação`; `JustifyAbsenceUseCase` → `secretaria` (guard fino vem de `F1-E09`; por ora `JwtAuthGuard` padrão).

## Frontend (`diceduc-web`)

- `app/(portal)/attendance/class-groups/[classGroupId]/roll-call/page.tsx` — chamada do dia (`F1-E03-U01`), tela do professor.
- `app/(portal)/attendance/below-threshold/page.tsx` — lista filtrável por turma (`F1-E03-U02`), tela da coordenação.
- `features/attendance/components/AttendanceSheet.tsx`, `BelowThresholdList.tsx`, `JustifyAbsenceDialog.tsx`.
- `features/attendance/api/attendance.ts` (`recordAttendance`, `getBelowThreshold`, `justifyAbsence`), `features/attendance/schemas/attendance.ts`.

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/attendance/enterprise/entities/attendance-record.ts`
- [ ] `diceduc-api/src/domain/attendance/enterprise/entities/valueObjects/attendance-status.ts`
- [ ] `diceduc-api/src/domain/attendance/application/useCases/record-attendance.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/attendance/application/useCases/get-attendance-below-threshold.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/attendance/application/useCases/justify-absence.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/attendance/application/repositories/attendance-records-repository.ts`
- [ ] `diceduc-api/prisma/schema.prisma` (editar)
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-attendance-records-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/attendance-record-mapper.ts`
- [ ] `diceduc-api/src/infra/http/controllers/record-attendance.controller.ts`
- [ ] `diceduc-api/src/infra/http/controllers/get-attendance-below-threshold.controller.ts`
- [ ] `diceduc-api/src/infra/http/controllers/justify-absence.controller.ts`
- [ ] `diceduc-api/src/infra/http/presenters/attendance-record-presenter.ts`
- [ ] `diceduc-web/app/(portal)/attendance/class-groups/[classGroupId]/roll-call/page.tsx`
- [ ] `diceduc-web/app/(portal)/attendance/below-threshold/page.tsx`
- [ ] `diceduc-web/features/attendance/components/AttendanceSheet.tsx`, `BelowThresholdList.tsx`, `JustifyAbsenceDialog.tsx`
- [ ] `diceduc-web/features/attendance/api/attendance.ts`
- [ ] `diceduc-web/features/attendance/schemas/attendance.ts`

## Testes

- Repositório in-memory: `in-memory-attendance-records-repository.ts`.
- Factory: `make-attendance-record.ts`.
- Unit specs: bloqueio por data fora do calendário letivo (`RecordAttendanceUseCase`), professor sem horário vigente naquela turma/disciplina/data → `NotAllowedError` (`RecordAttendanceUseCase`, usando o `in-memory-schedule-slots-repository.ts` de `F1-E05`), cálculo de percentual e filtro por limiar (`GetAttendanceBelowThresholdUseCase`), transição inválida para justificar falta já justificada ou presença (`JustifyAbsenceUseCase`).
- E2E: um `.e2e-spec.ts` por controller listado na seção HTTP.

## Definition of Done

- [ ] Critérios de aceite de `F1-E03-U01` a `F1-E03-U03` no BACKLOG satisfeitos.
- [ ] Todos os use cases e controllers implementados com testes unitários e e2e passando.
- [ ] Migração Prisma aplicada sem erro.
- [ ] Chamada, lista de frequência abaixo do limite e justificativa de falta funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E03` para 🟢 quando implementado.
