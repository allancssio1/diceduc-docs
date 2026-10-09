# F1-E10 — Identidade e acesso

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E10 — Identidade e acesso |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | `iam` |
| Depende de | F1-E0A (tenancy), F1-E01 (pessoas) |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E10` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Dar a cada pessoa cadastrada uma conta de acesso: login, sessão (access + refresh token), primeiro acesso com senha temporária, troca/redefinição de senha, desativação e revogação de sessões. `F1-E09` (perfis/auditoria) e `F1-E08` (portais) constroem sobre o que esta ficha entrega.

## Histórias cobertas

- `F1-E10-U01` — Login com bloqueio temporário após tentativas malsucedidas.
- `F1-E10-U02` — Secretaria cria acesso de uma pessoa já cadastrada, com senha temporária.
- `F1-E10-U03` — Secretaria redefine senha ou desativa conta, com efeito imediato.
- `F1-E10-U04` — Usuário troca a própria senha e revoga as demais sessões.
- `F1-E10-U05` — Equipe DICEDUC cria/redefine o admin de um group.
- `F1-E10-U06` — Recuperação de senha por e-mail — pronta, desligada por flag no MVP.

## Decisões resolvidas nesta spec

- **Sessão:** access token JWT RS256 de 15 minutos + refresh token opaco (256 bits aleatórios, nunca um JWT) de 7 dias, guardado no banco só como hash (`bcryptjs`). Sem `tokenVersion` — revogar é apagar/marcar `revokedAt` nos `RefreshToken`s do usuário; o pior caso é um access token já emitido, que ainda vale até 15 min.
- **Rotação de refresh:** cada `POST /sessions/refresh` gera um refresh novo e marca o usado como `replacedAt`. Reuso de um refresh já substituído (fora de uma tolerância de 30s, que cobre duas abas renovando ao mesmo tempo) é tratado como indício de roubo: revoga todos os refresh tokens daquele usuário.
- **Identificador de login:** `email` único **por group**, não global — a mesma pessoa (ex.: um professor que leciona em duas secretarias) tem uma conta e uma senha independentes em cada group. O login recebe `groupSlug`/`schoolSlug` (da URL) + `email`/`password`; resolve o group pelo slug antes de procurar o usuário.
- **Uma escola por sessão:** o login exige que o usuário tenha acesso à escola da URL (`ListMyAccessibleSchoolsUseCase`/`UserRolesRepository.canAccessSchool`, de `F1-E09`) e emite o access token já com aquele `schoolId`. Trocar de escola é um novo login (mecanismo no `diceduc-web`, `arquitetura-frontend.md` §6) — esta ficha só garante que o backend recusa (`SchoolAccessDeniedError`) quem tenta entrar numa escola sem vínculo.
- **Senha temporária:** gerada aleatoriamente (ex.: 12 caracteres, alfanumérico), mostrada **uma única vez** para quem criou a conta (a API nunca a devolve de novo). Expira em 72h; se expirar sem uso, a secretaria gera outra via `ResetUserPasswordUseCase`. `mustChangePassword = true` até o primeiro acesso ser concluído.
- **Recuperação por e-mail:** implementada e testada como qualquer outra funcionalidade, mas atrás da flag de env `PASSWORD_RESET_ENABLED` (default `false`). Com a flag desligada, as rotas HTTP respondem `404` e o `diceduc-web` nem renderiza as telas/link — ativar no futuro é só mudar a flag e configurar SMTP, sem reabrir código.
- **Anti-enumeração:** toda resposta que poderia revelar se um e-mail existe (login, pedido de recuperação) usa a mesma mensagem genérica de erro e o mesmo tempo de resposta (comparação com um hash fictício quando o usuário não existe).

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md). Específico deste épico:

- `User`, `RefreshToken`, `PasswordResetToken` carregam `groupId` e são filtrados pela mesma Prisma extension de qualquer model de negócio — não existe método "unscoped" nos repositórios.
- As rotas **sem sessão ainda** (`POST /sessions`, `/sessions/first-access`, `/sessions/refresh`, `/password-resets*`) resolvem `groupId` a partir do `groupSlug` da URL/body e rodam dentro de um `GroupContext` **preliminar** (só `groupId`, sem `schoolId`/`allowedStudentIds` — populado do mesmo jeito que `GetTenantBrandingUseCase`, de `F1-E0A`), o suficiente para o repositório localizar o `User` sem vazar dado de outro group.
- `Authenticate` valida o acesso à escola (`schoolSlug` → `schoolId`) **depois** de confirmar a senha — nunca antes, para não revelar por timing se a escola existe/a conta tem acesso a ela antes de validar a credencial.
- `CreateUserAccountUseCase`/`ResetUserPasswordUseCase`/`SetUserStatusUseCase` rodam dentro do `GroupContext` normal (usuário já logado como secretaria/admin) — nunca recebem `groupId` do corpo da requisição.

## Modelo de domínio (`enterprise`)

### Contexto `iam` — `diceduc-api/src/domain/iam/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `User` | `AggregateRoot` | `user.ts` | `groupId`, `name`, `email` (único **por group**), `passwordHash`, `status: UserStatus`, `mustChangePassword`, `temporaryPasswordExpiresAt?`, `failedLoginAttempts`, `lockedUntil?`, `lastLoginAt?`, `person?: { type: PersonType; id: string }`, `createdAt`, `updatedAt` |
| `RefreshToken` | `Entity` | `refresh-token.ts` | `userId`, `tokenHash`, `expiresAt`, `revokedAt?`, `replacedAt?` |
| `PasswordResetToken` | `Entity` | `password-reset-token.ts` | `userId`, `tokenHash`, `expiresAt` (1h), `usedAt?` |

