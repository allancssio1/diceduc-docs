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
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E01` para histórias/critérios completos |

## Objetivo

Cadastro e histórico de alunos, responsáveis, professores e funcionários, mais a estrutura acadêmica base (ano letivo, séries, turmas, disciplinas, turnos, salas). É a fundação sobre a qual todos os outros épicos da Fase 1 são construídos — **implemente esta ficha antes de qualquer outra de domínio**.

## Histórias cobertas

- `F1-E01-U01` — Secretaria cadastra aluno com dados pessoais, documentos e contatos de emergência.
- `F1-E01-U02` — Secretaria vincula um ou mais responsáveis (pedagógico/financeiro) a um aluno.
- `F1-E01-U03` — Coordenação cadastra séries, turmas, disciplinas, turnos, salas e o calendário letivo do ano.
- `F1-E01-U04` — Secretaria atualiza a situação do aluno (ativo, transferido, egresso, trancado).

(Lista completa de critérios de aceite está no BACKLOG — aqui é só o suficiente para dar contexto ao quebrar em specs.)

## Escopo / fora de escopo

- **Dentro:** cadastro de aluno (dados pessoais, documentos, contatos de emergência), vínculo aluno↔responsável, cadastro básico de professor/funcionário (nome + documento + contato), estrutura acadêmica base (ano letivo, séries, turmas, disciplinas, turnos, salas), alteração de situação do aluno.
- **Fora:** lançamentos pedagógicos de professor (horários, notas, frequência — épicos futuros), vínculo aluno↔turma (matrícula, `F1-E02`), checagem fina de perfil de acesso (`F1-E09`).

## Dependências e integrações

Depende de F1-E00 (bootstrap dos repositórios). É a fundação de todos os épicos de domínio da Fase 1: `F1-E02` (matrícula) usa `Student`/`ClassGroup`/`SchoolYear`; `F1-E03`/`F1-E04`/`F1-E05` seguem a mesma cadeia; `F1-E09` estende o guard de perfis que aqui só aplica `JwtAuthGuard` padrão.

## Decisões em aberto / riscos

Nenhuma no momento — já especificado.

## Próximo passo

Já especificado em [`spec-cadastros-sis.md`](spec-cadastros-sis.md). Ver status no [índice](../README.md).
