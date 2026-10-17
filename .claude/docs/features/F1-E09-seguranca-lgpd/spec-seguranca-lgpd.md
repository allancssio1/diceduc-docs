# F1-E09 — Segurança, perfis de acesso e LGPD

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E09 — Segurança, perfis de acesso e LGPD |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | M |
| Bounded context(s) | `iam` |
| Depende de | F1-E01, F1-E10 (transversal) |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E09` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Definir perfis de acesso com permissões por módulo, restringir dados de aluno a quem tem vínculo com ele e manter trilha de auditoria de ações sensíveis.

> Direitos do titular (exportação de dados, canal do encarregado) ficam em [`spec-direitos-titular.md`](spec-direitos-titular.md), que depende desta ficha.

## Histórias cobertas

- `F1-E09-U01` — Perfis de acesso configuráveis (secretaria, coordenação, professor, financeiro, direção, responsável, aluno, admin).
- `F1-E09-U02` — Trilha de auditoria de ações sensíveis (notas, dados pessoais, baixas financeiras).
- `F1-E09-U03` — Registro/revogação de consentimento LGPD por responsável.

## Decisão resolvida nesta spec

O épico deixava em aberto se o guard fino de perfil seria enforced desde o início ou "fechado" só agora. Resolução: **fechado agora, com retrofit explícito**. Todos os controllers de `F1-E01` a `F1-E08` foram especificados usando só `JwtAuthGuard` padrão (autenticado, sem checagem de papel) e anotando qual perfil *deveria* ser exigido. Esta ficha:
1. Implementa o guard fino (`PermissionsGuard` + decorator `@RequirePermission(module, action)`).
2. Traz, como parte do checklist desta ficha (não uma spec separada), a tarefa de **adicionar o decorator em cada controller já implementado**, usando o perfil já documentado na seção HTTP de cada spec anterior como fonte de verdade — sem reabrir decisão de negócio, só aplicando o guard.

Da mesma forma, a auditoria (`F1-E09-U02`) é feita retroativamente: esta ficha expõe um port `AuditLogger` (`abstract class`, `core/audit/audit-logger.ts`) e o adiciona nos use cases sensíveis já existentes (`UpdateStudentStatusUseCase` de `F1-E01`, `RecordAttendanceUseCase`/`JustifyAbsenceUseCase` de `F1-E03`, `LaunchGradesUseCase`/`PublishAssessmentUseCase` de `F1-E04`, `RegisterPaymentUseCase` de `F1-E06`, e `AssignRoleToUserUseCase`/`ConfigureRolePermissionsUseCase`/`CreateUserAccountUseCase`/`ResetUserPasswordUseCase`/`SetUserStatusUseCase` desta própria ficha e de `F1-E10`) via injeção — 1 linha de chamada a `auditLogger.record(...)` no fim de cada `execute()`, sem mudar a assinatura pública do use case.

### Mascaramento de dados sensíveis na auditoria

`AuditLog.previousValue`/`newValue` guardam o antes/depois de qualquer alteração — inclusive de campos que são eles próprios dado pessoal sensível (`cpf`, `rg`, `birthCertificateNumber`, `justification` de falta). Gravar o valor em texto puro duplicaria dado sensível fora do registro de origem. `core/audit/sensitive-fields.ts` exporta `maskSensitiveFields(value: unknown): unknown`, que substitui esses campos por `"[redacted]"` recursivamente; a implementação de `AuditLogger` (`AuditLogsRepository`) chama essa função antes de persistir — nenhum use case precisa saber disso, é uma responsabilidade só do `AuditLogger`.

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md). Esta ficha é onde o **`allowedStudentIds` do `GroupContext` realmente é resolvido**, não só consumido — e onde os papéis por escola são atribuídos:

- `Role`, `AuditLog`, `ConsentRecord` carregam `groupId` — perfis, auditoria e consentimento são configurados por group, não globalmente. `RoleName` é único **por group** (dois groups podem ter, cada um, seu próprio `'COORDINATION'` com permissões diferentes).
- **`UserRole` carrega `schoolId?: string`** — decide o escopo de cada atribuição de papel: `null`/ausente = papel **group-wide** (ex.: mantenedora, admin) — na prática, "pode logar em qualquer escola do group" (uma sessão continua sendo de uma escola só, ver `arquitetura-ignite.md` §11); preenchido = papel **school-scoped** (ex.: diretor, secretário de uma unidade específica), só pode logar naquela escola. Um mesmo usuário pode ter múltiplos `UserRole`, alguns group-wide e outros restritos a escolas diferentes.
- **Papéis efetivos de uma sessão** = os `UserRole` group-wide do usuário **mais** os que têm `schoolId` igual ao da sessão corrente. Um usuário que é diretor na escola A e professor na escola B só tem poder de diretor enquanto a sessão for da escola A — testado explicitamente (ver Testes).
- `GroupScopeGuard` (implementado em `F1-E0A`, com um stub `AllowAllStudentsProvider` até esta ficha existir) passa a usar `UserRolesRepository` real para resolver `allowedStudentIds`: papel de equipe (qualquer `UserRole` efetivo que não seja `GUARDIAN`/`STUDENT`) → `null` (sem restrição); só `GUARDIAN`/`STUDENT` → `GUARDIAN` resolve os filhos via `StudentGuardian` (`F1-E01`), `STUDENT` resolve o próprio `Student` a partir de `User.person` (`F1-E10`); sem nenhum vínculo → `[]` (nenhum aluno). **Trocar o binding do provider no módulo por esta implementação real é parte do checklist desta ficha.**
- `AuditLogger.record(...)` grava `groupId` e, quando aplicável, `schoolId` da entidade auditada.

## Modelo de domínio (`enterprise`)

### Contexto `iam` — `diceduc-api/src/domain/iam/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `Role` | `Entity` | `role.ts` | `name: RoleName`, `permissions: Permission[]` |
| `UserRole` | `Entity` | `user-role.ts` | `userId`, `roleId`, `schoolId?: string` (N:N — um usuário pode ter múltiplos papéis; `schoolId` ausente = papel group-wide, preenchido = restrito àquela escola — ver seção "Multi-tenancy") |
| `AuditLog` | `Entity` | `audit-log.ts` | `userId`, `module`, `action: AuditAction`, `entityId`, `previousValue: unknown`, `newValue: unknown`, `occurredAt` |
| `ConsentRecord` | `Entity` | `consent-record.ts` | `studentId`, `guardianId`, `status: ConsentStatus`, `termVersion: string`, `occurredAt` |

