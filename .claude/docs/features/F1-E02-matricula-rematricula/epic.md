# F1-E02 — Matrícula, rematrícula e turmas

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E02 — Matrícula, rematrícula e turmas |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | `academic` |
| Depende de | F1-E01 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E02` |

## Objetivo

Matricular e rematricular alunos em turmas respeitando limite de vagas, e aplicar regras de promoção/reprovação configuráveis ao final do ano letivo.

## Histórias cobertas

- `F1-E02-U01` — Matricular aluno em turma respeitando limite de vagas (com pré-matrícula/reserva).
- `F1-E02-U02` — Processar rematrícula reaproveitando dados existentes e sinalizando pendência financeira.
- `F1-E02-U03` — Configurar regras de promoção/reprovação por série e calcular situação final do aluno.

## Escopo / fora de escopo

- **Dentro:** matrícula, pré-matrícula, rematrícula, vínculo aluno-turma-ano letivo, cálculo de aprovação/reprovação/recuperação.
- **Fora:** bloqueio/cobrança financeira propriamente dita (F1-E06), grade de horários (F1-E05).

## Dependências e integrações

Depende de F1-E01 (`Student`, `ClassGroup`, `SchoolYear` já existentes). Integra com F1-E06 para sinalizar pendência financeira na rematrícula (leitura, não escrita). É pré-requisito de F1-E03 e F1-E04, que precisam do vínculo aluno-turma para existir.

## Decisões em aberto / riscos

- O bloqueio de rematrícula por inadimplência é "sinalizar e opcionalmente bloquear conforme configuração" — decidir se é feature flag por escola ou regra fixa.
- O cálculo de aprovação/reprovação cruza com notas (F1-E04) e frequência (F1-E03), que podem não existir ainda quando este épico for implementado — avaliar se o cálculo automático fica adiado (stub) até essas dependências existirem.

## Próximo passo

Já especificado em [`spec-matricula.md`](spec-matricula.md). Ver status no [índice](../README.md).
