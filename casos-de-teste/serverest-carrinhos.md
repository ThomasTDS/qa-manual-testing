# Design de casos de teste — ServeRest: Carrinhos

## Metadados

| Campo | Valor |
| --- | --- |
| Aplicação sob teste | ServeRest (API REST), versão 3.2.2, executada localmente (`npx serverest@3.2.2`) |
| Funcionalidade | Carrinhos (`/carrinhos`) |
| Tipo de artefato | Design de casos de teste com rastreabilidade de regra de negócio |
| Técnicas aplicadas | Tabela de decisão, transição de estados, análise de valor limite, particionamento de equivalência |
| Data do levantamento | 12/11/2026 |

## Objetivo

Documentar a regra de negócio real da funcionalidade de Carrinhos do ServeRest e derivar, a partir dela, um conjunto de casos de teste de API rastreáveis — incluindo cenários positivos, negativos, de fronteira e de ciclo de vida do carrinho — usando técnicas formais de design de teste.

Diferente dos dois artefatos anteriores, o alvo aqui não é uma interface gráfica: os casos são executados diretamente contra a API, com requisições HTTP montadas manualmente (Postman). O resultado esperado de cada caso inclui, portanto, o código de status HTTP e a mensagem retornada no corpo da resposta, além do efeito sobre os dados (estoque, existência do carrinho).

## Fonte da regra de negócio

O ServeRest é uma API de e-commerce de código aberto, criada para estudo de teste de software e mantida pela comunidade de QA brasileira (`github.com/ServeRest/ServeRest`). A documentação Swagger da própria aplicação descreve os endpoints e os códigos de resposta, mas não detalha a ordem das validações nem o comportamento de estoque. Por isso, como no ParaBank, a regra foi extraída diretamente do código-fonte, fixado no commit `0a8068f` da branch `trunk` (correspondente à versão 3.2.2 publicada no npm):

- `src/routes/carrinhos-route.js` — define quais endpoints exigem autenticação e em que ordem os *middlewares* rodam (token antes do schema).
- `src/models/carrinhos-model.js` — schema de validação do corpo da requisição e do parâmetro `id`.
- `src/middlewares/authentication-middleware.js` e `src/services/auth-service.js` — validação do token JWT e tempo de expiração (600 segundos, em `src/utils/conf.js`).
- `src/controllers/carrinhos-controller.js` — ordem das validações de negócio no cadastro e comportamento de concluir/cancelar compra.
- `src/services/carrinhos-service.js` e `src/services/produtos-service.js` — detecção de produto duplicado, checagem de estoque, baixa e reposição de estoque.
- `src/controllers/usuarios-controller.js` e `src/controllers/produtos-controller.js` — bloqueio de exclusão de usuário e de produto vinculados a um carrinho.
- `src/utils/constants.js` — texto literal de todas as mensagens citadas abaixo.

> **Atenção:** o ServeRest recebe atualizações com frequência. Os resultados esperados deste documento valem para a versão 3.2.2. Executar com `npx serverest@latest` pode trazer correções ou mudanças de mensagem que invalidem parte dos casos — por isso a versão está fixada.

A execução é feita numa instância local, e não na instância pública (`serverest.dev`), porque a instância pública é compartilhada: produtos e estoques podem ser alterados por outras pessoas durante a execução, o que tornaria os casos de estoque não reproduzíveis.

## Regras de negócio identificadas

