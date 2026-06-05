---
name: race-condition-guard
description: |
  Valida e previne race conditions em handlers concorrentes: double-submit,
  TOCTOU, lost update, check-then-act. Use ao escrever endpoints que mutam
  estado compartilhado (saldo, estoque, idempotência, status), revisar código
  concorrente/async, ou montar testes que disparam requests simultâneas e
  afirmam invariantes. Triggers: "race condition", "concorrência", "double
  submit", "lost update", "TOCTOU", "idempotência", "lock", "SELECT FOR UPDATE".
source: authored
upstream: https://github.com/FelipeOFF/race-condition-guard-skill
license: MIT
added: 2026-06-05
---

# Race Condition Guard

Guardrail para **race conditions** em código que serve requests concorrentes.
O foco é o caso mais comum em backend web: duas (ou N) requests chegam quase
ao mesmo tempo e ambas leem-decidem-escrevem sobre o mesmo estado, violando um
invariante (saldo negativo, estoque duplo, dois registros que deviam ser um).

## Quando ativar

- Endpoint lê estado, decide com base nele, e escreve (**check-then-act**).
- Mutação de recurso compartilhado: saldo, estoque, contador, status, slug único.
- Operações que **devem** ser idempotentes (pagamento, criação por chave externa).
- Revisando código async/threads/workers que compartilham estado.
- Webhooks/retries que podem chegar duplicados ou fora de ordem.

## O modelo mental: a janela TOCTOU

Toda race nasce de uma **janela** entre *Time Of Check* e *Time Of Use*:

```
# RUIM — check-then-act sem atomicidade
acc = Account.get(id)            # CHECK: lê saldo 100
if acc.balance >= amount:        #        decide com base no valor lido
    acc.balance -= amount        # USE:   escreve 100-amount
    acc.save()                   # duas requests concorrentes leem 100,
                                 # ambas debitam, saldo fica errado (lost update)
```

A cura é **fechar a janela**: tornar check+act uma operação atômica, ou
serializar o acesso ao recurso. Nunca confie em "é rápido demais pra dar race"
— sob carga, a janela sempre abre.

## Estratégias de guarda (escolha por caso)

| Estratégia | Quando | Mecanismo |
|---|---|---|
| **Constraint no banco** | Unicidade (slug, idempotency key, 1 voto/user) | `UNIQUE`/`EXCLUDE` + tratar `IntegrityError` como "perdeu a corrida" |
| **Update atômico condicional** | Contador, saldo, estoque | `UPDATE ... SET x = x - n WHERE x >= n` e cheque `rowcount` |
| **Lock pessimista** | Check-then-act que não cabe em 1 UPDATE | `SELECT ... FOR UPDATE` dentro de transação |
| **Lock otimista** | Baixa contenção, evitar lock longo | coluna `version`; `UPDATE ... WHERE version = :v`; retry no conflito |
| **Idempotency key** | POST que pode ser reenviado (pagamento, webhook) | chave única por operação; 2ª chamada retorna o 1º resultado |
| **Advisory lock** | Serializar por chave lógica fora de 1 linha | `pg_advisory_xact_lock(key)` |
| **Upsert atômico** | Get-or-create | `INSERT ... ON CONFLICT DO UPDATE` |

### Padrões concretos

```python
# 1) Update atômico condicional — sem ler antes (fecha a janela no banco)
updated = (Account.objects
           .filter(id=acc_id, balance__gte=amount)
           .update(balance=F("balance") - amount))
if not updated:                       # 0 linhas = saldo insuficiente OU perdeu a corrida
    raise InsufficientFunds()
```

```python
# 2) Lock pessimista — check-then-act que precisa de lógica entre ler e escrever
with transaction.atomic():
    acc = Account.objects.select_for_update().get(id=acc_id)  # serializa por linha
    if acc.balance < amount:
        raise InsufficientFunds()
    acc.balance -= amount
    acc.save()
```

