# Backlog — Sistema de Gestão Escolar (DICEDUC)

Backlog detalhado derivado de [ROADMAP.md](ROADMAP.md), com base em [Funcionalidades para um Sistema de Gestão Escolar em Três Níveis.md](Funcionalidades%20para%20um%20Sistema%20de%20Gestão%20Escolar%20em%20Três%20Níveis.md).

**Convenções:** ID de épico `F<fase>-E<nn>`, ID de história `F<fase>-E<nn>-U<nn>`. Estimativa relativa P/M/G/GG por épico. Prioridade MoSCoW.

---

## Fase 1 — MVP (Nível 1: Indispensáveis)

### F1-E01 — Gestão de cadastros / SIS básico
**Objetivo:** Cadastro e histórico de alunos, responsáveis, professores e funcionários, além de estruturas de turmas/séries/disciplinas/calendário.
**Prioridade:** Must · **Estimativa:** G · **Depende de:** —

#### F1-E01-U01
Como **secretaria**, quero cadastrar um aluno com dados pessoais, documentos e contatos de emergência,
para manter um registro completo e centralizado de cada estudante.

**Critérios de aceite:**
- [ ] Formulário permite registrar nome, data de nascimento, documentos (CPF/RG/certidão), endereço e contatos de emergência.
- [ ] Sistema impede salvar cadastro sem os campos obrigatórios definidos pela escola.
- [ ] Aluno recebe situação inicial "ativo" e um identificador único.
- [ ] Histórico de alterações do cadastro fica registrado (quem alterou, quando, o quê).

#### F1-E01-U02
Como **secretaria**, quero vincular um ou mais responsáveis (pedagógico e financeiro) a cada aluno,
para saber quem deve ser contatado e quem é responsável pelos pagamentos.

**Critérios de aceite:**
- [ ] É possível cadastrar múltiplos responsáveis por aluno, com papel definido (pedagógico, financeiro ou ambos).
- [ ] Ao menos um responsável financeiro é obrigatório antes de ativar a matrícula.
- [ ] Dados de contato do responsável (telefone, e-mail) são obrigatórios.

#### F1-E01-U03
Como **coordenação**, quero cadastrar séries, turmas, disciplinas, turnos, salas e o calendário letivo do ano,
para estruturar a base sobre a qual matrículas, horários e notas serão lançados.

**Critérios de aceite:**
- [ ] É possível criar um ano letivo com data de início/fim e eventos (feriados, recessos).
- [ ] Séries/turmas podem ser criadas vinculadas a um ano letivo, turno e capacidade de vagas.
- [ ] Disciplinas podem ser associadas a uma ou mais séries.
- [ ] Não é possível excluir uma turma/série que já possui alunos matriculados.

#### F1-E01-U04
Como **secretaria**, quero atualizar a situação de um aluno (ativo, transferido, egresso, trancado),
para refletir corretamente sua trajetória escolar nos relatórios e portais.

**Critérios de aceite:**
- [ ] Situação do aluno é exibida em todas as telas relevantes (perfil, listas, relatórios).
- [ ] Mudança de situação registra data e usuário responsável.
- [ ] Alunos com situação "transferido"/"egresso" deixam de aparecer em chamadas e lançamentos de nota ativos, mas mantêm histórico consultável.

---

### F1-E10 — Identidade e acesso
**Objetivo:** Dar a cada pessoa cadastrada uma conta de acesso: login, sessão, senha temporária de primeiro acesso e redefinição.
**Prioridade:** Must · **Estimativa:** G · **Depende de:** F1-E0A, F1-E01

#### F1-E10-U01
Como **usuário do sistema**, quero fazer login com e-mail e senha,
para acessar as funcionalidades do meu perfil.

**Critérios de aceite:**
- [ ] Login exige e-mail e senha cadastrados na escola/grupo correto.
- [ ] Após 5 tentativas malsucedidas seguidas, a conta fica temporariamente bloqueada.
- [ ] Mensagem de erro não revela se o problema foi o e-mail ou a senha.

#### F1-E10-U02
Como **secretaria**, quero criar o acesso de uma pessoa já cadastrada (responsável, professor, funcionário) com uma senha temporária,
para liberar o uso do sistema sem depender de autoatendimento.

**Critérios de aceite:**
- [ ] A senha temporária é exibida uma única vez, na tela de criação do acesso.
- [ ] No primeiro login, o usuário é obrigado a definir uma senha própria.
- [ ] A senha temporária expira se não for usada em 72 horas.

