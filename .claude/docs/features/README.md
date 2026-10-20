# Índice de fichas de tarefa

Cada épico tem uma pasta própria (`<ID>-<slug>/`) contendo `epic.md` (planejamento) e, quando especificado, uma ou mais specs (`spec-*.md`, nível de implementação). **Para implementar, siga o protocolo em [`CLAUDE.md`](../../../CLAUDE.md)** — não é necessário ler `BACKLOG.md`/`ROADMAP.md` para codar, só para planejar/priorizar.

**Ciclo de status:**
- ⚪ `épico` — só o planejamento (`epic.md`) existe. Não implementar direto: primeiro quebrar em specs.
- 🔵 `especificado` — tem spec(s) de implementação pronta(s) na pasta. Pode implementar.
- 🟢 `implementado` — código já existe em `diceduc-api`/`diceduc-web` para este épico.

## Fundação

| ID | Título | Camadas | Status | Depende de |
| --- | --- | --- | --- | --- |
| [F1-E00](F1-E00-bootstrap/epic.md) | Bootstrap dos repositórios (diceduc-api + diceduc-web) | backend, frontend | 🟢 implementado | — |
| [F1-E0A](F1-E0A-tenancy/epic.md) | Multi-tenancy (Group e School) | backend, frontend | 🟢 implementado | F1-E00 |

**F1-E0A é fundacional e roda antes de qualquer épico de domínio (F1-E01 em diante):** todo model de negócio a partir daqui carrega `groupId`/`schoolId` desde a primeira versão — ver [`arquitetura-ignite.md` §11](../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero).

## Fase 1 — MVP

| ID | Título | Bounded context | Camadas | Status | Depende de |
| --- | --- | --- | --- | --- | --- |
| [F1-E01](F1-E01-cadastros-sis/epic.md) | Cadastros / SIS básico | `people`, `academic` | backend | 🟢 implementado | F1-E00, F1-E0A |
| [F1-E10](F1-E10-identidade-acesso/epic.md) | Identidade e acesso (login, sessão, senha) | `iam` | backend, frontend | 🟢 implementado | F1-E0A, F1-E01 |
| [F1-E02](F1-E02-matricula-rematricula/epic.md) | Matrícula, rematrícula e turmas | `academic` | backend, frontend | 🟢 implementado | F1-E01 |
| [F1-E03](F1-E03-frequencia/epic.md) | Gestão de frequência | `attendance` | backend, frontend | 🟢 implementado | F1-E02, F1-E05 |
| [F1-E04](F1-E04-avaliacoes-notas/epic.md) | Avaliações, notas e boletins | `assessment` | backend | 🟢 implementado | F1-E02, F1-E05 |
| [F1-E05](F1-E05-horarios/epic.md) | Horários e grade de aulas | `academic` | backend, frontend | 🟢 implementado | F1-E02 |
| [F1-E06](F1-E06-financeiro/epic.md) | Financeiro essencial | `finance` | backend, frontend | 🟢 implementado | F1-E01, F1-E02 |
| [F1-E07](F1-E07-relatorios/epic.md) | Relatórios administrativos e oficiais | `reporting` | backend, frontend | 🟢 implementado | F1-E01…F1-E06 |
| [F1-E08](F1-E08-portais-web/epic.md) | Portais web (pais, alunos, professores) | vários | backend, frontend | 🟢 implementado | F1-E01…F1-E06 |
| [F1-E09](F1-E09-seguranca-lgpd/epic.md) | Segurança, perfis de acesso e LGPD | `iam` | backend, frontend | 🟢 implementado (2 specs: `spec-seguranca-lgpd.md`, `spec-direitos-titular.md`) | F1-E01, F1-E10 (transversal) |

## Fase 2 — Eficiência & Comunicação