Value objects — `entities/valueObjects/`:
- `role-name.ts` — `RoleName = 'SECRETARY' | 'COORDINATION' | 'TEACHER' | 'FINANCE' | 'PRINCIPAL' | 'GUARDIAN' | 'STUDENT' | 'ADMIN'` (`ADMIN` é sempre group-wide — administrador do cliente dentro do próprio group, diferente do administrador da plataforma/`PlatformAdminGuard` de `F1-E0A`, que é da equipe DICEDUC).
- `permission.ts` — `Permission` (`module: string`, `actions: ('VIEW' | 'EDIT')[]`).
- `audit-action.ts` — `AuditAction = 'CREATE' | 'UPDATE' | 'DELETE'`.
- `consent-status.ts` — `ConsentStatus = 'GRANTED' | 'REVOKED'`.

Invariantes:
- Um `userId` pode ter mais de um `UserRole` (ex.: coordenador que também é professor) — não há unicidade de papel por usuário.
- `ConsentRecord` é sempre criado (append-only), nunca atualizado: revogar consentimento cria um novo registro com `status = 'REVOKED'`, preservando o histórico completo.
- Não existe entidade `ConsentTerm` separada nesta ficha — `termVersion` é só uma string livre gerenciada fora do sistema (ex.: `"2026-v1"`); versionar o texto do termo em si é responsabilidade de conteúdo/jurídico, não deste módulo.

## Use cases (`application`)