#### F1-E10-U03
Como **secretaria**, quero redefinir a senha ou desativar o acesso de um usuário,
para lidar com esquecimento de senha ou desligamento de alguém da escola.

**Critérios de aceite:**
- [ ] Redefinir a senha gera uma nova senha temporária e invalida imediatamente as sessões abertas daquele usuário.
- [ ] Desativar um usuário impede login e invalida imediatamente as sessões abertas.
- [ ] Nenhuma dessas ações pode ser feita por um usuário sem o perfil adequado, nem contra um administrador, salvo por outro administrador.

#### F1-E10-U04
Como **usuário do sistema**, quero trocar minha própria senha e poder encerrar todas as outras sessões abertas,
para manter minha conta segura se suspeitar de acesso indevido.

**Critérios de aceite:**
- [ ] Trocar a senha exige informar a senha atual.
- [ ] Existe uma ação de "sair de todos os dispositivos" que invalida todas as sessões, exceto a atual (ou todas, dependendo da implementação).

#### F1-E10-U05
Como **equipe DICEDUC**, quero criar ou redefinir o administrador de uma escola/rede recém-onboardada,
para entregar o acesso inicial ao cliente sem depender de auto-cadastro.

**Critérios de aceite:**
- [ ] A ação exige autenticação de administrador da plataforma, não de um usuário comum.
- [ ] O administrador criado já nasce com o perfil de administrador daquele grupo.

#### F1-E10-U06
Como **usuário do sistema**, quero recuperar minha senha por e-mail sem depender da secretaria,
para não ficar bloqueado quando a secretaria não está disponível.

**Critérios de aceite:**
- [ ] Funcionalidade implementada e testada, mas desligada por padrão no MVP (fica disponível para ativação futura sem retrabalho).
- [ ] Quando ativa, o pedido de recuperação nunca revela se o e-mail existe ou não no sistema.
- [ ] O link de redefinição expira em 1 hora e só pode ser usado uma vez.

---

### F1-E02 — Matrícula, rematrícula e turmas
**Objetivo:** Processo completo de matrícula/rematrícula com controle de vagas e regras de promoção.
**Prioridade:** Must · **Estimativa:** G · **Depende de:** F1-E01

#### F1-E02-U01
Como **secretaria**, quero matricular um aluno em uma turma respeitando o limite de vagas,
para evitar turmas superlotadas.

**Critérios de aceite:**
- [ ] Sistema bloqueia matrícula quando a turma atinge a capacidade máxima configurada.
- [ ] Matrícula gera um vínculo aluno-turma-ano letivo consultável no histórico do aluno.
- [ ] É possível registrar pré-matrícula/reserva de vaga antes da confirmação definitiva.

#### F1-E02-U02
Como **secretaria**, quero processar a rematrícula de um aluno para o próximo ano letivo,
para dar continuidade à trajetória escolar sem recadastro manual.

**Critérios de aceite:**
- [ ] Rematrícula reaproveita dados cadastrais existentes do aluno e responsáveis.
- [ ] Sistema sinaliza (e opcionalmente bloqueia, conforme configuração) rematrícula de alunos com pendência financeira, integrando com `F1-E06`.
- [ ] Aluno aprovado é sugerido automaticamente para a série seguinte; aluno reprovado, para repetir a série.

#### F1-E02-U03
Como **coordenação**, quero configurar regras de promoção/reprovação por série,
para que o sistema aplique os critérios corretos ao final do ano letivo.

**Critérios de aceite:**
- [ ] É possível definir critérios mínimos de nota/frequência por série ou segmento.
- [ ] Ao final do ano, o sistema calcula automaticamente a situação de aprovação/reprovação/recuperação de cada aluno com base nas regras configuradas.
- [ ] Resultado do cálculo é revisável e ajustável manualmente pela coordenação antes de confirmar.

---

### F1-E03 — Gestão de frequência
**Objetivo:** Registro diário de presença por turma/aula e cálculo automático de percentual.
**Prioridade:** Must · **Estimativa:** M · **Depende de:** F1-E02, F1-E05

#### F1-E03-U01
Como **professor**, quero fazer a chamada de uma turma por aula,
para registrar rapidamente presenças e faltas.

**Critérios de aceite:**
- [ ] Dado que estou logado como professor de uma turma, quando abro a chamada do dia, então vejo a lista de alunos ativos daquela turma/aula.
- [ ] Posso marcar presente/ausente/justificado por aluno e salvar.
- [ ] Sistema impede lançamento em data fora do calendário letivo.
- [ ] Ao salvar, o percentual de frequência do aluno é recalculado automaticamente.

