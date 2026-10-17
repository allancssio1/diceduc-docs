# F1-E06 — Financeiro essencial (mensalidades)

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E06 — Financeiro essencial (mensalidades) |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | `finance` |
| Depende de | F1-E01, F1-E02 |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E06` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Cadastrar planos de mensalidade, gerar títulos a receber automaticamente ao vincular um aluno, registrar baixa manual de pagamento e reportar inadimplência por aluno/turma/série.

## Histórias cobertas

- `F1-E06-U01` — Cadastrar plano de mensalidade e gerar títulos automaticamente ao vincular a um aluno.
- `F1-E06-U02` — Emitir boleto e registrar baixa de pagamento (manual).
- `F1-E06-U03` — Consultar situação financeira por aluno, turma ou série.

## Integração com F1-E02

`F1-E02` (matrícula/rematrícula) definiu o port `FinanceStatusProvider` (`academic/application/services/finance-status-provider.ts`) com um stub que sempre retorna `false`. Esta ficha implementa o adapter real: `infra/database/prisma/services/prisma-finance-status-provider.ts implements FinanceStatusProvider`, consultando `InvoicesRepository` por títulos `OPEN`/`OVERDUE` do aluno. Trocar o binding do provider no módulo (`AcademicModule`/`DatabaseModule`) para usar essa implementação em vez do stub — não altera nenhum use case de `F1-E02`.

## Multi-tenancy

Mecanismo completo em [`arquitetura-ignite.md` §11](../../../arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [`F1-E0A`](../F1-E0A-tenancy/spec-tenancy.md). Específico deste épico:

- `BillingPlan`, `Invoice` carregam `groupId`/`schoolId` — planos de mensalidade são configurados por escola (duas escolas do mesmo group podem ter valores/parcelamentos diferentes).
- `PrismaFinanceStatusProvider` (implementação real do port de `F1-E02`) executa dentro do `GroupContext` corrente — a checagem de dívida do aluno nunca vaza para fora do group do usuário que está processando a rematrícula.

## Modelo de domínio (`enterprise`)

### Contexto `finance` — `diceduc-api/src/domain/finance/enterprise/`

| Entidade | Tipo | Caminho (`entities/`) | Props principais |
| --- | --- | --- | --- |
| `BillingPlan` | `Entity` | `billing-plan.ts` | `name`, `baseAmount`, `installments`, `dueDay` (dia do mês de vencimento) |
| `Invoice` | `AggregateRoot` | `invoice.ts` | `studentId`, `billingPlanId`, `installmentNumber`, `amount`, `discountAmount`, `dueDate`, `status: InvoiceStatus`, `paidAt?`, `paidAmount?`, `paymentMethod?: PaymentMethod`, `externalGatewayId?: string` (campo reservado para `F2-E03`, não usado nesta ficha) |

Value objects — `entities/valueObjects/`:
- `invoice-status.ts` — `InvoiceStatus = 'OPEN' | 'PAID' | 'OVERDUE' | 'CANCELLED'`.
- `payment-method.ts` — `PaymentMethod = 'CASH' | 'BANK_SLIP' | 'PIX' | 'CARD' | 'OTHER'`.

Invariantes:
- `Invoice.amount = billingPlan.baseAmount / billingPlan.installments - discountAmount` no momento da geração (desconto fixo por aluno, não percentual, para simplicidade).
- `status` transiciona para `'OVERDUE'` quando `dueDate < hoje` e ainda `'OPEN'` (calculado em leitura, não precisa de job agendado nesta ficha — ver nota em Persistência).
- `RegisterPaymentUseCase` só aceita `Invoice` com `status` em `('OPEN', 'OVERDUE')`.

## Use cases (`application`)

### `diceduc-api/src/domain/finance/application/useCases/`

| Use case | Arquivo | Entrada | Saída (`Either`) | Erros específicos |
| --- | --- | --- | --- | --- |
| `CreateBillingPlanUseCase` | `create-billing-plan.ts` | `name`, `baseAmount`, `installments`, `dueDay` | `Either<InvalidAmountError, { billingPlan: BillingPlan }>` | `invalid-amount-error.ts` |
| `LinkBillingPlanToStudentUseCase` | `link-billing-plan-to-student.ts` | `studentId`, `billingPlanId`, `schoolYearId`, `discountAmount?` | `Either<ResourceNotFoundError, { invoices: Invoice[] }>` | — |
| `RegisterPaymentUseCase` | `register-payment.ts` | `invoiceId`, `paidAmount`, `paidAt`, `paymentMethod` | `Either<ResourceNotFoundError \| InvoiceAlreadyPaidError, { invoice: Invoice }>` | `invoice-already-paid-error.ts` |
| `GetFinancialStatusUseCase` | `get-financial-status.ts` | `studentId?`, `classGroupId?`, `gradeLevelId?` (ao menos um) | `Either<ResourceNotFoundError, { totalOpen: number; totalOverdue: number; totalUpcoming: number; invoices: Invoice[] }>` | — |

Ports (`application/repositories/`): `BillingPlansRepository`, `InvoicesRepository` — inclui `findManyByStudent`, `findManyByClassGroup` (via join com `Enrollment` de `F1-E02`), `findManyByGradeLevel`.

Regras de negócio principais:
- `LinkBillingPlanToStudentUseCase`: gera `installments` títulos de uma vez, com `dueDate` calculado a partir de `dueDay` para cada mês do ano letivo a partir da matrícula.
- `RegisterPaymentUseCase`: seta `status = 'PAID'`, `paidAt`, `paidAmount`, `paymentMethod`.
- `GetFinancialStatusUseCase`: agrega `Invoice`s calculando `OVERDUE` em tempo de leitura (`dueDate < hoje && status == 'OPEN'` conta como vencido, sem exigir job de atualização em background nesta ficha).

## Persistência (`infra/database/prisma`)

- `diceduc-api/prisma/schema.prisma`: novos models `BillingPlan`, `Invoice`; enums `InvoiceStatus`, `PaymentMethod`. `externalGatewayId` como `String?` já reservado no schema (sem uso ainda).
- Repositórios: `prisma-billing-plans-repository.ts`, `prisma-invoices-repository.ts` em `infra/database/prisma/repositories/`.
- Adapter de serviço: `infra/database/prisma/services/prisma-finance-status-provider.ts` (implementa `FinanceStatusProvider` de `F1-E02`, ver seção "Integração").
- Mappers: `billing-plan-mapper.ts`, `invoice-mapper.ts` em `infra/database/prisma/mappers/`.

## HTTP (`infra/http`)

| Rota | Controller | Schema Zod (entrada) | Presenter |
| --- | --- | --- | --- |
| `POST /billing-plans` | `create-billing-plan.controller.ts` | dados de `CreateBillingPlanUseCase` | `billing-plan-presenter.ts` |
| `POST /students/:studentId/billing-plans` | `link-billing-plan-to-student.controller.ts` | `{ billingPlanId, schoolYearId, discountAmount? }` | `invoice-presenter.ts` (lista) |
| `POST /invoices/:invoiceId/payment` | `register-payment.controller.ts` | dados de `RegisterPaymentUseCase` (exceto `invoiceId`) | `invoice-presenter.ts` |
| `GET /financial-status` | `get-financial-status.controller.ts` | query `{ studentId?, classGroupId?, gradeLevelId? }` | JSON agregado (`totalOpen`, `totalOverdue`, `totalUpcoming`, `invoices`) |

Perfis exigidos: todas as rotas → `financeiro`/`secretaria` (guard fino vem de `F1-E09`; por ora `JwtAuthGuard` padrão).

## Frontend (`diceduc-web`)

- `app/(portal)/finance/plans/page.tsx` — CRUD de `BillingPlan` (`F1-E06-U01`).
- `app/(portal)/finance/students/[studentId]/page.tsx` — títulos do aluno, baixa manual (`F1-E06-U02`).
- `app/(portal)/finance/delinquency/page.tsx` — relatório filtrável por turma/série (`F1-E06-U03`).
- `features/finance/components/BillingPlanForm.tsx`, `InvoiceList.tsx`, `RegisterPaymentDialog.tsx`, `DelinquencySummary.tsx`.
- `features/finance/api/finance.ts` (`createBillingPlan`, `linkBillingPlanToStudent`, `registerPayment`, `getFinancialStatus`), `features/finance/schemas/finance.ts`.

## Arquivos a criar/editar (checklist)

- [ ] `diceduc-api/src/domain/finance/enterprise/entities/billing-plan.ts`
- [ ] `diceduc-api/src/domain/finance/enterprise/entities/invoice.ts`
- [ ] `diceduc-api/src/domain/finance/enterprise/entities/valueObjects/invoice-status.ts`
- [ ] `diceduc-api/src/domain/finance/enterprise/entities/valueObjects/payment-method.ts`
- [ ] `diceduc-api/src/domain/finance/application/useCases/create-billing-plan.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/finance/application/useCases/link-billing-plan-to-student.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/finance/application/useCases/register-payment.ts` (+ errors/, + .spec.ts)
- [ ] `diceduc-api/src/domain/finance/application/useCases/get-financial-status.ts` (+ .spec.ts)
- [ ] `diceduc-api/src/domain/finance/application/repositories/billing-plans-repository.ts`
- [ ] `diceduc-api/src/domain/finance/application/repositories/invoices-repository.ts`
- [ ] `diceduc-api/prisma/schema.prisma` (editar)
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-billing-plans-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/repositories/prisma-invoices-repository.ts`
- [ ] `diceduc-api/src/infra/database/prisma/services/prisma-finance-status-provider.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/billing-plan-mapper.ts`
- [ ] `diceduc-api/src/infra/database/prisma/mappers/invoice-mapper.ts`
- [ ] `diceduc-api/src/infra/http/controllers/*.ts` (4 controllers, ver seção HTTP)
- [ ] `diceduc-api/src/infra/http/presenters/billing-plan-presenter.ts`, `invoice-presenter.ts`
- [ ] `diceduc-web/app/(portal)/finance/plans/page.tsx`
- [ ] `diceduc-web/app/(portal)/finance/students/[studentId]/page.tsx`
- [ ] `diceduc-web/app/(portal)/finance/delinquency/page.tsx`
- [ ] `diceduc-web/features/finance/components/*.tsx`
- [ ] `diceduc-web/features/finance/api/finance.ts`
- [ ] `diceduc-web/features/finance/schemas/finance.ts`

