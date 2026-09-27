# Fluxo de commit e push (compartilhado)

Referenciado pelas skills `commit-push` de cada repo (`Simpleto.BackApi`,
`Simpleto_front_adm`). Não é uma skill invocável — só a parte do processo que
é idêntica nos dois repos. Cada skill local cobre o que é específico dela
(caminho raiz, branch/remote, tipos extras, exemplos).

## Fluxo obrigatório

1. **Revisar o estado do repo** (nunca pule esta etapa):
   ```bash
   git status
   git diff
   git diff --staged
   ```
2. **Nunca use `git add -A` ou `git add .`** — liste os arquivos e adicione
   nominalmente (`git add <arquivo1> <arquivo2>`). Se algo parecer secreto
   (credenciais, chaves, tokens), avise o usuário antes de adicionar.
3. **Classificar o tipo de mudança** (ver tabela na skill local) e verificar
   se há referência a issue/épico no pedido do usuário, na branch atual
   (`git branch --show-current`) ou em comentários/TODO no código alterado.
4. **Montar a mensagem** no formato Conventional Commits traduzido (ver
   seção "Formato da mensagem").
5. **Commitar** com heredoc para preservar formatação:
   ```bash
   git commit -m "$(cat <<'EOF'
   tipo(escopo): descrição curta no imperativo

   Corpo explicando o porquê da mudança, se necessário.

   Issue: #123
   Epic: #45
   EOF
   )"
   ```
6. **Antes do push, pedir confirmação explícita ao usuário.** Nunca faça
   `git push --force`. Se o push for rejeitado (non-fast-forward), avise o
   usuário e pergunte como proceder — não faça `pull --rebase` ou force
   push por conta própria.
   ```bash
   git push origin <branch-atual>
   ```
7. Rode `git status` novamente após o push para confirmar sucesso.

## Tipos de commit (base comum)

| Tipo em PT | Prefixo | Quando usar |
|---|---|---|
| Funcionalidade | `feat` | Nova funcionalidade |
| Correção | `fix` | Correção de bug/comportamento incorreto |
| Melhoria | `refactor` ou `perf` | Melhoria de código/performance sem mudar comportamento externo |
| Documentação | `docs` | Só documentação (README, comentários, skills) |
| Testes | `test` | Adição/ajuste de testes |
| Manutenção | `chore` | Config, dependências, build, CI |

(A skill local pode adicionar linhas específicas do stack, ex.: `style` no
front para mudanças de layout/CSS.)

Regras de identificação:
- Se o pedido do usuário ou o diff introduz comportamento novo → `feat`.
- Se corrige um problema relatado (bug, comportamento/resultado errado) → `fix`.
- Se só reorganiza/limpa/otimiza sem mudar o comportamento → `refactor`/`perf`
  (chamado de "melhoria" na conversa com o usuário).
- Se o pedido menciona um número de issue (`#123`, "issue 123") ou épico
  ("épico X", "epic #45"), inclua nas linhas de rodapé `Issue:` e/ou `Epic:`.
- Escopo (`(escopo)`) é opcional — use o nome do módulo/pasta afetado
  quando ficar claro qual é.

## Formato da mensagem

```
tipo(escopo): descrição curta no imperativo, minúscula, sem ponto final

Corpo opcional explicando o porquê (não o que — isso o diff já mostra).

Issue: #123
Epic: #45
```

- Título com no máximo ~70 caracteres.
- Sempre em Português.
- Rodapé `Issue:`/`Epic:` só aparece quando há referência real — não invente números.
