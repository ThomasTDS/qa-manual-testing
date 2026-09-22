# Relato de bug — Aprovação de empréstimo permite saldo negativo na conta de origem, sem validação

## Metadados

| Campo | Valor |
| --- | --- |
| ID | BUG-EMP-002 |
| Aplicação | ParaBank (https://parabank.parasoft.com) |
| Funcionalidade | Solicitar Empréstimo (*Request Loan*) |
| Ambiente | Instância pública de produção/demonstração |
| Data de identificação | 22/09/2026 |
| Identificado por | Thomas Teixeira, durante sessão de teste exploratório |
| Severidade | Crítica |
| Prioridade | Alta |
| Status | Aberto (não reportado ao mantenedor — instância pública de terceiros, mantida como registro de portfólio) |

## Resumo

A regra de aprovação de empréstimo verifica se o cliente tem fundos suficientes somando o saldo de **todas** as suas contas Corrente e Poupança. Porém, ao aprovar o pedido, o valor de entrada (*down payment*) é debitado apenas da **conta específica** escolhida pelo cliente em "From Account Number" — sem checar se essa conta, isoladamente, tem saldo suficiente para cobrir a entrada. O resultado é que um cliente com múltiplas contas pode ter um empréstimo aprovado e, no mesmo processo, ficar com uma de suas contas em saldo negativo, sem nenhum aviso, bloqueio ou confirmação adicional.

## Ambiente e pré-condições

- Cliente autenticado no ParaBank, com **duas ou mais contas** (Corrente e/ou Poupança).
- Uma das contas do cliente precisa ter saldo menor do que o valor de entrada que será informado, enquanto a soma de todas as contas do cliente precisa ser suficiente para cobrir essa mesma entrada e atender à razão mínima de aprovação (fundos disponíveis ÷ valor do empréstimo ≥ 20%, na configuração vigente da instância).

## Passos para reproduzir

1. Fazer login no ParaBank com um cliente que tenha pelo menos duas contas (ou abrir uma segunda conta em "Open New Account" a partir de uma conta já existente).
2. Confirmar, em "Accounts Overview", que uma das contas tem saldo baixo e a outra tem saldo mais alto, de forma que a soma das duas seja bem maior que o saldo da conta de menor valor.
3. Acessar "Request Loan".
4. Informar um valor de empréstimo cuja razão em relação à **soma de todas as contas** seja igual ou superior a 20%.
5. Informar um valor de entrada maior do que o saldo da conta de menor valor, mas menor do que a soma de todas as contas do cliente.
6. Em "From Account Number", selecionar justamente a conta de **menor saldo** (a que não cobriria a entrada sozinha).
7. Clicar em "Apply Now".
8. Após a confirmação, voltar a "Accounts Overview" e conferir o saldo da conta selecionada no passo 6.

## Resultado esperado

Uma das duas coisas deveria acontecer: o sistema deveria reprovar o pedido por saldo insuficiente **na conta de origem selecionada**, ou deveria impedir a seleção de uma conta que, sozinha, não cobre o valor de entrada informado. Em nenhum cenário uma conta bancária deveria terminar com saldo negativo como resultado direto de uma operação aprovada pelo próprio sistema, sem aviso prévio ao cliente.

## Resultado obtido

O pedido é aprovado normalmente (`approved: true`), uma nova conta de empréstimo é criada, e o valor de entrada é debitado integralmente da conta selecionada — que passa a ter **saldo negativo**. Esse saldo negativo aparece normalmente na coluna "Balance" tanto da tela "Accounts Overview" quanto da tela de extrato ("Account Activity") daquela conta; apenas uma coluna separada, "Available Balance", exibe zero nesse caso, o que não é suficiente para alertar o cliente de que sua conta está, de fato, no negativo.

## Evidência

Cliente de teste da sessão exploratória (id 15098), com duas contas antes da solicitação:

| Conta | Tipo | Saldo antes |
| --- | --- | --- |
| 16563 | Corrente | 415,50 |
| 16674 | Poupança | 100,00 |

Requisição enviada diretamente ao serviço REST consumido internamente pela tela (mesma chamada que o botão "Apply Now" dispara via AJAX), selecionando a conta poupança (saldo 100,00) como origem, com entrada de 300,00:

```
POST /parabank/services_proxy/bank/requestLoan?customerId=15098&amount=1000&downPayment=300&fromAccountId=16674
```

Resposta obtida (HTTP 200):

```json
{"responseDate":1790102108498,"loanProviderName":"Wealth Securities Dynamic Loans (WSDL)","approved":true,"accountId":16785}
```

Saldos consultados logo em seguida (`GET /parabank/services_proxy/bank/customers/15098/accounts`):

```json
[
  {"id":16563,"customerId":15098,"type":"CHECKING","balance":415.50},
  {"id":16674,"customerId":15098,"type":"SAVINGS","balance":-200.00},
  {"id":16785,"customerId":15098,"type":"LOAN","balance":1000.00}
]
```

A conta poupança (16674), que tinha 100,00, ficou com **-200,00** (100,00 − 300,00 de entrada), sem que a aprovação do empréstimo tivesse sido bloqueada ou sinalizada de nenhuma forma.

### Efeito cascata confirmado

O saldo negativo criado por este defeito passa a ser somado normalmente ao cálculo de fundos disponíveis do cliente em solicitações futuras — o que foi confirmado logo em seguida, na mesma sessão: com o cliente já nesse estado (fundos agregados de 415,50 + (-200,00) = 215,50), um novo pedido de empréstimo de 1.000,00 foi aprovado (razão 21,55%) e um de 1.100,00 foi reprovado (razão 19,59%), exatamente como a fórmula prevê considerando o saldo negativo na soma. Ou seja, o problema não fica isolado na conta afetada: ele reduz de forma silenciosa a capacidade de crédito do cliente em pedidos seguintes.

## Análise técnica (causa provável)

A verificação de fundos suficientes para a entrada compara o valor informado com `availableFunds`, calculado como a soma dos saldos de **todas** as contas não-Empréstimo do cliente. Essa soma agregada é suficiente para aprovar o pedido. No entanto, ao efetivar a aprovação, o débito da entrada é aplicado diretamente à conta identificada por `fromAccountId` — a conta especificamente escolhida pelo cliente — sem que exista, em nenhum ponto do fluxo, uma verificação de que **essa conta em particular** tem saldo suficiente para a operação. O método responsável por debitar a conta subtrai o valor do saldo diretamente, sem nenhuma proteção contra saldo negativo. Em resumo: a regra de aprovação valida o todo, mas a operação afeta apenas uma parte, sem revalidar essa parte antes de executá-la.

## Justificativa de severidade e prioridade

**Severidade: Crítica.** Trata-se de uma aplicação bancária aprovando uma operação de crédito e, no mesmo fluxo, deixando uma conta do cliente com saldo negativo sem qualquer aviso, confirmação ou possibilidade de o cliente evitar isso pela interface. Isso é uma falha de integridade financeira, não apenas de usabilidade — o tipo de comportamento que, em um sistema bancário real, geraria disputa de cliente, prejuízo contábil e risco regulatório. O efeito cascata confirmado (o saldo negativo afetando a capacidade de crédito em pedidos futuros) agrava ainda mais o cenário, pois o problema se acumula silenciosamente em vez de ficar contido.

**Prioridade: Alta.** Qualquer aplicação financeira real precisaria corrigir esse comportamento antes de ir a produção. A reprodução não depende de nenhuma condição incomum — apenas de o cliente ter mais de uma conta, algo perfeitamente normal em um banco.

## Rastreabilidade

- Regra de negócio relacionada: `RN-06`, em [`casos-de-teste/parabank-solicitar-emprestimo.md`](../casos-de-teste/parabank-solicitar-emprestimo.md).
- Caso de teste de origem: `CT-EMP-15`, no mesmo documento.
- Execução que confirmou o defeito: sessão registrada em [`sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md`](../sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md), itens das 18:35 UTC (aprovação) e 18:36 UTC (efeito cascata confirmado).
