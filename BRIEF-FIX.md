# Briefing: Fix do watchdog do Gateway Windows + detector de OOM

## Contexto
Diagnóstico completo já feito (leia `C:/Users/gabri/hermes-diag/RELATORIO-DIAGNOSTICO.md` primeiro).
Você está num clone compartilhado do repo hermes-agent (fork `gabmfranco-ds/hermes-agent`), branch `fix/gateway-windows-watchdog` — já criada, trabalhe nela.

**IMPORTANTE**: merge só no repositório do FORK (`gabmfranco-ds/hermes-agent`), NUNCA no upstream `NousResearch/hermes-agent` (origin). O remote `fork` já aponta pro fork.

## Os 2 bugs a consertar

### Bug 1 (alto impacto): Watchdog estruturalmente morto no Windows
- Arquivo: `hermes_cli/gateway_windows.py` (funções `_build_gateway_vbs_script` ~linha 516 e a geração do XML da Task ~593-648)
- Hoje: o `.vbs` termina com `sh.Run command_line, 0, False` (fire-and-forget) → wscript.exe sai na hora → Task Scheduler nunca vê o gateway cair → `RestartOnFailure` nunca dispara → 100% das religadas têm sido manuais
- Fix desejado: laço de supervisão no `.vbs`:
  ```vbscript
  Do While True
    sh.Run command_line, 0, True   ' bWaitOnReturn = True: espera o gateway morrer
    WScript.Sleep 5000             ' backoff antes de religar
  Loop
  ```
  Assim o wscript.exe fica vivo enquanto o gateway existir, e se o gateway morrer o loop relança — e se o próprio wscript morrer (crash), o RestartOnFailure da Task religa a tarefa.
- ⚠️ CUIDADO CRÍTICO: o design atual foi criado para consertar o bug #45599 — rodar o gateway via `cmd.exe` fazia o Windows matá-lo com `CTRL_CLOSE_EVENT` no logon. Sua solução NÃO pode reintroduzir isso: rodar via `sh.Run(..., 0, True)` dentro de wscript oculto não aloca console pro gateway (a janela oculta é do wscript, não do python) — VERIFIQUE essa premissa lendo o histórico do bug (`git log -p -S "CTRL_CLOSE_EVENT"` e o commit que introduziu o .vbs) e explique no relatório por que a solução é segura.
- Considere também: cap de restarts no loop (ex.: 10 religadas em 1h = desistir e sair com exit code não-zero pra deixar o RestartOnFailure da Task agir) para não criar loop infinito de crash-restart.
- Atualize os testes existentes de `gateway_windows.py` (procure tests que cobrem `_build_gateway_vbs_script` / task XML) e ajuste as asserções.

### Bug 2 (médio impacto): sample_memory() Linux-only
- Arquivo: `gateway/lifecycle_ledger.py` função `sample_memory()` (~linhas 77-91) — lê `/proc/self/status` e `/proc/meminfo`, retorna `{}` fora do Linux
- Fix: usar `psutil` (JÁ é dependência usada por `gateway/agent_cache_pressure.py` e `gateway/memory_monitor.py` — confirme o import/padrão deles e siga o mesmo estilo). Retornar as MESMAS chaves/dimensões que a versão Linux retorna (KiB) para não quebrar consumidores do ledger: own RSS + system availability + swap + suspected_oom.
- Manter o comportamento "never raises" (try/except amplo como hoje).
- Ajustar/criar testes (procure tests existentes de lifecycle_ledger).

## Critérios de aceite
1. `python -m pytest tests/ -k "gateway_windows or lifecycle" -q` verde (descubra o runner certo do projeto — provavelmente `pytest`)
2. Testes novos cobrindo: loop de supervisão no .vbs (assert que contém `bWaitOnReturn` True / Do While), cap de restarts, sample_memory retornando chaves esperadas (mock de psutil)
3. Ruff/format limpo no estilo do repo (verifique o pre-commit do repo)
4. Commit(s) descritivos na branch fix/gateway-windows-watchdog
5. Push para o FORK: `git push fork fix/gateway-windows-watchdog`
6. Relatório final: o que mudou, por que a solução não reintroduz #45599, resultado dos testes

## Proibições
- NÃO tocar no checkout vivo `C:/Users/gabri/hermes-agent` (esse é o Hermes em produção rodando) — trabalhe APENAS neste clone
- NÃO fazer push/merge/PR no origin (NousResearch)
- NÃO instalar dependências globais; se precisar rodar pytest, use o venv existente do repo vivo apenas para LER pacotes (`.venv` se existir) ou crie venv local no clone
- Sem flags de bypass