#### F1-E03-U02
Como **coordenação**, quero visualizar alunos com frequência abaixo do limite mínimo,
para agir preventivamente antes da reprovação por falta.

**Critérios de aceite:**
- [ ] Existe uma lista/relatório filtrável por turma mostrando o percentual de frequência de cada aluno.
- [ ] Alunos abaixo do limite configurado (ex.: 75%) são destacados visualmente.

#### F1-E03-U03
Como **secretaria**, quero justificar uma falta com base em atestado ou documento apresentado,
para que a frequência do aluno reflita a justificativa formal.

**Critérios de aceite:**
- [ ] É possível alterar o status de uma falta para "justificada", anexando observação/documento.
- [ ] Alteração fica registrada com data e usuário responsável (auditoria).

---

### F1-E04 — Avaliações, notas e boletins
**Objetivo:** Lançamento de notas/conceitos com pesos e médias, geração de boletim e histórico.
**Prioridade:** Must · **Estimativa:** G · **Depende de:** F1-E02, F1-E05

#### F1-E04-U01
Como **coordenação**, quero configurar a fórmula de cálculo de média por segmento (peso de provas/trabalhos, recuperação),
para que o cálculo de notas siga a política pedagógica da escola.

**Critérios de aceite:**
- [ ] É possível configurar pesos por tipo de avaliação e a fórmula de média por segmento (Infantil, Fundamental, Médio).
- [ ] É possível configurar se a série usa nota numérica, conceito (A/B/C) ou ambos.
- [ ] Regras de recuperação (nota mínima, cálculo de média final pós-recuperação) são configuráveis.

#### F1-E04-U02
Como **professor**, quero lançar notas de uma atividade/prova para toda a turma de uma vez,
para agilizar o preenchimento do diário de classe.

**Critérios de aceite:**
- [ ] Tela de lançamento lista todos os alunos ativos da turma/disciplina para uma atividade específica.
- [ ] Sistema valida a nota conforme a escala configurada (numérica ou conceito) e impede valores fora do intervalo permitido.
- [ ] Lançamentos podem ser salvos como rascunho e publicados posteriormente.

#### F1-E04-U03
Como **responsável**, quero visualizar o boletim do meu filho com notas por disciplina e período,
para acompanhar o desempenho escolar.

**Critérios de aceite:**
- [ ] Boletim exibe notas por disciplina, período e média final calculada.
- [ ] Boletim só exibe lançamentos já publicados pelo professor (não rascunhos).
- [ ] É possível baixar o boletim em PDF.

#### F1-E04-U04
Como **secretaria**, quero gerar o histórico escolar consolidado de um aluno,
para atender solicitações de transferência ou emissão de documentos oficiais.

**Critérios de aceite:**
- [ ] Histórico consolida notas/situação final de todos os anos letivos cursados pelo aluno.
- [ ] Documento gerado em PDF, pronto para assinatura/impressão.

---

### F1-E05 — Horários e grade de aulas
**Objetivo:** Montagem de horários evitando conflitos de professor e sala.
**Prioridade:** Must · **Estimativa:** M · **Depende de:** F1-E02

#### F1-E05-U01
Como **coordenação**, quero montar a grade de horários de uma turma alocando professor, disciplina e sala por horário/dia,
para organizar a rotina letiva.

**Critérios de aceite:**
- [ ] Sistema impede alocar o mesmo professor em dois horários simultâneos em turmas diferentes.
- [ ] Sistema impede alocar a mesma sala para duas turmas no mesmo horário.
- [ ] Grade pode ser editada durante o ano letivo, com efeito a partir de uma data configurável.

#### F1-E05-U02
Como **professor**, quero visualizar minha grade de horários da semana,
para saber onde e quando devo estar.

**Critérios de aceite:**
- [ ] Professor vê sua própria grade consolidada de todas as turmas em que leciona.
- [ ] Grade exibe turma, disciplina, sala, dia e horário.

---

### F1-E06 — Financeiro essencial (mensalidades)
**Objetivo:** Cadastro de planos, geração de títulos, contas a receber, baixa de pagamentos e inadimplência.
**Prioridade:** Must · **Estimativa:** G · **Depende de:** F1-E01, F1-E02

#### F1-E06-U01
Como **financeiro**, quero cadastrar planos de mensalidade (valor, número de parcelas, vencimentos) e vinculá-los a um aluno matriculado,
para gerar automaticamente os títulos a receber do ano letivo.

