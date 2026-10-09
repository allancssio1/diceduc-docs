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

- `Horario` carrega `groupId`/`schoolId` (herdado da `turmaId`).
- Detecção de conflito de professor/sala (`findConflictingByTeacher`/`findConflictingBySala`) é sempre calculada **dentro da mesma escola** — um professor pode, em tese, lecionar em duas escolas diferentes do mesmo group sem conflito de agenda ser detectado entre elas nesta ficha (cenário de professor multi-escola fica fora de escopo do MVP; a query de conflito é escopada por `schoolId`, não só por `groupId`).

## Modelo de domínio (`enterprise`)

### Contexto `academic` — `diceduc-api/src/domain/academic/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `Horario` | `Entity` | `horario.ts` | `turmaId`, `disciplinaId`, `teacherId`, `salaId`, `dayOfWeek: DayOfWeek`, `startTime` (`"HH:mm"`), `endTime` (`"HH:mm"`), `effectiveFrom: Date`, `effectiveUntil?: Date` |

Value objects — `entities/valueObjects/`:
- `day-of-week.ts` — `DayOfWeek = 'MON' | 'TUE' | 'WED' | 'THU' | 'FRI' | 'SAT'`.

Invariantes:
- `startTime < endTime`.
- Editar a grade durante o ano não sobrescreve o `Horario` antigo: fecha o registro vigente (`effectiveUntil = novaData - 1 dia`) e cria um novo com `effectiveFrom = novaData`, preservando histórico consultável.

## Use cases (`application`)

### `diceduc-api/src/domain/academic/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `CreateHorarioUseCase` | `create-horario.ts` | `turmaId`, `disciplinaId`, `teacherId`, `salaId`, `dayOfWeek`, `startTime`, `endTime`, `effectiveFrom` | `Either<ResourceNotFoundError \| TeacherConflictError \| SalaConflictError, { horario: Horario }>` | `teacher-conflict-error.ts`, `sala-conflict-error.ts` |
| `UpdateHorarioUseCase` | `update-horario.ts` | `horarioId`, campos alterados, `effectiveFrom` (data de efeito da mudança) | `Either<ResourceNotFoundError \| TeacherConflictError \| SalaConflictError, { horario: Horario }>` | reusa erros de `CreateHorarioUseCase` |
| `DeleteHorarioUseCase` | `delete-horario.ts` | `horarioId`, `effectiveFrom` | `Either<ResourceNotFoundError, void>` | — |
| `GetTeacherScheduleUseCase` | `get-teacher-schedule.ts` | `teacherId`, `date?` (para saber a versão vigente) | `Either<ResourceNotFoundError, { horarios: Horario[] }>` | — |
| `GetTurmaScheduleUseCase` | `get-turma-schedule.ts` | `turmaId`, `date?` | `Either<ResourceNotFoundError, { horarios: Horario[] }>` | — |

Ports (`application/repositories/`): `HorariosRepository` — inclui `findConflictingByTeacher(teacherId, dayOfWeek, startTime, endTime, effectiveFrom)`, `findConflictingBySala(salaId, ...)` e `existsActiveFor(teacherId, turmaId, disciplinaId, date): Promise<boolean>` (usado por `F1-E03`/`F1-E04` para confirmar que um professor só lança presença/nota de uma turma/disciplina onde de fato tem horário vigente naquela data — ver "Perfis exigidos" de cada spec).

Regras de negócio principais:
- `CreateHorarioUseCase`/`UpdateHorarioUseCase`: checam conflito de professor (mesmo `teacherId`, mesmo `dayOfWeek`, intervalo de horário sobreposto, período de vigência sobreposto) e de sala (mesma lógica trocando `teacherId` por `salaId`) antes de salvar.
- `UpdateHorarioUseCase`: nunca faz `UPDATE` destrutivo — sempre fecha a versão vigente e cria uma nova (ver invariante acima).

## Persistência (`infra/database/prisma`)

