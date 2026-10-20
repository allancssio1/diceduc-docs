# F1-E07 — Relatórios administrativos e oficiais

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E07 — Relatórios administrativos e oficiais |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | M |
| Bounded context(s) | `reporting` |
| Depende de | F1-E01 a F1-E06 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E07` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Gerar relatórios operacionais filtráveis (listas de alunos, frequência, notas, resultados finais, inadimplência), exportáveis em PDF/planilha, e exportar dados no formato exigido pelo Censo Escolar/INEP.

## Histórias cobertas

- `F1-E07-U01` — Relatórios filtráveis por turma/série/período/situação, exportáveis em PDF/planilha.
- `F1-E07-U02` — Exportação de dados no formato do Censo Escolar/INEP.

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md). Específico deste épico:

- Todo `Generate*ReportUseCase` já herda o filtro de `groupId`/`schoolId` da Prisma extension através dos repositórios injetados — nenhum filtro manual de tenant precisa ser escrito nos use cases de relatório em si, só os filtros de negócio (`classGroupId`, `gradeLevelId`, `period`).
- **Censo Escolar/INEP é por escola**, não por group — cada escola tem seu próprio código INEP (`School.inepCode`, de `F1-E0A`). `ExportSchoolCensusUseCase` **não** recebe `schoolId` como parâmetro — usa o da sessão corrente, igual a qualquer outro use case de leitura (uma sessão é sempre de uma escola, ver `arquitetura-ignite.md` §11). Exportar o Censo de outra escola do mesmo group exige logar naquela escola.
- `GenerateFinalResultsReportUseCase`/demais relatórios filtráveis por "turma/série" só listam turmas/séries da escola da sessão — resultado natural do filtro de `EnrollmentsRepository`/`ClassGroupsRepository`, sem lógica extra.

## Modelo de domínio (`enterprise`)

**N/A.** `reporting` é uma camada de leitura pura sobre os agregados já existentes em `people`, `academic`, `attendance`, `assessment` e `finance` — não introduz entidades/VOs novos. Os use cases desta ficha injetam diretamente os repositórios já existentes desses contextos (leitura cross-context é aceitável aqui porque `reporting` não escreve em nenhum deles).

## Use cases (`application`)

### `diceduc-api/src/domain/reporting/application/useCases/`

| Use case | Arquivo | Entrada | Repositórios injetados | Saída (`Either`) |
| --- | --- | --- | --- | --- |
| `GenerateStudentListReportUseCase` | `generate-student-list-report.ts` | `classGroupId?`, `gradeLevelId?`, `status?` | `StudentsRepository`, `EnrollmentsRepository` | `Either<never, { rows: StudentReportRow[] }>` |
| `GenerateAttendanceReportUseCase` | `generate-attendance-report.ts` | `classGroupId?`, `gradeLevelId?`, `startDate?`, `endDate?` | `AttendanceRecordsRepository` | `Either<never, { rows: AttendanceReportRow[] }>` |
| `GenerateGradesReportUseCase` | `generate-grades-report.ts` | `classGroupId?`, `gradeLevelId?`, `period?` | `GradesRepository`, `AssessmentsRepository` | `Either<never, { rows: GradesReportRow[] }>` |
| `GenerateFinalResultsReportUseCase` | `generate-final-results-report.ts` | `classGroupId?`, `gradeLevelId?`, `schoolYearId` | `EnrollmentsRepository` | `Either<never, { rows: FinalResultReportRow[] }>` |
| `GenerateDelinquencyReportUseCase` | `generate-delinquency-report.ts` | `classGroupId?`, `gradeLevelId?`, `startDate?`, `endDate?` | `InvoicesRepository` (mesma agregação de `GetFinancialStatusUseCase` de `F1-E06`, reaproveitar via injeção do repositório, não duplicar lógica) | `Either<never, { rows: DelinquencyReportRow[] }>` |
| `ExportSchoolCensusUseCase` | `export-school-census.ts` | `schoolYearId` | `StudentsRepository`, `EnrollmentsRepository` | `Either<IncompleteCensusDataError, { records: SchoolCensusRecord[] }>` |

`*ReportRow`/`SchoolCensusRecord` são tipos simples (DTOs) declarados junto de cada use case, não entidades de domínio.

Regras de negócio principais:
- Todos os relatórios de listagem (`Generate*ReportUseCase`) retornam `Either<never, ...>` — filtro vazio é um resultado válido (lista vazia), não um erro.
- `ExportSchoolCensusUseCase`: para cada aluno da lista, valida campos obrigatórios do Censo (CPF, data de nascimento, endereço completo — o conjunto exato de campos obrigatórios do layout vigente deve ser confirmado contra a especificação oficial do INEP do ano corrente antes de codar; até lá, implementar com um `TODO` explícito no código listando os campos assumidos). Retorna `IncompleteCensusDataError` listando quais alunos têm pendência, sem impedir a geração de um relatório parcial (a exportação em si é bloqueada só se _nenhum_ aluno estiver completo).

**Desvios decididos na implementação:** frequência e inadimplência filtram por `startDate`/`endDate` (inclusivos, dia UTC) em vez de `period` — só `Assessment` tem `period`; o relatório de notas só inclui avaliações `PUBLISHED`; frequência sai uma linha por aluno×turma; a rota `/pending` alimenta a tela do Censo.

## Persistência (`infra/database/prisma`)

Nenhum novo model. Os repositórios usados já existem (`F1-E01`, `F1-E02`, `F1-E03`, `F1-E04`, `F1-E06`); esta ficha só adiciona, quando necessário, métodos de leitura agregada nesses repositórios (ex.: `EnrollmentsRepository.findManyByGradeLevelWithResult`), sempre no arquivo do repositório já existente — **não** criar um novo repositório "de relatórios" que duplica queries.

## HTTP (`infra/http`)

| Rota | Controller | Query params | Formato de saída |
| --- | --- | --- | --- |
| `GET /reports/students` | `generate-student-list-report.controller.ts` | `classGroupId?, gradeLevelId?, status?, format=pdf\|xlsx` | arquivo (PDF ou planilha) |
| `GET /reports/attendance` | `generate-attendance-report.controller.ts` | `classGroupId?, gradeLevelId?, startDate?, endDate?, format` | arquivo |
| `GET /reports/grades` | `generate-grades-report.controller.ts` | `classGroupId?, gradeLevelId?, period?, format` | arquivo |
| `GET /reports/final-results` | `generate-final-results-report.controller.ts` | `classGroupId?, gradeLevelId?, schoolYearId, format` | arquivo |
| `GET /reports/delinquency` | `generate-delinquency-report.controller.ts` | `classGroupId?, gradeLevelId?, startDate?, endDate?, format` | arquivo |
| `GET /reports/school-census` | `export-school-census.controller.ts` | `schoolYearId, format=csv` | arquivo CSV (`422` com `{ message, pending }` se nenhum aluno está completo) |
| `GET /reports/school-census/pending` | `get-school-census-pending.controller.ts` | `schoolYearId` | JSON `{ pending }` (sempre `200`) |

Geração de arquivo: PDF reaproveita a mesma lib decidida em `F1-E04` (`pdf-lib` ou equivalente definido no bootstrap); planilha usa `exceljs` (XLSX) ou `csv-stringify` (CSV) — preferir `csv-stringify` para o Censo por ser o formato mais comum de importação em sistemas do MEC, e `exceljs` só para os relatórios operacionais que pedem XLSX explicitamente.

Perfis exigidos: todas as rotas → `direção`/`coordenação`/`secretaria` (guard fino vem de `F1-E09`; por ora `JwtAuthGuard` padrão).

## Frontend (`diceduc-web`)

- `app/(portal)/reports/page.tsx` — hub de relatórios com seleção de tipo + filtros (turma/série/período/situação) + botões de exportação (`F1-E07-U01`).
- `app/(portal)/reports/school-census/page.tsx` — tela dedicada de exportação do Censo, com lista de alunos pendentes destacada antes de exportar (`F1-E07-U02`).
- `features/reporting/components/ReportFilters.tsx`, `ReportTable.tsx`, `CensusPendencyList.tsx`.
- `features/reporting/api/reports.ts` (`generateStudentListReport`, `generateAttendanceReport`, `generateGradesReport`, `generateFinalResultsReport`, `generateDelinquencyReport`, `exportSchoolCensus` — todas retornam blob para download), `features/reporting/schemas/report-filters.ts`.

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/reporting/application/useCases/generate-student-list-report.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/reporting/application/useCases/generate-attendance-report.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/reporting/application/useCases/generate-grades-report.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/reporting/application/useCases/generate-final-results-report.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/reporting/application/useCases/generate-delinquency-report.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/reporting/application/useCases/export-school-census.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/infra/http/controllers/*.ts` (6 controllers, ver seção HTTP)
- [ ] `diceduc-api/src/infra/http/presenters/report-file-presenter.ts` (helper compartilhado: monta `Content-Disposition`/`Content-Type` conforme `format`)
- [ ] `diceduc-web/app/(portal)/reports/page.tsx`
- [ ] `diceduc-web/app/(portal)/reports/school-census/page.tsx`
- [ ] `diceduc-web/features/reporting/components/*.tsx`
- [ ] `diceduc-web/features/reporting/api/reports.ts`
- [ ] `diceduc-web/features/reporting/schemas/report-filters.ts`

## Testes

- Sem repositórios in-memory novos — reusar os já existentes de `F1-E01`/`F1-E02`/`F1-E03`/`F1-E04`/`F1-E06` nos unit specs.
- Unit specs: agregação correta por filtro em cada `Generate*ReportUseCase` (dado um conjunto de dados via repositórios in-memory populados, o relatório retorna as linhas esperadas), `ExportSchoolCensusUseCase` sinaliza alunos com campo obrigatório faltante sem travar os demais.
- E2E: um `.e2e-spec.ts` por controller, validando `Content-Type`/`Content-Disposition` do arquivo retornado para cada `format`.

## Definition of Done

- [ ] Critérios de aceite de `F1-E07-U01` e `F1-E07-U02` no BACKLOG satisfeitos.
- [ ] Todos os use cases e controllers implementados com testes unitários e e2e passando.
- [ ] Layout de campos do Censo Escolar/INEP validado contra a especificação oficial vigente (remover o `TODO` do código de `ExportSchoolCensusUseCase`).
- [ ] Hub de relatórios com filtros e exportação PDF/planilha funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E07` para 🟢 quando implementado.