**Critérios de aceite:**
- [ ] Plano de mensalidade define valor base, número de parcelas e dia de vencimento.
- [ ] Ao vincular um plano a um aluno, o sistema gera automaticamente os títulos (parcelas) do período.
- [ ] É possível aplicar desconto/bolsa a um aluno específico sem alterar o plano padrão.

#### F1-E06-U02
Como **financeiro**, quero emitir boletos e registrar a baixa de pagamentos (manual ou automática),
para manter o contas a receber atualizado.

**Critérios de aceite:**
- [ ] É possível gerar boleto (ou visualizar dados para pagamento) por título em aberto.
- [ ] Baixa manual de pagamento registra data, valor pago e forma de pagamento.
- [ ] Título pago passa a exibir status "quitado" e não aparece mais como pendente em relatórios de inadimplência.

#### F1-E06-U03
Como **financeiro**, quero consultar a situação financeira de um aluno, turma ou série,
para acompanhar inadimplência e apoiar decisões de rematrícula.

**Critérios de aceite:**
- [ ] Relatório de inadimplência é filtrável por turma, série e período.
- [ ] Relatório exibe valor total em aberto, vencido e a vencer.
- [ ] Situação financeira do aluno fica visível no seu cadastro (resumo).

---

### F1-E07 — Relatórios administrativos e oficiais
**Objetivo:** Relatórios de secretaria/coordenação/direção e relatórios alinhados ao Censo Escolar/INEP.
**Prioridade:** Must · **Estimativa:** M · **Depende de:** F1-E01 a F1-E06

#### F1-E07-U01
Como **direção**, quero gerar relatórios filtráveis por turma, série, período ou situação (listas de alunos, frequência, notas, resultados finais, inadimplência),
para acompanhar indicadores operacionais da escola.

**Critérios de aceite:**
- [ ] Cada relatório permite ao menos os filtros: turma, série, período e situação do aluno.
- [ ] Relatórios podem ser exportados em PDF e/ou planilha (CSV/XLSX).

#### F1-E07-U02
Como **secretaria**, quero exportar dados dos alunos no formato exigido pelo Censo Escolar/INEP,
para cumprir a obrigação legal anual sem retrabalho manual.

**Critérios de aceite:**
- [ ] Exportação inclui os campos mínimos exigidos pelo Censo Escolar (a validar com layout oficial vigente).
- [ ] Sistema sinaliza alunos com campos obrigatórios do Censo incompletos antes da exportação.

---

### F1-E08 — Portais web (pais, alunos, professores)
**Objetivo:** Portais por navegador centralizando boletins, frequência, comunicados e documentos.
**Prioridade:** Must · **Estimativa:** G · **Depende de:** F1-E01 a F1-E06

#### F1-E08-U01
Como **responsável**, quero acessar um portal web para ver frequência, notas, boletos e comunicados do meu filho,
para acompanhar a vida escolar sem precisar ir à escola.

**Critérios de aceite:**
- [ ] Login autenticado exibe apenas dados dos filhos vinculados ao responsável logado.
- [ ] Portal exibe frequência, boletim, 2ª via de boleto e comunicados recentes.
- [ ] Responsável com mais de um filho pode alternar entre eles.

#### F1-E08-U02
Como **professor**, quero acessar um portal para lançar notas, frequência e conteúdo de aula das minhas turmas,
para reduzir dependência da secretaria nesses lançamentos.

**Critérios de aceite:**
- [ ] Professor só visualiza e lança dados das turmas/disciplinas em que está alocado.
- [ ] Portal reaproveita as mesmas telas/fluxos de `F1-E03` e `F1-E04`.

#### F1-E08-U03
Como **aluno**, quero acessar meu boletim e frequência pelo portal,
para acompanhar meu próprio desempenho.

**Critérios de aceite:**
- [ ] Aluno visualiza apenas seus próprios dados (notas, frequência, comunicados).
- [ ] Acesso do aluno é somente leitura (sem permissão de lançamento).

---

### F1-E09 — Segurança, perfis de acesso e LGPD
**Objetivo:** Perfis de acesso, trilha de auditoria, direitos do titular e gestão de consentimento conforme LGPD.
**Prioridade:** Must · **Estimativa:** M · **Depende de:** F1-E01, F1-E10 (transversal)

#### F1-E09-U01
Como **administrador do sistema**, quero definir perfis de acesso (secretaria, coordenação, professor, financeiro, direção, responsável, aluno) com permissões específicas,
para garantir que cada usuário veja e altere apenas o que sua função permite.

**Critérios de aceite:**
- [ ] Cada perfil tem um conjunto de permissões configurável (visualizar/editar por módulo).
- [ ] Usuário sem permissão para um módulo não consegue acessá-lo nem via URL direta.
- [ ] É possível atribuir múltiplos perfis a um mesmo usuário quando aplicável (ex.: coordenador que também é professor).