Value objects — `entities/valueObjects/`:
- `user-status.ts` — `UserStatus = 'ACTIVE' | 'DISABLED'`.
- `person-type.ts` — `PersonType = 'GUARDIAN' | 'STUDENT' | 'TEACHER' | 'STAFF'` (referencia a pessoa em `people`, `F1-E01`, sem FK de banco cross-schema — validado na aplicação).

Invariantes:
- No máximo um `User` por `(groupId, person.type, person.id)` — uma pessoa não tem duas contas no mesmo group.
- `failedLoginAttempts >= 5` seta `lockedUntil = agora + 15min`; qualquer login bem-sucedido zera `failedLoginAttempts`/`lockedUntil`.
- `RefreshToken.replacedAt` setado só pela rotação (`RefreshSession`); `revokedAt` setado por logout, troca/reset de senha ou desativação.
- `PasswordResetToken` só é aceito se `usedAt` é nulo e `expiresAt > agora`; usar marca `usedAt` (nunca é reutilizável mesmo dentro da validade).

## Use cases (`application`)

### `diceduc-api/src/domain/iam/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `AuthenticateUseCase` | `authenticate.ts` | `groupSlug`, `schoolSlug`, `email`, `password` | `Either<InvalidCredentialsError \| PasswordChangeRequiredError \| SchoolAccessDeniedError \| AccountLockedError, { accessToken: string; refreshToken: string }>` | `invalid-credentials-error.ts`, `password-change-required-error.ts`, `school-access-denied-error.ts`, `account-locked-error.ts` |
| `CompleteFirstAccessUseCase` | `complete-first-access.ts` | `groupSlug`, `schoolSlug`, `email`, `temporaryPassword`, `newPassword` | `Either<InvalidCredentialsError \| TemporaryPasswordExpiredError \| SchoolAccessDeniedError, { accessToken: string; refreshToken: string }>` | `temporary-password-expired-error.ts` |
| `RefreshSessionUseCase` | `refresh-session.ts` | `groupSlug`, `refreshToken` | `Either<InvalidRefreshTokenError, { accessToken: string; refreshToken: string }>` | `invalid-refresh-token-error.ts` (cobre expirado, revogado e reuso — mesma mensagem genérica) |
| `LogoutUseCase` | `logout.ts` | `refreshToken` | `Either<never, void>` | — (idempotente: token já revogado/inexistente não é erro) |
| `RevokeMySessionsUseCase` | `revoke-my-sessions.ts` | `userId` (de `@CurrentUser()`) | `Either<never, void>` | — |
| `GetMeUseCase` | `get-me.ts` | `userId` (de `@CurrentUser()`) | `Either<ResourceNotFoundError, { user: User; roles: ...; accessibleSchools: ... }>` | reusa dados de `F1-E09` (papéis) quando existir; nesta ficha, sem `F1-E09`, devolve só os dados do `User` |
| `ChangePasswordUseCase` | `change-password.ts` | `userId`, `currentPassword`, `newPassword` | `Either<InvalidCredentialsError, { accessToken: string; refreshToken: string }>` | reusa `invalid-credentials-error.ts` |
| `CreateUserAccountUseCase` | `create-user-account.ts` | `personType`, `personId`, `email`, `name` | `Either<ResourceNotFoundError \| EmailAlreadyInUseError \| PersonAlreadyHasAccountError, { user: User; temporaryPassword: string }>` | `email-already-in-use-error.ts`, `person-already-has-account-error.ts` |
| `ListUsersUseCase` | `list-users.ts` | `status?` | `Either<never, { users: User[] }>` | — |
| `ResetUserPasswordUseCase` | `reset-user-password.ts` | `userId` | `Either<ResourceNotFoundError, { temporaryPassword: string }>` | — |
| `SetUserStatusUseCase` | `set-user-status.ts` | `userId`, `status` | `Either<ResourceNotFoundError, { user: User }>` | — |
| `ProvisionGroupAdminUseCase` | `provision-group-admin.ts` | `groupId`, `email`, `name` | `Either<ResourceNotFoundError, { user: User; temporaryPassword: string }>` | reusa `email-already-in-use-error.ts` (redefine se já existir) |
| `RequestPasswordResetUseCase` (atrás da flag) | `request-password-reset.ts` | `groupSlug`, `schoolSlug`, `email` | `Either<never, void>` | — (sempre `right`, mesma resposta exista ou não o e-mail) |
| `ResetPasswordWithTokenUseCase` (atrás da flag) | `reset-password-with-token.ts` | `token`, `newPassword` | `Either<InvalidResetTokenError, void>` | `invalid-reset-token-error.ts` (cobre inexistente/expirado/usado) |

