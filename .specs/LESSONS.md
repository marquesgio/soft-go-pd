# LESSONS - auto-maintained by scripts/lessons.py

> Machine-owned. Do NOT hand-edit. Changes are overwritten on the next `lessons.py` write.
> Canonical state lives in `.specs/lessons.json`. Edit lessons only via the script.
> promote_threshold=2 distinct features · window_days=45 · quarantine_threshold=2

## Confirmed (load these at Specify/Design)

Corroborated across multiple features. Safe to apply as guidance.

_none_

## Candidates (under observation - do NOT load as guidance yet)

Seen once or not yet corroborated. Tracked, not trusted.

### L-001 - Para requisitos de rota aberta, testar também que nenhum APP_GUARD global é registrado, não só metadata de guard por controller.
- signal: `surviving_mutant` · recurrence: 1 feature(s) · scope: `api/guards` · harmful: 0
- features: auth
- evidence: A11 soft-go-ii-api/src/app.module.ts:24 (api/guards)
- last seen: 2026-10-07T21:31:06Z

### L-002 - Quando testes usam repositório fake, cobrir opções de coluna críticas (ex.: select: false) lendo getMetadataArgsStorage.
- signal: `surviving_mutant` · recurrence: 1 feature(s) · scope: `api/entities` · harmful: 0
- features: auth
- evidence: A1 soft-go-ii-api/src/users/entities/user.entity.ts:26 (api/entities)
- last seen: 2026-10-07T21:31:06Z

### L-003 - Em páginas de formulário, testar a mensagem de erro de cada campo validado, não só de um campo representativo.
- signal: `surviving_mutant` · recurrence: 1 feature(s) · scope: `front/forms` · harmful: 0
- features: auth
- evidence: F10 soft-go-II/src/pages/SignUp.tsx:75 (front/forms)
- last seen: 2026-10-07T21:31:06Z

### L-004 - ACs de validação devem citar a mensagem exata esperada, para que testes não precisem de regex genérica.
- signal: `spec_precision_gap` · recurrence: 1 feature(s) · scope: `spec` · harmful: 0
- features: auth
- evidence: AUTH-07 (spec)
- last seen: 2026-10-07T21:31:06Z

### L-005 - Quando um AC lista o que uma tela mostra (ex.: resumo da carona), testar cada elemento listado, não só a ausência dos campos removidos.
- signal: `surviving_mutant` · recurrence: 1 feature(s) · scope: `front/components` · harmful: 0
- features: vou-junto
- evidence: F10 soft-go-II/src/components/Modal.tsx:60 (front/components)
- last seen: 2026-10-08T14:31:38Z

### L-006 - Em ACs de rejeição com 'sem criar X', o teste deve verificar o estado persistido (nenhum registro), não só o status HTTP.
- signal: `spec_precision_gap` · recurrence: 1 feature(s) · scope: `api/validation` · harmful: 0
- features: vou-junto
- evidence: JOIN-19 (api/validation)
- last seen: 2026-10-08T14:31:38Z

## Quarantined (failed when applied - ignore)

A confirmed lesson that recurred alongside failure. Kept for the maintainer to review.

_none_
