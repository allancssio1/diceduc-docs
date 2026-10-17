# F1-E01 — Gestão de cadastros / SIS básico

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E01 — Gestão de cadastros / SIS básico |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | `people`, `academic` |
| Depende de | F1-E00 (bootstrap) |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E01` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Cadastro e histórico de alunos, responsáveis, professores e funcionários, mais a estrutura acadêmica base (ano letivo, séries, turmas, disciplinas, turnos, salas). É a fundação sobre a qual todos os outros épicos da Fase 1 são construídos — **implemente esta ficha antes de qualquer outra de domínio**.

## Histórias cobertas

- `F1-E01-U01` — Secretaria cadastra aluno com dados pessoais, documentos e contatos de emergência.
- `F1-E01-U02` — Secretaria vincula um ou mais responsáveis (pedagógico/financeiro) a um aluno.
- `F1-E01-U03` — Coordenação cadastra séries, turmas, disciplinas, turnos, salas e o calendário letivo do ano.
- `F1-E01-U04` — Secretaria atualiza a situação do aluno (ativo, transferido, egresso, trancado).

Cadastro de professores e funcionários é incluído como base mínima (nome + documento + contato), sem lançamentos pedagógicos — isso vem em épicos futuros (ex. `F1-E05` aloca professor em horário).

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md) — esta ficha depende de `F1-E0A` já estar implementada. Específico deste épico:

- `Student`, `Teacher`, `Staff` carregam `schoolId` (props + coluna Prisma) — pertencem a uma escola do group.
- `Guardian` é **group-level**, sem `schoolId` — um responsável pode ter filhos em escolas diferentes do mesmo group (ex.: uma secretaria com dois filhos em duas escolas da rede).
- `SchoolYear`, `GradeLevel`, `ClassGroup`, `Subject`, `Room` carregam `schoolId` — estrutura acadêmica é por escola, não compartilhada entre escolas do mesmo group.
- Unicidade: CPF (`Student`, `Guardian`, `Teacher`, `Staff`) único **por group**, não globalmente; `SchoolYear.year` único **por escola** (duas escolas do mesmo group podem ter, cada uma, seu "2026").
- `RegisterStudentUseCase`/`CreateClassGroupUseCase`/etc. **não** recebem `schoolId` como parâmetro de entrada — a Prisma extension injeta o `schoolId` da sessão automaticamente na escrita (ver `arquitetura-ignite.md` §11; não existe mais um helper de validação contra uma lista de escolas, porque uma sessão é sempre de uma única escola).

## Modelo de domínio (`enterprise`)

### Contexto `people` — `diceduc-api/src/domain/people/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `Student` | `AggregateRoot` | `student.ts` | `name`, `birthDate`, `cpf?`, `rg?`, `birthCertificateNumber?`, `address: Address`, `emergencyContacts: EmergencyContact[]`, `status: StudentStatus`, `createdAt`, `updatedAt` |
| `Guardian` | `Entity` | `guardian.ts` | `name`, `cpf`, `phone`, `email`, `address: Address` |
| `StudentGuardian` | `Entity` | `student-guardian.ts` | `studentId`, `guardianId`, `role: GuardianRole` (link aluno↔responsável) |
| `Teacher` | `Entity` | `teacher.ts` | `name`, `cpf`, `email`, `phone` |
| `Staff` | `Entity` | `staff.ts` | `name`, `cpf`, `email`, `phone`, `role` (cargo administrativo) |

Value objects — `entities/valueObjects/`:

- `address.ts` — `Address` (`street`, `number`, `city`, `state`, `zipCode`).
- `emergency-contact.ts` — `EmergencyContact` (`name`, `phone`, `relationship`).
- `student-status.ts` — enum `StudentStatus = 'ACTIVE' | 'TRANSFERRED' | 'GRADUATED' | 'SUSPENDED'` + helper de transição válida (ver regras em `UpdateStudentStatusUseCase`).
- `guardian-role.ts` — enum `GuardianRole = 'PEDAGOGICAL' | 'FINANCIAL' | 'BOTH'`.