Ports (`application/repositories/`): `UsersRepository` (inclui `findByEmail(groupId, email)`), `RefreshTokensRepository`, `PasswordResetTokensRepository` — todos `abstract class`, todos operando dentro do `GroupContext` (preliminar ou completo conforme a rota).
Ports reusados de `arquitetura-ignite.md` §4: `HashGenerator`/`HashComparator` (senha e tokens), `Encrypter` (não usado aqui — token opaco é gerado com `crypto.randomBytes`, não criptografado).
Port de e-mail (`core/mail/`): `Mailer` — `abstract send(to: string, subject: string, body: string): Promise<void>` (reutilizável por `F2-E02`).

Regras de negócio principais:
- `AuthenticateUseCase`: busca `User` por `(groupId, email)`; se não existe, compara a senha informada com um hash fictício antes de retornar `InvalidCredentialsError` (iguala o tempo de resposta). Se existe: checa `lockedUntil`, compara senha (`HashComparator`), em caso de falha incrementa `failedLoginAttempts` e retorna `InvalidCredentialsError`; em caso de acerto, zera o contador e prossegue. Recusa se `Group.status = 'SUSPENDED'` (checagem já existente de `F1-E0A`). Se `mustChangePassword`, retorna `PasswordChangeRequiredError` (sem emitir token). Resolve `schoolId` do `schoolSlug` e checa `canAccessSchool` (`F1-E09`; nesta ficha, sem `F1-E09`, aceitar qualquer escola do group como stub, documentado); se não tem acesso, `SchoolAccessDeniedError`. Emite `accessToken` (`{ sub, groupId, schoolId }`) e `refreshToken` novo.
- `CompleteFirstAccessUseCase`: mesmas checagens de `Authenticate`, mas contra `passwordHash` temporário; se `temporaryPasswordExpiresAt < agora`, `TemporaryPasswordExpiredError`; ao concluir, seta o novo `passwordHash`, `mustChangePassword = false`, limpa `temporaryPasswordExpiresAt` e emite sessão.
- `RefreshSessionUseCase`: busca `RefreshToken` pelo hash; se não existe, expirado, revogado ou **já substituído há mais de 30s** (reuso fora da tolerância), `InvalidRefreshTokenError` — no caso de reuso, revoga todos os refresh tokens daquele usuário antes de retornar o erro. Caso válido: marca `replacedAt = agora`, cria um novo `RefreshToken`, emite `accessToken` novo mantendo o mesmo `schoolId` da sessão.
- `ChangePasswordUseCase`/`ResetUserPasswordUseCase`/`SetUserStatusUseCase` (ao desativar): revogam todos os `RefreshToken`s do usuário (`revokedAt = agora`).
- `CreateUserAccountUseCase`: valida que `personId` existe no repositório de `people` correspondente a `personType` (ex.: `GuardiansRepository.findById`); valida e-mail único no group e que a pessoa ainda não tem conta; gera senha temporária, devolve em texto puro **só nesta resposta** (o `passwordHash` já vai persistido).
- `ProvisionGroupAdminUseCase`: usada pela rota da plataforma (ver HTTP); se já existir um `User` com aquele e-mail no group, redefine a senha dele em vez de falhar (idempotente para reprocessar onboarding).
- `RequestPasswordResetUseCase`/`ResetPasswordWithTokenUseCase`: só chamadas quando `PASSWORD_RESET_ENABLED=true` (checado no controller, ver HTTP); a primeira sempre responde `right(void)` e só envia e-mail de fato (via `Mailer`) se o `User` existir e estiver `ACTIVE`; a segunda revoga todos os refresh tokens ao concluir, igual a uma troca de senha.

