# F2-E08 — LMS/EAD integrado básico

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F2-E08 — LMS/EAD integrado básico |
| Fase | F2 (Eficiência & Comunicação) |
| Prioridade | Could |
| Estimativa | G |
| Bounded context(s) | `lms` (novo) |
| Depende de | F1-E02, F1-E05 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F2-E08` |

## Objetivo

Publicar conteúdo e atividades por turma, e integrar turmas do sistema com o Google Classroom.

## Histórias cobertas

- `F2-E08-U01` — Publicar conteúdo/atividade para uma turma; aluno envia resposta.
- `F2-E08-U02` — Integrar turma do sistema com turma existente no Google Classroom.

## Escopo / fora de escopo

- **Dentro:** conteúdo, atividade, envio de resposta pelo aluno, sincronização com Google Classroom.
- **Fora:** fóruns, videoaulas ao vivo, gamificação (mencionados no documento-fonte só como direção geral de LMS, não são critério de aceite).

## Dependências e integrações

Depende de F1-E02 (`ClassGroup`) e F1-E05 (contexto de aula). A integração com Google Classroom exige OAuth e API externa do Google.

## Decisões em aberto / riscos

- Credenciais/OAuth do Google Workspace for Education são pré-requisito de infraestrutura antes de especificar `F2-E08-U02`.

## Próximo passo

Quebrar este épico em specs dentro desta mesma pasta, usando [`_TEMPLATE-spec.md`](../_TEMPLATE-spec.md). Depois de criar a(s) spec(s), atualizar o status deste épico no [índice](../README.md).
