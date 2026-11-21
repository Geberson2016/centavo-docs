# Fluxo de trabalho no GitHub

Vale para todos os repositórios do Centavo: `centavo-backend`, `centavo-bff`, `centavo-frontend`, `centavo-infra` e `centavo-docs`.

**Regra:** nada entra na `main` sem pull request. Quem aprova é o Geberson, fazendo o merge.

## Passo a passo de uma demanda

1. **Issue.** Título curto e descrição com contexto, o que fazer e critérios de aceite.
   - Demanda de um repositório só: a issue fica nele.
   - Demanda que toca vários repositórios: a issue fica no `centavo-docs`, e cada repositório recebe o seu PR citando ela.
2. **Branch** criada a partir da `main` atualizada, no formato `<tipo>/<nº-da-issue>-<slug>`. Exemplo: `feat/12-transaction-filters`.
   - O tipo é o mesmo do Conventional Commits (`feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`).
   - O slug fica em inglês, por ser identificador técnico.
   - Em demanda multi-repositório, o número é o da issue do `centavo-docs`, e a branch tem o mesmo nome em todos os repositórios.
3. **Commits** na branch, em Conventional Commits com o texto em português. Podem ser vários: no merge eles viram um só, e as mensagens deles formam o corpo do commit na `main`.
4. **Pull request** para a `main`.
   - O título segue o Conventional Commits, porque vira o título do commit na `main`.
   - A descrição segue o template: a issue (`Closes #12`, ou `Refs Geberson2016/centavo-docs#3` quando a issue é de outro repositório), o que muda e como testar.
   - Numa mudança coordenada entre repositórios (por exemplo, o contrato do BFF e os tipos do front), cada PR aponta o outro e diz a ordem de merge.
5. **Revisão e merge** pelo Geberson, sempre por squash. A branch é apagada sozinha e o `Closes` fecha a issue.

## O que o GitHub garante

A regra "Proteção da main" (ruleset do repositório) está ativa nos cinco repositórios:

- A `main` só muda por PR. Push direto é recusado com `GH013`, inclusive para administradores.
- Force push e exclusão da `main` são bloqueados.
- O merge só é liberado com o CI verde:
  - backend: `Build e testes`;
  - BFF: `Typecheck, lint, testes e build`;
  - front: `Lint, testes e build`.
- Os jobs de contrato (`Contrato em dia com o backend` e `Tipos em dia com o contrato do BFF`) rodam, mas não bloqueiam o merge. Numa mudança coordenada eles ficam vermelhos até o outro lado entrar na `main`.
- O merge é só por squash: o título do PR vira o título do commit, e as mensagens dos commits da branch viram o corpo.
- Os PRs saem da conta do próprio Geberson, e o GitHub não deixa o autor aprovar o próprio PR. Por isso não há exigência de "Approve": a aprovação é o merge.

## Para agentes (Claude Code)

- Nunca commitar nem dar push na `main`, e nunca fazer merge de PR.
- Criar issue, branch e PR com o `gh`, sempre com `-R Geberson2016/<repo>`. Os remotos usam o alias SSH `github-centavo`, que o `gh` não reconhece como GitHub.
- Sem trailer `Co-Authored-By` nos commits e sem linha de atribuição nos PRs.
- Depois do merge, voltar para a `main` atualizada (`git switch main && git pull`) e só então apagar a branch local. Como o merge é squash, o git não reconhece a branch como mergeada, então confirme o merge no PR antes de usar `git branch -D`.