### `diceduc-api/src/domain/iam/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `ConfigureRolePermissionsUseCase` | `configure-role-permissions.ts` | `roleName`, `permissions` | `Either<NotAllowedError, { role: Role }>` | só `ADMIN` pode chamar (ver "Anti-escalada de privilégio") |
| `AssignRoleToUserUseCase` | `assign-role-to-user.ts` | `userId`, `roleName`, `schoolId?` | `Either<ResourceNotFoundError \| NotAllowedError, { userRole: UserRole }>` | atribuir `ADMIN`/`PRINCIPAL` exige que quem chama já seja `ADMIN`; ninguém atribui papel a si mesmo |
| `ProvisionGroupRolesUseCase` | `provision-group-roles.ts` | `groupId` | `Either<never, { roles: Role[] }>` | — (idempotente: cria os 8 `RoleName` com permissões padrão só se ainda não existirem naquele group) |
| `GetAuditTrailUseCase` | `get-audit-trail.ts` | `userId?`, `module?`, `startDate?`, `endDate?` | `Either<never, { logs: AuditLog[] }>` | — |
| `RegisterConsentUseCase` | `register-consent.ts` | `studentId`, `guardianId`, `termVersion` | `Either<ResourceNotFoundError, { consent: ConsentRecord }>` | — |
| `RevokeConsentUseCase` | `revoke-consent.ts` | `studentId`, `guardianId` | `Either<ResourceNotFoundError \| ConsentNotFoundError, { consent: ConsentRecord }>` | `consent-not-found-error.ts` |
| `GetConsentStatusUseCase` | `get-consent-status.ts` | `studentId` | `Either<ResourceNotFoundError, { records: ConsentRecord[]; currentStatus: ConsentStatus }>` | — |
| `ListMyAccessibleSchoolsUseCase` | `list-my-accessible-schools.ts` | `userId` (de `@CurrentUser()`) | `Either<never, { schools: School[] }>` | — (papel group-wide → todas as escolas do group; senão, só as dos seus `UserRole`) |

Ports (`application/repositories/`): `RolesRepository`, `UserRolesRepository` (inclui `canAccessSchool(userId, schoolId): Promise<boolean>`, usado por `F1-E10` no login, e `findAccessibleSchoolIds(userId): Promise<string[] | 'ALL'>`, usado por `ListMyAccessibleSchoolsUseCase`), `AuditLogsRepository`, `ConsentRecordsRepository`.
Port de infraestrutura transversal (`core/audit/`): `AuditLogger` — `abstract record(entry: { userId, module, action, entityId, previousValue, newValue }): Promise<void>`. Implementado por `AuditLogsRepository` internamente (o port fica em `core` para poder ser injetado em use cases de qualquer contexto sem criar dependência de `iam/application` a partir de `people`/`assessment`/`finance`). A implementação usa `core/audit/sensitive-fields.ts` (ver "Mascaramento de dados sensíveis") antes de persistir `previousValue`/`newValue`.

### Anti-escalada de privilégio

- Só um ator com papel `ADMIN` efetivo pode: atribuir `ADMIN`/`PRINCIPAL` (`AssignRoleToUserUseCase`), configurar permissões de qualquer papel (`ConfigureRolePermissionsUseCase`), redefinir senha (`ResetUserPasswordUseCase`, `F1-E10`) ou desativar (`SetUserStatusUseCase`, `F1-E10`) um usuário que também é `ADMIN`.
- Ninguém altera os próprios papéis (`AssignRoleToUserUseCase`) nem o próprio status (`SetUserStatusUseCase`) — evita autopromoção e auto-bloqueio acidental.
- `ProvisionGroupAdminUseCase` (`F1-E10`) é a única forma de criar o **primeiro** `ADMIN` de um group (atrás do `PlatformAdminGuard`, da equipe DICEDUC) — ele chama `ProvisionGroupRolesUseCase` e atribui `ADMIN` group-wide ao usuário criado. Para groups já existentes antes desta ficha, um script único de migração roda `ProvisionGroupRolesUseCase` para cada um.

## Infra transversal — guard de permissões

- `diceduc-api/src/infra/auth/permissions.guard.ts` — `PermissionsGuard`: lê `sub`/`groupId`/`schoolId` de `@CurrentUser()` (o JWT **não** carrega `role` — papéis são sempre lidos do banco, para revogação ser imediata, ver `arquitetura-ignite.md` §11), busca os `UserRole` **efetivos da sessão** (group-wide + os da `schoolId` corrente, ver "Multi-tenancy" acima) + `Role.permissions`, compara com o metadata setado pelo decorator. Reusa a mesma busca de `UserRole` que o `GroupScopeGuard` já faz para montar `allowedStudentIds` — não duplica a query.
- `diceduc-api/src/infra/auth/require-permission.decorator.ts` — `@RequirePermission(module: string, action: 'VIEW' | 'EDIT')`: `SetMetadata` lido pelo `PermissionsGuard`.
- Registrar `PermissionsGuard` como segundo guard global (`APP_GUARD`), depois do `JwtAuthGuard`/`GroupScopeGuard` já existentes — rotas sem `@RequirePermission` continuam exigindo só autenticação (comportamento atual preservado).

