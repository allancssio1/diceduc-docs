# F1-E08 — Portais web (pais, alunos, professores)

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E08 — Portais web (pais, alunos, professores) |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | vários (`people`, `academic`, `attendance`, `assessment`, `finance`) — camada frontend |
| Depende de | F1-E01 a F1-E06, F1-E09, F1-E10 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E08` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Entregar o portal web (`diceduc-web`) que expõe, por papel (responsável, professor, aluno), os dados já disponíveis na API dos épicos F1-E01 a F1-E06.

## Histórias cobertas

- `F1-E08-U01` — Portal do responsável: frequência, boletim, boletos, comunicados dos filhos.
- `F1-E08-U02` — Portal do professor: lançamento de notas/frequência/conteúdo das suas turmas.
- `F1-E08-U03` — Portal do aluno: leitura de boletim e frequência.

Comunicados (parte de `F1-E08-U01`) ainda não têm origem de dados própria — nenhum épico de F1 cria uma entidade "Comunicado". **Fora de escopo desta spec**: a tela de comunicados fica com uma lista vazia/placeholder até `F2-E02` (comunicação multicanal) especificar a entidade e os endpoints.

## Gap de backend identificado: listar filhos de um responsável

Nenhum use case de `F1-E01` retorna "todos os alunos vinculados a um responsável" — só o inverso (`GET /students/:studentId` inclui responsáveis). O portal do responsável (`F1-E08-U01`) precisa dessa consulta para o seletor de filhos. Resolução: adicionar um use case pequeno ao contexto `people` como parte desta ficha (não abrir uma spec nova só para isso).

### `ListGuardianStudentsUseCase` — `diceduc-api/src/domain/people/application/useCases/list-guardian-students.ts`

- Entrada: `guardianId` (resolvido a partir de `User.person` do usuário autenticado — `F1-E10` — via `GetMeUseCase`/`@CurrentUser()`; um `User` com `person.type != 'GUARDIAN'` não chama este use case).
- Saída: `Either<ResourceNotFoundError, { students: Student[] }>`.
- Ports usados: `StudentGuardiansRepository`, `StudentsRepository` (ambos já existem, de `F1-E01`).
- Regra: retorna todos os `Student`s vinculados ao `guardianId` via `StudentGuardian`, independente do `role`.
- HTTP: `GET /guardians/me/students` — `list-guardian-students.controller.ts`, presenter `student-presenter.ts` (já existe, reusar).

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md); nota de frontend em [`arquitetura-frontend.md` §10](../../../arquitetura-frontend.md#10-multi-tenancy). Específico deste épico:

- O usuário autenticado (qualquer papel) pertence a um `Group` e sua sessão está presa a uma `School` — login resolve `groupId`/`schoolId` (via `GroupScopeGuard`), expostos ao frontend por `useSession()`/`getSession()` (`F1-E10`).
- `ListGuardianStudentsUseCase` já retorna só filhos dentro do `groupId` do responsável (a Prisma extension de `F1-E0A` garante isso sem lógica extra no use case); a restrição por aluno em todos os demais endpoints (`allowedStudentIds`, `F1-E09`) é o que impede um responsável de trocar `studentId` na URL e ver dado de outro aluno — ver "IDOR" nos Testes.
- `proxy.ts`/`shared/auth` não precisam reimplementar nenhuma checagem de group/escola — só a checagem de **papel** descrita na seção seguinte é responsabilidade desta ficha.
- **Sem seletor de escola:** professor que leciona em mais de uma escola do group entra por uma URL diferente para cada uma (novo login a cada troca, ver `arquitetura-frontend.md` §10) — não há mais `SchoolSelector` a reaproveitar aqui.

## Modelo de domínio (`enterprise`)

Nenhuma outra entidade nova além do use case acima. O restante desta ficha é composição de telas em `diceduc-web` sobre endpoints já existentes.

## Rotas e controle de acesso (`diceduc-web`)

- `proxy.ts` (`F1-E10`) já resolve/renova a sessão (`arquitetura-frontend.md` §6), mas só faz a checagem otimista de que o cookie existe — não sabe o papel do usuário. Esta ficha adiciona a checagem de **papel** por grupo de rota — responsável só acessa `/(portal)/responsavel/**`, professor só `/(portal)/professor/**`, aluno só `/(portal)/aluno/**` — no `layout.tsx` de cada área, via `getSession()` (`GET /me`, `F1-E10`; papéis de verdade vêm de `F1-E09`, já resolvido no backend). Como um layout não é re-renderizado em navegação entre páginas irmãs, as páginas de cada área também chamam `getSession()` (o `cache()` do React evita chamada duplicada — ver `arquitetura-frontend.md` §6); a proteção de fato é sempre a API, isso aqui só evita renderizar tela/ação errada.
- `shared/auth`: hook `useSession()` expõe `role` e, para responsável, o `guardianId`/lista de filhos (via `ListGuardianStudentsUseCase`).

### Portal do responsável (`F1-E08-U01`)

- `app/(portal)/responsavel/page.tsx` — seletor de filho (chama `GET /guardians/me/students`) + resumo.
- `app/(portal)/responsavel/[studentId]/frequencia/page.tsx` — consome `GET /turmas/:turmaId/attendance/below-threshold` não se aplica aqui; usa antes um endpoint de histórico do aluno — **nota:** `F1-E03` não expõe "frequência de um aluno específico" diretamente, só "abaixo do limite por turma". Adicionar aqui, como parte desta ficha, `GET /students/:studentId/attendance-summary` (`get-student-attendance-summary.controller.ts` + `GetStudentAttendanceSummaryUseCase` em `attendance/application/useCases/`, usando `AttendanceRecordsRepository.calculatePercentage` já existente de `F1-E03`) para retornar o percentual e o histórico de faltas do aluno.
- `app/(portal)/responsavel/[studentId]/boletim/page.tsx` — consome `GET /students/:studentId/report-card` (`F1-E04`).
- `app/(portal)/responsavel/[studentId]/financeiro/page.tsx` — consome `GET /financial-status?studentId=` (`F1-E06`).
- `app/(portal)/responsavel/[studentId]/comunicados/page.tsx` — placeholder vazio (ver seção de escopo acima).

### Portal do professor (`F1-E08-U02`)

- `app/(portal)/professor/minha-grade/page.tsx` — reusa a tela de `F1-E05-U02` (`GET /teachers/:teacherId/schedule`).
- `app/(portal)/professor/turmas/[turmaId]/chamada/page.tsx` — reusa componente `AttendanceSheet` de `F1-E03`.
- `app/(portal)/professor/turmas/[turmaId]/notas/page.tsx` — reusa componente `LaunchGradesTable` de `F1-E04`.
- Sem tela nova de "conteúdo de aula": nenhum épico de F1 modela essa entidade — fica fora de escopo, mesmo tratamento dado a comunicados (sinalizar como pendência para épico futuro, provavelmente `F2-E08` LMS/EAD).

### Portal do aluno (`F1-E08-U03`)

- `app/(portal)/aluno/boletim/page.tsx` — consome `GET /students/:studentId/report-card` com `studentId` resolvido do próprio usuário logado (aluno só vê a si mesmo).
- `app/(portal)/aluno/frequencia/page.tsx` — consome o novo `GET /students/:studentId/attendance-summary`.
- Ambas as telas são somente leitura — nenhum botão de edição renderizado para o papel `aluno` (reforçado também no backend: os controllers de lançamento já exigem perfil `professor`/`secretaria`, não `aluno`; e mesmo que o frontend tentasse mandar um `studentId` diferente, `allowedStudentIds` de `F1-E09` restringe o aluno a si mesmo — a UI nunca é a única barreira).

## Componentes e API client (`diceduc-web`)

- `features/people/api/guardians.ts` (`getMyStudents`).
- `features/attendance/api/attendance.ts` — adicionar `getStudentAttendanceSummary`.
- `features/attendance/components/AttendanceSummaryCard.tsx` (novo, usado nos portais de responsável e aluno).
- `features/reporting` **não** é usado aqui — portais consomem os endpoints de domínio (`report-card`, `financial-status`) diretamente, relatórios administrativos de `F1-E07` são uma área interna separada.
- `shared/ui`: reaproveitar componentes de tabela/card já usados nas telas internas de `F1-E03`/`F1-E04`/`F1-E06` — não recriar variantes "de portal".

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/people/application/useCases/list-guardian-students.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/attendance/application/useCases/get-student-attendance-summary.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/infra/http/controllers/list-guardian-students.controller.ts`
- [ ] `diceduc-api/src/infra/http/controllers/get-student-attendance-summary.controller.ts`
- [ ] `diceduc-web/proxy.ts` (editar — checagem de papel por grupo de rota)
- [ ] `diceduc-web/shared/auth/useSession.ts` (editar — expor `role`/`guardianId`)
- [ ] `diceduc-web/app/(portal)/responsavel/page.tsx`
- [ ] `diceduc-web/app/(portal)/responsavel/[studentId]/frequencia/page.tsx`
- [ ] `diceduc-web/app/(portal)/responsavel/[studentId]/boletim/page.tsx`
- [ ] `diceduc-web/app/(portal)/responsavel/[studentId]/financeiro/page.tsx`
- [ ] `diceduc-web/app/(portal)/responsavel/[studentId]/comunicados/page.tsx` (placeholder)
- [ ] `diceduc-web/app/(portal)/professor/minha-grade/page.tsx`
- [ ] `diceduc-web/app/(portal)/professor/turmas/[turmaId]/chamada/page.tsx`
- [ ] `diceduc-web/app/(portal)/professor/turmas/[turmaId]/notas/page.tsx`
- [ ] `diceduc-web/app/(portal)/aluno/boletim/page.tsx`
- [ ] `diceduc-web/app/(portal)/aluno/frequencia/page.tsx`
- [ ] `diceduc-web/features/people/api/guardians.ts`
- [ ] `diceduc-web/features/attendance/api/attendance.ts` (editar)
- [ ] `diceduc-web/features/attendance/components/AttendanceSummaryCard.tsx`

## Testes

- Backend: unit spec para `ListGuardianStudentsUseCase` e `GetStudentAttendanceSummaryUseCase` (repositórios in-memory já existentes de `F1-E01`/`F1-E03`); e2e para os dois novos controllers, incluindo **IDOR**: responsável autenticado pedindo `attendance-summary`/`report-card`/`financial-status` de um `studentId` que não é filho dele → `404` (o `allowedStudentIds` de `F1-E09` resolve isso nos repositórios já existentes, sem lógica nova nestes dois use cases).
- Frontend: teste de componente para `AttendanceSummaryCard`; teste do `layout.tsx` de cada área garantindo que um usuário com papel `RESPONSAVEL` não acessa `/(portal)/professor/**` (redirect) — a checagem agora é via `getSession()`, não mais uma prop simples de `role` no `proxy.ts` (ver "Rotas e controle de acesso").
- E2E (Playwright, crítico, roda no CI do `diceduc-web` a partir desta ficha — job sobe Postgres + `diceduc-api` no mesmo workflow, ver `F1-E00`): login como responsável → seleciona filho → vê boletim e financeiro; login como aluno → vê boletim (sem botão de edição); responsável tentando abrir a URL do boletim de outro aluno (fora dos seus filhos) → erro/404, nunca o boletim de fato.

## Definition of Done

- [ ] Critérios de aceite de `F1-E08-U01` a `F1-E08-U03` no BACKLOG satisfeitos (exceto comunicados, explicitamente fora de escopo).
- [ ] `ListGuardianStudentsUseCase` e `GetStudentAttendanceSummaryUseCase` implementados com testes passando.
- [ ] Cada portal (`responsavel`, `professor`, `aluno`) acessível apenas pelo papel correspondente.
- [ ] Portal do responsável permite alternar entre múltiplos filhos.
- [ ] Acesso do aluno é comprovadamente somente leitura (nenhum controle de edição renderizado nem endpoint de escrita aceito para esse papel).
- [ ] Testes de IDOR (acima) passando para os três endpoints consumidos pelos portais (`report-card`, `attendance-summary`, `financial-status`).
- [ ] Playwright rodando no CI do `diceduc-web` (não só localmente).
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E08` para 🟢 quando implementado, com nota sobre comunicados/conteúdo de aula ficarem para épicos futuros.
