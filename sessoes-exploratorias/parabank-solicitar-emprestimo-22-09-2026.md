# Sessão de teste exploratório — ParaBank: Solicitar Empréstimo

## Ficha da sessão

| Campo | Valor |
| --- | --- |
| Aplicação sob teste | ParaBank (https://parabank.parasoft.com) |
| Área | Solicitar Empréstimo (*Request Loan*) e contas correlatas |
| Testador | Thomas Teixeira |
| Data | 22/09/2026 |
| Início | 18:33 UTC |
| Término | 18:38 UTC |
| Duração | ≈ 5 minutos de execução cronometrada |
| Cliente de teste | usuário `qa_sess_1790102048`, id de cliente 15098, criado especificamente para esta sessão |
| Referência | Continuação do documento [`casos-de-teste/parabank-solicitar-emprestimo.md`](../casos-de-teste/parabank-solicitar-emprestimo.md) |

## Metodologia

A sessão foi conduzida por chamadas HTTP diretas (login, registro de cliente, abertura de conta e solicitação de empréstimo) aos mesmos endpoints que a própria interface do ParaBank consome internamente — o formulário de empréstimo, por exemplo, envia a solicitação via chamada AJAX a um serviço REST, e foi esse serviço que foi exercitado diretamente. A opção por essa abordagem, em vez de clicar manualmente na interface, tem dois motivos: os cenários de fronteira desta charter exigem valores numéricos exatos (até o centavo), difíceis de garantir com digitação manual repetida; e o ciclo de formular uma hipótese, testá-la e observar o resultado fica bem mais rápido, permitindo cobrir mais variações dentro do tempo da sessão. Por isso a duração cronológica é curta perto do que uma sessão manual levaria — a profundidade da investigação não é.

Consequência prática: o ambiente é a instância pública de demonstração da Parasoft, compartilhada com qualquer outra pessoa testando no mesmo momento. O cliente de teste, as contas e os empréstimos criados nesta sessão permanecem no banco de dados da instância — não há como fazer limpeza (*teardown*) dos dados a partir da interface disponível.

## Charter

> Explorar a funcionalidade "Solicitar Empréstimo" do ParaBank em busca de inconsistências na regra de aprovação, com foco nos três cenários sinalizados como "a confirmar" no documento de design de casos de teste anterior (valor de empréstimo igual a zero, valor de empréstimo negativo, conta de origem com saldo individual insuficiente frente ao saldo agregado do cliente), e seguindo qualquer pista relevante que surgisse durante a exploração.

## Cobertura por área

| Área | Proporção aproximada da sessão |
| --- | --- |
| Configuração de sessão (registro de cliente de teste, abertura de conta poupança) | 25% |
| Design e execução de cenários de teste | 45% |
| Investigação e confirmação de defeitos | 30% |

## Notas de teste (registro cronológico)

| Horário (UTC) | Ação | Dado de entrada | Resultado observado |
| --- | --- | --- | --- |
| 18:33 | Registro de cliente de teste | Formulário de cadastro padrão | Duas tentativas iniciais falharam com erro 500 genérico ao enviar o POST diretamente, sem antes buscar o formulário (sessão HTTP não estabelecida). Terceira tentativa, reaproveitando a sessão de um GET anterior, teve sucesso — cliente 15098 criado, com uma conta corrente (id 16563) e saldo inicial de 515,50, confirmando o valor padrão já documentado no artefato anterior |
| 18:34 | CT-EMP-13 — Empréstimo com valor zero | Empréstimo: 0,00 · Entrada: 0,00 | **HTTP 500**, página de erro genérica do ParaBank ("Error! / An internal error has occurred and has been logged."). Consistente com uma exceção de divisão por zero não tratada no cálculo da razão fundos ÷ empréstimo |
| 18:34:39 | CT-EMP-14 — Empréstimo negativo, com entrada | Empréstimo: -1.000,00 · Entrada: 100,00 | HTTP 200, `approved: false`, mensagem `error.insufficient.funds`. Nenhum erro, nenhuma aprovação indevida |
| 18:34:40 | Variante de CT-EMP-14 — Empréstimo negativo, sem entrada | Empréstimo: -1.000,00 · Entrada: 0,00 | Mesmo resultado do caso anterior. Saldo da conta corrente permaneceu em 515,50 em ambos os casos (reprovação não gera débito) |
| 18:34 | Abertura de conta poupança para testar CT-EMP-15 | Conta aberta a partir da conta corrente | Nova conta poupança (id 16674) criada com 100,00, transferidos automaticamente da conta corrente, que caiu para 415,50. Fundos agregados do cliente permaneceram em 515,50 |
| 18:35:08 | CT-EMP-15 — Entrada maior que o saldo da conta de origem, mas menor que o saldo agregado | Empréstimo: 1.000,00 · Entrada: 300,00 · Conta de origem: poupança (saldo 100,00) | HTTP 200, `approved: true`, novo empréstimo (id 16785) criado. A entrada foi debitada apenas da conta poupança, que ficou com saldo **-200,00** |
| 18:35 | Confirmação visual do saldo negativo | Consulta às telas "Accounts Overview" e "Account Activity" da conta poupança | O saldo negativo (-200,00) é exibido normalmente na coluna "Balance" das duas telas; apenas a coluna separada "Available Balance" (usada em outras validações) é zerada quando o saldo é negativo. Ou seja, o cliente veria a própria conta no vermelho ao consultar o extrato |
| 18:35–18:36 | Verificação do comportamento com campos vazio/não numérico (revisita a CT-EMP-10/11/12) | Empréstimo vazio; depois Empréstimo = "abc" | HTTP **400** (não 500), com corpo JSON detalhado: `"Required parameter 'amount' is not present."` e `"Failed to convert 'amount' with value: 'abc'"`, respectivamente. A interface, porém, ignora esse detalhe e mostra sempre a mesma tela genérica de erro ao usuário final |
| 18:36:31 | Efeito cascata do saldo negativo — empréstimo dentro da nova capacidade | Empréstimo: 1.000,00 · Entrada: 0,00 · Fundos agregados esperados: 415,50 + (-200,00) = 215,50 | HTTP 200, `approved: true` — razão 215,50 ÷ 1.000,00 = 21,55%, acima do limiar. Confirma que o saldo negativo da poupança é somado normalmente ao cálculo de fundos disponíveis para pedidos futuros |
| 18:36:32 | Efeito cascata — empréstimo fora da nova capacidade | Empréstimo: 1.100,00 · Entrada: 0,00 | HTTP 200, `approved: false`, `error.insufficient.funds` — razão 215,50 ÷ 1.100,00 = 19,59%, abaixo do limiar, exatamente como esperado |
| 18:36:43 | Regra R2 da tabela de decisão (entrada > fundos, razão ≥ 20%) | Empréstimo: 500,00 · Entrada: 300,00 (fundos agregados: 215,50) | `approved: false`, mensagem `error.insufficient.funds.for.down.payment` |
| 18:36:44 | Regra R1 da tabela de decisão (entrada > fundos, razão < 20%) | Empréstimo: 2.000,00 · Entrada: 300,00 | `approved: false`, **mesma mensagem** `error.insufficient.funds.for.down.payment` — confirma que R1 e R2 realmente produzem o mesmo texto ao usuário, como previsto na tabela de decisão do artefato anterior |
| 18:36:55 | Fronteira exata do limiar de 20% | Empréstimo: 1.077,50 (215,50 ÷ 0,20) · Entrada: 0,00 | `approved: true`, como esperado (razão exatamente 20,000%) |
| 18:36:56 | Fronteira "R$ 0,01 acima" do limiar (esperava-se reprovação) | Empréstimo: 1.077,51 · Entrada: 0,00 | **`approved: true`** — resultado inesperado. Levou à investigação de arredondamento abaixo |
| 18:37:23 | Investigação do arredondamento — valor mais distante da fronteira teórica | Empréstimo: 1.080,00 (razão bruta 19,954%) | `approved: true` |
| 18:37:24 | Investigação do arredondamento — valor ainda mais distante | Empréstimo: 1.085,00 (razão bruta 19,862%) | `approved: false`, `error.insufficient.funds` — isolado o ponto em que o resultado muda. Conclusão: o sistema arredonda a razão fundos ÷ empréstimo para 3 casas decimais (arredondamento round-half-up) antes de compará-la ao limiar de 20%, o que cria uma margem de tolerância de alguns centavos que o documento de design original não previa |
| 18:37:44 | Encerramento — consulta final de saldos do cliente de teste | — | Sete contas associadas ao cliente 15098: conta corrente (415,50), poupança (-200,00) e cinco contas de empréstimo criadas ao longo da sessão, confirmando o rastro completo dos cenários executados |

## Defeitos confirmados nesta sessão

1. **Empréstimo com valor zero derruba a aplicação (severidade alta).** A solicitação retorna HTTP 500 com página de erro genérica, em vez de uma rejeição controlada. Indica exceção não tratada na camada de cálculo da regra de aprovação. Candidato a relato de bug formal na próxima etapa do portfólio.
2. **Débito de entrada sem checagem de saldo individual da conta de origem (severidade alta).** A regra de aprovação usa o saldo agregado de todas as contas do cliente, mas o débito da entrada é aplicado apenas à conta selecionada, que pode ficar com saldo negativo sem qualquer aviso ou bloqueio — e esse saldo negativo é exibido normalmente ao cliente no extrato. O efeito também se acumula: cada conta que fica negativa reduz a capacidade de empréstimo do cliente em solicitações futuras, o que foi verificado diretamente. Também candidato a relato de bug formal.

## Hipótese testada e refutada

Valor de empréstimo negativo não é um problema: a razão fundos ÷ empréstimo fica negativa e é sempre menor que o limiar positivo de 20%, então o pedido é reprovado normalmente pela mesma regra de fundos insuficientes, sem erro e sem aprovação indevida. O caso havia sido sinalizado como risco no documento de design anterior apenas por não haver, no código, nenhuma validação explícita contra valores negativos — a execução real mostrou que a própria fórmula da regra já cobre esse caso indiretamente.

## Descoberta que não estava no documento de design original

O arredondamento da razão fundos/empréstimo para 3 casas decimais antes da comparação com o limiar (ver nota acima, cenários das 18:36:56 a 18:37:24) não é visível a partir da simples leitura da fórmula no código-fonte — só apareceu ao testar valores muito próximos da fronteira teórica e obter um resultado diferente do esperado. Os casos de fronteira do documento de design (`CT-EMP-03` e `CT-EMP-04`) foram corrigidos com os valores reais encontrados nesta sessão.

## Perguntas em aberto / ideias para a próxima sessão

- O que acontece se o cliente tentar fazer uma transferência (*Transfer Funds*) a partir de uma conta já negativa? A validação de saldo dessa tela é diferente da regra de empréstimo?
- A conta de empréstimo criada quando o pedido é aprovado nunca foi explorada em si — ela permite depósito, saque ou transferência como uma conta comum? Faz sentido ela aparecer na lista de contas de origem de um novo pedido de empréstimo?
- O comportamento de erro 500 ao enviar um POST de registro sem antes buscar o formulário (primeira tentativa desta sessão) não foi aprofundado — vale confirmar se é uma proteção intencional contra automação ou apenas uma falha de tratamento de sessão ausente.

## Avaliação de risco residual

Os dois defeitos confirmados nesta sessão (erro 500 em valor zero e saldo negativo sem bloqueio) são reproduzíveis, têm causa técnica identificada com boa confiança e evidência de execução real anexada nesta sessão. Estão prontos para virar relatos de bug formais na próxima etapa do portfólio.