#### F1-E09-U02
Como **direção**, quero consultar uma trilha de auditoria de ações sensíveis (alteração de notas, dados pessoais, baixas financeiras),
para investigar inconsistências e cumprir requisitos de conformidade.

**Critérios de aceite:**
- [ ] Toda alteração em nota, frequência, dado pessoal sensível e título financeiro registra usuário, data/hora e valor anterior/novo.
- [ ] Trilha de auditoria é consultável por filtro de usuário, período e módulo.

#### F1-E09-U03
Como **responsável**, quero registrar meu consentimento (ou revogação) para uso de imagem e dados do meu filho,
para exercer meus direitos previstos na LGPD.

**Critérios de aceite:**
- [ ] Existe uma tela de termo de consentimento apresentada no primeiro acesso ou quando o termo é atualizado.
- [ ] Responsável pode revogar consentimento a qualquer momento, com registro de data.
- [ ] Status de consentimento por aluno é consultável pela secretaria/direção.

#### F1-E09-U04
Como **responsável**, quero exportar todos os dados cadastrados do meu filho,
para exercer meu direito de acesso e portabilidade previsto na LGPD.

**Critérios de aceite:**
- [ ] Exportação inclui dados pessoais, matrículas, frequência, notas já publicadas, títulos financeiros e consentimentos do aluno.
- [ ] Responsável só consegue exportar dados dos próprios filhos, nunca de outro aluno.
- [ ] A exportação em si fica registrada na trilha de auditoria.

#### F1-E09-U05
Como **responsável**, quero um canal claro para pedir correção ou eliminação dos dados do meu filho,
para exercer meus direitos de titular quando a exportação não resolve.

**Critérios de aceite:**
- [ ] Existe uma página de privacidade com o contato do encarregado (DPO) da escola/rede.
- [ ] A página explica, em linguagem simples, como pedir correção ou eliminação de dados.

---

## Fase 2 — Eficiência & Comunicação (Nível 2: Bom ter)

### F2-E01 — App mobile (pais, alunos, professores)
**Objetivo:** Aplicativo mobile com paridade essencial ao portal web e notificações push.
**Prioridade:** Must · **Estimativa:** GG · **Depende de:** F1-E08

#### F2-E01-U01
Como **responsável**, quero acompanhar frequência, notas, comunicados e boletos do meu filho pelo app mobile,
para ter acesso rápido sem precisar abrir o navegador.

**Critérios de aceite:**
- [ ] App replica as informações essenciais do portal web de responsáveis (`F1-E08-U01`).
- [ ] App funciona em iOS e Android com o mesmo login usado no portal web.

#### F2-E01-U02
Como **professor**, quero registrar chamada e lançar notas pelo app,
para agilizar o lançamento direto da sala de aula.

**Critérios de aceite:**
- [ ] Fluxos de chamada (`F1-E03-U01`) e lançamento de notas (`F1-E04-U02`) estão disponíveis no app.
- [ ] App funciona com conectividade instável (fila de envio/sincronização ao reconectar).

#### F2-E01-U03
Como **responsável**, quero receber notificações push sobre eventos, reuniões, vencimentos e ocorrências,
para não perder prazos e comunicados importantes.

**Critérios de aceite:**
- [ ] Push é disparado para eventos configurados (novo comunicado, boleto próximo do vencimento, ocorrência registrada).
- [ ] Usuário pode configurar quais tipos de notificação deseja receber.

---

### F2-E02 — Comunicação multicanal e notificações
**Objetivo:** Envio de mensagens por e-mail, SMS, WhatsApp e notificações automáticas.
**Prioridade:** Must · **Estimativa:** G · **Depende de:** F1-E01, F1-E03

#### F2-E02-U01
Como **secretaria**, quero enviar um comunicado institucional por e-mail, SMS e/ou WhatsApp para um grupo de responsáveis (turma, série ou toda a escola),
para garantir que a informação chegue por canais que os pais realmente usam.

**Critérios de aceite:**
- [ ] É possível selecionar destinatários por turma, série ou escola inteira.
- [ ] É possível escolher um ou mais canais de envio para a mesma mensagem.
- [ ] Histórico de envios fica registrado e consultável (o que foi enviado, para quem, quando).

#### F2-E02-U02
Como **sistema**, quero disparar automaticamente um aviso quando a frequência de um aluno cair abaixo do limite ou uma mensalidade vencer,
para alertar responsáveis sem depender de ação manual da secretaria.