| ID | Título | Bounded context | Camadas | Status | Depende de |
| --- | --- | --- | --- | --- | --- |
| [F2-E01](F2-E01-app-mobile/epic.md) | App mobile | vários | mobile | ⚪ épico | F1-E08 |
| [F2-E02](F2-E02-comunicacao/epic.md) | Comunicação multicanal e notificações | `communication` | backend, frontend | ⚪ épico | F1-E01, F1-E03 |
| [F2-E03](F2-E03-pagamentos/epic.md) | Integração com meios de pagamento | `finance` | backend, frontend | ⚪ épico | F1-E06 |
| [F2-E04](F2-E04-biblioteca-patrimonio/epic.md) | Biblioteca e patrimônio | `library` (novo) | backend, frontend | ⚪ épico | F1-E01 |
| [F2-E05](F2-E05-transporte/epic.md) | Transporte escolar | `transport` (novo) | backend, frontend | ⚪ épico | F1-E01, F1-E02 |
| [F2-E06](F2-E06-rh-folha/epic.md) | RH e folha de pagamento | `hr` (novo) | backend, frontend | ⚪ épico | F1-E01 |
| [F2-E07](F2-E07-multiunidade/epic.md) | Multiunidade / rede escolar (dashboards e governança — isolamento/escopo já fundacional desde F1-E0A/F1-E09) | `iam` (extensão), `reporting` (extensão) | backend, frontend | ⚪ épico | F1-E0A, F1-E01…F1-E09 |
| [F2-E08](F2-E08-lms-ead/epic.md) | LMS/EAD integrado básico | `lms` (novo) | backend, frontend | ⚪ épico | F1-E02, F1-E05 |

## Fase 3 — Analytics, IA & Diferenciação

| ID | Título | Bounded context | Camadas | Status | Depende de |
| --- | --- | --- | --- | --- | --- |
| [F3-E01](F3-E01-analytics-ia/epic.md) | Analytics/BI avançado e IA | `analytics` (novo) | backend, frontend | ⚪ épico | F1-E03, F1-E04, F1-E06 |
| [F3-E02](F3-E02-bncc/epic.md) | Aderência avançada à BNCC | `assessment` (extensão) | backend, frontend | ⚪ épico | F1-E04 |
| [F3-E03](F3-E03-automacoes/epic.md) | Motor de fluxos/automações configuráveis | `automation` (novo) | backend, frontend | ⚪ épico | F1-E03, F1-E06, F2-E02 |
| [F3-E04](F3-E04-transporte-gps/epic.md) | Transporte com GPS e segurança avançada | `transport` (extensão) | backend, frontend | ⚪ épico | F2-E05 |
| [F3-E05](F3-E05-mobile-superior/epic.md) | Experiência mobile superior | vários | mobile | ⚪ épico | F2-E01 |
| [F3-E06](F3-E06-integracoes/epic.md) | Ecossistema de integrações | `integrations` (novo) | backend | ⚪ épico | F1-E09 |
| [F3-E07](F3-E07-multi-rede/epic.md) | Gestão multi-rede com governança | `iam` (extensão) | backend, frontend | ⚪ épico | F2-E07 |

## Fluxo de trabalho

1. **Planejar um épico ⚪:** o `epic.md` já existe para todos os épicos — é o ponto de partida.
2. **Especificar (⚪ → 🔵):** dentro da pasta do épico, copiar [`_TEMPLATE-spec.md`](_TEMPLATE-spec.md) para `spec-<nome>.md` (uma spec por fatia implementável — um épico grande pode gerar mais de uma) e preencher usando o `epic.md` e o `BACKLOG.md` como fonte. Atualizar o status nesta tabela para 🔵.
3. **Implementar (🔵 → 🟢):** seguir a(s) spec(s) conforme o protocolo do [`CLAUDE.md`](../../../CLAUDE.md). Atualizar o status nesta tabela para 🟢 quando o código estiver em produção/mergeado.
4. **Atualizar a documentação as-built do repo tocado:** se a ficha mudou arquitetura, regras ou decisões de um domínio, atualizar `.claude/arquitetura.md` e/ou `.claude/dominios/<contexto>.md` no repo (`diceduc-api`/`diceduc-web`), descrevendo só o que foi implementado. Esse passo faz parte da conclusão da ficha, antes de marcar 🟢.
5. **Novo épico** (fora do BACKLOG atual): copiar [`_TEMPLATE-epic.md`](_TEMPLATE-epic.md) para uma nova pasta e seguir o mesmo ciclo.
