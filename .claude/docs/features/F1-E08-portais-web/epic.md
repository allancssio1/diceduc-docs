# F1-E08 — Portais web (pais, alunos, professores)

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E08 — Portais web (pais, alunos, professores) |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | vários (`people`, `academic`, `attendance`, `assessment`, `finance`) — camada frontend |
| Depende de | F1-E01 a F1-E06 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E08` |

## Objetivo

Entregar o portal web (`diceduc-web`) que expõe, por papel (responsável, professor, aluno), os dados já disponíveis na API dos épicos anteriores.

## Histórias cobertas

- `F1-E08-U01` — Portal do responsável: frequência, boletim, boletos, comunicados dos filhos.
- `F1-E08-U02` — Portal do professor: lançamento de notas/frequência/conteúdo das suas turmas.
- `F1-E08-U03` — Portal do aluno: leitura de boletim e frequência.

## Escopo / fora de escopo

- **Dentro:** telas de consumo dos dados de F1-E01 a F1-E06 por papel, dentro de `diceduc-web` (ver `arquitetura-frontend.md`).
- **Fora:** app mobile nativo (F2-E01), envio de comunicação por e-mail/SMS/WhatsApp (F2-E02 — aqui só exibe comunicados já existentes).

## Dependências e integrações

É a "vitrine" de F1-E01 a F1-E06: cada tela consome endpoints já implementados nesses épicos. Depende também de F1-E09 para controle de visibilidade por perfil (o que cada papel pode ver/fazer).

## Decisões em aberto / riscos

- Como este épico agrega telas de vários contextos, ao especificar pode valer a pena dividir em specs por área (ex.: uma spec para a área de alunos/frequência/notas, outra para financeiro) em vez de uma spec monolítica.

## Próximo passo

Já especificado em [`spec-portais-web.md`](spec-portais-web.md). Ver status no [índice](../README.md).
