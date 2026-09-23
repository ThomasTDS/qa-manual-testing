# Design de casos de teste — BugBank: Transferência

## Metadados

| Campo | Valor |
| --- | --- |
| Aplicação sob teste | BugBank (https://bugbank.netlify.app) |
| Funcionalidade | Transferência entre contas |
| Tipo de artefato | Design de casos de teste com rastreabilidade de regra de negócio |
| Técnicas aplicadas | Tabela de decisão, particionamento de equivalência, análise de valor limite |
| Data do levantamento | 22/09/2026 |

## Objetivo

Documentar a regra de negócio oficial da funcionalidade de Transferência do BugBank e derivar, a partir dela, um conjunto de casos de teste rastreáveis — incluindo cenários positivos, negativos e de fronteira — usando técnicas formais de design de teste.

## Fonte da regra de negócio

O BugBank é uma aplicação de front-end voltada para prática de teste, mantida pela comunidade de QA brasileira. Diferente do ParaBank, o código-fonte do serviço que processa as regras é um repositório privado — não é possível lê-lo diretamente. Em compensação, a própria aplicação publica sua especificação de regras de negócio na rota `/requirements`. Essa página é renderizada via JavaScript (não é HTML estático), então o texto das regras foi extraído diretamente do pacote JavaScript da página (`_next/static/chunks/pages/requirements-*.js`), e não copiado visualmente — o que garante que o conteúdo abaixo é literal, não uma paráfrase.

Também é uma informação relevante para o design dos casos: a própria aplicação declara que **não usa banco de dados** — "a aplicação não conta com um banco de dados, todas as informações são armazenadas em memória local". Ou seja, contas e saldos criados numa sessão não persistem entre recarregamentos de página, o que precisa ser levado em conta como pré-condição de qualquer execução.

Regras publicadas oficialmente para a funcionalidade de Transferência (e as regras de Extrato diretamente relacionadas a ela):

> - Só é permitido transferência para contas válidas
> - Só é permitido transferência quando saldo é igual ou maior que valor para transferir
> - Tentativa de transferência para conta inválida deve exibir mensagem de erro "Conta inválida ou inexistente"
> - Número e dígito da conta aceitam apenas números
> - Campo descrição é um campo de preenchimento obrigatório
> - Valor de transferência não pode ser igual ou menor que zero
> - Ao realizar transferência com sucesso deve ser debitado o valor da conta e exibir a mensagem de "Transferência realizada com sucesso"
> - Ao realizar uma transferência com sucesso deve ser redirecionado para o extrato
> - Deve exibir o saldo disponível no momento
> - Cada transação deve exibir data que foi realizada, tipo da transação (Abertura de conta / Transferência enviada / Transferência recebida)
> - Transações sem comentário devem exibir (-)

Regra de Cadastro usada como pré-condição para o saldo da conta de origem:

> - Deixar ativo a opção "Criar conta com saldo" deve criar conta com saldo de R$ 1.000,00
> - Deixar inativo a opção "Criar conta com saldo" deve criar conta com saldo de R$ 0,00

## Regras de negócio identificadas

| ID | Regra |
| --- | --- |
| RN-01 | Os campos "Número da conta" e "Dígito" da conta de destino aceitam apenas caracteres numéricos. |
| RN-02 | A transferência só é permitida para uma conta de destino válida e existente. Se a conta for inválida ou não existir, a aplicação deve exibir a mensagem "Conta inválida ou inexistente". |
| RN-03 | A transferência só é permitida quando o saldo da conta de origem é igual ou maior que o valor a transferir. |
| RN-04 | O valor da transferência não pode ser igual ou menor que zero. |
| RN-05 | O campo "Descrição" é de preenchimento obrigatório. |
| RN-06 | Ao ser concluída com sucesso, a transferência debita o valor da conta de origem, exibe a mensagem "Transferência realizada com sucesso" e redireciona o usuário para o Extrato. |
| RN-07 | Cada transação exibida no Extrato mostra a data em que foi realizada e o tipo (Abertura de conta / Transferência enviada / Transferência recebida). |
| RN-08 *(ponto de atenção — possível inconsistência na especificação, não confirmada por execução)* | A regra de Extrato prevê que "transações sem comentário devem exibir (-)", o que sugere a existência de transações sem descrição — mas RN-05 exige descrição obrigatória em toda transferência. As duas regras só se conciliam se o "(-)" for exclusivo de transações que não são transferências (como "Abertura de conta"). Precisa ser confirmado executando a aplicação. |
| RN-09 *(não coberta pela especificação)* | A especificação não diz explicitamente o que acontece ao transferir para o número da própria conta (autotransferência). Não há regra publicada proibindo ou permitindo esse cenário. |
| RN-10 *(descoberta na execução real, não visível pela especificação)* | O formulário de transferência trata os campos obrigatórios de forma inconsistente: o campo "Valor", quando deixado vazio, expõe uma mensagem de erro técnica de uma biblioteca de validação (não uma mensagem de negócio amigável); os demais campos obrigatórios ("Número da conta", "Dígito", "Descrição") não exibem nenhuma indicação de erro quando deixados vazios e o envio é tentado. Ver [`sessoes-exploratorias/bugbank-transferencia-22-09-2026.md`](../sessoes-exploratorias/bugbank-transferencia-22-09-2026.md) e [`relatorios-de-bugs/bug-trf-001-validacao-formulario-inconsistente.md`](../relatorios-de-bugs/bug-trf-001-validacao-formulario-inconsistente.md). |

## Tabela de decisão — RN-02 × RN-03 × RN-04

A especificação não define a ordem de prioridade entre as três validações quando mais de uma falha ao mesmo tempo — diferente do ParaBank, aqui não há acesso ao código para confirmar qual mensagem prevalece. Isso está sinalizado explicitamente na tabela e é uma pergunta em aberto para a sessão exploratória.

| Regra | C1: Conta de destino válida? | C2: Saldo ≥ valor? | C3: Valor > 0? | Resultado |
| --- | --- | --- | --- | --- |
| R1 | Não | Não | Não | Reprovado — mensagem indeterminada (três violações simultâneas) |
| R2 | Não | Não | Sim | Reprovado — mensagem indeterminada (duas violações simultâneas) |
| R3 | Não | Sim | Não | Reprovado — mensagem indeterminada (duas violações simultâneas) |
| R4 | Não | Sim | Sim | Reprovado — **"Conta inválida ou inexistente"** (único texto de erro documentado oficialmente) |
| R5 | Sim | Não | Não | Reprovado — mensagem indeterminada (duas violações simultâneas) |
| R6 | Sim | Não | Sim | Reprovado — mensagem não documentada para saldo insuficiente isolado |
| R7 | Sim | Sim | Não | Reprovado — mensagem não documentada para valor inválido isolado |
| R8 | Sim | Sim | Sim | **Aprovado** — débito da conta de origem, mensagem "Transferência realizada com sucesso", redirecionamento ao Extrato |

## Casos de teste

Pré-condição comum a todos os casos: cliente cadastrado e autenticado no BugBank, com conta de origem criada com a opção "Criar conta com saldo" ativa (saldo inicial de R$ 1.000,00), salvo indicação contrária. Como a aplicação mantém os dados apenas em memória local, cada execução real exige recriar essas pré-condições — não há dados persistidos entre sessões.

| ID | Cenário | Técnica | Dados de entrada | Resultado esperado | Prioridade |
| --- | --- | --- | --- | --- | --- |
| CT-TRF-01 | Transferência aprovada com todas as condições satisfeitas (cenário principal) | Particionamento de equivalência | Saldo de origem: 1.000,00 · Conta de destino: válida e existente · Valor: 100,00 · Descrição: "Aluguel" | Regra R8 → **Aprovado**. Conta de origem debitada em 100,00 (saldo final 900,00); mensagem "Transferência realizada com sucesso"; redirecionamento ao Extrato | Alta |
| CT-TRF-02 | Valor da transferência igual ao saldo total disponível (fronteira válida de RN-03) | Análise de valor limite | Saldo de origem: 1.000,00 · Valor: 1.000,00 · Conta de destino válida · Descrição preenchida | C2 satisfeita (saldo igual ao valor, regra usa "igual ou maior") → **Aprovado**, saldo de origem passa a 0,00 | Alta |
| CT-TRF-03 | Valor R$ 0,01 maior que o saldo disponível (fronteira inválida de RN-03) | Análise de valor limite | Saldo de origem: 1.000,00 · Valor: 1.000,01 · Conta de destino válida · Descrição preenchida | Regra R6 → **Reprovado** por saldo insuficiente | Alta |
| CT-TRF-04 | Valor da transferência igual a zero (fronteira inválida de RN-04) | Análise de valor limite | Saldo de origem: 1.000,00 · Valor: 0,00 · Conta de destino válida · Descrição preenchida | Regra R7 → **Reprovado** — "valor não pode ser igual ... a zero" | Alta |
| CT-TRF-05 | Valor da transferência com um centavo (fronteira válida de RN-04) | Análise de valor limite | Saldo de origem: 1.000,00 · Valor: 0,01 · Conta de destino válida · Descrição preenchida | C3 satisfeita (valor > 0) → **Aprovado** | Média |
| CT-TRF-06 | Valor da transferência negativo | Particionamento de equivalência (classe inválida) | Saldo de origem: 1.000,00 · Valor: -50,00 · Conta de destino válida · Descrição preenchida | Regra R7 → **Reprovado** — "valor não pode ser ... menor que zero" | Alta |
| CT-TRF-07 | Conta de destino inexistente, mas em formato numérico válido | Particionamento de equivalência | Saldo de origem: 1.000,00 · Valor: 100,00 · Conta de destino: número numérico que não corresponde a nenhuma conta cadastrada · Descrição preenchida | **Confirmado por execução real em 22/09/2026**: regra R4 → **Reprovado** — mensagem exibida exatamente como documentado, "Conta inválida ou inexistente" | Alta |
| CT-TRF-08 | Conta de destino com caractere não numérico (letra) | Particionamento de equivalência (classe inválida) | Número da conta: "12A34" ou dígito não numérico | Viola RN-01. Resultado esperado a confirmar por execução real: campo pode bloquear a digitação (validação de máscara) ou aceitar e reprovar como conta inválida — não documentado qual dos dois comportamentos ocorre | Média |
| CT-TRF-09 | Campo "Descrição" vazio | Particionamento de equivalência (classe inválida) | Saldo de origem: 1.000,00 · Valor: 100,00 · Conta de destino válida · Descrição: *(vazio)* | **Parcialmente observado em 22/09/2026**: ao tentar enviar com o campo vazio, nenhuma mensagem de erro é exibida (diferente do esperado originalmente). Não ficou confirmado se a transferência foi de fato bloqueada nos bastidores ou se apenas não há retorno visual — ver RN-10 e a sessão exploratória referenciada | Alta |
| CT-TRF-10 | Saldo insuficiente **e** valor inválido ao mesmo tempo (combinação não documentada — regra R5 da tabela de decisão) | Tabela de decisão | Saldo de origem: 100,00 · Valor: -50,00 · Conta de destino válida | C2 e C3 violadas simultaneamente. Resultado esperado a confirmar: qual mensagem de erro é exibida quando duas regras falham ao mesmo tempo não está documentado | Média |
| CT-TRF-11 | Conta de destino inválida **e** saldo insuficiente ao mesmo tempo (regra R2 da tabela de decisão) | Tabela de decisão | Saldo de origem: 50,00 · Valor: 500,00 · Conta de destino inexistente | C1 e C2 violadas simultaneamente. Resultado esperado a confirmar: mesma dúvida de priorização de mensagens | Baixa |
| CT-TRF-12 | Transferência para o número da própria conta (autotransferência, cenário não documentado) | Particionamento de equivalência (valor não coberto pela especificação) | Saldo de origem: 1.000,00 · Valor: 100,00 · Conta de destino: a própria conta do cliente autenticado · Descrição preenchida | A especificação não prevê esse caso. Resultado a confirmar por execução real — candidato tanto a comportamento aceitável (transferência "circular" sem efeito líquido) quanto a inconsistência (ex.: saldo duplicado ou operação travada) | Alta |
| CT-TRF-13 | Verificação do Extrato após transferência aprovada | Particionamento de equivalência | Após CT-TRF-01: consultar o Extrato da conta de origem e da conta de destino | RN-06/RN-07: a transação aparece no extrato da conta de origem como "Transferência enviada" (valor em vermelho, com sinal de menos) e no extrato da conta de destino como "Transferência recebida" (valor em verde), ambas com a data da operação | Alta |
| CT-TRF-14 | Campos "Número da conta" e "Dígito" do destino vazios | Particionamento de equivalência (classe inválida) | Número da conta: *(vazio)* · Dígito: *(vazio)* · Valor e descrição preenchidos | Resultado esperado a confirmar por execução real: não há regra publicada especificamente para esse caso, apenas a regra genérica de conta inválida (RN-02) — presume-se o mesmo comportamento de RN-02, mas isso não está garantido pela especificação | Média |

## Rastreabilidade regra → casos de teste

| Regra | Casos de teste que a cobrem |
| --- | --- |
| RN-01 | CT-TRF-08 |
| RN-02 | CT-TRF-01, CT-TRF-02, CT-TRF-07, CT-TRF-11, CT-TRF-14 |
| RN-03 | CT-TRF-01, CT-TRF-02, CT-TRF-03, CT-TRF-10, CT-TRF-11 |
| RN-04 | CT-TRF-01, CT-TRF-04, CT-TRF-05, CT-TRF-06, CT-TRF-10 |
| RN-05 | CT-TRF-09 |
| RN-06 | CT-TRF-01, CT-TRF-02, CT-TRF-13 |
| RN-07 | CT-TRF-13 |
| RN-08 | Não coberta por um caso de teste específico — depende de uma transação sem descrição existir, o que RN-05 não permite pela interface de transferência; fica registrada como pergunta em aberto |
| RN-09 | CT-TRF-12 |
| RN-10 | CT-TRF-09 (descrição vazia) e a investigação geral de campos obrigatórios na sessão exploratória |
| Sem regra correspondente na especificação | CT-TRF-08 (comportamento do campo em si), CT-TRF-12, CT-TRF-14 |

## Observações e riscos

- Diferente do artefato do ParaBank, aqui não houve leitura de código-fonte: as regras vieram integralmente da especificação oficial publicada pelo próprio BugBank. Isso torna o levantamento mais rápido e diretamente rastreável a uma fonte pública, mas significa que os casos não haviam sido confirmados por execução real no momento em que este documento foi escrito — todos os resultados esperados estavam baseados na leitura literal da especificação, não em comportamento observado. Dois casos (CT-TRF-07 e CT-TRF-09) já foram confirmados/revisados por execução real em 22/09/2026 — ver [`sessoes-exploratorias/bugbank-transferencia-22-09-2026.md`](../sessoes-exploratorias/bugbank-transferencia-22-09-2026.md).
- Vários casos (CT-TRF-08, CT-TRF-10, CT-TRF-11, CT-TRF-12, CT-TRF-14) continuam com resultado esperado sinalizado como "a confirmar" — a sessão exploratória realizada não teve escopo (parte da aplicação ainda está em desenvolvimento) para cobri-los; permanecem como candidatos para uma próxima sessão.
- Como a aplicação não tem banco de dados e mantém tudo em memória local, qualquer execução real precisa recriar as contas de origem e destino do zero a cada sessão de teste — não é possível reaproveitar contas de uma execução anterior.
- A aplicação é inteiramente client-side (não há um serviço de back-end público para ser chamado diretamente, diferente do ParaBank); qualquer execução real desses casos precisa necessariamente ser feita pela interface, não por chamadas diretas a uma API.
