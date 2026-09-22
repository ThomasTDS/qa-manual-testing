# Relato de bug — Solicitação de empréstimo com valor zero causa erro interno do servidor

## Metadados

| Campo | Valor |
| --- | --- |
| ID | BUG-EMP-001 |
| Aplicação | ParaBank (https://parabank.parasoft.com) |
| Funcionalidade | Solicitar Empréstimo (*Request Loan*) |
| Ambiente | Instância pública de produção/demonstração |
| Data de identificação | 22/09/2026 |
| Identificado por | Thomas Teixeira, durante sessão de teste exploratório |
| Severidade | Alta |
| Prioridade | Média |
| Status | Aberto (não reportado ao mantenedor — instância pública de terceiros, mantida como registro de portfólio) |

## Resumo

Ao solicitar um empréstimo informando o valor "0" (zero) no campo "Loan Amount", a aplicação não retorna uma mensagem de reprovação controlada, como faz para os demais casos de fundos insuficientes. Em vez disso, a requisição falha com um erro interno do servidor (HTTP 500) e a aplicação exibe uma página de erro genérica, sem qualquer orientação ao usuário sobre o que deu errado.

## Ambiente e pré-condições

- Cliente autenticado no ParaBank, com pelo menos uma conta Corrente ou Poupança.
- Nenhuma condição especial de saldo é necessária: o defeito ocorre independentemente do valor disponível em conta.

## Passos para reproduzir

1. Fazer login no ParaBank com um cliente válido.
2. Acessar o menu "Request Loan".
3. Preencher o campo "Loan Amount" com o valor `0`.
4. Preencher o campo "Down Payment" com qualquer valor válido (por exemplo, `0`).
5. Selecionar qualquer conta em "From Account Number".
6. Clicar em "Apply Now".

## Resultado esperado

A aplicação deveria reprovar o pedido com uma mensagem controlada — por exemplo, reaproveitando a mensagem já existente para fundos insuficientes, ou uma mensagem específica de valor inválido — sem expor um erro de servidor ao usuário.

## Resultado obtido

A requisição retorna **HTTP 500**, e a interface exibe a tela genérica de erro do ParaBank ("Error! / An internal error has occurred and has been logged."), idêntica à usada para falhas inesperadas de infraestrutura — não uma mensagem de negócio como "reprovado por fundos insuficientes".

## Evidência

Requisição enviada diretamente ao serviço REST consumido internamente pela tela (mesma chamada que o botão "Apply Now" dispara via AJAX), contra o cliente de teste registrado durante a sessão exploratória (id 15098, conta corrente 16563, saldo 515,50 no momento do teste):

```
POST /parabank/services_proxy/bank/requestLoan?customerId=15098&amount=0&downPayment=0&fromAccountId=16563
```

Resposta obtida (HTTP 500):

```html
<h1 class="title">Error!</h1>
<p class="error">An internal error has occurred and has been logged.</p>
```

Para efeito de comparação, o mesmo endpoint retorna corretamente um erro de negócio estruturado (HTTP 200 com corpo JSON `{"approved": false, "message": "error.insufficient.funds", ...}`) quando o valor do empréstimo é positivo e apenas os fundos são insuficientes — o que reforça que o comportamento com valor zero é uma falha de tratamento, não a resposta padrão da aplicação para reprovação.

## Dados de teste utilizados

| Campo | Valor |
| --- | --- |
| Valor do empréstimo | 0,00 |
| Valor de entrada | 0,00 |
| Conta de origem | Conta corrente com saldo 515,50 |

O defeito não depende desses valores específicos de saldo ou de entrada — apenas do valor do empréstimo ser exatamente zero.

## Análise técnica (causa provável)

A regra de aprovação vigente ("Available Funds") calcula a razão entre os fundos disponíveis do cliente e o valor do empréstimo solicitado (`fundosDisponiveis ÷ valorDoEmprestimo`) para compará-la ao limiar de aprovação. Quando o valor do empréstimo é zero, essa divisão é matematicamente indefinida. Pela forma como a mensagem de erro genérica aparece — idêntica à usada em outras falhas não tratadas da aplicação, e não a uma das mensagens de negócio conhecidas ("insufficient funds", "insufficient down payment" etc.) — o comportamento é consistente com uma exceção aritmética (divisão por zero) não capturada em nenhum ponto entre o cálculo da regra e a resposta ao cliente, que acaba sendo tratada apenas pelo mecanismo genérico de erro do servidor.

## Justificativa de severidade e prioridade

**Severidade: Alta.** A aplicação não retorna uma resposta controlada para uma entrada plausível de ser digitada por um usuário real (ou colada/corrigida incorretamente), quebrando o fluxo principal da funcionalidade nesse caso específico e expondo uma tela de erro genérica em vez de uma orientação de negócio.

**Prioridade: Média.** O cenário exige uma entrada bem específica (exatamente zero), que não representa o caminho mais comum de uso da funcionalidade. Não há indício de comprometimento de dados ou de segurança — nenhuma conta ou empréstimo é criado quando o erro ocorre. O impacto é sobre a robustez percebida da aplicação e a experiência do usuário nesse caso de borda, não sobre a integridade dos dados financeiros.

## Rastreabilidade

- Caso de teste de origem: `CT-EMP-13`, em [`casos-de-teste/parabank-solicitar-emprestimo.md`](../casos-de-teste/parabank-solicitar-emprestimo.md).
- Execução que confirmou o defeito: sessão registrada em [`sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md`](../sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md), item das 18:34 UTC.