Invariantes:
- `Student` nasce com `status = 'ACTIVE'`.
- Mudança de status registra `updatedAt` (histórico completo de auditoria é responsabilidade de `F1-E09`, não duplicar aqui).

### Contexto `academic` — `diceduc-api/src/domain/academic/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `SchoolYear` | `AggregateRoot` | `school-year.ts` | `year`, `startDate`, `endDate`, `events: CalendarEvent[]` (feriados/recessos) |
| `GradeLevel` | `Entity` | `grade-level.ts` | `name`, `segment: Segment` (`'EARLY_CHILDHOOD' \| 'ELEMENTARY' \| 'HIGH_SCHOOL'`), `schoolYearId` |
| `ClassGroup` | `AggregateRoot` | `class-group.ts` | `name`, `gradeLevelId`, `schoolYearId`, `shift: Shift`, `capacity` (vagas) |
| `Subject` | `Entity` | `subject.ts` | `name`, `gradeLevelIds: string[]` (séries em que é lecionada) |
| `Room` | `Entity` | `room.ts` | `name`, `capacity` |

Value objects — `entities/valueObjects/`:
- `shift.ts` — enum `Shift = 'MORNING' | 'AFTERNOON' | 'EVENING' | 'FULL_TIME'`.
- `calendar-event.ts` — `CalendarEvent` (`date`, `type: 'HOLIDAY' | 'RECESS'`, `description`).

Invariantes:
- `ClassGroup` não pode ser criada com `capacity <= 0`.
- `GradeLevel`/`Subject` só podem referenciar um `SchoolYear` existente.

## Use cases (`application`)

### Contexto `people` — `diceduc-api/src/domain/people/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `RegisterStudentUseCase` | `register-student.ts` | dados pessoais + documentos + contatos de emergência + endereço | `Either<InvalidStudentDataError, { student: Student }>` | `invalid-student-data-error.ts` |
| `LinkGuardianToStudentUseCase` | `link-guardian-to-student.ts` | `studentId`, dados do responsável (ou `guardianId` existente), `role` | `Either<ResourceNotFoundError, { studentGuardian: StudentGuardian }>` | reusa `ResourceNotFoundError` de `core/errors` |
| `UpdateStudentStatusUseCase` | `update-student-status.ts` | `studentId`, `newStatus` | `Either<ResourceNotFoundError \| InvalidStatusTransitionError, { student: Student }>` | `invalid-status-transition-error.ts` |
| `RegisterTeacherUseCase` | `register-teacher.ts` | dados básicos do professor | `Either<InvalidTeacherDataError, { teacher: Teacher }>` | `invalid-teacher-data-error.ts` |
| `RegisterStaffUseCase` | `register-staff.ts` | dados básicos do funcionário | `Either<InvalidStaffDataError, { staff: Staff }>` | `invalid-staff-data-error.ts` |

Ports (`application/repositories/`, `abstract class`): `StudentsRepository`, `GuardiansRepository`, `StudentGuardiansRepository`, `TeachersRepository`, `StaffRepository`.

Regras de negócio principais:
- `RegisterStudentUseCase`: valida campos obrigatórios (nome, data de nascimento, ao menos um documento, endereço, ≥1 contato de emergência) antes de criar; atribui `status = 'ACTIVE'`.
- `LinkGuardianToStudentUseCase`: se `guardianId` não informado, cria um novo `Guardian` primeiro; valida que `studentId` existe; permite múltiplos responsáveis por aluno (não deduplica papéis — um aluno pode ter 2 responsáveis financeiros).
- `UpdateStudentStatusUseCase`: valida transição permitida (ex.: não permitir `GRADUATED → ACTIVE` diretamente); alunos fora de `ACTIVE` continuam consultáveis (não são deletados).

