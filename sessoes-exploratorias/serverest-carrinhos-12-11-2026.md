# Sessão de teste exploratório — ServeRest: Carrinhos

## Ficha da sessão

| Campo | Valor |
| --- | --- |
| Aplicação sob teste | ServeRest (API REST), versão 3.2.2, instância local (`npx serverest@3.2.2`) |
| Área | Carrinhos (`/carrinhos`), com Usuários, Login e Produtos como pré-condição |
| Testador | Thomas Teixeira |
| Data | 12/11/2026 |
| Ferramentas | Postman (coleção versionada em [`colecoes-postman/serverest-carrinhos.postman_collection.json`](../colecoes-postman/serverest-carrinhos.postman_collection.json)) e uma passada de apoio por script |
| Registro | Sequencial, por caso de teste, com as requisições e respostas reais |
| Referência | Continuação do documento [`casos-de-teste/serverest-carrinhos.md`](../casos-de-teste/serverest-carrinhos.md) |

## Metodologia

Esta é a primeira sessão do portfólio feita diretamente contra uma API, sem interface gráfica. Ela foi executada em duas etapas:

1. **Passada de apoio.** Os 29 casos do artefato de design foram executados em sequência por um script que monta as requisições HTTP e registra cada requisição e resposta. O objetivo era cobrir rapidamente todos os casos e levantar, antes da execução manual, onde a API divergia do esperado.
2. **Execução no Postman pelo testador.** A partir dessa passada, foi montada uma coleção Postman com os 29 casos e asserções automáticas sobre status, mensagem e estoque. O testador executou a coleção completa no Collection Runner e reexecutou manualmente, requisição por requisição, os casos que mais importavam (as duas hipóteses levantadas na leitura do código). Os resultados observados no Postman coincidiram com os da passada de apoio.

Cada caso cria seu próprio usuário e seus próprios produtos. Como cada carrinho criado, concluído ou cancelado altera o estoque, reaproveitar dados entre casos faria o resultado de um caso depender da ordem de execução dos anteriores.

O caso de token expirado (CT-CAR-17) foi executado numa segunda instância, iniciada com `--timeout 5` (validade do token reduzida de 600 para 5 segundos), para verificar o mecanismo de expiração sem esperar dez minutos.

## Charter

> Executar contra a API os 29 casos do design de Carrinhos, com atenção especial às duas hipóteses levantadas na leitura do código (produto duplicado em posições não vizinhas e lista de produtos vazia), e seguir qualquer efeito sobre o estoque que surgisse durante a exploração.

## Cobertura por área

| Área | Proporção aproximada da sessão |
| --- | --- |
| Execução dos casos planejados (criação, validações, autenticação, ciclo de vida) | 45% |
| Investigação do produto duplicado e do seu efeito no estoque | 35% |
| Montagem e validação da coleção Postman | 20% |

## Notas de teste (registro sequencial)

1. **Cenário principal (CT-CAR-01).** Carrinho com P1 × 2 e P2 × 1 criado com **201**. A consulta do carrinho mostrou `precoUnitario` 100 e 50, `precoTotal` 250 e `quantidadeTotal` 3. O estoque caiu de 5 para 3 (P1) e de 10 para 9 (P2) no momento da criação, como indicava o código.

2. **Fronteiras de estoque (CT-CAR-02 a 05).** Quantidade igual ao estoque foi aceita e zerou o estoque. Uma unidade acima foi recusada com **400**, *"Produto não possui quantidade suficiente"*, e a resposta trouxe a quantidade pedida e a quantidade em estoque. Produto com estoque zero também foi recusado. Em todos os casos recusados, o estoque permaneceu intacto.

3. **Validação do corpo (CT-CAR-06 a 08).** Quantidade 0, negativa ou decimal foram recusadas com **400**. Um detalhe na resposta: além do erro específico (`"produtos[0].quantidade deve ser um número positivo"`), a API sempre acrescenta `"produtos não contém 1 valor obrigatório"`. A segunda mensagem é enganosa, porque o item existe — ele só é inválido. Não chega a ser um defeito funcional, mas é uma mensagem que confundiria quem consome a API.

4. **Produto inexistente e duplicado vizinho (CT-CAR-09 e 10).** Os dois foram recusados com as mensagens documentadas. No produto inexistente, a resposta indicou corretamente a posição do item (`index` 1), e o estoque do outro produto da lista não sofreu baixa parcial.

5. **Duplicado em posições não vizinhas (CT-CAR-11) — hipótese confirmada.** O carrinho `P1 × 3, P2 × 1, P1 × 3`, com estoque de P1 igual a 5, foi **aceito com 201**. O carrinho registrou 6 unidades de P1 (mais do que o estoque) e `precoTotal` 650. O estoque de P1, porém, caiu só de 5 para 2: apenas a última ocorrência do produto foi descontada.

6. **Seguindo o achado: o que acontece ao cancelar e ao concluir.** O comportamento das duas saídas do ciclo de vida foi explorado:
   - **Cancelar** devolveu o estoque de P1 para 5, o valor original. O cancelamento devolve exatamente o que foi baixado, então o estoque volta a ficar coerente.
   - **Concluir** manteve o estoque de P1 em 2, embora 6 unidades tenham sido vendidas. Em seguida, **outro usuário comprou mais 2 unidades de P1** com sucesso. Ao todo, 8 unidades foram vendidas de um produto que tinha 5 em estoque.

