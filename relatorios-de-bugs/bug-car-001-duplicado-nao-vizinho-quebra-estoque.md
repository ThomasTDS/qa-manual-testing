# Relato de bug — Produto repetido fora de sequência passa pela validação e quebra o controle de estoque

## Metadados

| Campo | Valor |
| --- | --- |
| ID | BUG-CAR-001 |
| Aplicação | ServeRest (API REST) |
| Funcionalidade | Carrinhos — `POST /carrinhos` |
| Ambiente | Instância local, versão 3.2.2 (`npx serverest@3.2.2`) |
| Data de identificação | 12/11/2026 |
| Identificado por | Thomas Teixeira, na leitura do código-fonte, confirmado em sessão de teste exploratório |
| Severidade | Alta |
| Prioridade | Alta |
| Status | Aberto (não reportado ao mantenedor — mantido como registro de portfólio) |

## Resumo

A API não permite que o mesmo produto apareça duas vezes num carrinho e recusa corretamente uma lista como `[P1, P1]`. Porém, se as duas ocorrências do produto **não estiverem lado a lado** — por exemplo, `[P1, P2, P1]` —, o carrinho é aceito. Nesse caso, cada linha é comparada com o estoque separadamente, o que permite reservar mais unidades do que existem. Além disso, o estoque é descontado apenas pela quantidade da **última** ocorrência do produto. Ao concluir a compra, o produto continua com unidades "disponíveis" que já foram vendidas, e outros clientes conseguem comprá-las.

## Ambiente e pré-condições

- ServeRest 3.2.2 em execução local.
- Um usuário autenticado (qualquer perfil) sem carrinho.
- Dois produtos cadastrados: **P1** (preço 100, estoque 5) e **P2** (preço 50, estoque 10).

## Passos para reproduzir

1. Autenticar o usuário (`POST /login`) e guardar o token.
2. Enviar `POST /carrinhos` com o token e o corpo:
   ```json
   {
     "produtos": [
       { "idProduto": "<P1>", "quantidade": 3 },
       { "idProduto": "<P2>", "quantidade": 1 },
       { "idProduto": "<P1>", "quantidade": 3 }
     ]
   }
   ```
3. Consultar o carrinho criado (`GET /carrinhos/{_id}`) e o estoque de P1 (`GET /produtos/{P1}`).
4. Concluir a compra (`DELETE /carrinhos/concluir-compra`) e consultar novamente o estoque de P1.
5. Com um segundo usuário, criar um carrinho com P1 × 2.

A coleção [`colecoes-postman/serverest-carrinhos.postman_collection.json`](../colecoes-postman/serverest-carrinhos.postman_collection.json) reproduz esses passos nas pastas CT-CAR-11a, 11b e 11c.

## Resultado esperado

No passo 2, a API deveria recusar a requisição com **400**, *"Não é permitido possuir produto duplicado"*, exatamente como faz quando as ocorrências estão lado a lado. Mesmo que a repetição fosse aceita como regra de negócio, a soma das quantidades de P1 (6) deveria ser comparada com o estoque (5), e a requisição recusada por estoque insuficiente. Em nenhum cenário uma compra deveria deixar disponíveis unidades que já foram vendidas.

## Resultado obtido

- O carrinho é criado com **201** e registra 6 unidades de P1, com estoque de 5.
- O estoque de P1 cai de 5 para **2**: só a última ocorrência (3 unidades) é descontada.
- Depois de concluir a compra, o estoque de P1 continua em 2.
- O segundo usuário compra mais 2 unidades de P1 com sucesso. No total, **8 unidades vendidas de um produto que tinha 5**.

## Evidência

Requisição do passo 2 (ids reais da execução):

```json
POST /carrinhos
{"produtos": [{"idProduto": "7khu4GKVni3zhszw", "quantidade": 3},
              {"idProduto": "4ciQQ2xlhMpE2V0L", "quantidade": 1},
              {"idProduto": "7khu4GKVni3zhszw", "quantidade": 3}]}

201 {"message": "Cadastro realizado com sucesso", "_id": "HFqe33tYMtaqACLX"}
```

Carrinho registrado (numa execução idêntica, CT-CAR-11a):