### Contexto `academic` — `diceduc-api/src/domain/academic/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) |
| --- | --- | --- | --- |
| `CreateSchoolYearUseCase` | `create-school-year.ts` | `year`, `startDate`, `endDate`, `events?` | `Either<InvalidDateRangeError, { schoolYear: SchoolYear }>` |
| `CreateGradeLevelUseCase` | `create-grade-level.ts` | `name`, `segment`, `schoolYearId` | `Either<ResourceNotFoundError, { gradeLevel: GradeLevel }>` |
| `CreateClassGroupUseCase` | `create-class-group.ts` | `name`, `gradeLevelId`, `schoolYearId`, `shift`, `capacity` | `Either<ResourceNotFoundError \| InvalidCapacityError, { classGroup: ClassGroup }>` |
| `CreateSubjectUseCase` | `create-subject.ts` | `name`, `gradeLevelIds` | `Either<ResourceNotFoundError, { subject: Subject }>` |
| `CreateRoomUseCase` | `create-room.ts` | `name`, `capacity` | `Either<InvalidCapacityError, { room: Room }>` |
| `DeleteClassGroupUseCase` | `delete-class-group.ts` | `classGroupId` | `Either<ResourceNotFoundError \| ClassGroupHasStudentsError, void>` |

Ports: `SchoolYearsRepository`, `GradeLevelsRepository`, `ClassGroupsRepository`, `SubjectsRepository`, `RoomsRepository`.

Regra de negócio destacada: `DeleteClassGroupUseCase` bloqueia exclusão se houver alunos matriculados (a checagem em si — vínculo aluno↔turma — pertence ao domínio `academic` que `F1-E02` vai estender; nesta ficha, deixar o port `ClassGroupsRepository.hasEnrolledStudents(classGroupId)` definido como método a ser implementado quando `F1-E02` existir, retornando `false` por padrão nesta ficha).

## Persistência (`infra/database/prisma`)

### `diceduc-api/prisma/schema.prisma` — novos models

- `Student`, `Guardian`, `StudentGuardian` (tabela de junção com `role`), `Teacher`, `Staff`.
- `SchoolYear`, `GradeLevel`, `ClassGroup`, `Subject`, `SubjectGradeLevel` (junção N:N), `Room`, `CalendarEvent`.
- Enums Prisma espelhando os VOs: `StudentStatus`, `GuardianRole`, `Segment`, `Shift`, `CalendarEventType`.

### Repositórios — `diceduc-api/src/infra/database/prisma/repositories/`

Um arquivo por port listado acima (`prisma-students-repository.ts`, `prisma-guardians-repository.ts`, `prisma-student-guardians-repository.ts`, `prisma-teachers-repository.ts`, `prisma-staff-repository.ts`, `prisma-school-years-repository.ts`, `prisma-grade-levels-repository.ts`, `prisma-class-groups-repository.ts`, `prisma-subjects-repository.ts`, `prisma-rooms-repository.ts`).

### Mappers — `diceduc-api/src/infra/database/prisma/mappers/`

Um mapper estático por entidade (`student-mapper.ts`, `guardian-mapper.ts`, etc.) com `toDomain`/`toPrisma`, convertendo `string ↔ UniqueEntityId` e VOs (`Address`, `EmergencyContact`) para/de JSON (campo `Json` no Prisma) ou colunas próprias — decidir na implementação conforme necessidade de consulta.

### Módulos

- `PeopleModule` e `AcademicModule` (ou registrar tudo em `DatabaseModule`, seguindo o padrão de `{ provide: AbstractRepository, useClass: PrismaRepository }` já usado na arquitetura de referência).

## HTTP (`infra/http`)

Controllers em `diceduc-api/src/infra/http/controllers/`, um por use case, todos exigindo perfil `secretaria` ou `coordenação` (ver `F1-E09` para o guard de perfis — nesta ficha, aplicar apenas `JwtAuthGuard` padrão, sem checagem fina de papel ainda).

