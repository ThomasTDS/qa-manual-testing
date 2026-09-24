# Metodologia

Este documento explica como os artefatos deste repositório foram construídos e por quê — não o que cada um encontrou (isso está nos próprios artefatos), mas o raciocínio que conecta os dois aplicativos testados como um processo único, não dois exercícios soltos.

## Por que duas aplicações diferentes

O primeiro ciclo completo (design de casos, sessão exploratória, relatos de bug) foi feito no ParaBank, em cima da funcionalidade de Solicitar Empréstimo. O segundo ciclo foi no BugBank, em cima de Transferência, Cadastro e Login. A escolha de diversificar não foi só para "ter mais conteúdo" — foi para testar a mesma disciplina de trabalho em dois contextos com condições de acesso bem diferentes, o que acabou sendo o ponto mais interessante do processo.

## Como a regra de negócio foi obtida em cada caso

No ParaBank, o código-fonte da aplicação é público (`github.com/parasoft/parabank`). A regra de aprovação de empréstimo foi extraída diretamente das classes Java responsáveis pela decisão de crédito, não da interface nem de suposição. Isso deu acesso a um detalhe que nenhuma leitura de tela revelaria: o formulário de empréstimo, na interface real, não passa pelo validador que existe no próprio repositório — ele foi substituído por uma chamada direta a um serviço REST, deixando esse validador como código morto. Só a leitura do código, cruzada com a leitura do JSP que renderiza a tela, expôs essa divergência.

No BugBank, o código-fonte do serviço é um repositório privado — não havia essa opção. A regra de negócio veio de outra fonte oficial: a própria aplicação publica sua especificação na rota `/requirements`, só que renderizada via JavaScript, então o texto teve que ser extraído de dentro do pacote compilado da página para garantir que as regras citadas nos artefatos são literais, não paráfrase de memória.

Duas fontes diferentes, mas o mesmo princípio: a regra de negócio documentada num caso de teste precisa vir de algo verificável, nunca de "acho que funciona assim".

## Da regra ao caso de teste

As mesmas três técnicas foram aplicadas nas duas aplicações — tabela de decisão, particionamento de equivalência, análise de valor limite —, mas a maneira como elas se encaixaram dependeu da regra encontrada em cada lugar. A aprovação de empréstimo do ParaBank tem duas condições que se combinam com uma ordem de prioridade específica (a checagem de entrada domina a checagem de fundos, mesmo quando as duas falham juntas), o que rendeu uma tabela de decisão onde duas combinações diferentes colapsam na mesma mensagem visível ao usuário — um detalhe que só faz sentido documentar depois de entender o código. Já a Transferência do BugBank tem três condições combinadas, mas sem acesso ao código para saber qual delas tem prioridade quando mais de uma falha ao mesmo tempo — então a tabela de decisão desse artefato documenta explicitamente essa incerteza, em vez de inventar uma resposta.

## Da execução ao achado

O ParaBank expõe o mesmo serviço REST que a interface consome internamente, então a sessão exploratória foi conduzida via chamadas HTTP diretas — o que permitiu testar valores de fronteira com precisão de centavos e, durante a investigação, descobrir que a razão usada na regra de aprovação é arredondada para três casas decimais antes de ser comparada ao limiar, algo que não aparecia na leitura do código sozinha e que exigiu corrigir os valores de fronteira do artefato de design original.

O BugBank não expõe nenhuma API — é uma aplicação inteiramente client-side, com os dados mantidos em memória do navegador. A sessão exploratória, por isso, foi conduzida manualmente pela interface, com o registro e a redação do relatório montados a partir dos cenários testados e observados diretamente. Foi nessa sessão, aliás, que surgiu o achado mais forte do repositório até agora — e ele nem estava no charter original: veio de investigar por que não havia um fluxo de recuperação de senha.

## Rigor como prática, não como discurso

Três vezes, uma suposição inicial foi corrigida depois que a evidência real contradisse o que o artefato dizia:

- Os valores de fronteira do empréstimo do ParaBank foram recalculados depois que a execução real revelou o arredondamento de três casas decimais.
- O caso de teste que previa "valor de empréstimo negativo" como risco foi marcado como testado e refutado, depois que a execução mostrou que a própria fórmula da regra já lida com esse caso corretamente.
- A descrição do bug de validação do formulário de transferência do BugBank foi corrigida depois que um print mostrou que o gatilho real da mensagem de erro era diferente do que havia sido descrito de memória (vírgula como separador decimal, não campo vazio).

Nenhuma dessas correções foi escondida ou reescrita como se a versão original nunca tivesse existido — todas ficam registradas nos próprios artefatos e no histórico de decisões do projeto. É esse hábito — corrigir em vez de defender a primeira suposição — que este repositório tenta demonstrar, mais do que qualquer bug específico encontrado no caminho.

## Em números

- 2 aplicações públicas testadas (ParaBank, BugBank)
- 30 casos de teste desenhados, com rastreabilidade até a regra de negócio que cada um valida
- 2 sessões de teste exploratório, com metodologias diferentes (execução via API; execução manual pela interface)
- 4 defeitos reais confirmados e documentados, com evidência