| ID | Regra |
| --- | --- |
| RN-01 | `POST /carrinhos`, `DELETE /carrinhos/concluir-compra` e `DELETE /carrinhos/cancelar-compra` exigem o cabeçalho `Authorization` no formato `Bearer <token>`, com um token válido, não expirado (validade de 600 segundos) e pertencente a um usuário que ainda exista com o mesmo e-mail e senha usados no login. Caso contrário: **401**, *"Token de acesso ausente, inválido, expirado ou usuário do token não existe mais"*. Qualquer usuário autenticado pode ter carrinho — não é exigido perfil de administrador. |
| RN-02 | O corpo do `POST /carrinhos` deve conter `produtos`, uma lista em que cada item tem `idProduto` (texto, obrigatório) e `quantidade` (inteiro maior que zero, obrigatório). Caso contrário: **400** com a mensagem de validação do campo. Essa validação só roda depois da autenticação (RN-01). |
| RN-03 | Cada usuário pode ter no máximo um carrinho. Se já tiver: **400**, *"Não é permitido ter mais de 1 carrinho"*. |
| RN-04 | O mesmo produto não pode aparecer mais de uma vez na lista. Se aparecer: **400**, *"Não é permitido possuir produto duplicado"*, com a lista `idProdutosDuplicados`. |
| RN-05 | Todo `idProduto` precisa existir. Se não existir: **400**, *"Produto não encontrado"*, indicando o item e sua posição (`index`) na lista. |
| RN-06 | A quantidade pedida de cada produto deve ser menor ou igual ao estoque atual. Se for maior: **400**, *"Produto não possui quantidade suficiente"*, indicando o item, a quantidade pedida e a quantidade em estoque. |
| RN-07 | Satisfeitas RN-01 a RN-06, o carrinho é criado (**201**, *"Cadastro realizado com sucesso"* e o `_id` do carrinho). O carrinho guarda o preço unitário de cada produto no momento da criação, o preço total (soma de preço unitário × quantidade) e a quantidade total. O estoque de cada produto é baixado **no momento da criação do carrinho**, não na conclusão da compra. |
| RN-08 | `DELETE /carrinhos/concluir-compra` exclui o carrinho do usuário do token **sem** devolver o estoque (**200**, *"Registro excluído com sucesso"*). |
| RN-09 | `DELETE /carrinhos/cancelar-compra` exclui o carrinho do usuário do token e **devolve** ao estoque as quantidades reservadas (**200**, *"Registro excluído com sucesso. Estoque dos produtos reabastecido"*). |
| RN-10 | Concluir ou cancelar compra sem ter carrinho não é tratado como erro: **200**, *"Não foi encontrado carrinho para esse usuário"*. |
| RN-11 | Enquanto o carrinho existir, o usuário dono dele não pode ser excluído (**400**, *"Não é permitido excluir usuário com carrinho cadastrado"*) e nenhum produto contido nele pode ser excluído (**400**, *"Não é permitido excluir produto que faz parte de carrinho"*). |
| RN-12 | `GET /carrinhos/{id}` exige um `id` com exatamente 16 caracteres alfanuméricos (senão **400** de validação). Com formato válido mas inexistente: **400**, *"Carrinho não encontrado"*. `GET /carrinhos` e `GET /carrinhos/{id}` **não exigem autenticação**. |
| RN-13 *(ordem de avaliação)* | As validações do `POST /carrinhos` rodam na ordem RN-01 → RN-02 → RN-03 → RN-04 → RN-05 → RN-06, e a primeira que falha encerra a requisição. Quando mais de uma condição é violada, só a mensagem da primeira aparece. |
| RN-14 — **confirmada por execução real em 12/11/2026** | A detecção de duplicados (RN-04) ordena a lista com `sort()` e depois compara apenas itens vizinhos. Como `sort()` sem função de comparação não altera a ordem de uma lista de objetos, só são detectados duplicados que já estejam lado a lado. Uma lista como `[P1, P2, P1]` passaria pela checagem. Nesse caso, a checagem de estoque (RN-06) também avaliaria cada linha isoladamente contra o estoque inteiro, e a baixa de estoque (RN-07) calcularia o novo saldo das duas linhas a partir do mesmo valor lido — baixando o estoque uma vez só. Confirmado: o carrinho `P1 × 3, P2 × 1, P1 × 3` foi aceito com estoque 5, e o estoque foi descontado só pela última ocorrência (de 5 para 2). Depois de concluir a compra, outro usuário ainda conseguiu comprar 2 unidades — 8 vendidas de 5. Ver [`relatorios-de-bugs/bug-car-001-duplicado-nao-vizinho-quebra-estoque.md`](../relatorios-de-bugs/bug-car-001-duplicado-nao-vizinho-quebra-estoque.md). |
| RN-15 — **hipótese refutada por execução real em 12/11/2026** | Pela leitura do código, parecia que o schema (RN-02) não definia tamanho mínimo para a lista `produtos` e que `"produtos": []` criaria um carrinho vazio. Na execução, a lista vazia foi recusada com **400**, *"produtos não contém 1 valor obrigatório"*: a regra de validação dos itens obriga a lista a ter ao menos um elemento. Mantida aqui como registro da hipótese descartada. |

