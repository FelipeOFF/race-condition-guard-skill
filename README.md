# race-condition-guard

Claude Code skill: valida e previne **race conditions** em handlers
concorrentes — double-submit, TOCTOU, lost update, check-then-act — e ensina a
**provar a guarda com testes de concorrência** (não com chamada sequencial).

Framework-agnóstico, com foco em backend web (Python/Django/SQLAlchemy nos exemplos).

## Conteúdo

- Modelo mental da janela TOCTOU (check-then-act).
- Estratégias: constraint, update atômico condicional, lock pessimista/otimista,
  idempotency key, advisory lock, upsert atômico.
- Validar: testes que disparam requests concorrentes (asyncio/threads) e
  afirmam o invariante; property-based testing de intercalações.
- Checklist de review + anti-padrões.

## Instalar

```bash
npx skills add git@github.com:FelipeOFF/race-condition-guard-skill.git -y
```

Ou pelo marketplace [myskills](https://github.com/FelipeOFF/my-claude-code-skills)
(`programming@myskills`), onde a skill já vem vendorizada.

## Licença

MIT © 2026 Felipe Oliveira