| Rota | Controller | Schema Zod (entrada) | Presenter |
| --- | --- | --- | --- |
| `POST /students` | `register-student.controller.ts` | dados de `RegisterStudentUseCase` | `student-presenter.ts` |
| `POST /students/:studentId/guardians` | `link-guardian-to-student.controller.ts` | dados de `LinkGuardianToStudentUseCase` | `student-guardian-presenter.ts` |
| `PATCH /students/:studentId/status` | `update-student-status.controller.ts` | `{ newStatus }` | `student-presenter.ts` |
| `GET /students/:studentId` | `get-student.controller.ts` | — (param) | `student-presenter.ts` (inclui responsáveis vinculados) |
| `POST /teachers` | `register-teacher.controller.ts` | dados de `RegisterTeacherUseCase` | `teacher-presenter.ts` |
| `POST /staff` | `register-staff.controller.ts` | dados de `RegisterStaffUseCase` | `staff-presenter.ts` |
| `POST /school-years` | `create-school-year.controller.ts` | dados de `CreateSchoolYearUseCase` | `school-year-presenter.ts` |
| `POST /grade-levels` | `create-grade-level.controller.ts` | dados de `CreateGradeLevelUseCase` | `grade-level-presenter.ts` |
| `POST /class-groups` | `create-class-group.controller.ts` | dados de `CreateClassGroupUseCase` | `class-group-presenter.ts` |
| `DELETE /class-groups/:classGroupId` | `delete-class-group.controller.ts` | — (param) | — (204) |
| `POST /subjects` | `create-subject.controller.ts` | dados de `CreateSubjectUseCase` | `subject-presenter.ts` |
| `POST /rooms` | `create-room.controller.ts` | dados de `CreateRoomUseCase` | `room-presenter.ts` |

## Frontend (`diceduc-web`)

Contexto `features/people/` e `features/academic/`, sob a rota `app/(portal)/records/`:

- `app/(portal)/records/students/page.tsx` — lista de alunos (busca/filtro por situação).
- `app/(portal)/records/students/new/page.tsx` — formulário de `F1-E01-U01` (`RegisterStudentForm`).
- `app/(portal)/records/students/[studentId]/page.tsx` — detalhe do aluno: dados pessoais, responsáveis vinculados (`F1-E01-U02`), ação de alterar situação (`F1-E01-U04`).
- `app/(portal)/records/structure/page.tsx` — gestão de ano letivo/séries/turmas/disciplinas/salas (`F1-E01-U03`), telas simples de CRUD.
- `features/people/components/RegisterStudentForm.tsx`, `LinkGuardianForm.tsx`, `StudentStatusBadge.tsx`.
- `features/people/api/students.ts` (`registerStudent`, `getStudent`, `linkGuardian`, `updateStudentStatus`), `features/people/schemas/student.ts`.
- `features/academic/components/SchoolYearForm.tsx`, `ClassGroupForm.tsx`, etc.
- `features/academic/api/academic.ts`, `features/academic/schemas/academic.ts`.

## Arquivos a criar/editar (checklist)

