# F1-E05 — Horários e grade de aulas

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E05 — Horários e grade de aulas |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | M |
| Bounded context(s) | `academic` |
| Depende de | F1-E02 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E05` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Montar a grade de horários de cada turma (professor, disciplina, sala, dia/horário) evitando conflitos de alocação de professor e sala, com edição durante o ano letivo a partir de uma data de efeito.

## Histórias cobertas

- `F1-E05-U01` — Coordenação monta grade de horários evitando conflito de professor/sala.
- `F1-E05-U02` — Professor visualiza sua própria grade consolidada da semana.

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md). Específico deste épico:

- `ScheduleSlot` carrega `groupId`/`schoolId` (herdado da `classGroupId`).
- Detecção de conflito de professor/sala (`findConflictingByTeacher`/`findConflictingByRoom`) é sempre calculada **dentro da mesma escola** — um professor pode, em tese, lecionar em duas escolas diferentes do mesmo group sem conflito de agenda ser detectado entre elas nesta ficha (cenário de professor multi-escola fica fora de escopo do MVP; a query de conflito é escopada por `schoolId`, não só por `groupId`).

## Modelo de domínio (`enterprise`)

### Contexto `academic` — `diceduc-api/src/domain/academic/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `ScheduleSlot` | `Entity` | `schedule-slot.ts` | `classGroupId`, `subjectId`, `teacherId`, `roomId`, `dayOfWeek: DayOfWeek`, `startTime` (`"HH:mm"`), `endTime` (`"HH:mm"`), `effectiveFrom: Date`, `effectiveUntil?: Date` |

Value objects — `entities/valueObjects/`:
- `day-of-week.ts` — `DayOfWeek = 'MON' | 'TUE' | 'WED' | 'THU' | 'FRI' | 'SAT'`.

Invariantes:
- `startTime < endTime`.
- Editar a grade durante o ano não sobrescreve o `ScheduleSlot` antigo: fecha o registro vigente (`effectiveUntil = newDate - 1 dia`) e cria um novo com `effectiveFrom = newDate`, preservando histórico consultável.

## Use cases (`application`)

### `diceduc-api/src/domain/academic/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `CreateScheduleSlotUseCase` | `create-schedule-slot.ts` | `classGroupId`, `subjectId`, `teacherId`, `roomId`, `dayOfWeek`, `startTime`, `endTime`, `effectiveFrom` | `Either<ResourceNotFoundError \| TeacherConflictError \| RoomConflictError, { scheduleSlot: ScheduleSlot }>` | `teacher-conflict-error.ts`, `room-conflict-error.ts` |
| `UpdateScheduleSlotUseCase` | `update-schedule-slot.ts` | `scheduleSlotId`, campos alterados, `effectiveFrom` (data de efeito da mudança) | `Either<ResourceNotFoundError \| TeacherConflictError \| RoomConflictError, { scheduleSlot: ScheduleSlot }>` | reusa erros de `CreateScheduleSlotUseCase` |
| `DeleteScheduleSlotUseCase` | `delete-schedule-slot.ts` | `scheduleSlotId`, `effectiveFrom` | `Either<ResourceNotFoundError, void>` | — |
| `GetTeacherScheduleUseCase` | `get-teacher-schedule.ts` | `teacherId`, `date?` (para saber a versão vigente) | `Either<ResourceNotFoundError, { scheduleSlots: ScheduleSlot[] }>` | — |
| `GetClassGroupScheduleUseCase` | `get-class-group-schedule.ts` | `classGroupId`, `date?` | `Either<ResourceNotFoundError, { scheduleSlots: ScheduleSlot[] }>` | — |

Ports (`application/repositories/`): `ScheduleSlotsRepository` — inclui `findConflictingByTeacher(teacherId, dayOfWeek, startTime, endTime, effectiveFrom)`, `findConflictingByRoom(roomId, ...)` e `existsActiveFor(teacherId, classGroupId, subjectId, date): Promise<boolean>` (usado por `F1-E03`/`F1-E04` para confirmar que um professor só lança presença/nota de uma turma/disciplina onde de fato tem horário vigente naquela data — ver "Perfis exigidos" de cada spec).

**Atenção, este arquivo já existe:** `F1-E03` criou `diceduc-api/src/domain/attendance/application/repositories/schedule-slots-repository.ts` com só `existsActiveFor` (porta+stub, obrigado pela ordem de implementação — ver `epic.md`, "Decisões em aberto"), produção sempre `true` via `StubScheduleSlotsRepository`. Ao implementar esta ficha: estenda esse arquivo com `findConflictingByTeacher`/`findConflictingByRoom` (ou mova para `academic/application/repositories/` se fizer mais sentido ficar junto da entidade `ScheduleSlot` — nesse caso ajuste o import em `RecordAttendanceUseCase`/`JustifyAbsenceUseCase`), implemente `PrismaScheduleSlotsRepository` de verdade e troque o binding em `DatabaseModule`. Nenhum use case de F1-E03 precisa mudar — só a resposta real de `existsActiveFor` deixa de ser sempre `true`.

