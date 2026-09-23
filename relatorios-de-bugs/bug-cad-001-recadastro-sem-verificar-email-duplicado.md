# Relato de bug — Recadastro com e-mail já existente cria conta paralela, sem aviso de duplicidade

## Metadados

| Campo | Valor |
| --- | --- |
| ID | BUG-CAD-001 |
| Aplicação | BugBank (https://bugbank.netlify.app) |
| Funcionalidade | Cadastro / Login |
| Ambiente | Instância pública de demonstração |
| Data de identificação | 22/09/2026 |
| Identificado por | Thomas Teixeira, durante sessão de teste exploratório |
| Severidade | Alta |
| Prioridade | Alta |
| Status | Aberto (não reportado ao mantenedor — instância pública mantida pela comunidade de QA como ambiente de treino) |

## Resumo

O BugBank não tem nenhum fluxo de recuperação de senha ("Esqueci minha senha"). Ao tentar contornar essa limitação se cadastrando novamente com o mesmo e-mail de uma conta já existente, a aplicação aceita o novo cadastro sem qualquer aviso de e-mail duplicado, e cria uma **segunda conta completamente independente**, com um número de conta diferente da original. Não há nenhuma indicação, na hora do recadastro, de que aquele e-mail já pertence a outra conta.

## Ambiente e pré-condições

- Cliente já cadastrado no BugBank (conta de origem, aqui chamada de conta "383").
- Acesso à tela de Cadastro (mesma usada para o primeiro cadastro).

## Passos para reproduzir

1. Cadastrar um cliente novo no BugBank, com um e-mail qualquer (por exemplo, `teste@exemplo.com`), anotando o número de conta exibido no modal de confirmação.
2. Sem fazer logout nem qualquer ação adicional, acessar novamente a tela de Cadastro.
3. Preencher o formulário de cadastro novamente, usando o **mesmo e-mail** do passo 1, mas um nome e senha à escolha.
4. Enviar o formulário.

## Resultado esperado

A aplicação deveria recusar o cadastro, informando que o e-mail já está em uso — ou, alternativamente, já que não existe fluxo de recuperação de senha, deveria existir algum mecanismo dedicado para isso (redefinição de senha vinculada à conta existente), em vez de permitir um novo cadastro completo e independente sob o mesmo e-mail.

## Resultado obtido

O cadastro é aceito normalmente, sem nenhuma mensagem de erro ou aviso sobre duplicidade de e-mail. Uma nova conta é criada, com número diferente da primeira (aqui identificada como conta "545"), sob o mesmo endereço de e-mail da conta "383".

## Evidência

Print da tela de login do BugBank, mostrando o aviso oficial da própria aplicação de que não há banco de dados e os dados ficam apenas em memória local — informação usada na análise técnica abaixo:

![Tela de login do BugBank com o aviso "a aplicação não conta com um banco de dados, todas as informações são armazenadas em memória local"](evidencias/bugbank-login-sem-banco-de-dados.png)

Relato direto do testador, a partir de execução manual pela interface:

- Conta 1 criada: número de conta "383", e-mail `X`, senha original.
- Recadastro realizado com o mesmo e-mail `X`, senha diferente da original.
- Conta 2 criada com sucesso: número de conta "545", sem qualquer mensagem de erro ou aviso de duplicidade durante o processo.

## Pergunta em aberto (não confirmada nesta sessão)

Não foi testado, nesta sessão, se a conta 383 original continua acessível fazendo login com o e-mail e a senha originais depois do recadastro. Duas hipóteses permanecem em aberto para uma próxima verificação:

- **Hipótese A:** as duas contas coexistem de forma independente, e o login resolve para uma delas de forma ambígua ou imprevisível (por exemplo, sempre a mais recente), tornando a conta 383 inacessível na prática, mesmo que os dados ainda existam em memória.
- **Hipótese B:** a segunda conta de fato substitui a referência da primeira em qualquer estrutura de dados indexada por e-mail (coerente com a aplicação não usar banco de dados e manter tudo em memória local), fazendo a conta 383 e seu saldo ficarem permanentemente órfãos e inacessíveis.

Qualquer uma das duas hipóteses caracteriza um problema real; a diferença está apenas em qual é o mecanismo exato. Fica registrada como item prioritário para a próxima sessão exploratória sobre o BugBank.

## Análise técnica (causa provável)

A própria aplicação declara que não usa banco de dados, mantendo todas as informações em memória local durante a execução. Isso é consistente com o comportamento observado: o cadastro provavelmente não faz nenhuma consulta prévia para verificar se o e-mail informado já está associado a uma conta existente antes de criar uma nova — ausência que, em uma aplicação com persistência real, normalmente seria coberta por uma restrição de unicidade no banco de dados (e aqui, sem banco de dados, precisaria ser replicada explicitamente na lógica de cadastro).

## Justificativa de severidade e prioridade

**Severidade: Alta.** Trata-se de uma falha de controle de identidade em uma aplicação bancária (ainda que fictícia): permitir múltiplas contas sob o mesmo e-mail, sem nenhum aviso, compromete a premissa básica de que um e-mail identifica uma conta. Combinado com a ausência de recuperação de senha, o caminho mais natural que um usuário real tentaria ao esquecer a senha (cadastrar de novo) o deixa com duas identidades desencontradas, sem clareza sobre qual delas está "ativa" e sem visibilidade sobre o que aconteceu com o saldo da conta original.

**Prioridade: Alta.** "Esqueci minha senha" é uma funcionalidade básica esperada em qualquer aplicação com autenticação, e sua ausência já seria digna de nota. O problema se agrava porque o caminho alternativo mais óbvio ao usuário (recadastrar) não é bloqueado nem avisado — ele simplesmente parece funcionar, escondendo o problema real até o usuário precisar acessar a conta original de novo.

## Rastreabilidade

- Descoberto durante a sessão exploratória registrada em [`sessoes-exploratorias/bugbank-transferencia-22-09-2026.md`](../sessoes-exploratorias/bugbank-transferencia-22-09-2026.md), item 2 e 3 do registro sequencial.
- Não corresponde a nenhum caso do artefato [`casos-de-teste/bugbank-transferencia.md`](../casos-de-teste/bugbank-transferencia.md), pois esse documento cobre apenas a funcionalidade de Transferência — este defeito foi encontrado ao investigar Cadastro e Login como pré-condição, ilustrando um dos valores da técnica de teste exploratório: encontrar problemas fora do escopo original planejado.
