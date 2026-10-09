# F1-E09 — Direitos do titular (LGPD)

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E09 — Segurança, perfis de acesso e LGPD |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | P |
| Bounded context(s) | `reporting`, `iam` |
| Depende de | F1-E01 a F1-E06, [`spec-seguranca-lgpd.md`](spec-seguranca-lgpd.md) |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E09` (histórias `U04`/`U05`) |

## Objetivo

Dar ao responsável o direito de acesso/portabilidade dos dados do próprio filho (exportação) e um canal de contato do encarregado (DPO) do group para pedidos de correção/eliminação — o restante do escopo de LGPD que `spec-seguranca-lgpd.md` (consentimento, auditoria, perfis) não cobre.

## Histórias cobertas

- `F1-E09-U04` — Responsável exporta os dados do próprio filho.
- `F1-E09-U05` — Canal de privacidade com contato do encarregado do group.

## Escopo / fora de escopo

- **Dentro:** exportação em JSON dos dados de um aluno; página de privacidade com contato do encarregado e explicação de como pedir correção/eliminação; auditoria da própria exportação.
- **Fora:** fluxo automatizado de correção/eliminação dentro do sistema (é manual, via contato direto com o encarregado — não há prazo/SLA modelado nesta ficha); anonimização/retenção automática (proposta abaixo, sem automação até a Fase 2); exportação em outros formatos (PDF/CSV) — JSON cobre portabilidade, que é o critério de aceite.

## Modelo de domínio (`enterprise`)

Nenhuma entidade nova. Reusa `Group.dpoName`/`dpoEmail` (`F1-E0A`) e adiciona um valor ao enum já existente `AuditAction` (`spec-seguranca-lgpd.md`): `AuditAction = 'CREATE' | 'UPDATE' | 'DELETE' | 'EXPORT'`.

## Use cases (`application`)

### `diceduc-api/src/domain/reporting/application/useCases/export-student-data.ts`

- Entrada: `studentId`.
- Saída (`Either`): `Either<ResourceNotFoundError, { data: StudentDataExport }>`, onde `StudentDataExport` é um DTO simples (não entidade) com: dados do `Student`, `Guardian`s vinculados (`F1-E01`), matrículas (`F1-E02`), frequência (`F1-E03`), notas — **só de `Assessment.status = 'PUBLISHED'`** (`F1-E04`, mesma regra de `GenerateReportCardUseCase`), títulos financeiros (`F1-E06`), registros de consentimento (`spec-seguranca-lgpd.md`).
- Ports usados: mesmo padrão de leitura cross-context de `F1-E07` — injeta diretamente os repositórios já existentes desses contextos (`reporting` não escreve neles).
- Regras de negócio: a restrição a "só os próprios filhos" **não é responsabilidade do use case** — vem de graça do `allowedStudentIds` do `GroupContext` (`spec-seguranca-lgpd.md`), que a Prisma extension já aplica em cada repositório injetado. Ao final, chama `AuditLogger.record({ action: 'EXPORT', module: 'reporting', entityId: studentId, ... })`.

## Persistência (`infra/database/prisma`)

Nenhum novo model — mesma regra de `F1-E07` (`reporting` é leitura pura). `AuditAction` ganha o valor `'EXPORT'` no enum Prisma já existente (editar a migração de `spec-seguranca-lgpd.md` se ainda não aplicada, ou nova migração se já estiver em produção).

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Saída |
| --- | --- | --- | --- |
| `GET /students/:studentId/data-export` | `export-student-data.controller.ts` | — (param) | arquivo JSON, `Content-Disposition: attachment` (mesmo helper `report-file-presenter.ts` de `F1-E07`) |
| `GET /me` (editar, de `F1-E10`) | `get-me.controller.ts` | — | passa a incluir `group: { dpoName?, dpoEmail? }` na resposta |

Perfis exigidos: `export-student-data` → `responsável`/`aluno`/`secretaria`/`direção` (a restrição real de "só o próprio filho" é o `allowedStudentIds`, não o `@RequirePermission`).

## Frontend (`diceduc-web`)

- `app/(public)/privacidade/page.tsx` — página de privacidade: contato do encarregado (`group.dpoName`/`dpoEmail`, via `useSession()`/`getMe`), explicação de como pedir correção/eliminação de dados, e resumo em linguagem simples da política de retenção (ver seção abaixo). Pública porque não depende de estar logado numa escola específica.
- `app/(portal)/responsavel/[studentId]/page.tsx` (editar, de `F1-E08`) — botão "Baixar meus dados" chamando `GET /students/:studentId/data-export`.
- `app/(portal)/cadastros/alunos/[studentId]/page.tsx` (editar, de `F1-E01`) — mesmo botão na visão da secretaria.
- `features/reporting/api/reports.ts` (editar, de `F1-E07`) — `exportStudentData` (retorna blob para download, mesmo padrão dos outros relatórios).

## Proposta de retenção (a validar com o jurídico)

Tabela de referência para orientar a página de privacidade — **não implementada como automação nesta ficha**:

| Dado | Retenção proposta | Justificativa |
| --- | --- | --- |
| Cadastro do aluno/responsável, matrículas, notas, frequência | Enquanto o aluno estiver vinculado à escola + 5 anos após desligamento | Histórico escolar, obrigações de guarda de registro educacional |
| Títulos financeiros | 5 anos (prazo fiscal comum) | Obrigação contábil/fiscal |
| Trilha de auditoria (`AuditLog`) | 5 anos | Rastreabilidade de alterações sensíveis |
| Consentimento (`ConsentRecord`) | Enquanto a conta existir (append-only, nunca apagado) | Prova de consentimento/revogação ao longo do tempo |

`ponytail:` sem job de anonimização/expurgo automático nesta ficha — os prazos acima orientam a página de privacidade e uma futura rotina de expurgo, a implementar quando a Fase 2 justificar o custo de operar isso com segurança (apagar dado errado é irreversível).

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/reporting/application/useCases/export-student-data.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/iam/enterprise/entities/valueObjects/audit-action.ts` (editar — adicionar `'EXPORT'`)
- [ ] `diceduc-api/prisma/schema.prisma` (editar — enum `AuditAction`)
- [ ] `diceduc-api/src/infra/http/controllers/export-student-data.controller.ts`
- [ ] `diceduc-api/src/infra/http/controllers/get-me.controller.ts` (editar, de `F1-E10` — incluir `group.dpoName`/`dpoEmail`)
- [ ] `diceduc-web/app/(public)/privacidade/page.tsx`
- [ ] `diceduc-web/app/(portal)/responsavel/[studentId]/page.tsx` (editar)
- [ ] `diceduc-web/app/(portal)/cadastros/alunos/[studentId]/page.tsx` (editar)
- [ ] `diceduc-web/features/reporting/api/reports.ts` (editar)

## Testes

- Unit specs: `ExportStudentDataUseCase` só inclui notas de `Assessment.status = 'PUBLISHED'` (reusando o repositório in-memory de `F1-E04`); grava `AuditLog` com `action = 'EXPORT'`.
- E2E: responsável exportando o próprio filho → `200` com o JSON completo; responsável exportando um aluno que não é seu filho → `404` (prova de IDOR, mesmo padrão dos demais endpoints de `studentId`); `GET /me` retorna `group.dpoEmail` quando configurado.

## Definition of Done

- [ ] Critérios de aceite de `F1-E09-U04` e `F1-E09-U05` no BACKLOG satisfeitos.
- [ ] Exportação e página de privacidade funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E09` para 🟢 (agora com as duas specs implementadas).