7. **Duplicado com quantidades diferentes.** Para entender qual ocorrência é descontada, o mesmo cenário foi repetido com `P1 × 2, P2 × 1, P1 × 1`. O estoque caiu de 5 para 4: só a quantidade da **última** ocorrência (1) foi descontada, não a da primeira (2) nem a soma (3). Isso confirmou a causa apontada na leitura do código, detalhada no relato do defeito.

8. **Lista vazia (CT-CAR-12) — hipótese refutada.** `"produtos": []` foi recusado com **400**, *"produtos não contém 1 valor obrigatório"*. A regra de validação dos itens obriga a lista a ter pelo menos um elemento, detalhe que não ficou evidente na leitura do código. O usuário continuou livre para criar um carrinho de verdade em seguida.

9. **Um carrinho por usuário e ordem das validações (CT-CAR-13, 19 a 21).** Segundo carrinho recusado com *"Não é permitido ter mais de 1 carrinho"*. Nas combinações de duas condições violadas, prevaleceu sempre a primeira da sequência prevista: token antes do corpo, corpo antes da regra de um carrinho, e regra de um carrinho antes da de duplicado.

10. **Autenticação (CT-CAR-14 a 18).** Sem token, token sem o prefixo `Bearer`, token emitido antes de uma troca de senha e token expirado foram todos recusados com **401** e a mesma mensagem. Um usuário comum (não administrador) criou carrinho normalmente, enquanto o mesmo usuário recebeu **403** ao tentar cadastrar um produto.

11. **Ciclo de vida do carrinho (CT-CAR-22 a 26).**
    - **Concluir** apagou o carrinho e manteve o estoque baixado.
    - **Cancelar** apagou o carrinho e devolveu o estoque.
    - **Sem carrinho:** as duas operações responderam **200** com *"Não foi encontrado carrinho para esse usuário"*.
    - **Bloqueios:** com o carrinho ativo, a API recusou excluir o usuário dono e o produto contido nele.

12. **Preço registrado e consultas (CT-CAR-27 a 29).**
    - **Preço registrado:** alterar o preço de P1 para 200 depois da criação não mudou o carrinho (`precoUnitario` 100, `precoTotal` 250).
    - **Id inexistente:** a consulta respondeu **400**, *"Carrinho não encontrado"*.
    - **Id malformado:** a consulta respondeu **400** de validação.
    - **Listagem sem autenticação:** `GET /carrinhos` sem token devolveu os carrinhos de todos os usuários da instância.

## Defeitos confirmados nesta sessão

1. **Produto repetido em posições não vizinhas passa pela validação, permite reservar mais do que o estoque e faz o estoque descontar só a última ocorrência (severidade alta).** Relato completo em [`relatorios-de-bugs/bug-car-001-duplicado-nao-vizinho-quebra-estoque.md`](../relatorios-de-bugs/bug-car-001-duplicado-nao-vizinho-quebra-estoque.md).

## Hipótese testada e descartada

A suspeita de que uma lista de produtos vazia criaria um carrinho vazio (RN-15 do artefato de design, caso CT-CAR-12) **não se confirmou**: a API recusa a lista vazia. O artefato de design foi corrigido. Fica registrado porque a conclusão veio da execução, não da leitura do código, que sozinha sugeria o contrário.

## Confirmação de comportamento correto

Fora o CT-CAR-11, os 28 casos restantes se comportaram como o design previa, incluindo as fronteiras de estoque, as quatro variações de token inválido, a ordem das validações e as transições inválidas do ciclo de vida. A regra do carrinho está, no geral, bem implementada. O defeito encontrado fica concentrado num ponto específico: a detecção de duplicados.

## Observações que não viraram relato de defeito

- **Mensagem de validação enganosa (item 3).** `"produtos não contém 1 valor obrigatório"` aparece junto de qualquer erro de item, mesmo quando o item existe.
- **Códigos de status fora da convenção REST.** Carrinho inexistente responde 400 em vez de 404. Concluir ou cancelar sem carrinho responde 200, sem que nada tenha acontecido. São decisões de design da API, mas obrigam quem a consome a ler o texto da mensagem para distinguir os casos.
- **Listagem pública de carrinhos (item 12).** É aceitável numa API de estudo, mas numa aplicação real seria exposição de dados de outros clientes.
- **Estoque reservado na criação.** Como a baixa acontece na criação do carrinho e não na conclusão, um carrinho abandonado (nunca concluído nem cancelado) segura o estoque indefinidamente. Não há expiração de carrinho.

## Perguntas em aberto / ideias para a próxima sessão

- **Duas requisições simultâneas.** O cálculo do novo estoque parte de um valor lido antes da gravação. Vale investigar se dois usuários criando carrinho do mesmo produto ao mesmo tempo também podem fazer o estoque descontar só uma das compras. Testar isso exige requisições simultâneas, fora do escopo de uma sessão manual.
- **Alteração de estoque durante um carrinho ativo.** Se um administrador alterar o estoque de um produto via `PUT /produtos/{id}` enquanto ele está num carrinho, o cancelamento soma a reserva sobre o novo valor. Vale definir qual seria o comportamento correto e testar.

## Avaliação de risco residual

O BUG-CAR-001 é o único defeito confirmado, mas atinge diretamente a integridade do estoque. Ele não depende de nenhuma condição rara: basta repetir um produto, fora de sequência, na lista enviada. Numa loja real, isso significa vender o que não existe. O cancelamento devolve o estoque corretamente, o que limita o dano a compras concluídas. Por outro lado, também esconde o problema de quem testa só o fluxo de cancelamento.