- `diceduc-api/prisma/schema.prisma`: novo model `Horario`; enum `DayOfWeek`.
- Repositório: `infra/database/prisma/repositories/prisma-horarios-repository.ts` (queries de conflito via `WHERE` sobreposição de intervalo).
- Mapper: `infra/database/prisma/mappers/horario-mapper.ts`.

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Presenter |
| --- | --- | --- | --- |
| `POST /horarios` | `create-horario.controller.ts` | dados de `CreateHorarioUseCase` | `horario-presenter.ts` |
| `PATCH /horarios/:horarioId` | `update-horario.controller.ts` | dados de `UpdateHorarioUseCase` | `horario-presenter.ts` |
| `DELETE /horarios/:horarioId` | `delete-horario.controller.ts` | `{ effectiveFrom }` | — (204) |
| `GET /teachers/:teacherId/schedule` | `get-teacher-schedule.controller.ts` | query `{ date? }` | `horario-presenter.ts` (lista) |
| `GET /turmas/:turmaId/schedule` | `get-turma-schedule.controller.ts` | query `{ date? }` | `horario-presenter.ts` (lista) |

Perfis exigidos: `CreateHorarioUseCase`/`UpdateHorarioUseCase`/`DeleteHorarioUseCase` → `coordenação`; leitura de grade → `professor`/`coordenação`/`secretaria` (guard fino vem de `F1-E09`; por ora `JwtAuthGuard` padrão).

## Frontend (`diceduc-web`)

- `app/(portal)/horarios/turmas/[turmaId]/page.tsx` — montagem da grade da turma, com destaque de conflito (`F1-E05-U01`).
- `app/(portal)/horarios/minha-grade/page.tsx` — grade consolidada do professor logado (`F1-E05-U02`).
- `features/academic/components/ScheduleGrid.tsx`, `HorarioForm.tsx`, `ConflictWarning.tsx`.
- `features/academic/api/horarios.ts` (`createHorario`, `updateHorario`, `deleteHorario`, `getTeacherSchedule`, `getTurmaSchedule`), `features/academic/schemas/horario.ts`.

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/academic/enterprise/entities/horario.ts`
- [ ] `diceduc-api/src/domain/academic/enterprise/entities/valueObjects/day-of-week.ts`
- [ ] `diceduc-api/src/domain/academic/application/useCases/create-horario.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/update-horario.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/delete-horario.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/get-teacher-schedule.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/useCases/get-turma-schedule.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/academic/application/repositories/horarios-repository.ts`
- [ ] `diceduc-api/prisma/schema.prisma` (editar)
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-horarios-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/horario-mapper.ts`
- [ ] `diceduc-api/src/infra/http/controllers/*.ts` (5 controllers, ver seção HTTP)
- [ ] `diceduc-api/src/infra/http/presenters/horario-presenter.ts`
- [ ] `diceduc-web/app/(portal)/horarios/turmas/[turmaId]/page.tsx`
- [ ] `diceduc-web/app/(portal)/horarios/minha-grade/page.tsx`
- [ ] `diceduc-web/features/academic/components/ScheduleGrid.tsx`, `HorarioForm.tsx`, `ConflictWarning.tsx`
- [ ] `diceduc-web/features/academic/api/horarios.ts`
- [ ] `diceduc-web/features/academic/schemas/horario.ts`

## Testes

- Repositório in-memory: `in-memory-horarios-repository.ts` (com lógica de sobreposição de intervalo replicada).
- Factory: `make-horario.ts`.
- Unit specs: conflito de professor, conflito de sala, edição preservando histórico (`effectiveFrom`/`effectiveUntil`), grade consolidada do professor cruzando múltiplas turmas.
- E2E: um `.e2e-spec.ts` por controller listado na seção HTTP.

## Definition of Done

- [ ] Critérios de aceite de `F1-E05-U01` e `F1-E05-U02` no BACKLOG satisfeitos.
- [ ] Todos os use cases e controllers implementados com testes unitários e e2e passando.
- [ ] Migração Prisma aplicada sem erro.
- [ ] Montagem de grade com detecção de conflito e visualização da grade do professor funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E05` para 🟢 quando implementado.
