# Relato de bug — Validação do formulário de Transferência é inconsistente e vaza mensagem técnica

## Metadados

| Campo | Valor |
| --- | --- |
| ID | BUG-TRF-001 |
| Aplicação | BugBank (https://bugbank.netlify.app) |
| Funcionalidade | Transferência entre contas |
| Ambiente | Instância pública de demonstração |
| Data de identificação | 22/09/2026 |
| Identificado por | Thomas Teixeira, durante sessão de teste exploratório |
| Severidade | Média |
| Prioridade | Média |
| Status | Aberto (não reportado ao mantenedor — instância pública mantida pela comunidade de QA como ambiente de treino) |

## Resumo

O formulário de Transferência não trata a validação dos campos obrigatórios de forma consistente. O campo "Valor" expõe uma mensagem de erro que é claramente um texto interno de uma biblioteca de validação de schema, não uma mensagem de negócio traduzida e pensada para o usuário final — isso acontece tanto quando o campo é deixado vazio quanto quando o usuário digita o valor no formato brasileiro de moeda, usando vírgula como separador decimal (ex.: `100,`). Os demais campos obrigatórios (Número da conta, Dígito, Descrição) não exibem nenhuma mensagem quando deixados vazios. Além disso, quando a mensagem do campo "Valor" aparece, ela é renderizada sobreposta ao próprio campo, prejudicando a leitura.

## Ambiente e pré-condições

- Cliente autenticado no BugBank, na tela de Transferência.

## Passos para reproduzir

### Comportamento 1a — mensagem técnica vazada com o campo "Valor" vazio

1. Acessar a tela de Transferência.
2. Preencher um número de conta de destino e uma descrição quaisquer, deixando o campo "Valor" vazio.
3. Tentar enviar o formulário.

### Comportamento 1b — mensagem técnica vazada ao digitar vírgula como separador decimal

1. Acessar a tela de Transferência.
2. Preencher número da conta (ex.: `66774334`), dígito (ex.: `342`) e valor da transferência digitando **vírgula** como separador decimal, no formato brasileiro (ex.: `100,`).
3. Tentar enviar o formulário.

### Comportamento 2 — ausência de indicação nos demais campos obrigatórios

1. Acessar a tela de Transferência.
2. Deixar vazio o campo "Número da conta", "Dígito" ou "Descrição" (um de cada vez), preenchendo os demais normalmente.
3. Tentar enviar o formulário.

## Resultado esperado

Em ambos os casos, a aplicação deveria exibir uma mensagem de validação clara e amigável, no padrão de "campo obrigatório" já usado nas mensagens de Cadastro (por exemplo, "Nome não pode ser vazio"), consistente entre todos os campos do formulário — nunca um texto de erro interno de biblioteca.

## Resultado obtido

**Comportamento 1a:** ao enviar o formulário com o campo "Valor" vazio, a aplicação exibe, segundo relato do testador, o mesmo tipo de mensagem técnica do comportamento 1b abaixo, trocando apenas o valor citado ao final (`cast from the value` `""`` em vez de um valor com vírgula).

**Comportamento 1b (capturado em print, evidência mais forte):** ao tentar enviar o formulário com o valor `100,` no campo "Valor da transferência", a aplicação exibe literalmente o seguinte texto:

```
transferValue must be a `number` type, but the final value was: `NaN` (cast from the value `"100,"`).
```

Esse texto identifica o nome interno do campo em inglês (`transferValue`), o tipo de dado esperado pela camada de validação e a forma como o valor foi convertido (`NaN`, a partir da string `"100,"`) — informação de implementação que nunca deveria ser exposta ao usuário final. Além de vazar um texto técnico, esse comportamento revela um segundo problema: o campo aceita digitar vírgula (não bloqueia o caractere), mas não sabe interpretá-la como separador decimal — ou seja, o campo não suporta o formato brasileiro de valor monetário, apenas ponto.

**Comportamento 2:** ao enviar o formulário com "Número da conta", "Dígito" ou "Descrição" vazios, nenhuma mensagem de erro aparece. Não há como o usuário saber, pela interface, por que o formulário não foi concluído.

**Problema adicional de layout:** quando a mensagem do campo "Valor" é exibida, o texto aparece visualmente sobreposto ao campo de entrada (ver evidência abaixo), dificultando a leitura tanto da mensagem quanto do próprio campo. Também não há nenhum indicador visual (como um asterisco) nos campos obrigatórios antes de uma tentativa de envio.

Para contraste: a tela de Login do mesmo aplicativo, para esse mesmo tipo de cenário (campo obrigatório vazio), exibe corretamente a mensagem "É campo obrigatório" abaixo do campo "Senha" (ver segunda imagem em Evidência) — o que reforça que o problema descrito aqui é uma inconsistência pontual do formulário de Transferência, não uma limitação geral da aplicação.

## Evidência

Print da tela de Transferência, mostrando a mensagem técnica sobreposta ao campo, capturada ao tentar enviar o formulário com o valor `100,` (vírgula como separador decimal):

![Mensagem técnica de validação vazada no campo Valor, sobreposta ao campo](evidencias/bug-trf-001-mensagem-tecnica-vazada.png)

Print da tela de Login do BugBank, usado aqui apenas como contraste: o campo "Senha" vazio exibe corretamente "É campo obrigatório", diferente do que ocorre no formulário de Transferência:

![Tela de login do BugBank, com validação de campo obrigatório funcionando corretamente](evidencias/bugbank-login-sem-banco-de-dados.png)

## Análise técnica (causa provável)

O formato da mensagem — nome do campo em `camelCase`, descrição do tipo esperado entre crases, e o valor original citado entre crases — é o formato padrão de mensagens de erro de bibliotecas de validação de schema em JavaScript (como Yup, comumente usada em formulários React). O comportamento sugere que essa mensagem de erro, gerada automaticamente pela biblioteca ao tentar converter o texto digitado para número, nunca foi capturada nem substituída por uma mensagem de negócio — diferente do que parece ter sido feito para os campos obrigatórios do formulário de Cadastro e do campo "Senha" do Login, que exibem mensagens amigáveis e traduzidas ("É campo obrigatório", por exemplo). O campo "Valor" também não parece ter nenhuma máscara ou tratamento de entrada regional (vírgula como separador decimal), o que é uma lacuna relevante numa aplicação em português voltada ao mercado brasileiro. É provável que os campos "Número da conta", "Dígito" e "Descrição" do formulário de Transferência sequer tenham uma regra de validação equivalente implementada, o que explicaria a ausência total de mensagem para eles.

## Justificativa de severidade e prioridade

**Severidade: Média.** Não há perda de dados nem comprometimento de saldo — é uma falha de qualidade de validação e de experiência do usuário, mas o sistema não executa nenhuma ação incorreta com os dados (a transferência não chega a ser processada). Ainda assim, expor uma mensagem de erro interna de biblioteca é o tipo de detalhe que, segundo a própria avaliação do testador durante a sessão, não seria aceitável em um produto real — revela informação de implementação e passa uma impressão de descuido.

**Prioridade: Média.** O formulário de Transferência é uma funcionalidade central da aplicação, então a falta de clareza nas mensagens de erro afeta diretamente a usabilidade do fluxo mais importante do sistema. Não é uma prioridade máxima porque não impede a operação de ser concluída corretamente quando os dados estão certos — afeta apenas o caminho de erro.

## Rastreabilidade

- Regra de negócio relacionada: `RN-10`, em [`casos-de-teste/bugbank-transferencia.md`](../casos-de-teste/bugbank-transferencia.md).
- Caso de teste de origem: `CT-TRF-09` (campo Descrição vazio), no mesmo documento — o achado acabou sendo mais amplo do que esse caso previa, cobrindo também o campo Valor e o campo Número da conta/Dígito.
- Execução que confirmou o defeito: sessão registrada em [`sessoes-exploratorias/bugbank-transferencia-22-09-2026.md`](../sessoes-exploratorias/bugbank-transferencia-22-09-2026.md), itens 5 a 8 do registro sequencial.
