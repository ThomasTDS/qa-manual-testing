# Testes Manuais e Exploratórios

![Licença](https://img.shields.io/badge/licen%C3%A7a-MIT-blue.svg)

Portfólio de artefatos de teste manual e exploratório: design de casos de
teste com rastreabilidade de regra de negócio, relatórios de sessão de teste
exploratório (Session-Based Test Management) e relatos de defeitos bem
documentados.

Este repositório é complementar ao portfólio de automação de testes do
mesmo autor e está em construção — o conteúdo será adicionado de forma
incremental.

## Destaques

Três achados que resumem o tipo de raciocínio que este portfólio tenta demonstrar — o processo por trás de cada um importa mais do que a lista em si:

- **[Uma conta que fica pra trás sem avisar (BugBank)](relatorios-de-bugs/bug-cad-001-recadastro-sem-verificar-email-duplicado.md).** O BugBank não tem botão de "esqueci minha senha". O caminho mais óbvio que um usuário real tentaria — se cadastrar de novo com o mesmo e-mail — é aceito sem nenhum aviso de duplicidade, e cria uma segunda conta completamente desconectada da primeira. O achado não veio de testar um cenário óbvio de bug; veio de simular o comportamento de alguém que só quer voltar a acessar a própria conta.

- **[Um empréstimo aprovado que deixa a conta no vermelho (ParaBank)](relatorios-de-bugs/bug-emp-002-debito-sem-validar-conta-origem.md).** A regra de aprovação do ParaBank soma o saldo de todas as contas do cliente para decidir se aprova um empréstimo, mas debita a entrada de apenas uma conta específica, sem checar se ela sozinha tem saldo suficiente. Um pedido aprovado corretamente pela regra pode deixar uma conta com saldo negativo, sem aviso — e o efeito se acumula em pedidos futuros.

- **Uma hipótese testada e descartada, não só confirmada.** Nem todo risco sinalizado num design de teste vira defeito: o valor de empréstimo negativo, apontado como candidato a inconsistência no ParaBank, foi executado de verdade e a aplicação lidou com ele corretamente. Documentar isso importa tanto quanto documentar um bug — mostra que a conclusão veio de execução real, não de suposição.

Como cada achado foi levantado (leitura de código-fonte, especificação oficial da aplicação, execução real) está detalhado em [`metodologia.md`](metodologia.md).

## Estrutura

| Pasta | Conteúdo |
| --- | --- |
| [`casos-de-teste/`](casos-de-teste/) | Design de casos de teste com rastreabilidade de regra de negócio, documentando a técnica utilizada (tabela de decisão, particionamento de equivalência, análise de valor limite) para cada funcionalidade analisada |
| [`sessoes-exploratorias/`](sessoes-exploratorias/) | Relatórios de sessão de teste exploratório (Session-Based Test Management): charter, cobertura por área, registro cronológico de teste e defeitos confirmados |
| [`relatorios-de-bugs/`](relatorios-de-bugs/) | Relatos de defeitos reais encontrados durante as sessões de teste, com passos de reprodução, evidência, análise técnica de causa e justificativa de severidade/prioridade |

## Artefatos

### ParaBank

- [Solicitar Empréstimo](casos-de-teste/parabank-solicitar-emprestimo.md):
  regra de negócio extraída diretamente do código-fonte oficial da
  aplicação, tabela de decisão e dezesseis casos de teste rastreados até a
  regra que cada um valida.
- [Sessão exploratória — Solicitar Empréstimo](sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md):
  execução real contra a aplicação, com dois defeitos confirmados (erro
  interno com valor de empréstimo zero e débito de entrada sem checagem
  de saldo individual da conta de origem) e uma correção aplicada ao
  artefato de design de casos de teste a partir do que foi observado.
- [BUG-EMP-001 — Valor de empréstimo zero causa erro interno do servidor](relatorios-de-bugs/bug-emp-001-valor-zero-erro-500.md):
  severidade alta, prioridade média.
- [BUG-EMP-002 — Aprovação de empréstimo permite saldo negativo na conta de origem](relatorios-de-bugs/bug-emp-002-debito-sem-validar-conta-origem.md):
  severidade crítica, prioridade alta, com efeito cascata confirmado sobre
  pedidos de empréstimo futuros.

### BugBank

- [Transferência](casos-de-teste/bugbank-transferencia.md):
  regra de negócio extraída da especificação oficial publicada pela
  própria aplicação, tabela de decisão com três condições combinadas e
  catorze casos de teste, incluindo cenários que a especificação não
  cobre explicitamente.
- [Sessão exploratória — Cadastro, Login e Transferência](sessoes-exploratorias/bugbank-transferencia-22-09-2026.md):
  execução manual pela interface, com dois defeitos confirmados (conta
  duplicada ao recadastrar com o mesmo e-mail, sem fluxo de recuperação
  de senha; e validação inconsistente do formulário de transferência,
  incluindo uma mensagem técnica de biblioteca exposta ao usuário) e a
  confirmação de uma regra funcionando corretamente.
- [BUG-CAD-001 — Recadastro com e-mail já existente cria conta paralela](relatorios-de-bugs/bug-cad-001-recadastro-sem-verificar-email-duplicado.md):
  severidade alta, prioridade alta.
- [BUG-TRF-001 — Validação do formulário de transferência é inconsistente](relatorios-de-bugs/bug-trf-001-validacao-formulario-inconsistente.md):
  severidade média, prioridade média.

## Autor

Desenvolvido por [Thomas Teixeira](https://github.com/ThomasTDS) como
projeto de estudo e portfólio em teste de software.