## Tabela de decisão — validações do `POST /carrinhos` (RN-01 a RN-06)

São seis condições binárias, o que daria 64 combinações. Como o código avalia as condições em sequência e retorna na primeira falha (RN-13), a tabela pode ser reduzida: a partir da primeira condição violada, as seguintes não influenciam o resultado ("—" = indiferente).

| Regra | C1: Token válido? | C2: Corpo válido? | C3: Usuário sem carrinho? | C4: Sem produto duplicado? | C5: Todos os produtos existem? | C6: Quantidade ≤ estoque? | Resultado |
| --- | --- | --- | --- | --- | --- | --- | --- |
| R1 | Não | — | — | — | — | — | **401** — token inválido |
| R2 | Sim | Não | — | — | — | — | **400** — erro de validação do campo |
| R3 | Sim | Sim | Não | — | — | — | **400** — "Não é permitido ter mais de 1 carrinho" |
| R4 | Sim | Sim | Sim | Não | — | — | **400** — "Não é permitido possuir produto duplicado" |
| R5 | Sim | Sim | Sim | Sim | Não | — | **400** — "Produto não encontrado" |
| R6 | Sim | Sim | Sim | Sim | Sim | Não | **400** — "Produto não possui quantidade suficiente" |
| R7 | Sim | Sim | Sim | Sim | Sim | Sim | **201** — carrinho criado e estoque baixado |

Como no ParaBank, os casos CT-CAR-19 a CT-CAR-21 combinam deliberadamente duas condições violadas ao mesmo tempo. Eles não acrescentam um resultado novo hoje, mas são os que detectariam uma mudança na ordem das validações.

## Diagrama de transição de estados — ciclo de vida do carrinho

Do ponto de vista de um usuário, o carrinho tem dois estados: **S0 — sem carrinho** e **S1 — com carrinho**. O estoque dos produtos acompanha as transições.

| Estado atual | Evento | Próximo estado | Resposta | Efeito no estoque |
| --- | --- | --- | --- | --- |
| S0 | `POST /carrinhos` válido | S1 | 201 | Baixado |
| S0 | `DELETE concluir-compra` | S0 | 200 — sem carrinho | Nenhum |
| S0 | `DELETE cancelar-compra` | S0 | 200 — sem carrinho | Nenhum |
| S0 | `DELETE /usuarios/{id}` do próprio usuário | Usuário excluído | 200 | Nenhum |
| S1 | `POST /carrinhos` (qualquer corpo válido) | S1 | 400 — limite de 1 carrinho | Nenhum |
| S1 | `DELETE concluir-compra` | S0 | 200 | **Permanece baixado** |
| S1 | `DELETE cancelar-compra` | S0 | 200 | **Devolvido** |
| S1 | `DELETE /usuarios/{id}` do dono do carrinho | S1 | 400 — usuário com carrinho | Nenhum |
| S1 | `DELETE /produtos/{id}` de produto do carrinho | S1 | 400 — produto em carrinho | Nenhum |

As transições inválidas (as linhas que mantêm o estado com erro) são tão importantes quanto as válidas: são elas que garantem que um carrinho não fica "órfão" de usuário ou de produto.

## Casos de teste

Pré-condições comuns a todos os casos, salvo indicação contrária:

- ServeRest 3.2.2 em execução local (`npx serverest@3.2.2`), em `http://localhost:3000`.
- Um usuário administrador (`administrador: "true"`) para cadastrar os produtos de teste, e um usuário comum **U1** (`administrador: "false"`), autenticado via `POST /login`, sem carrinho.
- Produtos cadastrados para o teste: **P1** (preço 100, estoque 5), **P2** (preço 50, estoque 10) e **P3** (preço 30, estoque 0).
- Como o estoque muda a cada carrinho criado ou concluído, os produtos devem ser recriados (ou ter o estoque reajustado via `PUT /produtos/{id}`) antes de cada caso que dependa de estoque.