```
diceduc-api/src/domain/people/enterprise/entities/student.ts
diceduc-api/src/domain/people/enterprise/entities/guardian.ts
diceduc-api/src/domain/people/enterprise/entities/student-guardian.ts
diceduc-api/src/domain/people/enterprise/entities/teacher.ts
diceduc-api/src/domain/people/enterprise/entities/staff.ts
diceduc-api/src/domain/people/enterprise/entities/valueObjects/address.ts
diceduc-api/src/domain/people/enterprise/entities/valueObjects/emergency-contact.ts
diceduc-api/src/domain/people/enterprise/entities/valueObjects/student-status.ts
diceduc-api/src/domain/people/enterprise/entities/valueObjects/guardian-role.ts
diceduc-api/src/domain/people/application/useCases/register-student.ts (+ errors/, + .spec.ts)
diceduc-api/src/domain/people/application/useCases/link-guardian-to-student.ts (+ .spec.ts)
diceduc-api/src/domain/people/application/useCases/update-student-status.ts (+ errors/, + .spec.ts)
diceduc-api/src/domain/people/application/useCases/register-teacher.ts (+ errors/, + .spec.ts)
diceduc-api/src/domain/people/application/useCases/register-staff.ts (+ errors/, + .spec.ts)
diceduc-api/src/domain/people/application/repositories/students-repository.ts
diceduc-api/src/domain/people/application/repositories/guardians-repository.ts
diceduc-api/src/domain/people/application/repositories/student-guardians-repository.ts
diceduc-api/src/domain/people/application/repositories/teachers-repository.ts
diceduc-api/src/domain/people/application/repositories/staff-repository.ts

diceduc-api/src/domain/academic/enterprise/entities/school-year.ts
diceduc-api/src/domain/academic/enterprise/entities/grade-level.ts
diceduc-api/src/domain/academic/enterprise/entities/class-group.ts
diceduc-api/src/domain/academic/enterprise/entities/subject.ts
diceduc-api/src/domain/academic/enterprise/entities/room.ts
diceduc-api/src/domain/academic/enterprise/entities/valueObjects/shift.ts
diceduc-api/src/domain/academic/enterprise/entities/valueObjects/calendar-event.ts
diceduc-api/src/domain/academic/application/useCases/create-school-year.ts (+ .spec.ts)
diceduc-api/src/domain/academic/application/useCases/create-grade-level.ts (+ .spec.ts)
diceduc-api/src/domain/academic/application/useCases/create-class-group.ts (+ errors/, + .spec.ts)
diceduc-api/src/domain/academic/application/useCases/create-subject.ts (+ .spec.ts)
diceduc-api/src/domain/academic/application/useCases/create-room.ts (+ .spec.ts)
diceduc-api/src/domain/academic/application/useCases/delete-class-group.ts (+ errors/, + .spec.ts)
diceduc-api/src/domain/academic/application/repositories/school-years-repository.ts
diceduc-api/src/domain/academic/application/repositories/grade-levels-repository.ts
diceduc-api/src/domain/academic/application/repositories/class-groups-repository.ts
diceduc-api/src/domain/academic/application/repositories/subjects-repository.ts
diceduc-api/src/domain/academic/application/repositories/rooms-repository.ts

diceduc-api/prisma/schema.prisma                          (editar — adicionar models/enums acima)
diceduc-api/src/infra/database/prisma/repositories/*.ts   (10 repositórios, ver seção Persistência)
diceduc-api/src/infra/database/prisma/mappers/*.ts        (10 mappers, ver seção Persistência)

diceduc-api/src/infra/http/controllers/*.ts                (12 controllers, ver seção HTTP)
diceduc-api/src/infra/http/presenters/*.ts                 (8 presenters, ver seção HTTP)

diceduc-web/app/(portal)/records/students/page.tsx
diceduc-web/app/(portal)/records/students/new/page.tsx
diceduc-web/app/(portal)/records/students/[studentId]/page.tsx
diceduc-web/app/(portal)/records/structure/page.tsx
diceduc-web/features/people/components/*.tsx
diceduc-web/features/people/api/students.ts
diceduc-web/features/people/schemas/student.ts
diceduc-web/features/academic/components/*.tsx
diceduc-web/features/academic/api/academic.ts
diceduc-web/features/academic/schemas/academic.ts
```

## Testes

- Repositórios in-memory (`diceduc-api/test/repositories/`): `in-memory-students-repository.ts`, `in-memory-guardians-repository.ts`, `in-memory-student-guardians-repository.ts`, `in-memory-teachers-repository.ts`, `in-memory-staff-repository.ts`, `in-memory-school-years-repository.ts`, `in-memory-grade-levels-repository.ts`, `in-memory-class-groups-repository.ts`, `in-memory-subjects-repository.ts`, `in-memory-rooms-repository.ts`.
- Factories (`diceduc-api/test/factories/`): `make-student.ts`, `make-guardian.ts`, `make-teacher.ts`, `make-staff.ts`, `make-school-year.ts`, `make-grade-level.ts`, `make-class-group.ts`, `make-subject.ts`, `make-room.ts` (+ versões `@Injectable()` com persistência Prisma para e2e).
- Unit specs: um `.spec.ts` por use case listado nas tabelas acima, cobrindo caminho feliz + cada erro específico.
- E2E: um `.e2e-spec.ts` por controller listado na seção HTTP.

## Definition of Done

- [ ] Critérios de aceite de `F1-E01-U01` a `F1-E01-U04` no BACKLOG satisfeitos.
- [ ] Todos os use cases e controllers listados implementados com testes unitários e e2e passando.
- [ ] Migração Prisma aplicada sem erro (`prisma migrate dev`).
- [ ] Telas de cadastro de aluno, vínculo de responsável, alteração de situação e estrutura acadêmica funcionando ponta a ponta (formulário → API → persistência → exibição).
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E01` permanece 🟢, adicionar nota se algo ficou fora do escopo original.