## Persistência (`infra/database/prisma`)

- `diceduc-api/prisma/schema.prisma`: novos models `Role`, `UserRole`, `AuditLog`, `ConsentRecord`; enums `RoleName`, `AuditAction`, `ConsentStatus`. `Permission` como campo `Json` em `Role` (lista de `{ module, actions }`, não uma tabela própria — evita join extra para uma lista pequena e de baixa cardinalidade).
- Repositórios: `prisma-roles-repository.ts`, `prisma-user-roles-repository.ts`, `prisma-audit-logs-repository.ts`, `prisma-consent-records-repository.ts` em `infra/database/prisma/repositories/`.
- Mappers: `role-mapper.ts`, `user-role-mapper.ts`, `audit-log-mapper.ts`, `consent-record-mapper.ts`.

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Presenter |
| --- | --- | --- | --- |
| `PUT /roles/:roleName/permissions` | `configure-role-permissions.controller.ts` | `{ permissions }` | `role-presenter.ts` |
| `POST /users/:userId/roles` | `assign-role-to-user.controller.ts` | `{ roleName, schoolId? }` | `user-role-presenter.ts` |
| `GET /me/accessible-schools` | `list-my-accessible-schools.controller.ts` | — | `school-presenter.ts` (lista, reusa de `F1-E0A`) |
| `GET /audit-logs` | `get-audit-trail.controller.ts` | query `{ userId?, module?, startDate?, endDate? }` | `audit-log-presenter.ts` (lista) |
| `POST /students/:studentId/consent` | `register-consent.controller.ts` | `{ guardianId, termVersion }` | `consent-record-presenter.ts` |
| `DELETE /students/:studentId/consent` | `revoke-consent.controller.ts` | `{ guardianId }` | `consent-record-presenter.ts` |
| `GET /students/:studentId/consent` | `get-consent-status.controller.ts` | — (param) | `consent-record-presenter.ts` (lista + `currentStatus`) |

`configure-role-permissions`/`assign-role-to-user` exigem `@RequirePermission('iam', 'EDIT')`, perfil `direção`/`ADMIN` (mais a checagem de "Anti-escalada de privilégio" acima); `get-audit-trail` exige `@RequirePermission('iam', 'VIEW')`, perfil `direção`/`ADMIN`; `list-my-accessible-schools` só exige sessão válida (qualquer papel); `register-consent`/`revoke-consent`/`get-consent-status` são acessíveis também por `responsável` do próprio filho — a restrição vem de graça do `allowedStudentIds` (ver "Multi-tenancy").

### Retrofit nos controllers existentes (checklist desta ficha)

Adicionar `@RequirePermission(<module>, <action>)` em cada controller de `F1-E01` a `F1-E08` e `F1-E10`, usando o mapeamento já anotado na seção "Perfis exigidos"/"HTTP" de cada spec (`PRINCIPAL`/`ADMIN` onde a spec original dizia "administrador"):

| Épico | Módulo (`module`) | Controllers afetados |
| --- | --- | --- |
| F1-E01 | `people`, `academic` | todos os 12 controllers listados em `spec-cadastros-sis.md` |
| F1-E02 | `academic` | todos os 7 controllers de `spec-matricula.md` |
| F1-E03 | `attendance` | os 3 controllers de `spec-frequencia.md` |
| F1-E04 | `assessment` | os 6 controllers de `spec-avaliacoes-notas.md` |
| F1-E05 | `academic` | os 5 controllers de `spec-horarios.md` |
| F1-E06 | `finance` | os 4 controllers de `spec-financeiro.md` |
| F1-E07 | `reporting` | os 6 controllers de `spec-relatorios.md` |
| F1-E08 | `people`, `attendance` | os 2 controllers novos de `spec-portais-web.md` |
| F1-E10 | `iam` | `create-user-account`, `list-users`, `reset-user-password`, `set-user-status` (as rotas de sessão/`me` ficam só com `JwtAuthGuard`, sem `@RequirePermission`) |

