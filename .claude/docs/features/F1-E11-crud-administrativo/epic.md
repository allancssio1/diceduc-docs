# F1-E11 — CRUD administrativo

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E11 — CRUD administrativo |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | `tenancy`, `people`, `iam` |
| Depende de | F1-E0A, F1-E01, F1-E09 (papéis por escola, para `U05`/`U06`) |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E11` |

## Objetivo

Dar ao admin do grupo e à secretaria o ciclo completo (listar, buscar, editar, inativar) sobre escolas, alunos, professores e funcionários, incluindo transferir e promover/despromover funcionários entre as escolas do grupo. Hoje essas entidades só nascem por cadastro (`POST`), sem listagem, edição ou inativação.

## Histórias cobertas

- `F1-E11-U01` — Admin do grupo edita os dados do grupo e das escolas e inativa/reativa uma escola.
- `F1-E11-U02` — Secretaria lista, busca, edita e inativa alunos.
- `F1-E11-U03` — Secretaria lista, busca, edita e inativa professores.
- `F1-E11-U04` — Secretaria lista, busca, edita e inativa funcionários.
- `F1-E11-U05` — Transferir funcionário/professor de uma escola para outra do mesmo grupo.
- `F1-E11-U06` — Atribuir, promover e despromover o papel de um funcionário por escola.

(Lista completa de critérios de aceite está no BACKLOG — aqui é só o suficiente para dar contexto ao quebrar em specs.)

## Escopo / fora de escopo

- **Dentro:** `GET` de listagem/busca e `PATCH` de edição para `Student`, `Teacher`, `Staff`, `School` e para os dados editáveis do `Group` (nome, logo, cor, DPO); inativação lógica; transferência de escola; papel de funcionário por escola; as telas de `diceduc-web` correspondentes (hoje `admin/schools` é só leitura e `records/*` só cadastra).
- **Fora:** criar `Group` e provisionar o primeiro admin (continuam com a equipe DICEDUC atrás do `PlatformAdminGuard`, `F1-E0A`); CRUD da estrutura acadêmica (ano letivo, séries, turmas, disciplinas, salas), de planos de cobrança e de regras de promoção — pendência registrada, sem épico; dashboards consolidados de várias escolas numa sessão só (`F2-E07`); exclusão física de pessoas (LGPD/auditoria pedem inativação lógica).

## Dependências e integrações

- `F1-E0A`: `School`/`Group` já existem com `status` e branding; este épico só adiciona edição e inativação, sem mudar o isolamento por `groupId`.
- `F1-E01`: `Student`, `Teacher` e `Staff` já têm cadastro. `Teacher` e `Staff` guardam um único `schoolId`, e `Staff.role` é texto livre.
- `F1-E09`: o papel por escola (`UserRole` com `schoolId`) vem daí e **ainda não está implementado**. `U05` e `U06` dependem dele; `U01` a `U04` podem sair antes.
- Auditoria (`F1-E09-U02`): edição de dado pessoal, transferência e mudança de papel são ações sensíveis e entram na trilha.

## Decisões em aberto / riscos

- **Transferência:** mover o `schoolId` do registro existente ou passar a um vínculo N:N pessoa↔escola. Quem acumula papéis em duas escolas (promovido numa, despromovido na outra) aponta para o vínculo por escola.
- **Histórico:** o que acontece com turmas e aulas atribuídas a um professor/funcionário ao transferi-lo (reatribuir, bloquear ou manter até o fim do ano letivo).
- **Edição de CPF:** `@@unique([groupId, cpf])` vale para `Teacher` e `Staff`; definir se CPF é editável ou imutável.

## Próximo passo

Quebrar este épico em specs dentro desta mesma pasta, usando [`_TEMPLATE-spec.md`](../_TEMPLATE-spec.md). Sugestão de fatias: tenancy (`U01`), pessoas (`U02`–`U04`) e papéis/transferência (`U05`–`U06`, depois de `F1-E09` implementado). Depois de criar a(s) spec(s), atualizar o status deste épico no [índice](../README.md).