**Critérios de aceite:**
- [ ] Regra de disparo automático é configurável (limite de frequência, dias antes/depois do vencimento).
- [ ] Envio automático fica registrado no mesmo histórico de comunicação de `F2-E02-U01`, para fins de comprovação.

---

### F2-E03 — Integração com meios de pagamento
**Objetivo:** Pagamento on-line de mensalidades (cartão, Pix, boleto) com baixa automática.
**Prioridade:** Must · **Estimativa:** G · **Depende de:** F1-E06

#### F2-E03-U01
Como **responsável**, quero pagar uma mensalidade on-line via Pix, cartão ou boleto,
para quitar pendências sem precisar ir a um banco ou à escola.

**Critérios de aceite:**
- [ ] Portal/app exibe título em aberto com opção de pagamento via gateway integrado.
- [ ] Ao confirmar pagamento no gateway, a baixa é registrada automaticamente no financeiro (`F1-E06-U02`) sem intervenção manual.
- [ ] Responsável recebe confirmação/recibo do pagamento realizado.

#### F2-E03-U02
Como **financeiro**, quero que lembretes automáticos de vencimento com link de pagamento sejam enviados aos responsáveis,
para reduzir inadimplência sem esforço manual recorrente.

**Critérios de aceite:**
- [ ] Lembrete é enviado X dias antes do vencimento (configurável), reutilizando os canais de `F2-E02`.
- [ ] Mensagem inclui link direto de pagamento do título específico.

---

### F2-E04 — Biblioteca e patrimônio
**Objetivo:** Cadastro de acervo, empréstimos, devoluções, multas e controle de patrimônio.
**Prioridade:** Should · **Estimativa:** M · **Depende de:** F1-E01

#### F2-E04-U01
Como **bibliotecário(a)**, quero registrar o empréstimo de um livro a um aluno já cadastrado no sistema,
para controlar o acervo sem duplicar cadastro de usuários.

**Critérios de aceite:**
- [ ] Empréstimo reutiliza o cadastro de aluno existente (`F1-E01`), sem novo cadastro manual.
- [ ] Sistema define prazo de devolução automaticamente conforme política configurada.
- [ ] Sistema calcula multa por atraso na devolução, se configurado.

#### F2-E04-U02
Como **administrador de patrimônio**, quero cadastrar equipamentos e mobiliário da escola com localização e responsável,
para manter controle do patrimônio escolar.

**Critérios de aceite:**
- [ ] Item de patrimônio tem código único, descrição, localização e status (em uso, manutenção, baixado).
- [ ] É possível gerar relatório de itens por localização/status.

---

### F2-E05 — Transporte escolar
**Objetivo:** Cadastro de veículos, motoristas, rotas e vínculo de alunos.
**Prioridade:** Should · **Estimativa:** M · **Depende de:** F1-E01, F1-E02

#### F2-E05-U01
Como **coordenação de transporte**, quero cadastrar veículos, motoristas e rotas com pontos de embarque,
para organizar a logística do transporte escolar.

**Critérios de aceite:**
- [ ] Rota tem lista ordenada de pontos de embarque/desembarque e horário estimado.
- [ ] Veículo e motorista podem ser vinculados a uma ou mais rotas.

#### F2-E05-U02
Como **secretaria**, quero vincular um aluno matriculado a uma rota de transporte,
para que o responsável saiba qual rota o filho utiliza.

**Critérios de aceite:**
- [ ] Vínculo aluno-rota é visível no portal do responsável (`F1-E08`).
- [ ] Relatório de ocupação por veículo/rota está disponível para a coordenação.

---

### F2-E06 — RH e folha de pagamento
**Objetivo:** Cadastro de colaboradores, jornada, férias, afastamentos e cálculo de folha.
**Prioridade:** Could · **Estimativa:** G · **Depende de:** F1-E01

#### F2-E06-U01
Como **RH**, quero cadastrar colaboradores com jornada de trabalho, férias e afastamentos,
para manter um controle centralizado de pessoal.

**Critérios de aceite:**
- [ ] Cadastro de colaborador inclui cargo, jornada, data de admissão e histórico de afastamentos/férias.
- [ ] Sistema alerta sobre períodos de férias vencidas conforme regra configurável.

#### F2-E06-U02
Como **RH**, quero calcular a folha de pagamento integrada aos dados de jornada e afastamentos,
para reduzir a dependência de sistemas contábeis externos para o cálculo básico.

