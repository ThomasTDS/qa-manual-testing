# Testes Manuais e Exploratórios

![Licença](https://img.shields.io/badge/licen%C3%A7a-MIT-blue.svg)

Portfólio de artefatos de teste manual e exploratório: design de casos de
teste com rastreabilidade de regra de negócio, relatórios de sessão de teste
exploratório (Session-Based Test Management) e relatos de defeitos bem
documentados.

Este repositório é complementar ao portfólio de automação de testes do
mesmo autor e está em construção — o conteúdo será adicionado de forma
incremental.

## Estrutura

| Pasta | Conteúdo |
| --- | --- |
| [`casos-de-teste/`](casos-de-teste/) | Design de casos de teste com rastreabilidade de regra de negócio, documentando a técnica utilizada (tabela de decisão, particionamento de equivalência, análise de valor limite) para cada funcionalidade analisada |
| [`sessoes-exploratorias/`](sessoes-exploratorias/) | Relatórios de sessão de teste exploratório (Session-Based Test Management): charter, cobertura por área, registro cronológico de teste e defeitos confirmados |

Ainda será adicionada a pasta de relatos de defeitos, conforme o
portfólio avançar.

## Artefatos

- [Solicitar Empréstimo — ParaBank](casos-de-teste/parabank-solicitar-emprestimo.md):
  regra de negócio extraída diretamente do código-fonte oficial da
  aplicação, tabela de decisão e dezesseis casos de teste rastreados até a
  regra que cada um valida.
- [Sessão exploratória — Solicitar Empréstimo (ParaBank)](sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md):
  execução real contra a aplicação, com dois defeitos confirmados (erro
  interno com valor de empréstimo zero e débito de entrada sem checagem
  de saldo individual da conta de origem) e uma correção aplicada ao
  artefato de design de casos de teste a partir do que foi observado.

## Autor

Desenvolvido por [Thomas Teixeira](https://github.com/ThomasTDS) como
projeto de estudo e portfólio em teste de software.
