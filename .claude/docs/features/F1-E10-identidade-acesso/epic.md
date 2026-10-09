# F1-E10 — Identidade e acesso

## Cabeçalho

| Campo | Valor |
| --- | --- |
| Épico | F1-E10 — Identidade e acesso |
| Fase | F1 (MVP) |
| Prioridade | Must |
| Estimativa | G |
| Bounded context(s) | `iam` |
| Depende de | F1-E0A (tenancy), F1-E01 (pessoas) |
| Rastreabilidade | [`BACKLOG.md`](../../../BACKLOG.md) — buscar por `### F1-E10` para histórias/critérios completos (referência, não leitura obrigatória) |

## Objetivo

Dar a cada pessoa cadastrada (`F1-E01`) uma conta de acesso ao sistema: login, sessão, senha temporária de primeiro acesso, troca/redefinição de senha e desativação — a peça que faltava entre "a pessoa existe no cadastro" e "a pessoa consegue entrar no portal". Sem esta ficha, `F1-E09` (perfis de acesso) e `F1-E08` (portais) não têm de onde tirar `sub`/sessão real.

## Histórias cobertas

- `F1-E10-U01` — Usuário faz login com e-mail/senha; conta bloqueia temporariamente após tentativas malsucedidas repetidas.
- `F1-E10-U02` — Secretaria cria o acesso de uma pessoa já cadastrada, com senha temporária de uso único.
- `F1-E10-U03` — Secretaria redefine a senha ou desativa a conta de um usuário, com efeito imediato.
- `F1-E10-U04` — Usuário troca a própria senha e pode encerrar todas as outras sessões abertas.
- `F1-E10-U05` — Equipe DICEDUC cria ou redefine o administrador de um group recém-onboardado.
- `F1-E10-U06` — Usuário recupera a senha por e-mail sem depender da secretaria (pronta no código, **desligada por flag** no MVP — ver decisão abaixo).

(Lista completa de critérios de aceite está no BACKLOG — aqui é só o suficiente para dar contexto ao quebrar em specs.)

## Escopo / fora de escopo

- **Dentro:** conta de usuário (`User`), autenticação com sessão de access+refresh token, primeiro acesso com senha temporária, troca/redefinição/desativação de senha, revogação de sessões, provisionamento do admin de um group pela equipe DICEDUC, e o módulo de recuperação por e-mail (implementado, mas desligado por flag).
- **Fora:** perfis/permissões finas por módulo e trilha de auditoria (isso é `F1-E09`, que depende desta ficha para ter uma conta real de onde ler `sub`); MFA (fica para contas de rede, `F2-E07`); provisionamento automático em provedores externos (Google Workspace/Microsoft 365 — isso é `F3-E06`).

## Dependências e integrações

- Depende de `F1-E0A` porque o login precisa resolver o group/escola pelo slug da URL antes de autenticar (`GroupContext` preliminar) e porque a sessão emitida carrega `groupId`+`schoolId`.
- Depende de `F1-E01` porque `CreateUserAccountUseCase` valida a pessoa (`Guardian`/`Teacher`/`Staff`/`Student`) lendo os repositórios de `people` antes de criar a conta — não recadastra ninguém.
- `F1-E09` depende desta ficha: o `PermissionsGuard`/`GroupScopeGuard` passam a ler papéis de um `User` real, não de um stub.
- `F1-E08` (portais) depende desta ficha para toda a camada de sessão do `diceduc-web` (`proxy.ts`, `getSession()`).

## Decisões em aberto / riscos

Nenhuma em aberto — decisões já tomadas ao especificar (ver `spec-identidade-acesso.md`): sessão via access token (15 min) + refresh token (7 dias) rotativo, ambos em cookies httpOnly geridos pelo `diceduc-web` (padrão BFF); e-mail único **por group** (não global); recuperação de senha por e-mail implementada e testada, mas atrás da flag `PASSWORD_RESET_ENABLED=false` no MVP.

## Próximo passo

Já especificado em [`spec-identidade-acesso.md`](spec-identidade-acesso.md). Ver status no [índice](../README.md).
