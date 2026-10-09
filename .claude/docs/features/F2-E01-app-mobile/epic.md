# F2-E01 — App mobile (pais, alunos, professores)

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F2-E01 — App mobile (pais, alunos, professores) |
| Fase | F2 (Eficiência & Comunicação) |
| Prioridade | Must |
| Estimativa | GG |
| Bounded context(s) | vários — camada mobile |
| Depende de | F1-E08 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F2-E01` |

## Objetivo

Aplicativo mobile (iOS/Android) com paridade essencial ao portal web, incluindo notificações push.

## Histórias cobertas

- `F2-E01-U01` — App do responsável com paridade ao portal web.
- `F2-E01-U02` — App do professor: chamada e lançamento de notas, com fila offline.
- `F2-E01-U03` — Notificações push configuráveis por tipo de evento.

## Escopo / fora de escopo

- **Dentro:** app mobile consumindo a mesma API do `diceduc-web`, paridade de funcionalidades essenciais, push.
- **Fora:** recursos exclusivos de mobile como GPS de transporte (F3-E04) ou UX diferenciada avançada (F3-E05 — aqui é paridade, não "experiência superior").

## Dependências e integrações

Depende de F1-E08 (mesma API/contratos do portal web). Provavelmente reusa a camada `shared/api` do `diceduc-web` ou expõe um cliente equivalente para o app.

## Decisões em aberto / riscos

- Framework de app mobile (React Native, Flutter, nativo) ainda não decidido — precisa de uma decisão de arquitetura própria (análoga à `arquitetura-frontend.md`) antes de especificar.

## Próximo passo

Quebrar este épico em specs dentro desta mesma pasta, usando [`_TEMPLATE-spec.md`](../_TEMPLATE-spec.md). Depois de criar a(s) spec(s), atualizar o status deste épico no [índice](../README.md).