**Critérios de aceite:**
- [ ] Cálculo de folha considera jornada, faltas/afastamentos e eventos remuneratórios cadastrados.
- [ ] Folha pode ser exportada em formato aceito por escritórios de contabilidade parceiros.

---

### F2-E07 — Multiunidade / rede escolar
**Objetivo:** Gerenciar múltiplas unidades em um mesmo ambiente com indicadores consolidados.
**Prioridade:** Should · **Estimativa:** G · **Depende de:** F1-E01 a F1-E09

#### F2-E07-U01
Como **mantenedora**, quero visualizar indicadores consolidados (matrículas, inadimplência, frequência) de todas as unidades da rede,
para comparar desempenho entre escolas.

**Critérios de aceite:**
- [ ] Dashboard consolida indicadores de `F1-E06`, `F1-E07` e `F1-E03` por unidade.
- [ ] É possível filtrar/comparar duas ou mais unidades lado a lado.

#### F2-E07-U02
Como **administrador do sistema**, quero configurar permissões por nível (mantenedora, direção de unidade, coordenação),
para que cada papel veja apenas os dados da(s) unidade(s) sob sua responsabilidade.

**Critérios de aceite:**
- [ ] Usuário de uma unidade não visualiza dados de outra unidade, salvo perfil de mantenedora/rede.
- [ ] Perfis de `F1-E09` são estendidos com o conceito de unidade sem duplicar a lógica de permissões.

---

### F2-E08 — LMS/EAD integrado básico
**Objetivo:** Ambiente virtual de aprendizagem com publicação de conteúdo, atividades e integração com Google Classroom.
**Prioridade:** Could · **Estimativa:** G · **Depende de:** F1-E02, F1-E05

#### F2-E08-U01
Como **professor**, quero publicar conteúdos e atividades para uma turma,
para apoiar aulas remotas ou complementares.

**Critérios de aceite:**
- [ ] Conteúdo/atividade publicado é visível apenas para alunos da turma-alvo.
- [ ] Aluno pode enviar resposta/arquivo para uma atividade publicada.

#### F2-E08-U02
Como **coordenação**, quero integrar turmas do sistema com o Google Classroom,
para permitir adoção gradual sem migrar tudo de uma vez.

**Critérios de aceite:**
- [ ] É possível vincular uma turma do sistema a uma turma existente no Google Classroom.
- [ ] Alunos matriculados são sincronizados com a turma do Classroom vinculada.

---

## Fase 3 — Analytics, IA & Diferenciação (Nível 3: Diferenciais)

### F3-E01 — Analytics/BI avançado e IA
**Objetivo:** Dashboards e modelos preditivos de evasão/reprovação para direção e mantenedoras.
**Prioridade:** Must · **Estimativa:** GG · **Depende de:** F1-E03, F1-E04, F1-E06

#### F3-E01-U01
Como **direção**, quero um dashboard com indicadores de aprovação, evasão, inadimplência e ocupação de vagas,
para embasar decisões estratégicas com dados atualizados.

**Critérios de aceite:**
- [ ] Dashboard consolida dados de `F1-E03`, `F1-E04`, `F1-E06` e `F1-E02` (ocupação de vagas).
- [ ] Indicadores são filtráveis por período, turma/série e (se multiunidade) por unidade.

#### F3-E01-U02
Como **coordenação**, quero identificar alunos com risco de evasão ou reprovação com base em notas, frequência e ocorrências,
para agir preventivamente antes que o problema se agrave.

**Critérios de aceite:**
- [ ] Modelo/regra de risco combina histórico de notas, frequência e ocorrências disciplinares do aluno.
- [ ] Lista de alunos em risco é acompanhada de sugestão de ação (ex.: contato com responsável, reforço).
- [ ] Resultado é explicável (mostra os fatores que levaram à classificação de risco).

---

### F3-E02 — Aderência avançada à BNCC e avaliação por competências
**Objetivo:** Mapeamento de conteúdos/avaliações para habilidades e competências da BNCC.
**Prioridade:** Should · **Estimativa:** G · **Depende de:** F1-E04

#### F3-E02-U01
Como **coordenação pedagógica**, quero vincular avaliações e conteúdos a habilidades/competências da BNCC,
para acompanhar o domínio de cada aluno por competência, não apenas por nota.

**Critérios de aceite:**
- [ ] É possível associar uma avaliação a uma ou mais habilidades BNCC do catálogo oficial.
- [ ] Relatório por aluno/turma mostra o nível de domínio por competência, não só a média geral.

---