```json
{"produtos": [{"idProduto": "bMqeeMnQvMJj62Js", "quantidade": 3, "precoUnitario": 100},
              {"idProduto": "BNb3cjTtd0ELgtgO", "quantidade": 1, "precoUnitario": 50},
              {"idProduto": "bMqeeMnQvMJj62Js", "quantidade": 3, "precoUnitario": 100}],
 "precoTotal": 650, "quantidadeTotal": 7}
```

Estoque de P1 ao longo do cenário (CT-CAR-11b):

| Momento | Estoque de P1 | Unidades de P1 vendidas |
| --- | --- | --- |
| Antes do carrinho | 5 | 0 |
| Após criar o carrinho com P1 × 3 + P1 × 3 | 2 | 6 reservadas |
| Após concluir a compra | 2 | 6 |
| Após o segundo usuário comprar P1 × 2 (201) | 0 | 8 |

### Qual ocorrência é descontada

Com quantidades diferentes nas duas ocorrências (`P1 × 2, P2 × 1, P1 × 1`, estoque 5), o estoque caiu para **4**. Foi descontada só a quantidade da última ocorrência (1), não a da primeira (2) nem a soma (3).

### Por que o cancelamento esconde o problema

Ao cancelar a compra, o estoque volta exatamente ao valor original (5 no exemplo), porque a reposição sofre do mesmo efeito e devolve apenas a última ocorrência. Quem testar só o fluxo de cancelamento vê o estoque "voltar ao normal" e não percebe o defeito. O problema aparece na conclusão da compra.

## Análise técnica (causa provável)

A causa foi identificada na leitura do código-fonte (commit `0a8068f`) e é coerente com todos os resultados observados. Três pontos se combinam:

1. **Detecção de duplicados** (`extrairProdutosDuplicados`, em `src/services/carrinhos-service.js`). A lista é ordenada com `slice().sort()` e depois cada item é comparado só com o vizinho seguinte. Como `sort()` sem função de comparação ordena pelo texto de cada elemento, e todo objeto vira o mesmo texto (`"[object Object]"`), a ordem não muda. Na prática, só são detectados os duplicados que o próprio cliente já enviou lado a lado.
2. **Checagem de estoque** (`getPrecosUnitariosPorLote`, em `src/services/produtos-service.js`). Cada linha é comparada com o estoque inteiro do produto, de forma independente. Duas linhas de 3 unidades passam, cada uma, pela checagem "3 ≤ 5".
3. **Baixa de estoque** (`updateQuantidadePorLote`, no mesmo arquivo). O estoque é lido uma única vez, e o novo valor de cada linha é calculado a partir dessa mesma leitura (`5 − 3 = 2` para as duas linhas de P1). As duas gravações escrevem um valor absoluto, então a última sobrescreve a primeira. A reposição no cancelamento (`restoreQuantidadePorLote`) segue a mesma lógica, o que explica por que o cancelamento devolve exatamente o que foi descontado.

Uma correção possível é detectar duplicados sem depender da ordem (por exemplo, comparando o tamanho da lista com o de um conjunto de `idProduto`) e, se a repetição fosse aceitável, somar as quantidades por produto antes de checar e descontar o estoque.

## Justificativa de severidade e prioridade

**Severidade: Alta.** O defeito permite vender mais unidades do que existem e deixa o estoque registrado diferente do real, sem nenhum aviso. Numa loja real, isso gera pedidos impossíveis de entregar, cancelamentos forçados e estoque contábil errado. Não foi classificado como crítico por dois motivos: depende de uma lista com o mesmo produto repetido fora de sequência, algo que um front-end bem feito normalmente não envia, e o cancelamento não amplia o dano.

**Prioridade: Alta.** Qualquer cliente da API, ou alguém mal-intencionado chamando a API diretamente, consegue reproduzir com uma única requisição. Além disso, a regra que deveria impedir o problema já existe e está documentada: ela só não funciona em todos os casos.

## Rastreabilidade

- Regras de negócio relacionadas: `RN-04` e `RN-14`, em [`casos-de-teste/serverest-carrinhos.md`](../casos-de-teste/serverest-carrinhos.md).
- Caso de teste de origem: `CT-CAR-11`, no mesmo documento.
- Execução que confirmou o defeito: itens 5 a 7 da sessão registrada em [`sessoes-exploratorias/serverest-carrinhos-12-11-2026.md`](../sessoes-exploratorias/serverest-carrinhos-12-11-2026.md).
