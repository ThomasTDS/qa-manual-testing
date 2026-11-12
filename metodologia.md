# Metodologia

Este documento explica como os artefatos deste repositório foram construídos e por quê — não o que cada um encontrou (isso está nos próprios artefatos), mas o raciocínio que conecta as três aplicações testadas como um processo único, não três exercícios soltos.

## Por que três aplicações diferentes

O primeiro ciclo completo (design de casos, sessão exploratória, relatos de bug) foi feito no ParaBank, em cima da funcionalidade de Solicitar Empréstimo. O segundo ciclo foi no BugBank, em cima de Transferência, Cadastro e Login. A escolha de diversificar não foi só para "ter mais conteúdo" — foi para testar a mesma disciplina de trabalho em dois contextos com condições de acesso bem diferentes, o que acabou sendo o ponto mais interessante do processo.

O terceiro ciclo foi no ServeRest, uma API de e-commerce, em cima da funcionalidade de Carrinhos. Aqui o motivo foi outro: os dois primeiros ciclos eram de aplicações bancárias testadas como sistema, e faltava o teste de API como alvo principal — com tudo o que muda quando não há tela: o resultado esperado passa a ser o código de status, o texto exato da mensagem e o efeito nos dados, e não o que aparece para o usuário.

## Como a regra de negócio foi obtida em cada caso

No ParaBank, o código-fonte da aplicação é público (`github.com/parasoft/parabank`). A regra de aprovação de empréstimo foi extraída diretamente das classes Java responsáveis pela decisão de crédito, não da interface nem de suposição. Isso deu acesso a um detalhe que nenhuma leitura de tela revelaria: o formulário de empréstimo, na interface real, não passa pelo validador que existe no próprio repositório — ele foi substituído por uma chamada direta a um serviço REST, deixando esse validador como código morto. Só a leitura do código, cruzada com a leitura do JSP que renderiza a tela, expôs essa divergência.

No BugBank, o código-fonte do serviço é um repositório privado — não havia essa opção. A regra de negócio veio de outra fonte oficial: a própria aplicação publica sua especificação na rota `/requirements`, só que renderizada via JavaScript, então o texto teve que ser extraído de dentro do pacote compilado da página para garantir que as regras citadas nos artefatos são literais, não paráfrase de memória.

No ServeRest, o código-fonte também é aberto (`github.com/ServeRest/ServeRest`), então a regra voltou a vir do código, como no ParaBank. A diferença foi a necessidade de fixar a versão: o projeto recebe atualizações com frequência, e um artefato que cita linhas de código precisa dizer de qual versão está falando — senão uma correção futura invalida os achados sem deixar contexto. Foi a leitura do código, aliás, que levantou as duas hipóteses centrais do ciclo, antes de qualquer requisição: a detecção de produto duplicado só comparava itens vizinhos, e a lista de produtos parecia aceitar ficar vazia.

Fontes diferentes, mas o mesmo princípio: a regra de negócio documentada num caso de teste precisa vir de algo verificável, nunca de "acho que funciona assim".

## Da regra ao caso de teste

As mesmas três técnicas foram aplicadas nos dois primeiros ciclos — tabela de decisão, particionamento de equivalência, análise de valor limite —, mas a maneira como elas se encaixaram dependeu da regra encontrada em cada lugar. A aprovação de empréstimo do ParaBank tem duas condições que se combinam com uma ordem de prioridade específica (a checagem de entrada domina a checagem de fundos, mesmo quando as duas falham juntas), o que rendeu uma tabela de decisão onde duas combinações diferentes colapsam na mesma mensagem visível ao usuário — um detalhe que só faz sentido documentar depois de entender o código. Já a Transferência do BugBank tem três condições combinadas, mas sem acesso ao código para saber qual delas tem prioridade quando mais de uma falha ao mesmo tempo — então a tabela de decisão desse artefato documenta explicitamente essa incerteza, em vez de inventar uma resposta.

