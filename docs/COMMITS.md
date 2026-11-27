# Convenção de commits

Adotamos Conventional Commits para novas alterações neste portfólio, sem reescrever commits anteriores.

Formato: `tipo(escopo): descrição curta`

| Tipo | Uso |
| --- | --- |
| `docs` | Relatórios, índices, modelos e documentação |
| `feat` | Nova funcionalidade em código ou automação |
| `fix` | Correção de comportamento ou erro em código |
| `refactor` | Reorganização de código sem mudar comportamento |
| `test` | Testes |
| `chore` | Manutenção e configuração |

Escopos preferenciais: `soc-l1`, `cyber-security-101`, `portfolio`, `security`, `lab`.

Exemplos:

- `docs(soc-l1): add suspicious login investigation`
- `docs(soc-l1): clarify false positive evidence`
- `docs(portfolio): update report index`
- `fix(lab): correct log parsing`

Usar inglês na mensagem curta para consistência, verbo no imperativo, um objetivo por commit e descrição sem ponto final. Quando necessário, incluir um corpo explicando a motivação e as limitações. Nunca incluir credenciais, flags ou dados sensíveis na mensagem ou nos arquivos. Revisar o diff antes de publicar.

A convenção é documental; não há validação automática obrigatória neste momento.
