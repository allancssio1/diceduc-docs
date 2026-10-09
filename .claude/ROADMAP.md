# Roadmap — Sistema de Gestão Escolar (DICEDUC)

Roadmap derivado de [Funcionalidades para um Sistema de Gestão Escolar em Três Níveis.md](Funcionalidades%20para%20um%20Sistema%20de%20Gestão%20Escolar%20em%20Três%20Níveis.md), organizando os 3 níveis de funcionalidades em fases de release. Detalhamento de histórias e critérios de aceite está em [BACKLOG.md](BACKLOG.md).

## Convenções

- **ID de épico:** `F<fase>-E<nn>`
- **Estimativa relativa:** `P` pequeno · `M` médio · `G` grande · `GG` muito grande
- **Prioridade:** MoSCoW (Must / Should / Could) dentro de cada fase

## Sequenciamento

O DICEDUC é **multi-tenant desde o primeiro momento da arquitetura**: `F1-E0A` (Group/School,
isolamento por `groupId` e escopo de permissão por `schoolId`) é fundacional e roda logo após o
bootstrap, **antes** de qualquer épico de domínio — nenhum contexto de negócio é implementado sem
essa base pronta. Cadastros/SIS é pré-requisito de praticamente tudo (matrícula, turmas,
financeiro, portais). Dentro da Fase 1, a ordem recomendada é:

0. `F1-E0A` Multi-tenancy (Group/School — fundacional, antes de tudo) → 1. `F1-E01` Cadastros/SIS →
2. `F1-E10` Identidade e acesso (login/sessão — sem ela não existe usuário real para `F1-E09` atribuir
papel) → 3. `F1-E09` Segurança/LGPD (perfis de acesso, incluindo o escopo por escola em `UserRole`,
transversal desde o início) → 4. `F1-E02` Matrícula/turmas → 5. `F1-E05` Horários → 6. `F1-E03`
Frequência e `F1-E04` Notas/boletins (em paralelo, ambos dependem de turmas/horários) → 7. `F1-E06`
Financeiro → 8. `F1-E07` Relatórios → 9. `F1-E08` Portais (consome dados de todos os anteriores).

Fases 2 e 3 pressupõem a Fase 1 estável em produção. `F2-E07` (Multiunidade/rede) deixou de
introduzir qualquer parte do isolamento/escopo multi-tenant (isso já é fundacional desde `F1-E0A`/
`F1-E09`) — passou a ser só dashboards consolidados e governança de rede sobre a base existente.

---

## Fase 1 — MVP (Nível 1: Indispensáveis)

**Objetivo:** operar a escola com segurança jurídica e operacional, eliminando planilhas e documentos manuais.

**Critérios de saída da fase:**
- Secretaria consegue matricular, fazer chamada, lançar notas e emitir boletim 100% pelo sistema.
- Financeiro básico gera títulos, registra baixas e reporta inadimplência.
- Pais/alunos/professores acessam portal web com dados corretos e atualizados.
- Perfis de acesso e trilha de auditoria mínima implementados (LGPD).
- Relatório para Censo Escolar/INEP exportável.

| Épico | Descrição | Prioridade | Estimativa | Depende de |
| --- | --- | --- | --- | --- |
| F1-E0A | Multi-tenancy (Group e School) — fundacional | Must | M | F1-E00 |
| F1-E01 | Gestão de cadastros / SIS básico | Must | G | F1-E0A |
| F1-E10 | Identidade e acesso (login, sessão, senha) | Must | G | F1-E0A, F1-E01 |
| F1-E02 | Matrícula, rematrícula e turmas | Must | G | F1-E01 |
| F1-E03 | Gestão de frequência | Must | M | F1-E02, F1-E05 |
| F1-E04 | Avaliações, notas e boletins | Must | G | F1-E02, F1-E05 |
| F1-E05 | Horários e grade de aulas | Must | M | F1-E02 |
| F1-E06 | Financeiro essencial (mensalidades) | Must | G | F1-E01, F1-E02 |
| F1-E07 | Relatórios administrativos e oficiais | Must | M | F1-E01 a F1-E06 |
| F1-E08 | Portais web (pais, alunos, professores) | Must | G | F1-E01 a F1-E06 |
| F1-E09 | Segurança, perfis de acesso e LGPD | Must | M | F1-E01, F1-E10 (transversal) |

---

## Fase 2 — Eficiência & Comunicação (Nível 2: Bom ter)

**Objetivo:** ganho de eficiência operacional e aumento de atratividade comercial.