| ID | Cenário | Técnica | Dados de entrada | Resultado esperado | Prioridade |
| --- | --- | --- | --- | --- | --- |
| CT-CAR-01 | Criação de carrinho com todas as condições satisfeitas (cenário principal) | Tabela de decisão (R7) | Token de U1 · `produtos`: P1 × 2, P2 × 1 | **201**, "Cadastro realizado com sucesso" e `_id`. `GET /carrinhos/{_id}` retorna preço unitário 100 e 50, `precoTotal` 250, `quantidadeTotal` 3 e `idUsuario` de U1. Estoque: P1 = 3, P2 = 9 | Alta |
| CT-CAR-02 | Quantidade igual ao estoque (fronteira válida de RN-06) | Análise de valor limite | P1 × 5 (estoque 5) | **201**. Estoque de P1 passa a 0 | Alta |
| CT-CAR-03 | Quantidade uma unidade acima do estoque (fronteira inválida de RN-06) | Análise de valor limite | P1 × 6 (estoque 5) | **400**, "Produto não possui quantidade suficiente", com `quantidade` 6 e `quantidadeEstoque` 5. Nenhum carrinho criado; estoque de P1 continua 5 | Alta |
| CT-CAR-04 | Quantidade uma unidade abaixo do estoque | Análise de valor limite | P1 × 4 (estoque 5) | **201**. Estoque de P1 passa a 1 | Média |
| CT-CAR-05 | Produto com estoque zerado | Análise de valor limite | P3 × 1 (estoque 0) | **400**, "Produto não possui quantidade suficiente" | Média |
| CT-CAR-06 | Quantidade igual a zero (fronteira inválida de RN-02) | Análise de valor limite | P1 × 0 | **400** de validação: quantidade deve ser positiva. Nenhum carrinho criado | Alta |
| CT-CAR-07 | Quantidade negativa | Particionamento de equivalência (classe inválida) | P1 × -1 | **400** de validação: quantidade deve ser positiva | Média |
| CT-CAR-08 | Quantidade não inteira | Particionamento de equivalência (classe inválida) | P1 × 1.5 | **400** de validação: quantidade deve ser inteira | Baixa |
| CT-CAR-09 | Produto inexistente | Tabela de decisão (R5) | P1 × 1 e um `idProduto` com 16 caracteres que não corresponde a nenhum produto, na segunda posição | **400**, "Produto não encontrado", com o item e `index` 1. Estoque de P1 inalterado (nenhuma baixa parcial) | Alta |
| CT-CAR-10 | Produto duplicado em posições vizinhas | Tabela de decisão (R4) | P1 × 1, P1 × 1 | **400**, "Não é permitido possuir produto duplicado", `idProdutosDuplicados` contendo P1 | Alta |
| CT-CAR-11 *(confirmado como defeito em 12/11/2026 — BUG-CAR-001)* | Produto duplicado em posições **não** vizinhas (ver RN-14) | Tabela de decisão (R4) | P1 × 3, P2 × 1, P1 × 3 (estoque de P1 = 5) | Pela regra RN-04: **400**, produto duplicado. **Obtido:** 201, carrinho com 6 unidades de P1 e estoque descontado só pela última ocorrência (5 → 2). Cancelar devolve o estoque a 5; concluir mantém 2, e outro usuário consegue comprar mais 2 | Alta |
| CT-CAR-12 *(resultado esperado corrigido após a sessão exploratória — ver RN-15)* | Lista de produtos vazia | Particionamento de equivalência (classe inválida) | `produtos`: `[]` | **400**, *"produtos não contém 1 valor obrigatório"*. Nenhum carrinho criado; U1 continua livre para criar um carrinho | Média |
| CT-CAR-13 | Segundo carrinho para o mesmo usuário | Transição de estados (S1 + POST) | U1 já com carrinho (após CT-CAR-01) · novo POST com P2 × 1 | **400**, "Não é permitido ter mais de 1 carrinho". Estoque de P2 inalterado | Alta |
| CT-CAR-14 | Requisição sem o cabeçalho `Authorization` | Tabela de decisão (R1) | Corpo válido, sem token | **401**, "Token de acesso ausente, inválido, expirado ou usuário do token não existe mais" | Alta |
| CT-CAR-15 | Token sem o prefixo `Bearer` | Particionamento de equivalência (classe inválida) | `Authorization`: apenas o JWT, sem "Bearer " | **401**, mesma mensagem de token inválido | Média |
| CT-CAR-16 | Token emitido antes de uma troca de senha | Particionamento de equivalência (classe inválida) | Login de U1 → `PUT /usuarios/{id}` de U1 com nova senha → POST de carrinho com o token antigo | **401**: o token carrega o e-mail e a senha do login, e o usuário não existe mais com essa combinação | Média |
| CT-CAR-17 | Token expirado | Análise de valor limite | Token de U1 usado mais de 600 segundos após o login | **401**, mesma mensagem de token inválido | Baixa |
| CT-CAR-18 | Usuário comum (não administrador) cria carrinho | Particionamento de equivalência | Token de U1 (`administrador: "false"`), corpo válido | **201** — carrinho não é rota exclusiva de administrador (contraste com `POST /produtos`, que retorna 403 para o mesmo usuário) | Média |
| CT-CAR-19 | Sem token **e** corpo inválido (R1 × R2) | Tabela de decisão | Sem `Authorization` · `produtos`: P1 × 0 | **401**: a autenticação é avaliada antes do schema; a mensagem de validação não aparece | Média |
| CT-CAR-20 | Corpo inválido **e** usuário já com carrinho (R2 × R3) | Tabela de decisão | U1 com carrinho · `produtos`: P1 × 0 | **400** de validação do campo (o schema é avaliado antes da regra de um carrinho por usuário) | Baixa |
| CT-CAR-21 | Usuário já com carrinho **e** produto duplicado (R3 × R4) | Tabela de decisão | U1 com carrinho · P1 × 1, P1 × 1 | **400**, "Não é permitido ter mais de 1 carrinho" (prevalece sobre a mensagem de duplicado) | Baixa |
| CT-CAR-22 | Concluir compra com carrinho | Transição de estados (S1 → S0) | Após CT-CAR-01: `DELETE /carrinhos/concluir-compra` com token de U1 | **200**, "Registro excluído com sucesso". `GET /carrinhos?idUsuario=<U1>` retorna lista vazia. Estoque **não** é devolvido: P1 = 3, P2 = 9. U1 pode criar um novo carrinho | Alta |
| CT-CAR-23 | Cancelar compra com carrinho | Transição de estados (S1 → S0) | Após CT-CAR-01: `DELETE /carrinhos/cancelar-compra` com token de U1 | **200**, "Registro excluído com sucesso. Estoque dos produtos reabastecido". Carrinho excluído; estoque devolvido: P1 = 5, P2 = 10 | Alta |
| CT-CAR-24 | Concluir e cancelar compra sem ter carrinho | Transição de estados (S0 → S0) | U1 sem carrinho: `DELETE concluir-compra` e depois `DELETE cancelar-compra` | **200** nas duas, "Não foi encontrado carrinho para esse usuário". Nenhum estoque alterado | Média |
| CT-CAR-25 | Excluir usuário que tem carrinho | Transição de estados (S1, evento inválido) | U1 com carrinho: `DELETE /usuarios/{id de U1}` | **400**, "Não é permitido excluir usuário com carrinho cadastrado", com o `idCarrinho`. U1 e o carrinho continuam existindo | Alta |
| CT-CAR-26 | Excluir produto que está num carrinho | Transição de estados (S1, evento inválido) | U1 com carrinho contendo P1: `DELETE /produtos/{id de P1}` com token de administrador | **400**, "Não é permitido excluir produto que faz parte de carrinho", com o id do carrinho. P1 continua existindo | Alta |
| CT-CAR-27 | Alteração de preço depois da criação do carrinho | Particionamento de equivalência | Após CT-CAR-01: `PUT /produtos/{id de P1}` alterando o preço para 200 → `GET /carrinhos/{_id}` | O carrinho mantém `precoUnitario` 100 e `precoTotal` 250 — o preço é registrado no momento da criação (RN-07) | Média |
| CT-CAR-28 | Consulta de carrinho por id inexistente ou malformado | Particionamento de equivalência | `GET /carrinhos/{id}` com (a) 16 caracteres alfanuméricos inexistentes e (b) 15 caracteres | (a) **400**, "Carrinho não encontrado"; (b) **400** de validação do `id`. Ponto para discussão, não defeito: pela convenção REST, um recurso inexistente costuma retornar 404 | Baixa |
| CT-CAR-29 | Consulta de carrinhos sem autenticação | Particionamento de equivalência | `GET /carrinhos` sem `Authorization`, com carrinhos de usuários diferentes cadastrados | **200**, lista com os carrinhos de todos os usuários (RN-12). Comportamento esperado para uma API de estudo, mas registrado: numa aplicação real seria exposição de dados de outros clientes | Baixa |