No ServeRest, o cadastro de carrinho passa por seis validações. Uma tabela de decisão completa teria 64 combinações, mas o código avalia as validações em sequência e para na primeira que falha — então a tabela foi reduzida a sete regras, fiel ao comportamento real, mantendo casos com duas condições violadas ao mesmo tempo para detectar uma eventual mudança de ordem. O ciclo também trouxe uma técnica nova para o portfólio: transição de estados, aplicada ao ciclo de vida do carrinho (sem carrinho, com carrinho, compra concluída ou cancelada), incluindo as transições que a API deve recusar, como excluir um usuário ou um produto vinculado a um carrinho ativo.

## Da execução ao achado

O ParaBank expõe o mesmo serviço REST que a interface consome internamente, então a sessão exploratória foi conduzida via chamadas HTTP diretas — o que permitiu testar valores de fronteira com precisão de centavos e, durante a investigação, descobrir que a razão usada na regra de aprovação é arredondada para três casas decimais antes de ser comparada ao limiar, algo que não aparecia na leitura do código sozinha e que exigiu corrigir os valores de fronteira do artefato de design original.

O BugBank não expõe nenhuma API — é uma aplicação inteiramente client-side, com os dados mantidos em memória do navegador. A sessão exploratória, por isso, foi conduzida manualmente pela interface, com o registro e a redação do relatório montados a partir dos cenários testados e observados diretamente. Foi nessa sessão, aliás, que surgiu o achado mais forte do repositório até agora — e ele nem estava no charter original: veio de investigar por que não havia um fluxo de recuperação de senha.

No ServeRest, a execução foi feita numa instância local, e não na instância pública, porque a pública é compartilhada: outras pessoas podem alterar os mesmos produtos durante o teste, o que tornaria os casos de estoque impossíveis de reproduzir. Pelo mesmo motivo, cada caso cria seu próprio usuário e seus próprios produtos. A execução teve duas etapas — uma passada de apoio por script, para cobrir os 29 casos rapidamente, e a execução no Postman pelo testador — e gerou uma coleção versionada no repositório, em que as asserções dos casos do defeito verificam o comportamento correto e falham de propósito. A leitura do código apontou onde procurar; foi a execução que mostrou o tamanho do problema: só a última ocorrência do produto repetido é descontada do estoque, e o cancelamento devolve exatamente esse valor, escondendo o defeito de quem testa só esse fluxo.

## Rigor como prática, não como discurso

Quatro vezes, uma suposição inicial foi corrigida depois que a evidência real contradisse o que o artefato dizia:

- Os valores de fronteira do empréstimo do ParaBank foram recalculados depois que a execução real revelou o arredondamento de três casas decimais.
- O caso de teste que previa "valor de empréstimo negativo" como risco foi marcado como testado e refutado, depois que a execução mostrou que a própria fórmula da regra já lida com esse caso corretamente.
- A descrição do bug de validação do formulário de transferência do BugBank foi corrigida depois que um print mostrou que o gatilho real da mensagem de erro era diferente do que havia sido descrito de memória (vírgula como separador decimal, não campo vazio).
- A suspeita de que o ServeRest aceitaria uma lista de produtos vazia, levantada na leitura do código, foi refutada na execução: a API recusa a lista vazia. O caso de teste foi corrigido e a hipótese ficou registrada como descartada.

Nenhuma dessas correções foi escondida ou reescrita como se a versão original nunca tivesse existido — todas ficam registradas nos próprios artefatos e no histórico de decisões do projeto. É esse hábito — corrigir em vez de defender a primeira suposição — que este repositório tenta demonstrar, mais do que qualquer bug específico encontrado no caminho.

## Em números

- 3 aplicações públicas testadas (ParaBank, BugBank, ServeRest)
- 59 casos de teste desenhados, com rastreabilidade até a regra de negócio que cada um valida
- 4 técnicas de design de teste aplicadas (tabela de decisão, particionamento de equivalência, análise de valor limite, transição de estados)
- 3 sessões de teste exploratório, com metodologias diferentes (chamadas HTTP diretas ao serviço; execução manual pela interface; execução via Postman com coleção versionada)
- 5 defeitos reais confirmados e documentados, com evidência
- 2 hipóteses de defeito testadas e descartadas por execução real
