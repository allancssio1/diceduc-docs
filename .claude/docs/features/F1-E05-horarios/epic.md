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
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E05` |

## Objetivo

Montar a grade de horários de cada turma (professor, disciplina, sala, dia/horário) evitando conflitos de alocação.

## Histórias cobertas

- `F1-E05-U01` — Coordenação monta grade de horários evitando conflito de professor/sala.
- `F1-E05-U02` — Professor visualiza sua própria grade consolidada da semana.

## Escopo / fora de escopo

- **Dentro:** grade por turma, alocação professor/disciplina/sala/horário, detecção de conflito, edição durante o ano com data de efeito.
- **Fora:** geração automática/otimizada de grade — citada no documento-fonte como diferencial de mercado, não é critério de aceite deste MVP.

## Dependências e integrações

Depende de F1-E01 (`Teacher`, `Room`, `Subject`) e F1-E02 (`ClassGroup`). É pré-requisito de F1-E03 e F1-E04, que precisam saber "quando" ocorre uma aula.

## Decisões em aberto / riscos

Nenhuma decisão maior em aberto — épico bem contido, com uma pendência pontual:

- **`ScheduleSlotsRepository` já existe, criado por `F1-E03`** (que precisava de `existsActiveFor` antes de F1-E05 existir — ver `spec-frequencia.md`, decisão de porta+stub). Hoje é `abstract class ScheduleSlotsRepository { abstract existsActiveFor(...): Promise<boolean> }` em `diceduc-api/src/domain/attendance/application/repositories/schedule-slots-repository.ts` (contexto `attendance`, não `academic`), com produção sempre retornando `true` (`StubScheduleSlotsRepository`, bind em `DatabaseModule`). Ao implementar esta ficha: (1) decidir se o port definitivo continua em `attendance` ou migra para `academic` (onde a entidade `ScheduleSlot` vive) — se migrar, `RecordAttendanceUseCase`/`JustifyAbsenceUseCase` (F1-E03) precisam importar do novo caminho; (2) estender a assinatura/arquivo com `findConflictingByTeacher`/`findConflictingByRoom` desta ficha; (3) trocar o binding de `StubScheduleSlotsRepository` para `PrismaScheduleSlotsRepository` em `DatabaseModule`, sem alterar `RecordAttendanceUseCase` — só a implementação de `existsActiveFor` passa a consultar o `ScheduleSlot` de verdade em vez de sempre devolver `true`.

## Próximo passo

Já especificado em [`spec-horarios.md`](spec-horarios.md). Ver status no [índice](../README.md).