## Módulo de e-mail (`core/mail`, `infra/mail`)

- `core/mail/mailer.ts` — port `Mailer` (ver acima).
- `infra/mail/console-mailer.ts` — implementação de dev/teste: só loga o destinatário/assunto/link no console (nunca envia de verdade). Binding padrão quando `PASSWORD_RESET_ENABLED=false` ou em ambiente de teste.
- `infra/mail/smtp-mailer.ts` — implementação real via `nodemailer`, configurável para qualquer provedor SMTP (Gmail, SES, SendGrid, etc. via env). Só é bindada quando `PASSWORD_RESET_ENABLED=true`.
- Env (`infra/env/env.ts`, Zod): `PASSWORD_RESET_ENABLED` (boolean, default `false`), `WEB_URL`, `SMTP_HOST`/`SMTP_PORT`/`SMTP_USER`/`SMTP_PASS`/`MAIL_FROM` — os quatro últimos exigidos pelo schema **só quando** `PASSWORD_RESET_ENABLED=true` (validação condicional no `.superRefine`).

## Persistência (`infra/database/prisma`)

- `diceduc-api/prisma/schema.prisma`: novos models `User` (índice único composto `groupId + email`), `RefreshToken`, `PasswordResetToken`; enums `UserStatus`, `PersonType`.
- Repositórios: `prisma-users-repository.ts`, `prisma-refresh-tokens-repository.ts`, `prisma-password-reset-tokens-repository.ts` em `infra/database/prisma/repositories/`.
- Mappers: `user-mapper.ts`, `refresh-token-mapper.ts`, `password-reset-token-mapper.ts`.

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Presenter | Contexto |
| --- | --- | --- | --- | --- |
| `POST /sessions` | `authenticate.controller.ts` | `{ groupSlug, schoolSlug, email, password }` | `{ accessToken }` (refresh vai só em cookie — ver nota) | `@Public()` + `@Throttle` restritivo |
| `POST /sessions/first-access` | `complete-first-access.controller.ts` | `{ groupSlug, schoolSlug, email, temporaryPassword, newPassword }` | idem | `@Public()` + `@Throttle` |
| `POST /sessions/refresh` | `refresh-session.controller.ts` | `{ groupSlug, refreshToken }` | idem | `@Public()` + `@Throttle` |
| `POST /sessions/logout` | `logout.controller.ts` | `{ refreshToken }` | — (204) | autenticado |
| `GET /me` | `get-me.controller.ts` | — | `me-presenter.ts` | autenticado |
| `PATCH /me/password` | `change-password.controller.ts` | `{ currentPassword, newPassword }` | idem sessão | autenticado |
| `POST /me/sessions/revoke` | `revoke-my-sessions.controller.ts` | — | — (204) | autenticado |
| `POST /users` | `create-user-account.controller.ts` | `{ personType, personId, email, name }` | `user-presenter.ts` + `temporaryPassword` (só nesta resposta) | secretaria/admin |
| `GET /users` | `list-users.controller.ts` | query `{ status? }` | `user-presenter.ts` (lista) | secretaria/admin |
| `POST /users/:userId/password-reset` | `reset-user-password.controller.ts` | — (param) | `{ temporaryPassword }` | secretaria/admin |
| `PATCH /users/:userId/status` | `set-user-status.controller.ts` | `{ status }` | `user-presenter.ts` | secretaria/admin |
| `POST /groups/:groupId/admins` | `provision-group-admin.controller.ts` | `{ email, name }` | `user-presenter.ts` + `temporaryPassword` | `PlatformAdminGuard` (`F1-E0A`) |
| `POST /password-resets` (atrás da flag) | `request-password-reset.controller.ts` | `{ groupSlug, schoolSlug, email }` | — (204) | `@Public()` + `@Throttle`; `404` se `PASSWORD_RESET_ENABLED=false` |
| `POST /password-resets/confirm` (atrás da flag) | `reset-password-with-token.controller.ts` | `{ token, newPassword }` | — (204) | idem |