### F3-E03 — Motor de fluxos/automações configuráveis
**Objetivo:** Regras de negócio configuráveis por usuários não técnicos (ex.: alertas, bloqueios automáticos).
**Prioridade:** Must · **Estimativa:** G · **Depende de:** F1-E03, F1-E06, F2-E02

#### F3-E03-U01
Como **coordenação**, quero configurar uma regra do tipo "se frequência < X%, notificar responsável e coordenação",
para automatizar ações preventivas sem depender de acompanhamento manual constante.

**Critérios de aceite:**
- [ ] Interface permite compor regras (condição + ação) sem necessidade de código.
- [ ] Regra configurada dispara a ação (notificação via `F2-E02`) automaticamente quando a condição é satisfeita.
- [ ] Regras ativas/inativas e seu histórico de disparos são consultáveis.

#### F3-E03-U02
Como **financeiro**, quero configurar uma regra do tipo "se mensalidade atrasada Y dias, bloquear rematrícula e enviar lembrete",
para reduzir inadimplência de forma consistente.

**Critérios de aceite:**
- [ ] Regra bloqueia automaticamente a ação de rematrícula (`F1-E02-U02`) quando a condição é atingida.
- [ ] Bloqueio pode ser removido manualmente por um usuário com permissão, com registro em auditoria (`F1-E09-U02`).

---

### F3-E04 — Transporte com GPS e segurança avançada
**Objetivo:** Rastreamento em tempo real, alertas de embarque/desembarque e rotas otimizadas.
**Prioridade:** Could · **Estimativa:** G · **Depende de:** F2-E05

#### F3-E04-U01
Como **responsável**, quero ver a posição em tempo real do ônibus escolar e a previsão de chegada,
para me organizar e ter mais segurança sobre o trajeto do meu filho.

**Critérios de aceite:**
- [ ] App/portal exibe a localização do veículo em mapa, atualizada em tempo real durante o trajeto.
- [ ] Sistema envia notificação de embarque e desembarque do aluno.

---

### F3-E05 — Experiência mobile superior
**Objetivo:** UX completa cobrindo todos os papéis com desempenho e usabilidade superiores.
**Prioridade:** Should · **Estimativa:** G · **Depende de:** F2-E01

#### F3-E05-U01
Como **gestor**, quero acompanhar os principais indicadores da escola (de `F3-E01`) diretamente pelo app mobile,
para tomar decisões mesmo fora do computador.

**Critérios de aceite:**
- [ ] App mobile inclui uma versão resumida do dashboard de `F3-E01-U01`, otimizada para tela pequena.
- [ ] Navegação e tempo de carregamento atendem a metas de performance definidas pelo time (a especificar em métricas de UX).

---

### F3-E06 — Ecossistema de integrações
**Objetivo:** Integrações nativas (Google Workspace, Microsoft 365, contábeis) e API pública documentada.
**Prioridade:** Should · **Estimativa:** GG · **Depende de:** F1-E09

#### F3-E06-U01
Como **desenvolvedor parceiro (edtech)**, quero consultar uma API pública documentada para ler dados de turmas e alunos autorizados,
para construir integrações sem depender de acesso direto ao banco de dados.

**Critérios de aceite:**
- [ ] API é documentada (ex.: OpenAPI/Swagger) e requer autenticação por token/API key.
- [ ] Acesso à API respeita os mesmos perfis/permissões de `F1-E09`.
- [ ] Existe mecanismo de webhook para notificar eventos (ex.: nova matrícula) a sistemas externos.

#### F3-E06-U02
Como **coordenação**, quero sincronizar contas de alunos/professores com Google Workspace ou Microsoft 365,
para evitar gestão duplicada de identidades.

**Critérios de aceite:**
- [ ] Provisionamento de conta no provedor externo ocorre automaticamente ao matricular/cadastrar usuário.
- [ ] Desativação de usuário no sistema reflete (ou aciona) desativação correspondente no provedor externo.

---

### F3-E07 — Gestão multi-rede com governança
**Objetivo:** Padronização curricular, metas por unidade e comparativos entre escolas para mantenedoras.
**Prioridade:** Could · **Estimativa:** G · **Depende de:** F2-E07

#### F3-E07-U01
Como **mantenedora**, quero definir metas de desempenho acadêmico e financeiro por unidade e acompanhar o cumprimento,
para direcionar investimentos e ações corretivas na rede.

**Critérios de aceite:**
- [ ] É possível cadastrar metas (ex.: taxa de aprovação, inadimplência máxima) por unidade e período.
- [ ] Painel mostra o progresso de cada unidade em relação às metas definidas, reutilizando os indicadores de `F2-E07-U01` e `F3-E01`.