## Rastreabilidade regra → casos de teste

| Regra | Casos de teste que a cobrem |
| --- | --- |
| RN-01 | CT-CAR-14, CT-CAR-15, CT-CAR-16, CT-CAR-17, CT-CAR-18, CT-CAR-19 |
| RN-02 | CT-CAR-06, CT-CAR-07, CT-CAR-08, CT-CAR-19, CT-CAR-20 |
| RN-03 | CT-CAR-13, CT-CAR-20, CT-CAR-21 |
| RN-04 | CT-CAR-10, CT-CAR-11, CT-CAR-21 |
| RN-05 | CT-CAR-09 |
| RN-06 | CT-CAR-02, CT-CAR-03, CT-CAR-04, CT-CAR-05 |
| RN-07 | CT-CAR-01, CT-CAR-02, CT-CAR-04, CT-CAR-18, CT-CAR-27 |
| RN-08 | CT-CAR-22 |
| RN-09 | CT-CAR-23 |
| RN-10 | CT-CAR-24 |
| RN-11 | CT-CAR-25, CT-CAR-26 |
| RN-12 | CT-CAR-28, CT-CAR-29 |
| RN-13 | CT-CAR-19, CT-CAR-20, CT-CAR-21 |
| RN-14 | CT-CAR-11 |
| RN-15 | CT-CAR-12 |

