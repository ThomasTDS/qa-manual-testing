# Manual and Exploratory Testing

![License](https://img.shields.io/badge/license-MIT-blue.svg)

*[Leia em português](README.md) · [Leer en español](README.es.md)*

Portfolio of manual and exploratory testing artifacts: test case design
with business rule traceability, exploratory testing session reports
(Session-Based Test Management), and well-documented bug reports.

This repository complements the same author's test automation portfolio
and is under active development — content is added incrementally.

## Highlights

Four findings that summarize the kind of reasoning this portfolio tries to demonstrate — the process behind each one matters more than the list itself:

- **[An account left behind without warning (BugBank)](relatorios-de-bugs/bug-cad-001-recadastro-sem-verificar-email-duplicado.md).** BugBank has no "forgot password" button. The most obvious workaround a real user would try — registering again with the same email — is accepted with no duplicate-email warning, and creates a second account completely disconnected from the first. The finding didn't come from testing an obvious bug scenario; it came from simulating the behavior of someone who just wants back into their own account.

- **[An approved loan that leaves the account overdrawn (ParaBank)](relatorios-de-bugs/bug-emp-002-debito-sem-validar-conta-origem.md).** ParaBank's loan approval rule sums the balance across all of a customer's accounts to decide whether to approve a loan, but debits the down payment from only one specific account, without checking whether that account alone holds enough funds. A request correctly approved by the rule can leave an account with a negative balance, with no warning — and the effect compounds on future requests.

- **[Eight units sold of a product that had five (ServeRest)](relatorios-de-bugs/bug-car-001-duplicado-nao-vizinho-quebra-estoque.md).** The API rejects a cart with the same product repeated, but only when the repetitions are next to each other. With the product repeated out of sequence, the cart is accepted, each line is checked against stock on its own, and only the last one is deducted. After the purchase is completed, another customer can still buy units that were already sold. The suspicion came from reading the code; execution confirmed it and revealed an effect the code alone didn't make clear: cancelling restores the stock exactly, which hides the problem from anyone who only tests that flow.

- **Hypotheses tested and discarded, not just confirmed.** Not every risk flagged in a test design turns into a defect. A negative loan amount in ParaBank and an empty product list in ServeRest were flagged as candidate inconsistencies, actually executed, and both applications handled them correctly. Documenting that matters as much as documenting a bug — it shows the conclusion came from real execution, not assumption.

How each finding was uncovered (source code reading, the application's official specification, real execution) is detailed in [`metodologia.md`](metodologia.md) *(Portuguese only)*.

## Structure

| Folder | Content |
| --- | --- |
| [`casos-de-teste/`](casos-de-teste/) | Test case design with business rule traceability, documenting the technique used (decision table, equivalence partitioning, boundary value analysis) for each feature analyzed |
| [`sessoes-exploratorias/`](sessoes-exploratorias/) | Exploratory testing session reports (Session-Based Test Management): charter, coverage by area, chronological test log, and confirmed defects |
| [`relatorios-de-bugs/`](relatorios-de-bugs/) | Reports on real defects found during testing sessions, with reproduction steps, evidence, root-cause analysis, and severity/priority justification |
| [`colecoes-postman/`](colecoes-postman/) | Postman collections used in the API testing sessions, with assertions on status, message, and data side effects, to reproduce the execution |

> The artifacts linked below are written in Portuguese, the project's working language. This README is an English-language orientation to the repository's content.

## Artifacts

### ParaBank

- [Request a Loan](casos-de-teste/parabank-solicitar-emprestimo.md):
  business rule extracted directly from the application's official
  source code, a decision table, and sixteen test cases traced back to
  the rule each one validates.
- [Exploratory session — Request a Loan](sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md):
  real execution against the application, with two confirmed defects
  (an internal error on a zero loan amount, and a down payment debited
  without checking the source account's individual balance) and a
  correction applied to the test case design artifact based on what was
  observed.
- [BUG-EMP-001 — A zero loan amount causes an internal server error](relatorios-de-bugs/bug-emp-001-valor-zero-erro-500.md):
  high severity, medium priority.
- [BUG-EMP-002 — Loan approval allows a negative balance on the source account](relatorios-de-bugs/bug-emp-002-debito-sem-validar-conta-origem.md):
  critical severity, high priority, with a confirmed cascading effect on
  future loan requests.

### BugBank

- [Transfer](casos-de-teste/bugbank-transferencia.md):
  business rule extracted from the official specification published by
  the application itself, a decision table with three combined
  conditions, and fourteen test cases, including scenarios the
  specification does not explicitly cover.
- [Exploratory session — Registration, Login, and Transfer](sessoes-exploratorias/bugbank-transferencia-22-09-2026.md):
  manual execution through the interface, with two confirmed defects (a
  duplicate account created on re-registration with the same email, with
  no password-recovery flow; and inconsistent validation on the transfer
  form, including a raw library error message exposed to the user) and
  confirmation of one rule working correctly.
- [BUG-CAD-001 — Re-registering with an existing email creates a parallel account](relatorios-de-bugs/bug-cad-001-recadastro-sem-verificar-email-duplicado.md):
  high severity, high priority.
- [BUG-TRF-001 — The transfer form's validation is inconsistent](relatorios-de-bugs/bug-trf-001-validacao-formulario-inconsistente.md):
  medium severity, medium priority.

### ServeRest

- [Carts](casos-de-teste/serverest-carrinhos.md):
  the portfolio's first API testing cycle. Business rule extracted from
  the official source code, a decision table for the cart creation
  validations, a state transition diagram for the cart lifecycle, and
  twenty-nine test cases, including two hypotheses raised while reading
  the code (one confirmed as a defect, the other refuted during
  execution).
- [Exploratory session — Carts](sessoes-exploratorias/serverest-carrinhos-12-11-2026.md):
  execution of all 29 cases against the API, with a versioned Postman
  collection, one confirmed defect (a product repeated out of sequence
  that breaks stock control) and one discarded hypothesis (an empty
  product list).
- [BUG-CAR-001 — A product repeated out of sequence passes validation and breaks stock control](relatorios-de-bugs/bug-car-001-duplicado-nao-vizinho-quebra-estoque.md):
  high severity, high priority, with 8 units sold of a product that had
  5 in stock.

## Author

Built by [Thomas Teixeira](https://github.com/ThomasTDS) as a study and
portfolio project in software testing.
