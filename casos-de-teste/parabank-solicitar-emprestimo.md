# Design de casos de teste — ParaBank: Solicitar Empréstimo

## Metadados

| Campo | Valor |
| --- | --- |
| Aplicação sob teste | ParaBank (https://parabank.parasoft.com) |
| Funcionalidade | Solicitar Empréstimo (*Request Loan*) |
| Tipo de artefato | Design de casos de teste com rastreabilidade de regra de negócio |
| Técnicas aplicadas | Tabela de decisão, particionamento de equivalência, análise de valor limite |
| Data do levantamento | 22/09/2026 |

## Objetivo

Documentar a regra de negócio real por trás da aprovação de empréstimo do ParaBank e derivar, a partir dela, um conjunto de casos de teste rastreáveis — incluindo cenários positivos, negativos e de fronteira — usando técnicas formais de design de teste.

## Fonte da regra de negócio

O ParaBank é uma aplicação de código aberto mantida pela Parasoft, disponível em `github.com/parasoft/parabank`. A regra de aprovação de empréstimo não é documentada publicamente em nenhuma página de ajuda da aplicação; ela foi extraída diretamente das classes responsáveis pela decisão de crédito:

- `BankManagerImpl.requestLoan` — calcula os fundos disponíveis do cliente e monta a solicitação.
- `AbstractLoanProcessor.requestLoan` — aplica o filtro de entrada (*down payment*) e delega a análise de fundos ao processador configurado.
- `AvailableFundsLoanProcessor` — processador de crédito atualmente configurado na instância pública; calcula a razão entre fundos disponíveis e valor solicitado.
- `Account.debit` — confirma que não existe proteção contra saldo negativo ao debitar uma conta.

A configuração vigente (Provedor de Empréstimo, Processador de Crédito e Limiar) foi conferida diretamente no painel `/parabank/admin.htm` da instância pública na data do levantamento:

| Parâmetro | Valor confirmado em produção |
| --- | --- |
| Loan Provider | Web Service |
| Loan Processor | Available Funds |
| Loan Processor Threshold | 20% |
| Initial Balance (saldo de conta nova) | 515,50 |
| Minimum Balance | 100,00 |

> **Atenção:** o painel de administração é público e pode ser alterado por qualquer pessoa que acesse a instância compartilhada. Antes de executar os casos abaixo, reconfirme esses valores em `/parabank/admin.htm`, pois uma mudança no limiar ou no processador invalida os dados de teste numéricos apresentados neste documento.

Também foi confirmado, pela leitura do JSP da tela (`requestloan.jsp`) e da assinatura do serviço REST (`ParaBankService.requestLoan`), que o botão "Apply Now" **não passa pelo validador Spring** (`RequestLoanFormValidator`) que existe no código-fonte — esse validador pertence a um fluxo MVC legado que não é mais alcançado pela interface atual. O envio é feito via chamada AJAX direta ao serviço REST, com `amount` e `downPayment` tipados como `BigDecimal` no parâmetro de consulta. Isso significa que não há validação de campo específica no cliente nem no servidor para esse formulário: qualquer falha de conversão (campo vazio, texto não numérico) é tratada como erro genérico de requisição.

## Regras de negócio identificadas

| ID | Regra |
| --- | --- |
| RN-01 | Os campos "Valor do empréstimo" e "Valor de entrada" são obrigatórios para o envio, mas a tela não valida o preenchimento antes de enviar a requisição. Se ficarem vazios ou contiverem valor não numérico, a conversão falha no servidor e a aplicação exibe uma tela de erro genérica ("Error! / An internal error has occurred and has been logged."), sem indicar qual campo está incorreto. |
| RN-02 | A conta de origem (`fromAccountId`) é escolhida em uma lista suspensa preenchida com as contas do cliente autenticado; não é possível deixá-la vazia pela interface. |
| RN-03 | O valor de entrada não pode ser maior que o total de fundos disponíveis do cliente — a soma dos saldos de **todas** as contas Corrente e Poupança do cliente, excluindo contas de Empréstimo já existentes. Se violada, o pedido é reprovado com a mensagem *"You do not have sufficient funds for the given down payment."*, independentemente de qualquer outra condição. |
| RN-04 | Na configuração vigente (processador "Available Funds", limiar 20%), o pedido só é aprovado se a razão entre os fundos disponíveis totais e o valor do empréstimo solicitado for maior ou igual a 20%. Se violada, o pedido é reprovado com a mensagem *"We cannot grant a loan in that amount with your available funds."* O valor de entrada **não** entra nesse cálculo. |
| RN-05 | Quando RN-03 e RN-04 são satisfeitas, o pedido é aprovado (*"Congratulations, your loan has been approved."*), uma nova conta do tipo Empréstimo é criada com saldo igual ao valor solicitado, e o valor de entrada é debitado da conta de origem selecionada. |
| RN-06 — **confirmada por execução real em 22/09/2026** | RN-03 usa o saldo agregado de todas as contas do cliente, mas o débito de RN-05 é aplicado apenas à conta de origem selecionada, sem verificar se essa conta específica possui saldo suficiente isoladamente. Isso permite que a conta de origem fique com saldo negativo mesmo quando o pedido é aprovado corretamente pelas regras RN-03/RN-04. Reproduzido na sessão exploratória registrada em [`sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md`](../sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md). |
| RN-07 — **descoberta na execução real, não visível apenas pela leitura do código** | A razão entre fundos disponíveis e valor do empréstimo (RN-04) não é comparada em sua forma bruta: o sistema a arredonda para 3 casas decimais (arredondamento round-half-up) antes de compará-la ao limiar de 20%. Isso cria uma margem de tolerância de alguns centavos ao redor do valor teórico de corte, dentro da qual o resultado não muda. Detalhe também confirmado na sessão exploratória referenciada acima. |

## Tabela de decisão — RN-03 × RN-04

A aprovação depende da combinação de duas condições binárias. O algoritmo verifica RN-03 primeiro e retorna imediatamente se ela falhar, o que faz duas das quatro combinações produzirem o mesmo resultado visível ao usuário.

| Regra | C1: Entrada ≤ fundos disponíveis? | C2: Fundos disponíveis ÷ empréstimo ≥ 20%? | Resultado |
| --- | --- | --- | --- |
| R1 | Não | Não | Reprovado — mensagem de entrada insuficiente |
| R2 | Não | Sim | Reprovado — mensagem de entrada insuficiente (C2 nunca é avaliada, pois a função retorna antes) |
| R3 | Sim | Não | Reprovado — mensagem de fundos insuficientes |
| R4 | Sim | Sim | **Aprovado** |

R1 e R2 foram mantidas como casos de teste separados (ver CT-EMP-07 e CT-EMP-08) porque, embora produzam o mesmo resultado visível hoje, testam dados de entrada diferentes e continuam válidas para detectar uma regressão caso a ordem de avaliação das condições mude no futuro.

## Casos de teste

Pré-condição comum a todos os casos: cliente autenticado no ParaBank, com as contas e saldos indicados na coluna "Dados de entrada" já configurados antes da execução. Valores em dólares (USD), formato da própria aplicação.

| ID | Cenário | Técnica | Dados de entrada | Resultado esperado | Prioridade |
| --- | --- | --- | --- | --- | --- |
| CT-EMP-01 | Aprovação com todas as condições folgadas (cenário principal) | Particionamento de equivalência | Fundos: 515,50 (conta única) · Empréstimo: 1.000,00 · Entrada: 100,00 | Razão = 51,55% (≥ 20%); entrada ≤ fundos → **Aprovado**. Nova conta de Empréstimo criada com saldo 1.000,00; conta de origem debitada em 100,00 | Alta |
| CT-EMP-02 | Fundos disponíveis exatamente no limiar de 20% (fronteira válida) | Análise de valor limite | Fundos: 515,50 · Empréstimo: 2.577,50 · Entrada: 0,00 | Razão = 20,00% exatos (≥ 20%) → **Aprovado** | Alta |
| CT-EMP-03 *(valores corrigidos após a sessão exploratória — ver RN-07)* | Fundos disponíveis logo abaixo da margem de tolerância do arredondamento (fronteira inválida real) | Análise de valor limite | Fundos: 515,50 · Empréstimo: 2.584,47 · Entrada: 0,00 | Razão bruta ≈ 19,9500%, arredondada para 19,9% (< 20%) → **Reprovado** ("insufficient funds"). Valores de R$ 2.577,51 a R$ 2.584,46 — que pareceriam reprovar por uma leitura ingênua da razão bruta — na verdade ainda arredondam para 20,0% e são aprovados; ver nota RN-07 | Alta |
| CT-EMP-04 *(valores corrigidos após a sessão exploratória — ver RN-07)* | Fundos disponíveis exatamente na borda da margem de tolerância do arredondamento (fronteira válida real) | Análise de valor limite | Fundos: 515,50 · Empréstimo: 2.584,46 · Entrada: 0,00 | Razão bruta ≈ 19,9501%, arredondada para 20,0% (≥ 20%) → **Aprovado** — R$ 0,01 de diferença para CT-EMP-03 já muda o resultado | Média |
| CT-EMP-05 | Fundos muito abaixo do limiar (partição inválida "baixa") | Particionamento de equivalência | Fundos: 515,50 · Empréstimo: 10.000,00 · Entrada: 0,00 | Razão = 5,155% (< 20%) → **Reprovado** | Média |
| CT-EMP-06 | Entrada exatamente igual ao total de fundos disponíveis (fronteira válida de RN-03) | Análise de valor limite | Fundos: 515,50 · Empréstimo: 1.000,00 · Entrada: 515,50 | C1: 515,50 ≤ 515,50 satisfeita; C2 satisfeita (51,55%) → **Aprovado** | Alta |
| CT-EMP-07 | Entrada R$ 0,01 maior que o total de fundos disponíveis, com C2 satisfeita (regra R2 da tabela de decisão) | Tabela de decisão | Fundos: 515,50 · Empréstimo: 1.000,00 · Entrada: 515,51 | C1 violada → **Reprovado** ("insufficient funds for down payment"), mesmo com C2 satisfeita | Alta |
| CT-EMP-08 | Entrada maior que os fundos e fundos insuficientes para o empréstimo (regra R1 da tabela de decisão) | Tabela de decisão | Fundos: 200,00 · Empréstimo: 5.000,00 · Entrada: 300,00 | C1 e C2 violadas → **Reprovado**, mensagem de entrada insuficiente prevalece (retorno antecipado do algoritmo) | Média |
| CT-EMP-09 | Entrada igual a zero, com fundos suficientes para a razão mínima (demonstra que a entrada não interfere em RN-04) | Particionamento de equivalência | Fundos: 515,50 · Empréstimo: 1.000,00 · Entrada: 0,00 | Razão = 51,55% → **Aprovado** mesmo com entrada zero | Alta |
| CT-EMP-10 | Campo "Valor do empréstimo" vazio | Particionamento de equivalência (classe inválida) | Empréstimo: *(vazio)* · Entrada: 100,00 | Falha de conversão no servidor; tela de erro genérica exibida ("Error! / An internal error has occurred and has been logged."), sem indicar o campo | Alta |
| CT-EMP-11 | Campo "Valor de entrada" vazio | Particionamento de equivalência (classe inválida) | Empréstimo: 1.000,00 · Entrada: *(vazio)* | Mesmo comportamento de CT-EMP-10 | Alta |
| CT-EMP-12 | Valor do empréstimo com caractere não numérico | Particionamento de equivalência (classe inválida) | Empréstimo: "abc" · Entrada: 100,00 | Falha de conversão do parâmetro `BigDecimal` no servidor; tela de erro genérica exibida | Média |
| CT-EMP-13 | Valor do empréstimo igual a zero | Análise de valor limite (fronteira não coberta pela especificação) | Empréstimo: 0,00 · Entrada: 0,00 | **Confirmado por execução real em 22/09/2026**: a requisição retorna HTTP 500 com página de erro genérica ("Error! / An internal error has occurred and has been logged."), evidenciando exceção não tratada (divisão por zero) em vez de uma rejeição controlada. Defeito registrado — ver sessão exploratória referenciada | Alta |
| CT-EMP-14 | Valor do empréstimo negativo | Particionamento de equivalência (classe inválida, não coberta pela especificação) | Empréstimo: -1.000,00 · Entrada: 100,00 | **Testado e refutado em 22/09/2026**: o sistema não quebra nem aprova indevidamente. A razão fica negativa (sempre abaixo do limiar positivo de 20%) e o pedido é reprovado normalmente com a mensagem "insufficient funds", tanto com entrada zero quanto com entrada positiva. Não é uma inconsistência | Alta |
| CT-EMP-15 | Conta de origem com saldo individual insuficiente, mas soma de todas as contas do cliente suficiente (RN-06) | Tabela de decisão + contas múltiplas | Cliente com 2 contas — Corrente: 415,50 / Poupança: 100,00 (fundos totais: 515,50) · Empréstimo: 1.000,00 · Entrada: 300,00 · Conta de origem selecionada: Poupança | **Confirmado por execução real em 22/09/2026**: C1 e C2 satisfeitas pelo saldo agregado → pedido **aprovado**. A entrada foi debitada apenas da conta de origem selecionada, que ficou com saldo negativo (100,00 − 300,00 = −200,00), visível tanto no resumo de contas quanto no extrato da conta, sem qualquer bloqueio. Defeito confirmado — ver sessão exploratória referenciada | Alta |
| CT-EMP-16 | Valor do empréstimo de grande magnitude, com fundos proporcionalmente suficientes (valor extremo dentro da partição válida) | Particionamento de equivalência (valor extremo) | Fundos: 1.000.000,00 (saldo consolidado após depósitos de preparação) · Empréstimo: 4.500.000,00 · Entrada: 0,00 | Razão ≈ 22,2% (≥ 20%) → **Aprovado**; valida ausência de erro de arredondamento ou *overflow* em valores grandes (`BigDecimal` não tem limite teórico de magnitude) | Baixa |

## Rastreabilidade regra → casos de teste

| Regra | Casos de teste que a cobrem |
| --- | --- |
| RN-01 | CT-EMP-10, CT-EMP-11, CT-EMP-12 |
| RN-02 | Coberta apenas por inspeção da interface (lista suspensa sempre preenchida); não há caso de teste dedicado, pois não é possível reproduzir o campo vazio pela tela |
| RN-03 | CT-EMP-01, CT-EMP-06, CT-EMP-07, CT-EMP-08, CT-EMP-15 |
| RN-04 | CT-EMP-01, CT-EMP-02, CT-EMP-03, CT-EMP-04, CT-EMP-05, CT-EMP-08, CT-EMP-09, CT-EMP-16 |
| RN-05 | CT-EMP-01, CT-EMP-02, CT-EMP-04, CT-EMP-06, CT-EMP-09, CT-EMP-15 |
| RN-06 | CT-EMP-15 |
| RN-07 | CT-EMP-02, CT-EMP-03, CT-EMP-04 |
| Sem regra correspondente na especificação (testes de valor não documentado) | CT-EMP-13, CT-EMP-14 |

## Observações e riscos

- Os casos CT-EMP-13, CT-EMP-14 e CT-EMP-15 foram executados de fato contra a instância pública em 22/09/2026, na sessão de teste exploratório registrada em [`sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md`](../sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md). Dois deles confirmaram defeitos reais (CT-EMP-13 e CT-EMP-15); o terceiro (CT-EMP-14) foi testado e refutado como inconsistência.
- A execução real também revelou uma regra que não era visível apenas pela leitura do código (RN-07): o arredondamento da razão fundos/empréstimo para 3 casas decimais antes da comparação com o limiar, o que exigiu corrigir os valores originais de CT-EMP-03 e CT-EMP-04.
- A ausência de validação de campo obrigatório na tela (RN-01) é, por si só, uma lacuna de usabilidade: o usuário final recebe uma mensagem genérica de erro interno em vez de uma indicação clara de que esqueceu de preencher um campo. A execução real confirmou que o servidor até retorna um detalhe técnico específico (ex.: "Required parameter 'amount' is not present."), mas a interface descarta essa informação e exibe sempre a mesma mensagem genérica.
- Os valores numéricos de fronteira (CT-EMP-02 a CT-EMP-04, CT-EMP-06, CT-EMP-07) dependem do saldo inicial padrão de conta nova (515,50) e do limiar de 20% configurados na instância pública na data deste levantamento. Qualquer alteração desses parâmetros no painel de administração invalida os valores exatos e exige recalcular os casos.