## Testes

- Repositórios in-memory: `in-memory-billing-plans-repository.ts`, `in-memory-invoices-repository.ts`.
- Factories: `make-billing-plan.ts`, `make-invoice.ts`.
- Unit specs: geração correta de N parcelas com desconto (`LinkBillingPlanToStudentUseCase`), baixa em título já pago retorna erro (`RegisterPaymentUseCase`), cálculo de `OVERDUE` em leitura e agregação por turma/série (`GetFinancialStatusUseCase`), `PrismaFinanceStatusProvider.hasOpenDebt` retorna `true` com título vencido.
- E2E: um `.e2e-spec.ts` por controller listado na seção HTTP.

## Definition of Done

- [ ] Critérios de aceite de `F1-E06-U01` a `F1-E06-U03` no BACKLOG satisfeitos.
- [ ] Todos os use cases e controllers implementados com testes unitários e e2e passando.
- [ ] Migração Prisma aplicada sem erro.
- [ ] `FinanceStatusProvider` de `F1-E02` migrado do stub para a implementação real desta ficha.
- [ ] Cadastro de plano, geração de títulos, baixa manual e relatório de inadimplência funcionando ponta a ponta.
- [ ] `.claude/docs/features/README.md` atualizado: status de `F1-E06` para 🟢 quando implementado.