**Critérios de saída da fase:**
- App mobile publicado para pais/alunos/professores com paridade essencial ao portal.
- Pagamento on-line de mensalidades com baixa automática.
- Comunicação multicanal (e-mail/SMS/WhatsApp/push) operando para avisos institucionais e automáticos.
- Pelo menos um módulo adicional (biblioteca, transporte ou RH) em produção, conforme prioridade comercial da escola/rede.
- **Pendência operacional trazida do MVP:** backup com restauração testada (fora do MVP por custo/prioridade — ver [`arquitetura-ignite.md` §12](arquitetura-ignite.md#12-segurança-de-aplicação-baseline)).

| Épico | Descrição | Prioridade | Estimativa | Depende de |
| --- | --- | --- | --- | --- |
| F2-E01 | App mobile (pais, alunos, professores) | Must | GG | F1-E08 |
| F2-E02 | Comunicação multicanal e notificações | Must | G | F1-E01, F1-E03 |
| F2-E03 | Integração com meios de pagamento | Must | G | F1-E06 |
| F2-E04 | Biblioteca e patrimônio | Should | M | F1-E01 |
| F2-E05 | Transporte escolar | Should | M | F1-E01, F1-E02 |
| F2-E06 | RH e folha de pagamento | Could | G | F1-E01 |
| F2-E07 | Multiunidade / rede escolar (dashboards/governança consolidando várias escolas numa sessão só — no MVP, cada sessão é de uma escola; isolamento e escopo já fundacionais) | Should | G | F1-E0A, F1-E01 a F1-E09 |
| F2-E08 | LMS/EAD integrado básico | Could | G | F1-E02, F1-E05 |

---

## Fase 3 — Analytics, IA & Diferenciação (Nível 3: Diferenciais)

**Objetivo:** diferenciar o produto no mercado com camada analítica, automação e integrações.

**Critérios de saída da fase:**
- Dashboard de indicadores (evasão, inadimplência, aprovação) disponível para direção/mantenedora.
- Pelo menos um fluxo automático configurável em produção (ex.: alerta de frequência baixa).
- API pública documentada com ao menos uma integração externa ativa.

| Épico | Descrição | Prioridade | Estimativa | Depende de |
| --- | --- | --- | --- | --- |
| F3-E01 | Analytics/BI avançado e IA | Must | GG | F1-E03, F1-E04, F1-E06 |
| F3-E02 | Aderência avançada à BNCC | Should | G | F1-E04 |
| F3-E03 | Motor de fluxos/automações configuráveis | Must | G | F1-E03, F1-E06, F2-E02 |
| F3-E04 | Transporte com GPS e segurança avançada | Could | G | F2-E05 |
| F3-E05 | Experiência mobile superior | Should | G | F2-E01 |
| F3-E06 | Ecossistema de integrações | Should | GG | F1-E09 |
| F3-E07 | Gestão multi-rede com governança | Could | G | F2-E07 |

---

## Resumo de todos os épicos

| ID | Épico | Fase | Prioridade | Estimativa |
| --- | --- | --- | --- | --- |
| F1-E0A | Multi-tenancy (Group e School) | 1 | Must | M |
| F1-E01 | Cadastros / SIS básico | 1 | Must | G |
| F1-E10 | Identidade e acesso | 1 | Must | G |
| F1-E02 | Matrícula, rematrícula e turmas | 1 | Must | G |
| F1-E03 | Gestão de frequência | 1 | Must | M |
| F1-E04 | Avaliações, notas e boletins | 1 | Must | G |
| F1-E05 | Horários e grade de aulas | 1 | Must | M |
| F1-E06 | Financeiro essencial | 1 | Must | G |
| F1-E07 | Relatórios administrativos e oficiais | 1 | Must | M |
| F1-E08 | Portais web | 1 | Must | G |
| F1-E09 | Segurança, acesso e LGPD | 1 | Must | M |
| F2-E01 | App mobile | 2 | Must | GG |
| F2-E02 | Comunicação multicanal | 2 | Must | G |
| F2-E03 | Integração com pagamentos | 2 | Must | G |
| F2-E04 | Biblioteca e patrimônio | 2 | Should | M |
| F2-E05 | Transporte escolar | 2 | Should | M |
| F2-E06 | RH e folha de pagamento | 2 | Could | G |
| F2-E07 | Multiunidade / rede escolar | 2 | Should | G |
| F2-E08 | LMS/EAD básico | 2 | Could | G |
| F3-E01 | Analytics/BI e IA | 3 | Must | GG |
| F3-E02 | BNCC avançado | 3 | Should | G |
| F3-E03 | Motor de automações | 3 | Must | G |
| F3-E04 | Transporte com GPS | 3 | Could | G |
| F3-E05 | Experiência mobile superior | 3 | Should | G |
| F3-E06 | Ecossistema de integrações | 3 | Should | GG |
| F3-E07 | Gestão multi-rede com governança | 3 | Could | G |