**Atenção extra em `record-attendance.controller.ts` (F1-E03):** não é só adicionar `@RequirePermission` — hoje o controller faz `teacherId = recordedBy = user.sub` (usa o id do `User` como se fosse o id do `Teacher`, decisão deliberada enquanto não existe papel no token; ver `spec-frequencia.md`, "Regras de negócio principais", e `epic.md` de `F1-E05`). São entidades diferentes: `User.id` (`F1-E10`) ≠ `Teacher.id` (`F1-E01`). Ao fechar esta ficha, trocar essa resolução para `User.person.id` quando `person.type === 'TEACHER'` (campo já definido em `spec-identidade-acesso.md`) — sem essa correção, quando `F1-E05` trocar o `StubScheduleSlotsRepository` pelo real, `existsActiveFor(teacherId, ...)` vai comparar contra um id que nunca é o `Teacher.id` certo.

## Frontend (`diceduc-web`)

- `app/(portal)/admin/roles/page.tsx` — configuração de permissões por perfil (`F1-E09-U01`).
- `app/(portal)/admin/audit/page.tsx` — trilha de auditoria filtrável (`F1-E09-U02`).
- `app/(public)/consent/page.tsx` — termo de consentimento apresentado no primeiro acesso do responsável (`F1-E09-U03`), integrado ao fluxo de login existente.
- `features/iam/components/RolePermissionsForm.tsx`, `AuditLogTable.tsx`, `ConsentTermDialog.tsx`.
- `features/iam/api/iam.ts` (`configureRolePermissions`, `assignRoleToUser`, `getAuditTrail`, `registerConsent`, `revokeConsent`, `getConsentStatus`), `features/iam/schemas/iam.ts`.
- `shared/auth`: expor `role`/`permissions` do usuário logado para esconder ações na UI conforme permissão (complementa o guard do backend, não substitui).

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/iam/enterprise/entities/role.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/user-role.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/audit-log.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/consent-record.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/valueObjects/role-name.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/valueObjects/permission.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/valueObjects/audit-action.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/valueObjects/consent-status.ts`
- [ ] `diceduc-api/src/domain/iam/application/useCases/configure-role-permissions.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/assign-role-to-user.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/provision-group-roles.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/get-audit-trail.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/register-consent.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/revoke-consent.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/get-consent-status.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/list-my-accessible-schools.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/repositories/roles-repository.ts`
- [ ] `diceduc-api/src/domain/iam/application/repositories/user-roles-repository.ts` (editar se já existir stub — adiciona `canAccessSchool`/`findAccessibleSchoolIds`)
- [ ] `diceduc-api/src/domain/iam/application/repositories/audit-logs-repository.ts`
- [ ] `diceduc-api/src/domain/iam/application/repositories/consent-records-repository.ts`
- [ ] `diceduc-api/src/core/audit/audit-logger.ts`
- [ ] `diceduc-api/src/core/audit/sensitive-fields.ts`
- [ ] `diceduc-api/prisma/schema.prisma` (editar)
- [ ] `diceduc-api/src/infra/database/prisma/repositories/*.ts` (4 repositórios, ver seção Persistência)
- [ ] `diceduc-api/src/infra/database/prisma/mappers/*.ts` (4 mappers, ver seção Persistência)
- [ ] `diceduc-api/src/infra/auth/permissions.guard.ts`
- [ ] `diceduc-api/src/infra/auth/require-permission.decorator.ts`
- [ ] `diceduc-api/src/infra/auth/allow-all-students-provider.ts` → substituir binding pela implementação real (stub existente de `F1-E0A`)
- [ ] `diceduc-api/src/infra/http/controllers/*.ts` (7 controllers novos, ver seção HTTP)
- [ ] `diceduc-api/src/infra/http/presenters/*.ts` (4 presenters, ver seção HTTP)
- [ ] Retrofit: adicionar `@RequirePermission` em todos os controllers de `F1-E01` a `F1-E08` e `F1-E10` (ver tabela acima)
- [ ] Retrofit: injetar `AuditLogger` em `UpdateStudentStatusUseCase`, `RecordAttendanceUseCase`, `JustifyAbsenceUseCase`, `LaunchGradesUseCase`, `PublishAssessmentUseCase`, `RegisterPaymentUseCase`, `AssignRoleToUserUseCase`, `ConfigureRolePermissionsUseCase`, `CreateUserAccountUseCase`, `ResetUserPasswordUseCase`, `SetUserStatusUseCase`
- [ ] Retrofit: trocar o binding de `GroupScopeGuard` de `AllowAllStudentsProvider` (stub de `F1-E0A`) para a implementação real baseada em `UserRolesRepository` (ver seção "Multi-tenancy")
- [ ] Retrofit: em `record-attendance.controller.ts` (`F1-E03`), trocar `teacherId = user.sub` por `User.person.id` (quando `person.type === 'TEACHER'`) — ver nota acima, "Atenção extra em record-attendance.controller.ts"
- [ ] Script único de migração: rodar `ProvisionGroupRolesUseCase` para todo `Group` criado antes desta ficha
- [ ] `diceduc-web/app/(portal)/admin/roles/page.tsx`
- [ ] `diceduc-web/app/(portal)/admin/audit/page.tsx`
- [ ] `diceduc-web/app/(public)/consent/page.tsx`
- [ ] `diceduc-web/features/iam/components/*.tsx`
- [ ] `diceduc-web/features/iam/api/iam.ts`
- [ ] `diceduc-web/features/iam/schemas/iam.ts`

## Testes

- Repositórios in-memory: `in-memory-roles-repository.ts`, `in-memory-user-roles-repository.ts` (com `canAccessSchool`/`findAccessibleSchoolIds`), `in-memory-audit-logs-repository.ts`, `in-memory-consent-records-repository.ts`.
- Factories: `make-role.ts`, `make-user-role.ts`, `make-audit-log.ts`, `make-consent-record.ts`.
- Unit specs: múltiplos papéis por usuário (`AssignRoleToUserUseCase`); não-`ADMIN` tentando atribuir `ADMIN`/`DIRECAO` ou configurar permissões → `NotAllowedError`; usuário tentando alterar o próprio papel/status → `NotAllowedError`; `ProvisionGroupRolesUseCase` é idempotente (rodar duas vezes não duplica `Role`); papéis efetivos da sessão somam group-wide + os da escola corrente, e **não** incluem papel de outra escola (o cenário "diretor na A, professor na B" citado em Multi-tenancy); resolução de `allowedStudentIds` para papel de equipe (`null`), responsável (filhos via `StudentGuardian`) e aluno (o próprio `Student`); filtro de trilha de auditoria por usuário/período/módulo (`GetAuditTrailUseCase`); `maskSensitiveFields` mascara `cpf`/`rg`/`birthCertificateNumber`/`justification` e preserva os demais campos; consentimento append-only e `currentStatus` sempre reflete o registro mais recente (`RegisterConsentUseCase`/`RevokeConsentUseCase`/`GetConsentStatusUseCase`).
- E2E: um `.e2e-spec.ts` por controller novo; uma rota com `@RequirePermission` retorna `403` para um usuário sem a permissão (ex.: `POST /students` sem perfil `secretaria`); responsável pedindo `GET /students/:studentId/report-card` de um aluno que não é seu filho → `404` (prova de que `allowedStudentIds` funciona ponta a ponta); mesmo `404` para `GET /students/:studentId/consent` e `POST /students/:studentId/consent` de um aluno que não é seu filho; responsável em `GET /financial-status?classGroupId=` só recebe títulos dos próprios filhos; secretaria (não-`ADMIN`) tentando `POST /users/:userId/password-reset` contra um usuário `ADMIN` → `403`; usuário com papel de equipe **e** responsável ao mesmo tempo não sofre nenhuma restrição de `allowedStudentIds`.

## Definition of Done

- [ ] Critérios de aceite de `F1-E09-U01` a `F1-E09-U03` no BACKLOG satisfeitos.
- [ ] Todos os use cases e controllers novos implementados com testes unitários e e2e passando.
- [ ] Migração Prisma aplicada sem erro.
- [ ] Retrofit de `@RequirePermission` concluído em todos os controllers de `F1-E01` a `F1-E08` e `F1-E10` (checklist acima).
- [ ] Retrofit de `AuditLogger` concluído em todos os use cases sensíveis listados, com mascaramento de campos sensíveis comprovado por teste.
- [ ] Restrição por `allowedStudentIds` comprovada ponta a ponta (testes de IDOR listados acima).
- [ ] Tela de configuração de perfis, trilha de auditoria e termo de consentimento funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E09` para 🔵 (parcial — falta `spec-direitos-titular.md`) até as duas specs estarem implementadas, depois 🟢.