Regras de negócio principais:
- `CreateScheduleSlotUseCase`/`UpdateScheduleSlotUseCase`: checam conflito de professor (mesmo `teacherId`, mesmo `dayOfWeek`, intervalo de horário sobreposto, período de vigência sobreposto) e de sala (mesma lógica trocando `teacherId` por `roomId`) antes de salvar.
- `UpdateScheduleSlotUseCase`: nunca faz `UPDATE` destrutivo — sempre fecha a versão vigente e cria uma nova (ver invariante acima).

## Persistência (`infra/database/prisma`)

- `diceduc-api/prisma/schema.prisma`: novo model `ScheduleSlot`; enum `DayOfWeek`.
- Repositório: `infra/database/prisma/repositories/prisma-schedule-slots-repository.ts` (queries de conflito via `WHERE` sobreposição de intervalo).
- Mapper: `infra/database/prisma/mappers/schedule-slot-mapper.ts`.

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Presenter |
| --- | --- | --- | --- |
| `POST /schedule-slots` | `create-schedule-slot.controller.ts` | dados de `CreateScheduleSlotUseCase` | `schedule-slot-presenter.ts` |
| `PATCH /schedule-slots/:scheduleSlotId` | `update-schedule-slot.controller.ts` | dados de `UpdateScheduleSlotUseCase` | `schedule-slot-presenter.ts` |
| `DELETE /schedule-slots/:scheduleSlotId` | `delete-schedule-slot.controller.ts` | `{ effectiveFrom }` | — (204) |
| `GET /teachers/:teacherId/schedule` | `get-teacher-schedule.controller.ts` | query `{ date? }` | `schedule-slot-presenter.ts` (lista) |
| `GET /class-groups/:classGroupId/schedule` | `get-class-group-schedule.controller.ts` | query `{ date? }` | `schedule-slot-presenter.ts` (lista) |

Perfis exigidos: `CreateScheduleSlotUseCase`/`UpdateScheduleSlotUseCase`/`DeleteScheduleSlotUseCase` → `coordenação`; leitura de grade → `professor`/`coordenação`/`secretaria` (guard fino vem de `F1-E09`; por ora `JwtAuthGuard` padrão).

## Frontend (`diceduc-web`)

- `app/(portal)/schedules/class-groups/[classGroupId]/page.tsx` — montagem da grade da turma, com destaque de conflito (`F1-E05-U01`).
- `app/(portal)/schedules/my-schedule/page.tsx` — grade consolidada do professor logado (`F1-E05-U02`).
- `features/academic/components/ScheduleGrid.tsx`, `ScheduleSlotForm.tsx`, `ConflictWarning.tsx`.
- `features/academic/api/schedule-slots.ts` (`createScheduleSlot`, `updateScheduleSlot`, `deleteScheduleSlot`, `getTeacherSchedule`, `getClassGroupSchedule`), `features/academic/schemas/schedule-slot.ts`.

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/academic/enterprise/entities/schedule-slot.ts`
- [ ] `diceduc-api/src/domain/academic/enterprise/entities/valueObjects/day-of-week.ts`
- [ ] `diceduc-api/src/domain/academic/application/useCases/create-schedule-slot.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/update-schedule-slot.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/delete-schedule-slot.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/get-teacher-schedule.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/get-class-group-schedule.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/repositories/schedule-slots-repository.ts`
- [ ] `diceduc-api/prisma/schema.prisma` (editar)
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-schedule-slots-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/schedule-slot-mapper.ts`
- [ ] `diceduc-api/src/infra/http/controllers/*.ts` (5 controllers, ver seção HTTP)
- [ ] `diceduc-api/src/infra/http/presenters/schedule-slot-presenter.ts`
- [ ] `diceduc-web/app/(portal)/schedules/class-groups/[classGroupId]/page.tsx`
- [ ] `diceduc-web/app/(portal)/schedules/my-schedule/page.tsx`
- [ ] `diceduc-web/features/academic/components/ScheduleGrid.tsx`, `ScheduleSlotForm.tsx`, `ConflictWarning.tsx`
- [ ] `diceduc-web/features/academic/api/schedule-slots.ts`
- [ ] `diceduc-web/features/academic/schemas/schedule-slot.ts`

## Testes

- Repositório in-memory: `in-memory-schedule-slots-repository.ts` (com lógica de sobreposição de intervalo replicada).
- Factory: `make-schedule-slot.ts`.
- Unit specs: conflito de professor, conflito de sala, edição preservando histórico (`effectiveFrom`/`effectiveUntil`), grade consolidada do professor cruzando múltiplas turmas.
- E2E: um `.e2e-spec.ts` por controller listado na seção HTTP.

## Definition of Done

- [ ] Critérios de aceite de `F1-E05-U01` e `F1-E05-U02` no BACKLOG satisfeitos.
- [ ] Todos os use cases e controllers implementados com testes unitários e e2e passando.
- [ ] Migração Prisma aplicada sem erro.
- [ ] Montagem de grade com detecção de conflito e visualização da grade do professor funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E05` para 🟢 quando implementado.