## Observações e riscos

- Como no ParaBank, a regra veio da leitura do código-fonte, e nenhum caso havia sido executado no momento em que este documento foi escrito. Os 29 casos foram executados em 12/11/2026 — ver [`sessoes-exploratorias/serverest-carrinhos-12-11-2026.md`](../sessoes-exploratorias/serverest-carrinhos-12-11-2026.md). Vinte e sete se comportaram como previsto.
- Os dois casos que partiam de hipóteses da leitura do código tiveram destinos opostos: CT-CAR-11 (RN-14) se confirmou como defeito e virou o BUG-CAR-001; CT-CAR-12 (RN-15) foi refutado, e o resultado esperado foi corrigido. A leitura de código gerou as duas hipóteses, mas só a execução decidiu entre elas.
- A baixa de estoque acontece na criação do carrinho (RN-07), não na conclusão da compra. Isso não é um defeito, mas é uma decisão de negócio relevante: um carrinho abandonado (nunca concluído nem cancelado) reserva estoque indefinidamente. Fica como ponto de atenção para a sessão exploratória.
- Os casos de estoque são sensíveis à ordem de execução, porque cada carrinho criado, concluído ou cancelado altera o estoque usado pelos casos seguintes. Por isso a execução deve ser feita numa instância local, com os produtos de teste recriados entre os casos.
- A coleção Postman usada na execução está versionada em [`colecoes-postman/serverest-carrinhos.postman_collection.json`](../colecoes-postman/serverest-carrinhos.postman_collection.json). As asserções das pastas CT-CAR-11a/b/c verificam o comportamento correto e falham de propósito na versão 3.2.2: a falha é a evidência do BUG-CAR-001.