```python
# 3) Idempotência — 2ª request com a mesma key NÃO reprocessa
try:
    op = Operation.objects.create(idempotency_key=key, ...)   # UNIQUE(key)
except IntegrityError:
    return Operation.objects.get(idempotency_key=key).result  # devolve o 1º resultado
```

```python
# 4) Lock otimista — version column, retry no conflito
for _ in range(3):
    obj = Doc.objects.get(id=i)
    n = Doc.objects.filter(id=i, version=obj.version).update(
        body=new_body, version=obj.version + 1)
    if n: break            # ganhou
else:
    raise ConflictError()  # perdeu 3x → erro explícito, não silencioso
```

## Validar com teste de concorrência (o guardrail)

Race não reproduz com chamada sequencial. O teste **precisa** disparar
operações simultâneas e afirmar o invariante depois.

```python
# Dispara N requests concorrentes contra o MESMO recurso e checa o invariante.
import asyncio, httpx

async def test_no_double_spend(account_with_balance_100):
    async with httpx.AsyncClient(base_url=BASE) as c:
        # 10 débitos de 20 concorrentes sobre saldo 100 → no máx 5 podem passar
        results = await asyncio.gather(*[
            c.post(f"/accounts/{acc}/debit", json={"amount": 20})
            for _ in range(10)
        ], return_exceptions=True)
    ok = sum(r.status_code == 200 for r in results if hasattr(r, "status_code"))
    assert ok == 5, f"double-spend: {ok} débitos passaram (esperado 5)"
    assert get_balance(acc) == 0          # invariante: nunca negativo
```

```python
# Threads para código síncrono — barreira garante largada simultânea.
import threading
def test_unique_under_race(make_request):
    start = threading.Barrier(20)
    errs = []
    def worker():
        start.wait()                      # todos largam juntos → maximiza a janela
        try: make_request(slug="dup")
        except Exception as e: errs.append(e)
    ts = [threading.Thread(target=worker) for _ in range(20)]
    [t.start() for t in ts]; [t.join() for t in ts]
    assert Resource.objects.filter(slug="dup").count() == 1   # 1 venceu, resto barrado
```

### Property-based para invariantes sob concorrência

Use PBT (Hypothesis) para gerar **sequências/intercalações** de operações e
afirmar que o invariante vale para qualquer ordem — pega casos que exemplos
fixos não cobrem.

```python
from hypothesis import given, strategies as st

@given(ops=st.lists(st.tuples(st.sampled_from(["debit","credit"]),
                              st.integers(1, 50)), min_size=1, max_size=30))
def test_balance_never_negative_for_any_interleaving(ops):
    acc = fresh_account(balance=100)
    run_concurrently(acc, ops)            # aplica ops em threads/tasks
    assert get_balance(acc) >= 0          # invariante independe da ordem
    assert get_balance(acc) == expected_serial(acc, ops)  # = execução serial equivalente
```

## Checklist de review

- [ ] Todo check-then-act sobre estado compartilhado é atômico (UPDATE condicional, lock, ou constraint).
- [ ] Unicidade é garantida por **constraint no banco**, não só por `if not exists`.
- [ ] POSTs reenviáveis (pagamento/webhook) têm idempotency key.
- [ ] Conflito otimista vira **erro explícito** ou retry — nunca escrita silenciosa.
- [ ] Existe teste que dispara requests **concorrentes** e afirma o invariante.
- [ ] Transação engloba ler+escrever (não commita no meio da janela).
- [ ] Lock tem escopo mínimo (linha/chave), não a tabela toda.

## Anti-padrões

- "É rápido, não dá race" — sob carga real a janela abre. Sempre.
- `get_or_create` sem `UNIQUE` no banco — cria duplicado sob concorrência.
- Lock só na aplicação (mutex em processo) num serviço com N réplicas — não serializa entre processos.
- Ler-decidir-escrever em transações separadas (commit entre check e act).
- Engolir `IntegrityError`/conflito sem decidir vencedor/perdedor (corrupção silenciosa).
- Testar concorrência com loop sequencial — não reproduz a race.