**Nota sobre onde fica o refresh token:** o corpo da resposta HTTP devolve `accessToken` e (só nas 3 rotas de emissão de sessão) `refreshToken`; é o **`diceduc-web`** quem grava os dois em cookies httpOnly — a API em si não seta cookie (ela é chamada só pelo servidor do Next, nunca pelo browser, ver `arquitetura-frontend.md` §6). O contrato aqui é JSON puro, como qualquer outra rota.

Perfis exigidos: `POST /users`, `GET /users`, `POST /users/:userId/password-reset`, `PATCH /users/:userId/status` → `secretaria`/`admin` (guard fino vem de `F1-E09`; por ora `JwtAuthGuard` padrão). Demais rotas autenticadas exigem só sessão válida (qualquer papel).

## Frontend (`diceduc-web`)

Tudo sob `app/[grupo]/[escola]/`:

- `app/[grupo]/[escola]/(public)/login/page.tsx` (editar) — mostra nome/logo/cor do branding (`F1-E0A`); Server Action envia `groupSlug`/`schoolSlug` + credenciais a `POST /sessions` e grava `diceduc_access`/`diceduc_refresh` (`Path=/{grupo}`); texto "Esqueceu a senha? Procure a secretaria" — vira link para `/esqueci-senha` quando `PASSWORD_RESET_ENABLED=true` (env do próprio `diceduc-web`).
- `app/[grupo]/[escola]/(public)/primeiro-acesso/page.tsx` — formulário de `CompleteFirstAccessUseCase`.
- `app/[grupo]/[escola]/(public)/esqueci-senha/page.tsx` e `.../redefinir-senha/page.tsx` — chamam `notFound()` logo no topo quando `PASSWORD_RESET_ENABLED=false`.
- `app/[grupo]/[escola]/(portal)/minha-conta/page.tsx` — trocar senha (`ChangePasswordUseCase`), botão "sair de todos os dispositivos" (`RevokeMySessionsUseCase`), lista das outras escolas do group a que o usuário tem acesso (links, com aviso de que abrir exige novo login).
- `app/[grupo]/[escola]/(portal)/admin/usuarios/page.tsx` — lista de usuários, criar acesso (mostra a senha temporária **uma vez**, com botão de copiar), redefinir senha, ativar/desativar.
- `shared/auth/session.ts` (`server-only`) — `getSession()` (`cache()` do React, chama `GET /me`), helpers de cookie.
- `shared/api/client.ts` (editar) — injeta `Bearer` do `diceduc_access`; em `401`, propaga para o `proxy.ts` tratar (apaga cookies, redireciona).
- `proxy.ts` (editar) — renova o access quando perto de expirar (`POST /sessions/refresh`), e desloga se o `schoolId` da sessão não bate com a escola da URL (ver `arquitetura-frontend.md` §6).
- `features/iam/api/iam.ts` (`login`, `completeFirstAccess`, `changePassword`, `revokeMySessions`, `createUserAccount`, `listUsers`, `resetUserPassword`, `setUserStatus`, `requestPasswordReset`, `resetPasswordWithToken`), `features/iam/schemas/iam.ts`.
- `.env.example` do `diceduc-web` ganha `PASSWORD_RESET_ENABLED`.

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/iam/enterprise/entities/user.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/refresh-token.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/password-reset-token.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/valueObjects/user-status.ts`
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/valueObjects/person-type.ts`
- [ ] `diceduc-api/src/domain/iam/application/useCases/authenticate.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/complete-first-access.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/refresh-session.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/logout.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/revoke-my-sessions.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/get-me.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/change-password.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/create-user-account.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/list-users.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/reset-user-password.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/set-user-status.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/provision-group-admin.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/request-password-reset.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/useCases/reset-password-with-token.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/iam/application/repositories/users-repository.ts`
- [ ] `diceduc-api/src/domain/iam/application/repositories/refresh-tokens-repository.ts`
- [ ] `diceduc-api/src/domain/iam/application/repositories/password-reset-tokens-repository.ts`
- [ ] `diceduc-api/src/core/mail/mailer.ts`
- [ ] `diceduc-api/src/infra/mail/console-mailer.ts`
- [ ] `diceduc-api/src/infra/mail/smtp-mailer.ts`
- [ ] `diceduc-api/src/infra/env/env.ts` (editar — `PASSWORD_RESET_ENABLED` + variáveis de SMTP condicionais)
- [ ] `diceduc-api/src/infra/auth/jwt.strategy.ts` (editar — payload `{ sub, groupId, schoolId }`)
- [ ] `diceduc-api/prisma/schema.prisma` (editar)
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-users-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-refresh-tokens-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-password-reset-tokens-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/user-mapper.ts`, `refresh-token-mapper.ts`, `password-reset-token-mapper.ts`
- [ ] `diceduc-api/src/infra/http/controllers/*.ts` (13 controllers, ver seção HTTP)
- [ ] `diceduc-api/src/infra/http/presenters/user-presenter.ts`, `me-presenter.ts`
- [ ] `diceduc-web/app/[grupo]/[escola]/(public)/login/page.tsx` (editar)
- [ ] `diceduc-web/app/[grupo]/[escola]/(public)/primeiro-acesso/page.tsx`
- [ ] `diceduc-web/app/[grupo]/[escola]/(public)/esqueci-senha/page.tsx`
- [ ] `diceduc-web/app/[grupo]/[escola]/(public)/redefinir-senha/page.tsx`
- [ ] `diceduc-web/app/[grupo]/[escola]/(portal)/minha-conta/page.tsx`
- [ ] `diceduc-web/app/[grupo]/[escola]/(portal)/admin/usuarios/page.tsx`
- [ ] `diceduc-web/shared/auth/session.ts`
- [ ] `diceduc-web/shared/api/client.ts` (editar)
- [ ] `diceduc-web/proxy.ts` (editar)
- [ ] `diceduc-web/features/iam/api/iam.ts`
- [ ] `diceduc-web/features/iam/schemas/iam.ts`
- [ ] `diceduc-web/.env.example` (editar)

## Testes

- Repositório in-memory: `in-memory-users-repository.ts`, `in-memory-refresh-tokens-repository.ts`, `in-memory-password-reset-tokens-repository.ts`.
- Factory: `make-user.ts` (+ `UserFactory.makePrismaUser()` para e2e — nota: a partir desta ficha, os e2e de **outros** contextos que hoje assinam um JWT direto passam a criar um `User` real via factory antes de logar).
- Fake: `test/mail/fake-mailer.ts` (guarda os e-mails "enviados" em memória para asserção nos specs).
- Unit specs: erro genérico idêntico para e-mail inexistente e senha errada (`AuthenticateUseCase`); bloqueio após 5 falhas e desbloqueio após `lockedUntil`; senha temporária expirada (`CompleteFirstAccessUseCase`); rotação de refresh e detecção de reuso (`RefreshSessionUseCase`); troca/reset de senha revoga sessões; `CreateUserAccountUseCase` rejeita pessoa inexistente e e-mail duplicado; `ResetPasswordWithTokenUseCase` rejeita token expirado/usado; `RequestPasswordResetUseCase` responde igual para e-mail existente e inexistente; mesmo e-mail em dois groups diferentes gera contas independentes (senha, bloqueio e reset de uma não afetam a outra).
- E2E: login válido → `200`; senha errada/e-mail inexistente → `401` com a mesma mensagem; `mustChangePassword` → corpo com `PASSWORD_CHANGE_REQUIRED`; login numa escola sem acesso → `SchoolAccessDeniedError`; slug de group inexistente → mesmo erro genérico do login normal (não revela que o group não existe); refresh rotaciona e o token antigo não serve mais; usuário `DISABLED` não consegue renovar; 6ª tentativa de login em 1 minuto → `429`; rotas de `/password-resets*` → `404` com a flag desligada, fluxo completo (pedir → e-mail no `fake-mailer`/`console-mailer` → confirmar → login com a senha nova) com a flag ligada.
- Frontend: Playwright cobrindo login, primeiro acesso e renovação transparente de sessão (usuário navega por mais de 15 min sem perceber logout).

## Definition of Done

- [ ] Critérios de aceite de `F1-E10-U01` a `F1-E10-U06` no BACKLOG satisfeitos (U06 satisfeito com a flag testada nos dois estados).
- [ ] Todos os use cases e controllers implementados com testes unitários e e2e passando.
- [ ] Migração Prisma aplicada sem erro.
- [ ] Login, primeiro acesso, troca/redefinição de senha, desativação e provisionamento de admin de group funcionando ponta a ponta.
- [ ] `jwt.strategy.ts` validando o novo formato de payload (`groupId` + `schoolId`) sem quebrar o que `F1-E0A` já validava.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E10` para 🟢 quando implementado.
